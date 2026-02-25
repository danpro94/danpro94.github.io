---
title: "배포 직후 502 Bad Gateway: ALB→EC2→RDS 장애 격리 런북과 RDS SSL 근본원인 분석"
date: 2026-02-25 17:57:00 +0900
categories: [Project, Incident]
tags: [AWS, ALB, RDS, CI/CD, Runbook, git-revert]
---
이 글은 팀 프로젝트(StageLog) 2차 고도화 과정에서 uploads-presign 기능 배포 직후 시점에서 발생했던 502 Bad Gateway 장애를 격리·분석·롤백·재도입한
전 과정의 기록입니다. "로컬 성공 ≠ AWS 성공"이라는 경험과 런북·롤백 플랜의 중요성을 중심으로 정리했습니다.


push(develop)
  → GitHub Actions
      → Docker 빌드 + ECR/DockerHub Push
      → SSM send-command
          → EC2: .env 생성 + docker-compose pull + up -d
```

uploads-presign 기능(S3 Presigned PUT URL 발급)을 `develop`에 머지한 직후,
FE 전체에서 502 Bad Gateway가 발생했습니다.

---

## 장애 타임라인

| Day | 단계 | 내용 |
|---|---|---|
| Day 11 | 증상 감지 | ALB 502, FE UI 전체 불능 |
| Day 13 | 원인 격리 | 컨테이너 Exited(1), 로그 확보 |
| Day 14 | 롤백 | git revert → CI/CD → 서비스 복구 |
| Day 15 | 재도입 시도 | 502 재발 → 근본원인 재분석 |

---

## 1단계: 증상 수집 — ALB부터 순서대로

502는 ALB가 Target(EC2)으로부터 응답을 받지 못했을 때 반환합니다.
FE에서 UI가 깨진다고 해서 FE 문제로 단정하면 안 됩니다.
**ALB → EC2 → 컨테이너 순서로 레이어를 격리하는 것이 원칙입니다.**

### 레이어 1: ALB 응답 확인

```bash
# 브라우저 대신 curl로 정확한 HTTP 상태코드 확인
curl -i https://api.example.com/

# 기대: 200 OK (정상) 또는 502 Bad Gateway (ALB→EC2 통신 실패)
# 502가 나오면 ALB 뒷단(EC2/컨테이너) 문제로 범위 좁힘
```

ALB 콘솔 → Target Group → Targets 탭에서 Target 상태도 함께 확인합니다.

```
상태: unhealthy  ← 이 시점에서 EC2/컨테이너 문제로 확정
상태: healthy    ← ALB→EC2 통신은 정상, 앱 로직 문제
```

### 레이어 2: EC2 컨테이너 상태 확인

```bash
# SSM Session Manager 또는 SSH로 EC2 접근 후
docker ps -a

# 정상 상태
CONTAINER ID   IMAGE           STATUS
abc123         stagelog-api    Up 3 hours

# 장애 상태 — 확인됨
CONTAINER ID   IMAGE           STATUS
abc123         stagelog-api    Exited (1) 2 minutes ago
#                              ↑ exit code 1 = 비정상 종료
```

`Exited (1)`이 확인되면 컨테이너가 **시작 직후 죽은 것**입니다.
ALB가 응답을 못 받는 것은 결과이고, 진짜 원인은 컨테이너 부팅 실패입니다.

### 레이어 3: 컨테이너 로그 확인

```bash
# 마지막 200줄 로그 확인
docker logs --tail 200 

# 또는 컨테이너명으로
docker logs --tail 200 stagelog-api
```

이 단계에서 두 개의 핵심 에러를 확보했습니다.

---

## 2단계: 원인 격리 — 로그 분석

### 에러 1: STATIC_ROOT ImproperlyConfigured

```
django.core.exceptions.ImproperlyConfigured:
  You're using the staticfiles app without having set the STATIC_ROOT setting
  to a filesystem path.
```

컨테이너 시작 스크립트는 gunicorn 실행 전에 `collectstatic`을 먼저 수행합니다.
`STATIC_ROOT`가 누락된 상태에서 `collectstatic`이 실패하면 컨테이너가 즉시 종료됩니다.

**왜 로컬에서는 괜찮았나:**
로컬 개발 환경(`DEBUG=True`)에서는 Django가 `staticfiles`를 `STATIC_ROOT` 없이도 서빙합니다.
AWS 배포 환경에서는 `collectstatic`이 필수 부팅 단계로 실행되므로 `STATIC_ROOT`가 없으면 즉시 종료됩니다.

실제 `settings.py`에는 `STATIC_ROOT`가 이미 정의되어 있었습니다:

```python
STATIC_URL = 'static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
```

문제는 **배포된 Docker 이미지 내부의 `settings.py`** 였습니다.
uploads-presign 기능을 머지한 PR에 로컬 `settings.py`의 불완전한 버전이 포함되어,
배포 이미지 내부의 `settings.py`에서 `STATIC_ROOT`가 누락된 상태였습니다.

### 에러 2: RDS SSL OperationalError 3159

```
django.db.utils.OperationalError: (3159,
  "Connections using insecure transport are prohibited while
   --require_secure_transport=ON.")
```

컨테이너가 `STATIC_ROOT` 에러로 죽기 직전, DB 연결 시도에서도 에러가 확인되었습니다.

**근본 원인 분석:**

현재 `settings.py`의 DB 설정:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'OPTIONS': {
            'ssl': {
                'ca': None,  # ← CA 인증서 경로가 None
            },
            'ssl_mode': 'REQUIRED',  # ← SSL은 강제
        },
    }
}
```

`ssl_mode: REQUIRED`인데 `ca: None`입니다.
RDS는 SSL을 요구하는데, 컨테이너 내부에 RDS CA 인증서(`global-bundle.pem`)가 없습니다.

AWS RDS는 기본적으로 `require_secure_transport=ON`으로 설정되어 있어
SSL 없이 연결을 시도하면 에러 코드 3159를 반환합니다.

**왜 로컬에서는 괜찮았나:**
로컬 개발 환경은 `DB_MODE=sqlite`로 설정되어 있어 MySQL/RDS 연결 자체를 하지 않습니다.
배포 환경에서 처음으로 RDS에 연결을 시도했을 때 CA 부재가 드러난 것입니다.

**해결 방향 (재도입 시 적용):**

```python
# 방법 A: CA 인증서를 컨테이너에 포함
'ssl': {
    'ca': os.path.join(BASE_DIR, 'certs/global-bundle.pem'),
}
# → Dockerfile에서 AWS RDS CA 번들을 COPY하는 단계 추가 필요

# 방법 B: SSL 검증 비활성화 (개발/테스트 한정, 운영 비권장)
'ssl': {
    'ca': None,
    'check_hostname': False,
    'verify_mode': False,
}
```

MVP 단계에서는 방법 B로 우선 복구하고, CA 번들 주입은 별도 PR로 분리해 처리합니다.

---

## 3단계: 롤백 — 복구 우선

### ADR-001: 장애 시 핫픽스보다 롤백이 우선

원인이 완전히 파악되지 않은 상태에서 핫픽스를 시도하면
FE 전체 장애가 분석 시간만큼 연장됩니다.

**결정: 복구(롤백) 먼저 → 원인 분리 후 재도입**

```
서비스 다운 상태에서 원인 분석 시간 = FE 전체 블로킹 시간
롤백으로 서비스 복구 → 안정 상태에서 원인 분석 → 검증 후 재도입
```

### git revert 롤백 절차

롤백 대상: uploads-presign 관련 머지 커밋

```bash
# 1. 롤백 대상 커밋 해시 확인
git log --oneline -10

# 출력 예시
a1b2c3d (HEAD) Merge pull request #42 from feature/uploads-presign
e4f5g6h feat: add presign upload endpoint
...

# 2. 머지 커밋 revert (머지 커밋은 -m 1 옵션 필요)
git revert -m 1 a1b2c3d

# -m 1: 머지 커밋의 첫 번째 부모(main/develop)를 기준으로 되돌림
# revert는 새 커밋을 추가해 이력을 보존 (reset과 다름)
```

**주의: git revert 실패 케이스**

```bash
# 에러 메시지
error: Your local changes to the following files would be overwritten by merge:
        config/settings.py
Please commit your changes or stash them before you merge.

# 원인: 로컬에 uncommitted 변경사항이 있을 때
# 해결
git stash          # 변경사항 임시 저장
git revert -m 1 a1b2c3d
git stash pop      # 필요 시 복원
```

```bash
# 3. revert 커밋 push → CI/CD 자동 트리거
git push origin develop

# 4. GitHub Actions 완료 후 확인
curl -i https://api.example.com/
# HTTP/1.1 200 OK ← 복구 확인

docker ps -a
# STATUS: Up X minutes ← 컨테이너 정상 기동 확인
```

롤백 후 502 해소 및 컨테이너 정상 기동이 확인되었습니다.

---

## 4단계: 재도입 — 검증 루트 강제

롤백 후 안정화된 상태에서 uploads-presign을 재도입할 때,
Day15에서 동일한 502가 재발했습니다.

**재발 원인:**

CI/CD `.env` 생성 단계에 S3 관련 환경변수 4개가 누락되어 있었습니다.
CI/CD 작업은 팀원이 진행하였으나 해당 환경변수가 주입되어야 하는 점을 공유하지 못했고, 이 과정에서 느낀 점은 변경사항에 대한 코드 리뷰, 팀원 간 양방향 소통이 중요하다는 점이었습니다.

```yaml
# api-cicd.yaml — 당시 누락 상태
echo "AWS_REGION=${{ secrets.AWS_REGION }}" >> .env
echo "S3_UPLOAD_BUCKET=${{ secrets.S3_UPLOAD_BUCKET }}" >> .env
echo "S3_UPLOAD_PREFIX=${{ secrets.S3_UPLOAD_PREFIX }}" >> .env
echo "S3_PRESIGN_EXPIRES=${{ secrets.S3_PRESIGN_EXPIRES }}" >> .env
# ↑ 이 4줄이 CI/CD yaml에 없었음
# → 컨테이너 런타임에서 S3 설정 누락 → presign 뷰 500 → ALB 502
```

`settings.py`에서 `S3_UPLOAD_BUCKET`은 다음과 같이 읽힙니다:

```python
S3_UPLOAD_BUCKET = env(
    "S3_UPLOAD_BUCKET",
    default=env("AWS_STORAGE_BUCKET_NAME", default=env("S3_BUCKET", default="")),
)
```

fallback이 있어 앱이 죽지는 않지만 빈 문자열이 됩니다.
presign 뷰에서 bucket이 빈 문자열이면 500을 반환하고, ALB는 이를 502로 응답합니다.

**SSOT(Single Source of Truth) 원칙 위반:**

```
GitHub Secrets (값 존재) ≠ CI/CD yaml (주입 코드 없음)
               ↓
       컨테이너 런타임 env에 값 없음
               ↓
          502 재발
```

환경변수가 Secrets에 "있다"는 것과 컨테이너가 실제로 "읽는다"는 것은 다릅니다.

---

## 최종 결과: 재도입 완료 체크리스트

재도입 PR 승인 조건으로 다음 검증 루트를 팀 내 표준화했습니다.

```bash
# 검증 1: 배포 이미지 내부 settings.py 핵심 설정 확인
docker run --rm : \
  grep -n "STATIC_ROOT\|S3_UPLOAD_BUCKET\|S3_PUBLIC_BASE_URL" \
  /app/config/settings.py

# 기대 출력
# 73: STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
# 85: S3_UPLOAD_BUCKET = env("S3_UPLOAD_BUCKET", ...)
# 91: S3_PUBLIC_BASE_URL = env("S3_PUBLIC_BASE_URL", default=None)

# 검증 2: 컨테이너 부팅 로그 확인 (부팅 성공 여부)
docker logs --tail 200 stagelog-api | grep -E "Starting|Booting|Worker|error|Error"

# 기대 출력 (gunicorn 정상 기동)
# [INFO] Starting gunicorn 21.x.x
# [INFO] Booting worker with pid: 123

# 검증 3: presign 엔드포인트 스모크 테스트
curl -s -X POST https://api.example.com/api/uploads/presign \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"filename":"test.png","content_type":"image/png"}' \
  | jq '.data.file_url'

# 기대 출력
# "https://[CloudFront-Domain].cloudfront.net/uploads/..."
# 실패 출력
# null 또는 "https://stagelog-uploads.s3.ap-northeast-2.amazonaws.com/..."

# 검증 4: 컨테이너 런타임 환경변수 확인
docker exec stagelog-api env | grep -E "S3_|AWS_REGION|STATIC"

# 기대 출력
# AWS_REGION=ap-northeast-2
# S3_UPLOAD_BUCKET=stagelog-uploads
# S3_PUBLIC_BASE_URL=https://[CloudFront-Domain].cloudfront.net
```

위 4가지를 모두 통과한 뒤 FE 통합 진행을 팀 원칙으로 확립했습니다.

---

## ADR 정리

### ADR-002: 502 발생 시 레이어 격리를 우선한다

```
Context:  FE에서 UI가 깨져도 원인이 FE가 아닐 수 있음
Decision: ALB → EC2 → 컨테이너 로그 순서로 원인 격리
Result:   FE/BE 논쟁 없이 5분 안에 "컨테이너 부팅 실패"로 원인 범주 확정
```

### ADR-003: 컨테이너 부팅 필수 설정은 PR 최우선 확인 항목

```
Context:  collectstatic/migrate는 gunicorn보다 먼저 실행됨
          실패하면 컨테이너가 죽고 ALB 502로 이어짐
Decision: STATIC_ROOT, DATABASES 같은 "부팅 필수 설정" 변경은
          PR 리뷰에서 배포 이미지 내부 grep 결과를 첨부 필수
```

### ADR-001: 장애 시 핫픽스보다 롤백이 우선

```
Context:  원인 불명 상태에서 핫픽스 시도 = FE 장애 연장
Decision: 롤백(git revert) 먼저 → 안정 상태에서 원인 분석 → 재도입
Result:   롤백 후 서비스 복구까지 약 15분 소요
```

### ADR-005: SSOT(.env) + 배포 아티팩트(이미지)가 맞지 않으면 장애가 반복된다

```
Context:  GitHub Secrets에 값이 있어도 CI/CD yaml에 주입 코드가 없으면 무의미
Decision: 환경변수 추가 시 반드시 3곳을 동시에 확인
          1) GitHub Secrets (값)
          2) CI/CD yaml (주입 코드)
          3) settings.py (읽는 코드)
Result:   Day15 재발 원인이 이 불일치로 확정됨
```

---

## 핵심 교훈

### "로컬 성공 ≠ AWS 성공"의 세 가지 패턴

이번 장애에서 드러난 로컬과 AWS 환경의 차이를 정리했습니다.

| 항목 | 로컬(개발) | AWS(배포) |
|---|---|---|
| DB | `DB_MODE=sqlite`, MySQL 연결 없음 | RDS MySQL, SSL 강제(`require_secure_transport=ON`) |
| Static 파일 | `DEBUG=True`, `collectstatic` 불필요 | `collectstatic` 부팅 단계에서 필수 실행 |
| 환경변수 | `.env` 파일 직접 편집 | GitHub Secrets → CI/CD yaml → 컨테이너 런타임의 3단계 전달 |

이 세 가지 차이가 모두 "로컬에서 통과 → AWS에서 502"라는 패턴을 만들었습니다.

### docker exec 실전 규칙

```bash
# -i 없이 here-doc 입력이 필요한 명령은 동작하지 않음
docker exec  bash << 'EOF'
echo "test"
EOF
# → 동작 안 함

# -i 필요
docker exec -i  bash << 'EOF'
echo "test"
EOF
# → 정상 동작

# -it: 대화형 터미널이 필요한 경우 (vim, python shell 등)
docker exec -it  bash
```

---

## 배포 후 502 대응 런북 (팀 표준화)

```
[502 발생 시 체크리스트]

1. ALB Target Group 상태 확인
   → healthy: EC2는 정상, 앱 로직 확인으로 이동
   → unhealthy: 컨테이너 문제로 확정

2. EC2 접속 → docker ps -a
   → Up: 컨테이너는 살아있음, 포트/헬스체크 확인
   → Exited(1): 컨테이너 부팅 실패

3. docker logs --tail 200 <container>
   → ImproperlyConfigured: settings.py 설정 누락
   → OperationalError 3159: RDS SSL CA 인증서 문제
   → ModuleNotFoundError: requirements.txt 누락 패키지

4. 원인 불명 또는 분석 > 10분 소요 예상
   → git revert 롤백 먼저 실행
   → 서비스 복구 확인 후 원인 분석 재개

5. 재도입 전 검증 루트 4가지 통과 필수
   (이미지 내부 grep / 부팅 로그 / presign 스모크 / 런타임 env)
```

---