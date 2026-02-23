---

### title: "S3 이미지 직접 노출 취약점 제거기: CloudFront OAC + WAF + Terraform IaC 전 과정"
### date: 2026-02-23 19:35:00 +0900
### categories: [Project, Security]
### tags: [AWS, S3, CloudFront, WAF, Terraform, IaC, OAC, CORS, DevOps]

> 이 글은 팀 프로젝트(StageLog) 2차 고도화 과정에서 S3 Public 구조의 보안 취약점을 발견하고,
Terraform으로 S3 Private + CloudFront OAC + WAF + CORS allowlist 최소화까지
전 과정을 설계/구현/검증한 기록입니다.

(단, 코드에 지정된 리소스 이름 등은 예시값으로 수정됨)
> 

---

## 배경

StageLog는 공연 기반 커뮤니티 서비스로, 게시글에 이미지를 첨부할 수 있는 기능을 제공합니다.
1차(MVP)에서는 개발 속도를 우선하기 위해 S3 버킷을 **Public** 상태로 운영했습니다.

당시 인프라 구성:

```
FE(React) → BE(Django/EC2) → S3 Public Bucket
                              ↓
                        image_url (S3 직링크) 저장 → DB
```

2차 고도화 목표를 논의하던 중, 이 구조가 **서비스 운영 관점에서 방치하면 안 되는 취약점**임을 확인했습니다.

---

## 문제 정의

### 취약점 시나리오

S3 Public 상태에서 다음 세 가지 위협이 존재합니다:

| 위협 | 현상 | 영향 |
| --- | --- | --- |
| 이미지 직접 노출 | S3 URL만 알면 누구나 접근 가능 | 사용자 업로드 콘텐츠 무단 접근 |
| 대량 다운로드 | Rate 제한 없음 | AWS 비용 폭증 위험 |
| Public 회귀 가능 | 설정 실수 한 번으로 재노출 가능 | 지속적 보안 거버넌스 취약 |

### 현재 `file_url` 저장 방식의 문제

```python
# 문제: S3 direct 링크가 DB에 저장됨
file_url = f"https://{bucket}.s3.{region}.amazonaws.com/{key}"
```

S3를 Private로 전환하는 순간, DB에 저장된 모든 기존 `image_url`이 **403**을 반환합니다.
단순 버킷 권한 변경만으로는 해결되지 않고, **URL 체계 자체를 바꿔야** 하는 문제였습니다.

---

## 해결 설계

### 목표 아키텍처

```
FE(React)
  │
  ├─ 이미지 업로드: Presigned PUT URL → S3 (직접 업로드)
  │
  └─ 이미지 표시: CloudFront URL ← DB 저장값

S3 (Private)
  └─ CloudFront OAC만 GetObject 허용
       └─ WAF (WebACL) → Rate 기반 룰 + Managed Rules
```

### 핵심 ADR 3개

**ADR-001: Terraform Permanent/Ephemeral Zone 분리**

> *Context*: 한정된 팀프로젝트 비용 측면에서 매일 dev 리소스를 켜두면 비용이 과다 발생하는 부분 최적화
VPC/RDS/S3는 자주 내리면 운영 및 데이터 리스크가 큰 점을 고려.
> 

```
Decision: 영구 존(네트워크/DB/보안)과 임시 존(앱/라우팅/NAT)을 분리하고
          Terraform state도 분리해 운영한다.

1-permanent/ → VPC, RDS, S3, CloudFront, WAF  (절대 destroy 하지 않음)
2-ephemeral/ → NAT, EC2, ALB                   (apply/destroy 반복)
```

⇒ 비용 절감 + 핵심 데이터 계층 보호 목표 달성.

**ADR-002: S3 Private 전환 + CloudFront OAC 기반 공개 읽기**

```
Decision: S3는 Private(PAB all true)으로 전환하고,
          CloudFront OAC만 GetObject 허용한다.

Why:
  - Public Access Block(PAB)으로 Public 회귀를 구조적으로 차단
  - CloudFront 레이어에서 WAF 확장 가능
  - 캐싱으로 S3 요청 수 감소 → 비용 절감
```

**ADR-003: WAF를 Permanent Zone에 포함**

```
Decision: CloudFront는 서비스 핵심 경로이므로 WAF도 영구 존에서 관리한다.

Why:
  - WAF/CloudFront는 자주 destroy/apply 하는 자원보다 고정 운영이 적합
  - 보안 정책 연속성 보장
```

---

## 구현 과정

### Step 1 — Terraform 구조 설계

```hcl
# 디렉토리 구조 (upsert)
infra/
├── 1-permanent/
│   ├── s3.tf           # S3 버킷 + PAB + Lifecycle
│   ├── **cloudfront.tf**   # OAC + Distribution
│   ├── **waf.tf**          # WebACL + Managed Rules
│   └── outputs.tf      # vpc_id, subnet_ids 등 ephemeral로 전달
│   └──...
└── 2-ephemeral/
    ├── data.tf         # permanent outputs 참조
    ├── ec2.tf
    └── alb.tf
    └── ...
```

`1-permanent/outputs.tf`에서 `2-ephemeral/data.tf`로 값을 전달하는 구조로,
각 Zone이 독립적으로 apply/destroy 가능합니다.

```hcl
# 2-ephemeral/data.tf
data "terraform_remote_state" "permanent" {
  backend = "s3"
  config = {
    bucket = "your-tfstate-bucket"
    key    = "permanent/terraform.tfstate"
    region = "ap-northeast-2"
  }
}

locals {
  vpc_id     = data.terraform_remote_state.permanent.outputs.vpc_id
  subnet_ids = data.terraform_remote_state.permanent.outputs.private_subnet_ids
}
```

### Step 2 — S3 Private 전환 설계

```hcl
# 1-permanent/s3.tf
resource "aws_s3_bucket" "uploads" {
  bucket = "stagelog-uploads"
}

# Public Access Block — 모든 항목 true로 Public 회귀 구조적 차단
resource "aws_s3_bucket_public_access_block" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Bucket Ownership
resource "aws_s3_bucket_ownership_controls" "uploads" {
  bucket = aws_s3_bucket.uploads.id
  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

# Lifecycle — 불완전 멀티파트 업로드 정리 (7일)
resource "aws_s3_bucket_lifecycle_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  rule {
    id     = "abort-incomplete-mpu"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}

# CloudFront OAC만 GetObject 허용하는 Bucket Policy
resource "aws_s3_bucket_policy" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid    = "AllowCloudFrontOAC"
      Effect = "Allow"
      Principal = {
        Service = "cloudfront.amazonaws.com"
      }
      Action   = "s3:GetObject"
      Resource = "${aws_s3_bucket.uploads.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.uploads.arn
        }
      }
    }]
  })
}
```

### Step 3 — CloudFront OAC 구현

```hcl
# 1-permanent/cloudfront.tf

# OAC 생성
resource "aws_cloudfront_origin_access_control" "uploads" {
  name                              = "stagelog-s3-oac"
  description                       = "OAC for StageLog uploads S3"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# Managed CachePolicy 참조 (표준 정책으로 운영 비용 절감)
data "aws_cloudfront_cache_policy" "optimized" {
  name = "Managed-CachingOptimized"
}

resource "aws_cloudfront_distribution" "uploads" {
  enabled = true

  origin {
    domain_name              = aws_s3_bucket.uploads.bucket_regional_domain_name
    origin_id                = "s3-uploads"
    origin_access_control_id = aws_cloudfront_origin_access_control.uploads.id
  }

  default_cache_behavior {
    target_origin_id       = "s3-uploads"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = data.aws_cloudfront_cache_policy.optimized.id
  }

  # WAF 연결 (Step 4에서 생성)
  web_acl_id = aws_wafv2_web_acl.cloudfront.arn

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}
```

> **트러블슈팅 — terraform validate 에러**
> 
> 
> ```
> Error: Invalid reference
>   on cloudfront.tf: data "aws_cloudfront_cache_policy" ...
>   리소스 참조명 하이픈/언더스코어 불일치
> ```
> 
> `data.aws_cloudfront_cache_policy.caching-optimized` →
> `data.aws_cloudfront_cache_policy.optimized`으로 일관화해서 해결.
> Terraform 리소스명은 하이픈 대신 **언더스코어**를 사용해야 함.
> 

### Step 4 — CORS allowlist 축소 + WAF 추가

```hcl
# S3 CORS: AllowedOrigins="*" → 도메인 allowlist로 축소
resource "aws_s3_bucket_cors_configuration" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["PUT"]
    # 허용 도메인 최소화 (다음은 dummy 도메인 예시입니다)
    allowed_origins = ["https://stagelog-front-origin.com"]
    max_age_seconds = 3000
  }
}

# WAF — CloudFront 스코프 (us-east-1 필수)
resource "aws_wafv2_web_acl" "cloudfront" {
  provider    = aws.us_east_1  # CloudFront WAF는 반드시 us-east-1
  name        = "stagelog-cloudfront-waf"
  scope       = "CLOUDFRONT"

  default_action {
    allow {}
  }

  # AWS Managed Rules
  rule {
    name     = "AWSManagedRulesCommonRuleSet"
    priority = 1

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "CommonRuleSet"
      sampled_requests_enabled   = true
    }
  }

  # Rate 기반 룰 — 5분간 100회 초과 시 429
  rule {
    name     = "RateBasedRule"
    priority = 2

    action {
      block {}
    }

    statement {
      rate_based_statement {
        limit              = 100
        aggregate_key_type = "IP"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "RateBasedRule"
      sampled_requests_enabled   = true
    }
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "stagelog-waf"
    sampled_requests_enabled   = true
  }
}
```

> **트러블슈팅 — WAF HCL 에러**
> 
> 
> ```hcl
> # ❌ 잘못된 문법
> default_action = { allow = {} }
> 
> # ✅ 올바른 nested block 형태
> default_action {
>   allow {}
> }
> ```
> 
> WAF의 `default_action`, `action`은 `=` 할당이 아닌 **block** 문법을 사용.
> 

### Step 5 — Django Backend: Presigned URL 발급 및 `file_url` 생성 로직

MVP 단계에서는 Presigned URL과 `file_url`을 하나의 뷰 함수에서 생성하고,
`file_url`을 S3 직링크 형태(`https://{bucket}.s3.{region}.amazonaws.com/{key}`)로 반환했습니다.

S3 Private 전환 이후 이 구조를 유지하면 **DB에 저장된 모든 `image_url`이 403을 반환**합니다.
URL 생성 로직 전체를 CloudFront 기반으로 전환해야 했습니다.

### 설계 결정: views / services 책임 분리

단순히 URL 포맷을 교체하는 것을 넘어, 다음 두 가지 관심사를 명확히 분리했습니다.

| 레이어 | 파일 | 책임 |
| --- | --- | --- |
| 인프라 연동 로직 | `uploads/services.py` | S3 키 생성, Presigned PUT URL 발급, 표시용 URL |
| HTTP 요청/응답 처리 | `uploads/views.py` | 입력 검증, settings 읽기, 서비스 호출, 응답 반환 |

### services.py — 핵심 로직 3개

```python
# uploads/services.py

import re
import uuid
from datetime import datetime, timezone

import boto3
from botocore.config import Config

_FILENAME_SAFE_RE = re.compile(r"[^a-zA-Z0-9._-]+")

def _safe_filename(filename: str) -> str:
    """파일명에서 경로 구분자·특수문자를 제거해 S3 키에 안전한 형태로 변환"""
    name = (filename or "").strip()
    if not name:
        return "file"
    name = name.split("/")[-1].split("\\")[-1]  # 경로 제거
    name = name[:120]
    name = _FILENAME_SAFE_RE.sub("_", name).strip("._")
    return name or "file"

def build_object_key(prefix: str, user_id: int, filename: str) -> str:
    """
    S3 Object Key 생성.
    구조: {prefix}/{user_id}/{YYYY/MM/DD}/{uuid}_{safe_filename}
    예시: uploads/42/2026/02/23/a3f8d1_photo.png
    """
    p = (prefix or "uploads/").strip()
    if not p.endswith("/"):
        p += "/"
    safe = _safe_filename(filename)
    now = datetime.now(timezone.utc)
    date_path = now.strftime("%Y/%m/%d")
    uid = uuid.uuid4().hex
    return f"{p}{user_id}/{date_path}/{uid}_{safe}"

def make_public_url(bucket: str, region: str, key: str, public_base_url: str | None = None) -> str:
    """
    표시용 URL 생성.
    - public_base_url이 설정된 경우: CloudFront URL 반환
    - 미설정(None)인 경우: S3 virtual-hosted-style URL로 fallback
    """
    if public_base_url:
        base = public_base_url.rstrip("/")
        return f"{base}/{key}"
    # virtual-hosted-style (AWS 권장, region 명시)
    return f"https://{bucket}.s3.{region}.amazonaws.com/{key}"

def generate_presigned_put_url(
    *,
    bucket: str,
    region: str,
    key: str,
    expires_in: int,
) -> str:
    """EC2 IAM Role 기반 credential chain으로 Presigned PUT URL 발급"""
    client = boto3.client(
        "s3",
        region_name=region,
        config=Config(signature_version="s3v4"),
    )
    return client.generate_presigned_url(
        ClientMethod="put_object",
        Params={"Bucket": bucket, "Key": key},
        ExpiresIn=expires_in,
        HttpMethod="PUT",
    )
```

**`build_object_key`의 Key 구조:**

- `{user_id}/` : 특정 사용자 오브젝트를 S3 Lifecycle 정책이나 IAM Condition으로 격리 가능
- `{YYYY/MM/DD}/` : S3 목록 조회 시 날짜 기준 파티셔닝, 운영 비용 절감
- `{uuid}_{safe_filename}` : 충돌 방지 + 원본 파일명 추적 가능

### views.py — HTTP 처리 및 settings 연동

```python
# uploads/views.py

import json
from django.conf import settings
from django.views.decorators.csrf import csrf_exempt
from django.views.decorators.http import require_POST

from common.utils import common_response, login_check
from .services import build_object_key, generate_presigned_put_url, make_public_url

def _is_allowed_image(content_type: str, filename: str) -> bool:
    """Content-Type과 확장자를 모두 검사해 허용 이미지 형식 제한"""
    ct = (content_type or "").strip().lower()
    fn = (filename or "").strip().lower()
    if not ct.startswith("image/"):
        return False
    allowed_ext = (".png", ".jpg", ".jpeg", ".webp")
    if not any(fn.endswith(ext) for ext in allowed_ext):
        return False
    allowed_ct = ("image/png", "image/jpeg", "image/webp")
    return ct in allowed_ct

@csrf_exempt
@login_check
@require_POST
def presign_upload(request):
    """
    POST /api/uploads/presign
    Body: { "filename": "a.png", "content_type": "image/png" }

    Response:
    {
      "upload": { "method": "PUT", "url": "...presigned...", "headers": {...} },
      "key": "uploads/42/2026/02/23/uuid_a.png",
      "file_url": "https://xxx.cloudfront.net/uploads/42/.../uuid_a.png",
      "expires_in": 300
    }
    """
    try:
        data = json.loads(request.body or b"{}")
    except json.JSONDecodeError:
        return common_response(False, message="잘못된 JSON 형식입니다.", status=400)

    filename = (data.get("filename") or "").strip()
    content_type = (data.get("content_type") or "").strip()

    if not filename or not content_type:
        return common_response(False, message="filename/content_type는 필수입니다.", status=400)

    if not _is_allowed_image(content_type, filename):
        return common_response(False, message="이미지 업로드는 png/jpg/jpeg/webp만 허용됩니다.", status=400)

    # settings에서 읽기 — 여러 환경변수 키 fallback 지원
    bucket = getattr(settings, "S3_UPLOAD_BUCKET", "") or ""
    region = getattr(settings, "AWS_REGION", "") or ""
    prefix = getattr(settings, "S3_UPLOAD_PREFIX", "uploads/")
    expires_in = int(getattr(settings, "S3_PRESIGN_EXPIRES", 300) or 300)
    public_base_url = getattr(settings, "S3_PUBLIC_BASE_URL", None)  # None이면 S3 fallback

    if not bucket or not region:
        return common_response(False, message="S3 설정이 누락되었습니다.(bucket/region)", status=500)

    key = build_object_key(prefix=prefix, user_id=request.user_id, filename=filename)

    try:
        presigned_url = generate_presigned_put_url(
            bucket=bucket, region=region, key=key, expires_in=expires_in,
        )
    except Exception:
        return common_response(False, message="Presigned URL 생성에 실패했습니다.", status=500)

    file_url = make_public_url(
        bucket=bucket, region=region, key=key, public_base_url=public_base_url
    )

    return common_response(True, data={
        "upload": {
            "method": "PUT",
            "url": presigned_url,
            "headers": {"Content-Type": content_type},  # FE가 PUT 시 함께 전송
        },
        "key": key,
        "file_url": file_url,   # DB에 저장할 표시용 URL (image_url 컬럼)
        "expires_in": expires_in,
    }, message="Presign 발급 성공", status=200)
```

### settings.py — S3 관련 환경변수 설정

```python
# config/settings.py (S3 관련 부분)

AWS_REGION = env("AWS_REGION", default=env("AWS_DEFAULT_REGION", default="ap-northeast-2"))

# S3_UPLOAD_BUCKET: 여러 환경변수 키 fallback 지원
S3_UPLOAD_BUCKET = env(
    "S3_UPLOAD_BUCKET",
    default=env("AWS_STORAGE_BUCKET_NAME", default=env("S3_BUCKET", default="")),
)

S3_UPLOAD_PREFIX  = env("S3_UPLOAD_PREFIX", default="uploads/")
S3_PRESIGN_EXPIRES = env.int("S3_PRESIGN_EXPIRES", default=300)

# CloudFront 도메인 주입 시 CloudFront URL, 미주입(None) 시 S3 URL로 자동 fallback
S3_PUBLIC_BASE_URL = env("S3_PUBLIC_BASE_URL", default=None)
```

SSM Parameter Store에 CloudFront 도메인 등록:

```bash
aws ssm put-parameter \
  --name "/stagelog/prod/S3_PUBLIC_BASE_URL" \
  --value "https://[CloudFront-Domain].cloudfront.net" \
  --type "SecureString"
```

### URL 전환 흐름 정리

```
[S3 Private 전환 전]
  BE → file_url = "https://{bucket}.s3.{region}.amazonaws.com/{key}"
  FE → S3 직접 접근 → ✅ 200 OK

[S3 Private 전환 후, S3_PUBLIC_BASE_URL 미설정]
  BE → file_url = "https://{bucket}.s3.{region}.amazonaws.com/{key}"  (fallback)
  FE → S3 직접 접근 → ❌ 403 Forbidden  ← 장애 발생 지점

[S3 Private 전환 후, S3_PUBLIC_BASE_URL 설정 완료]
  BE → file_url = "https://[CF-Domain].cloudfront.net/{key}"
  FE → CloudFront OAC → S3 Private → ✅ 200 OK
```

---

## 결과 & 검증

### 3중 검증 시나리오

| 검증 항목 | 기대 결과 | 실제 결과 |
| --- | --- | --- |
| S3 직접 URL 접근 | 403 Forbidden | ✅ 403 확인 |
| CloudFront 경유 URL 접근 | 200 OK | ✅ 200 확인 |
| WAF Rate 룰 초과 요청 | 429 차단 | ✅ 429 확인 |

```bash
# 검증 커맨드

# 1. S3 직접 접근 — 403 확인
curl -I "https://stagelog-uploads.s3.ap-northeast-2.amazonaws.com/uploads/42/.../uuid_photo.png"
# HTTP/1.1 403 Forbidden  ← Public 차단 증명

# 2. CloudFront 경유 — 200 확인
curl -I "https://[CloudFront-Domain].cloudfront.net/uploads/42/.../uuid_photo.png"
# HTTP/1.1 200 OK  ← OAC 경유 읽기 증명

# 3. 게시글 이미지 정상 표시 확인
# ⚠️ [직접 확인 필요] 실제 게시글 작성 후 브라우저 Network 탭 스크린샷 첨부 권장
```

### 최종 인프라 구조

```
[브라우저]
  │
  ├─ 이미지 업로드 요청
  │    └─ BE(Django) → Presigned PUT URL 발급
  │         └─ FE → S3 직접 PUT (서버 부하 없음)
  │
  └─ 이미지 표시
       └─ CloudFront URL (DB 저장값)
            └─ WAF(WebACL) → CloudFront OAC → S3 Private
```

---

## Postmortem: 이미지 미노출 장애 (S3 Private 전환 완료 시점)

S3 Private 전환 완료 후 게시글의 이미지가 전부 표시되지 않는 장애가 발생.

### 원인 분석

```
증상: 업로드는 성공 → 게시글 image_url이 S3 직링크 → Private S3에서 403 → UI 깨짐

원인: settings.py에서 S3_PUBLIC_BASE_URL의 default=None 상태였고,
      SSM Parameter Store에 값이 등록되어 있었으나
      해당 파라미터가 컨테이너 환경변수로 실제 주입되지 않은 채 배포됨.

결과: public_base_url이 None → make_public_url()이 S3 fallback URL 반환
      → DB에 S3 직링크 저장 → Private 버킷에서 403
```

→ SSM에 파라미터가 있다는 것과 컨테이너가 그 값을 실제로 "읽고 있다"는 것은 별개인 지점 파악

### 교훈

> **"환경변수가 SSM에 등록되어 있는 것과, 컨테이너가 실제로 읽고 있는 것은 다르다"**
> 

배포 후 다음 검증 단계를 파이프라인에 추가했습니다:

```bash
# 방법 1: 컨테이너 내부에서 환경변수 실제 주입 확인
docker run --rm stagelog-api:latest env | grep S3_PUBLIC_BASE_URL

# 방법 2: 애플리케이션 코드가 해당 설정을 실제로 참조하는지 확인
docker run --rm stagelog-api:latest \
  grep -n "S3_PUBLIC_BASE_URL" /app/config/settings.py

# 방법 3: 배포 직후 presign API를 호출해 반환된 file_url이 CloudFront 도메인인지 확인
curl -s -X POST https://api.stagelog.com/api/uploads/presign \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"filename":"test.png","content_type":"image/png"}' \
  | jq '.data.file_url'
# 기대값: "https://[CF-Domain].cloudfront.net/..."
# 장애값: "https://stagelog-uploads.s3.ap-northeast-2.amazonaws.com/..."
```

---

## 회고

### 잘한 것

**설계 → 구현 → 검증을 ADR로 연결한 것.**

**3중 검증 체계.** S3 403 + CloudFront 200 + WAF 429를 각각 curl로 확인한 것이 "보안이 실제로 작동하고 있다"는 Evidence로 마련했고, 단순히 "설정했다"가 수준을 넘어 **증거를 남기는 습관**의 중요성과 일상화 시켜야하는 점 체감

**views / services 책임 분리.** URL 생성 로직을 `services.py`로 분리한 덕분에, `make_public_url()`의 `public_base_url` 파라미터 하나로 CloudFront/S3 fallback을 제어할 수 있었습니다. 코드 변경 없이 환경변수 주입만으로 동작을 전환할 수 있는 구조가 실제 장애 복구 시간을 단축할 수 있음을 알게됨.

### 다음에 다르게 할 것

**배포 검증 단계에 presign API E2E 테스트 포함.** 환경변수 미주입 장애를 사전에 막으려면, CI/CD 파이프라인의 smoke test 단계에서 `file_url`이 CloudFront 도메인으로 시작하는지 자동으로 검증해야 합니다.

**WAF CloudWatch 대시보드 선행 구성.** WAF Rate 룰이 실제로 발동되는지 확인하려면 CloudWatch 메트릭 대시보드가 먼저 있어야 합니다. 검증 당시에는 수동으로 확인했으나, 운영 단계에서는 알람까지 연결하는 것이 필요

---