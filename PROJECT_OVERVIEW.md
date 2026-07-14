# Redot Backend API Server

AI 프롬프트 마켓플레이스 플랫폼의 백엔드 서버입니다. 셀러가 AI 이미지 생성 프롬프트를 등록·판매하고, 구매자가 프롬프트를 구매하여 AI 이미지를 생성할 수 있는 서비스입니다.

---

## 목차

- [기술 스택](#기술-스택)
- [아키텍처 개요](#아키텍처-개요)
- [주요 기능](#주요-기능)
- [프로젝트 구조](#프로젝트-구조)
- [도메인 모델](#도메인-모델)
- [API 명세](#api-명세)
- [인증 및 보안](#인증-및-보안)
- [결제 및 크레딧 시스템](#결제-및-크레딧-시스템)
- [AI 이미지 생성 Flow](#ai-이미지-생성-flow)
- [외부 서비스 연동](#외부-서비스-연동)
- [환경 설정](#환경-설정)
- [실행 방법](#실행-방법)

---

## 기술 스택

| 분류 | 기술 |
|------|------|
| Language | Java 21 |
| Framework | Spring Boot 3.5.9 |
| Build | Gradle |
| Database | H2 (local), MySQL 8 (dev/prod) |
| ORM | Spring Data JPA / Hibernate |
| Auth | Spring Security + OAuth2 Client + JWT (jjwt 0.11.5) |
| HTTP Client | Spring Cloud OpenFeign |
| Storage | Cloudflare R2 (AWS S3 호환) |
| Payment | PortOne SDK v0.22.0 |
| Rate Limiting | Bucket4j v8.16.0 |
| API Docs | SpringDoc OpenAPI 3.0 (Swagger UI) |
| Env | spring-dotenv |

---

## 아키텍처 개요

```
[Frontend]
    │
    │  HTTPS
    ▼
[Nginx] (리버스 프록시)
    │
    ▼
[Spring Boot API Server]
    │
    ├── SecurityFilter (JWT 검증)
    ├── Controllers
    │     ├── UserController
    │     ├── ProductController
    │     ├── GenerationController
    │     ├── CreditController
    │     ├── LibraryController
    │     ├── PurchaseController
    │     ├── MetadataController
    │     ├── ImageController
    │     ├── AuthController
    │     └── CallbackController  ◄── Webhook (KIE AI)
    │
    ├── Services
    ├── Repositories
    └── Domain Entities
          │
          ▼
       [MySQL]         [Cloudflare R2]        [KIE AI]      [PortOne]
       (RDB)           (Object Storage)       (AI API)      (Payment)
```

**요청 처리 흐름:**
1. 클라이언트 요청 → Nginx (리버스 프록시) → Spring Boot
2. `JwtAuthenticationFilter`가 Authorization 헤더에서 JWT 토큰 검증
3. Spring Security 역할 기반 접근 제어 (GUEST / USER / SELLER / ADMIN)
4. Controller → Service → Repository 계층 처리

---

## 주요 기능

### 사용자 관리
- **소셜 로그인**: Google, Kakao, Naver OAuth2
- **역할 체계**: GUEST → USER → SELLER (업그레이드 가능)
- **프로필 관리**: 닉네임, 소개, 프로필 이미지 (Cloudflare R2)
- **회원 탈퇴**: 소프트 삭제 (deletedAt 기록)

### 프롬프트 마켓플레이스
- **프롬프트 등록**: 셀러가 AI 프롬프트 템플릿 판매 등록
- **변수 시스템**: 프롬프트 내 `{변수명}` 플레이스홀더로 개인화 생성 지원
- **룩북 이미지**: 프롬프트 미리보기 샘플 이미지 관리
- **태그/카테고리**: 검색 및 분류 지원
- **상태 관리**: PENDING / APPROVED / REJECTED 심사 상태

### AI 이미지 생성
- **다중 AI 모델**: KIE AI (기본), Midjourney 전략 패턴으로 확장
- **비동기 처리**: 생성 요청 → taskId 발급 → Webhook 콜백 수신
- **참조 이미지**: Midjourney img2img 지원
- **옵션 선택**: 화면 비율, 해상도 등 모델별 옵션

### 결제 및 크레딧
- **PortOne 결제**: 카드 등 다양한 결제 수단 지원
- **크레딧 시스템**: 결제 금액을 크레딧으로 환전하여 AI 생성에 사용
- **보너스 정책**: 충전 금액에 따른 보너스 크레딧 지급
- **거래 내역**: 충전/사용 전체 이력 조회

### 라이브러리
- **구매 목록**: 사용자가 구매한 프롬프트 및 생성 이미지 관리
- **판매 내역**: 셀러의 판매 현황 조회
- **이미지 공개/비공개**: 생성 이미지 공개 여부 설정

---

## 프로젝트 구조

```
src/main/java/com/redot/
├── RedotApplication.java
│
├── auth/                          # 인증 관련
│   ├── controller/AuthController.java
│   ├── service/AuthService.java
│   ├── repository/RefreshTokenRepository.java
│   ├── JwtTokenProvider.java      # JWT 토큰 생성/검증
│   ├── JwtAuthenticationFilter.java
│   ├── CustomOAuth2UserService.java
│   ├── OAuth2SuccessHandler.java
│   ├── OAuth2FailureHandler.java
│   └── OAuthAttributes.java       # 소셜 로그인 속성 매핑
│
├── config/
│   ├── SecurityConfig.java        # Security, CORS, 권한 설정
│   ├── R2Config.java              # Cloudflare R2 S3 클라이언트
│   ├── PortOneConfig.java         # PortOne SDK 설정
│   ├── RestConfig.java            # RestTemplate 설정
│   └── SwaggerConfig.java         # OpenAPI 문서 설정
│
├── controller/
│   ├── user/UserController.java
│   ├── GenerationController.java
│   ├── ProductController.java
│   ├── CreditController.java
│   ├── PurchaseController.java
│   ├── LibraryController.java
│   ├── MetadataController.java
│   ├── ImageController.java
│   ├── CallbackController.java    # AI 생성 완료 Webhook
│   ├── HealthCheckController.java
│   └── DevLoginController.java    # 개발용 토큰 발급
│
├── service/
│   ├── user/UserService.java
│   ├── GenerationService.java     # AI 이미지 생성 오케스트레이션
│   ├── ProductService.java
│   ├── CreditService.java
│   ├── PurchaseService.java
│   ├── LibraryService.java
│   ├── AiModelService.java
│   ├── CategoryService.java
│   ├── TagService.java
│   ├── KieAiClient.java           # OpenFeign KIE AI 클라이언트
│   ├── RateLimiterService.java    # Bucket4j 레이트 리미팅
│   ├── ai/                        # AI 요청 전략 패턴
│   │   ├── AiRequestStrategy.java
│   │   ├── DefaultKieStrategy.java
│   │   └── MidjourneyStrategy.java
│   └── image/                     # 스토리지 관리
│       ├── ImageManager.java
│       └── R2ImageManager.java
│
├── domain/
│   ├── user/
│   │   ├── User.java
│   │   └── Role.java              # GUEST, USER, SELLER, ADMIN
│   ├── Prompt.java
│   ├── PromptVariable.java
│   ├── LookbookImage.java
│   ├── LookbookImageVariableOption.java
│   ├── GeneratedImage.java
│   ├── GeneratedImageVariableValue.java
│   ├── Purchase.java
│   ├── Category.java
│   ├── AiModel.java
│   ├── ModelOption.java
│   ├── Tag.java
│   ├── CreditTransaction.java
│   ├── PaymentHistory.java
│   ├── CreditChargeOption.java
│   └── BonusCreditPolicy.java
│
├── dto/                           # Request / Response DTO
├── repository/                    # Spring Data JPA Repositories
├── exception/
│   ├── BusinessException.java
│   ├── ErrorCode.java             # 40+ 에러 코드 정의
│   └── GlobalExceptionHandler.java
└── util/
    └── ImageFileUtils.java
```

---

## 도메인 모델

```
User (seller) ──1:N──► Prompt ──N:M──► Tag
                           │
                           ├──1:N──► PromptVariable
                           │              │
                           │              └──1:N──► GeneratedImageVariableValue
                           │
                           └──1:N──► LookbookImage
                                         └──1:N──► LookbookImageVariableOption

User (buyer) ──1:N──► Purchase ──1:N──► GeneratedImage
                          └── (ref) Prompt

User ──1:N──► CreditTransaction
User ──1:N──► PaymentHistory
```

### 주요 엔티티

| 엔티티 | 설명 |
|--------|------|
| `User` | 사용자. creditBalance로 크레딧 잔액 관리. role로 권한 구분 |
| `Prompt` | 판매용 프롬프트 템플릿. price(크레딧), masterPrompt(변수 포함 원문) |
| `PromptVariable` | 프롬프트 내 변수 정의 (`{keyName}` 형태) |
| `Purchase` | 구매 기록. 구매 당시 price 스냅샷 저장 |
| `GeneratedImage` | 생성된 이미지. taskId로 AI 서비스와 상태 추적 |
| `AiModel` | 사용 가능한 AI 모델 (KIE, Midjourney 등) |
| `CreditTransaction` | 크레딧 충전(CHARGE)/사용(USAGE) 이력 |
| `BonusCreditPolicy` | 충전 금액 구간별 보너스 크레딧 정책 |

---

## API 명세

> Swagger UI: `{서버_주소}/swagger-ui.html` (local/dev 프로필에서 활성화)

### 인증 (`/auth`)

| Method | Endpoint | 설명 | 권한 |
|--------|----------|------|------|
| POST | `/auth/reissue` | Access Token 재발급 (Refresh Token 쿠키 사용) | Public |
| POST | `/auth/logout` | 로그아웃 (Refresh Token 삭제) | 인증 필요 |

### 사용자 (`/user`)

| Method | Endpoint | 설명 | 권한 |
|--------|----------|------|------|
| POST | `/user/signup` | 회원가입 (GUEST → USER) | GUEST |
| GET | `/user/me` | 내 프로필 조회 | USER/SELLER |
| PATCH | `/user/me` | 내 프로필 수정 (닉네임, 소개, 이미지) | USER/SELLER |
| DELETE | `/user/me` | 회원 탈퇴 | USER/SELLER |
| GET | `/user/{userId}` | 특정 사용자 공개 프로필 조회 | Public |
| POST | `/user/upgrade-seller` | 셀러 업그레이드 신청 | USER |

### 프롬프트/상품 (`/product`)

| Method | Endpoint | 설명 | 권한 |
|--------|----------|------|------|
| POST | `/product` | 프롬프트 등록 | SELLER |
| PATCH | `/product/{id}` | 프롬프트 수정 | SELLER |
| DELETE | `/product/{id}` | 프롬프트 삭제 | SELLER |
| GET | `/product` | 상품 목록 조회 (검색/필터/정렬) | Public |
| GET | `/product/{id}` | 상품 상세 조회 | Public |
| GET | `/product/{id}/purchase` | 구매 페이지용 상품 조회 | Public |
| POST | `/product/{id}/estimate` | 변수 적용 가격 견적 | USER/SELLER |
| GET | `/product/user/{userId}` | 특정 셀러의 상품 목록 | Public |

### AI 이미지 생성 (`/product`, `/image`)

| Method | Endpoint | 설명 | 권한 |
|--------|----------|------|------|
| POST | `/product/{promptId}/generate` | 이미지 생성 요청 | USER/SELLER |
| GET | `/image/{imageId}/status` | 생성 상태 조회 | USER/SELLER |
| GET | `/image/{imageId}/download` | 이미지 다운로드 URL 획득 | USER/SELLER |
| PATCH | `/image/{imageId}/visibility` | 이미지 공개/비공개 설정 | USER/SELLER |
| GET | `/image/presigned-upload` | 이미지 업로드용 Presigned URL 발급 | USER/SELLER |

### 구매 (`/product`)

| Method | Endpoint | 설명 | 권한 |
|--------|----------|------|------|
| POST | `/product/{promptId}/purchase` | 프롬프트 구매 | USER/SELLER |

### 크레딧/결제 (`/credit`)

| Method | Endpoint | 설명 | 권한 |
|--------|----------|------|------|
| GET | `/credit/balance` | 크레딧 잔액 조회 | USER/SELLER |
| GET | `/credit/history` | 크레딧 거래 내역 | USER/SELLER |
| GET | `/credit/options` | 충전 옵션 목록 | Public |
| POST | `/credit/charge` | 결제 완료 및 크레딧 충전 | USER/SELLER |

### 라이브러리 (`/user/me/library`)

| Method | Endpoint | 설명 | 권한 |
|--------|----------|------|------|
| GET | `/user/me/library/purchases` | 구매한 프롬프트 및 생성 이미지 목록 | USER/SELLER |
| GET | `/user/me/library/sales` | 판매 내역 조회 | SELLER |

### 메타데이터 (`/metadata`)

| Method | Endpoint | 설명 | 권한 |
|--------|----------|------|------|
| GET | `/metadata/categories` | 카테고리 목록 | Public |
| GET | `/metadata/ai-models` | AI 모델 목록 | Public |

### Webhook (`/callback`)

| Method | Endpoint | 설명 | 권한 |
|--------|----------|------|------|
| POST | `/callback/kie-ai` | AI 이미지 생성 완료 Webhook 수신 | Public |

---

## 인증 및 보안

### OAuth2 소셜 로그인 Flow

```
1. 클라이언트 → /login/oauth2/authorization/{provider}
                (Google | Kakao | Naver)

2. 소셜 제공자 → 콜백 (인증 코드)

3. CustomOAuth2UserService
   └── OAuthAttributes로 제공자별 속성 매핑
   └── 신규 사용자: GUEST 역할로 DB 저장
   └── 기존 사용자: 조회

4. OAuth2SuccessHandler
   └── Refresh Token 생성 → DB 저장
   └── HttpOnly 쿠키에 Refresh Token 설정
   └── 프론트엔드로 리다이렉트

5. 클라이언트 → POST /auth/reissue
   └── Refresh Token 쿠키 → Access Token (30분) 발급
```

### JWT 구조

| 항목 | 값 |
|------|-----|
| 알고리즘 | HS256 |
| Access Token 유효기간 | 30분 |
| Refresh Token 유효기간 | 14일 |
| Claims | sub (userId), role |

### 역할 권한 체계

| 역할 | 접근 가능 기능 |
|------|----------------|
| **GUEST** | 회원가입 |
| **USER** | 상품 조회, 구매, AI 이미지 생성, 크레딧 충전, 셀러 신청 |
| **SELLER** | USER 모든 기능 + 프롬프트 등록/수정/삭제, 판매 내역 조회 |
| **ADMIN** | 전체 관리 (카테고리, AI 모델 등) |

### API 응답 형식

성공:
```json
{
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": { ... }
}
```

에러:
```json
{
  "code": "INSUFFICIENT_CREDIT",
  "message": "크레딧이 부족합니다.",
  "data": null
}
```

---

## 결제 및 크레딧 시스템

### 결제 Flow (PortOne)

```
1. 클라이언트: PortOne 결제 위젯으로 결제 처리 → paymentId 획득

2. 클라이언트 → POST /credit/charge { paymentId }

3. CreditService
   ├── PortOne SDK로 paymentId 검증 (실 결제 확인)
   ├── 금액 유효성 검사 (3,000 ~ 50,000원, 100원 단위)
   ├── BonusCreditPolicy로 보너스 크레딧 산정
   ├── User.creditBalance += (기본 크레딧 + 보너스 크레딧)
   ├── PaymentHistory 저장
   └── CreditTransaction(CHARGE) 저장
```

### 크레딧 사용

- AI 이미지 생성 시 해당 Prompt의 price(크레딧)만큼 차감
- 구매된 프롬프트는 횟수 제한 없이 반복 생성 가능
- `CreditTransaction(USAGE)` 기록 저장

---

## AI 이미지 생성 Flow

```
1. POST /product/{promptId}/generate
   {
     "purchaseId": 123,
     "variables": { "스타일": "애니메이션", "배경": "우주" },
     "aspectRatio": "1:1",
     "referenceImageUrl": null
   }

2. GenerationService
   ├── Purchase 소유권 검증
   ├── 크레딧 차감 (CreditTransaction)
   ├── masterPrompt에 변수 치환
   │    예: "a {스타일} character in {배경}"
   │    → "a 애니메이션 character in 우주"
   ├── AiRequestStrategy 선택 (모델별)
   │    ├── DefaultKieStrategy (기본 KIE 포맷)
   │    └── MidjourneyStrategy (Midjourney 포맷)
   ├── KieAiClient.submit() → taskId 획득
   └── GeneratedImage(PROCESSING) 저장

3. Response: { "imageId": 456, "taskId": "abc-def" }

4. [비동기] KIE AI 처리 완료
   → POST /callback/kie-ai { "task_id": "abc-def", "status": "success", "image_url": "..." }

5. CallbackController
   ├── taskId로 GeneratedImage 조회
   ├── AI 이미지 다운로드
   ├── Cloudflare R2에 재업로드 (영구 저장)
   └── GeneratedImage.status = COMPLETED, imageUrl 저장

6. GET /image/{imageId}/status  →  { "status": "COMPLETED" }
7. GET /image/{imageId}/download  →  R2 Presigned URL
```

---

## 외부 서비스 연동

### Cloudflare R2 (이미지 저장소)

AWS S3 호환 API를 사용하며, 두 가지 버킷을 운영합니다.

| 버킷 | 접근 | 용도 |
|------|------|------|
| `redot-bucket` | Public | 상품 미리보기, 생성 이미지 |
| `redot-secret` | Private | 사용자 프로필 이미지 |

- **업로드**: Presigned PUT URL을 클라이언트에 발급하여 직접 업로드 (서버 무부하)
- **다운로드**: Private 파일은 Presigned GET URL 발급
- **AI 이미지**: Webhook 수신 후 서버가 AI URL에서 다운로드 → R2 재업로드

### KIE AI

OpenFeign 클라이언트로 연동. 전략 패턴(Strategy Pattern)으로 AI 모델별 요청 형식 분리.

```
POST https://api.kie.ai/{endpoint}
Authorization: Bearer {KIE_API_KEY}
{
  "prompt": "치환된 최종 프롬프트",
  "ratio": "1:1",
  "callBackUrl": "https://api.redot.store/callback/kie-ai"
}
```

### PortOne

공식 Java SDK를 사용하여 결제를 서버-투-서버로 검증합니다.

```java
PaymentClient client = new PaymentClient(PORTONE_API_SECRET);
Payment payment = client.getPayment(paymentId);
// status == PaidPayment 확인 후 크레딧 지급
```

### Rate Limiting (Bucket4j)

이미지 업로드 요청(`/image/presigned-upload`)에 사용자별 요청 수 제한 적용. 한도 초과 시 `429 Too Many Requests` 반환.

---

## 환경 설정

### 필수 환경 변수

```env
# JWT
JWT_SECRET_KEY=

# OAuth2 소셜 로그인
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
KAKAO_CLIENT_ID=
KAKAO_CLIENT_SECRET=
NAVER_CLIENT_ID=
NAVER_CLIENT_SECRET=

# Cloudflare R2
R2_ACCESS_KEY=
R2_SECRET_KEY=
R2_ACCOUNT_ID=
R2_PUBLIC_DOMAIN=

# PortOne
PORTONE_API_SECRET=

# KIE AI
KIE_API_KEY=

# AI 생성 완료 Webhook 수신 URL (ex: https://api.example.com)
APP_CALLBACK_URL=

# DB (dev/prod 프로필)
DB_URL=
DB_USERNAME=
DB_PASSWORD=
```

### 프로필별 설정

| 프로필 | DB | DDL | Swagger | 용도 |
|--------|----|-----|---------|------|
| `local` | H2 인메모리 | update | 활성화 | 로컬 개발 |
| `dev` | MySQL | update | 활성화 | 개발 서버 |
| `prod` | MySQL (NCP) | validate | 비활성화 | 운영 서버 |

---

## 실행 방법

### 로컬 실행

```bash
# 1. 저장소 클론
git clone https://github.com/swyp-web12-team06/server.git
cd server

# 2. .env 파일 생성 (루트 디렉토리)
# 필수 환경 변수 입력

# 3. 빌드 및 실행
./gradlew bootRun --args='--spring.profiles.active=local'
```

### API 문서 접근

- Swagger UI: `http://localhost:8080/swagger-ui.html`
- 개발용 JWT 발급: `GET http://localhost:8080/dev/token?userId={id}`

### 헬스 체크

```bash
curl http://localhost:8080/health
# {"code":"SUCCESS","data":{"status":"UP","profile":"local"}}
```

---

## 주요 에러 코드

| HTTP | 코드 | 설명 |
|------|------|------|
| 400 | `INSUFFICIENT_CREDIT` | 크레딧 부족 |
| 400 | `ALREADY_PURCHASED` | 이미 구매한 프롬프트 |
| 400 | `INVALID_PRICE_RANGE` | 유효하지 않은 가격 범위 |
| 400 | `MISSING_VARIABLE_VALUES` | 필수 변수 값 누락 |
| 400 | `INVALID_TAG_COUNT` | 태그 수 범위 초과 (2~5개) |
| 401 | `UNAUTHORIZED` | 인증 필요 |
| 401 | `INVALID_REFRESH_TOKEN` | 유효하지 않은 Refresh Token |
| 403 | `NOT_PURCHASED_ITEM` | 구매하지 않은 프롬프트 |
| 403 | `FORBIDDEN` | 접근 권한 없음 |
| 404 | `USER_NOT_FOUND` | 사용자 없음 |
| 404 | `PROMPT_NOT_FOUND` | 프롬프트 없음 |
| 404 | `IMAGE_NOT_FOUND` | 이미지 없음 |
| 409 | `NICKNAME_DUPLICATION` | 닉네임 중복 |
| 409 | `ALREADY_SELLER` | 이미 셀러 역할 보유 |
| 429 | `TOO_MANY_REQUESTS` | 요청 한도 초과 |
| 500 | `PAYMENT_PROCESSING_FAILED` | 결제 처리 실패 |