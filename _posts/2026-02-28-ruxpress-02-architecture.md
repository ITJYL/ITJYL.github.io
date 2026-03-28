---
title: "02. 시스템 아키텍처"
date: 2026-02-28
categories: [Ruxpress Project, 기획·설계]
order: 2
tags: [아키텍처, AWS, Spring Boot, React, Docker]
---

# 02. 시스템 아키텍처

전체 시스템 구성과 데이터 흐름을 ruxpress 소스 기준으로 정리합니다.

---

## 1. 전체 구성도

```
┌─────────────────────────────────────────────────────┐
│                   Docker Compose                     │
│                                                     │
│  ┌──────────────┐       ┌──────────────────────┐    │
│  │   Frontend   │       │      Backend         │    │
│  │   (Nginx)    │──────▶│   (Spring Boot)      │    │
│  │  React 18    │ :80   │   Java 21            │    │
│  │  Vite 5      │       │   :8080              │    │
│  │  TypeScript  │       │                      │    │
│  └──────────────┘       │  ┌─────────────────┐ │    │
│                         │  │ Spring Security  │ │    │
│                         │  │ JWT Auth Filter  │ │    │
│                         │  │ BCrypt           │ │    │
│                         │  └─────────────────┘ │    │
│                         │  ┌─────────────────┐ │    │
│                         │  │ Spring Data JPA  │ │    │
│                         │  └────────┬────────┘ │    │
│                         └───────────┼──────────┘    │
│                                     │               │
└─────────────────────────────────────┼───────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 ▼                  │
                    │         ┌──────────────┐          │
                    │         │   MariaDB     │          │
                    │         │  (AWS RDS)    │          │
                    │         └──────────────┘          │
                    │                                    │
                    │  ┌──────────┐  ┌──────────────┐   │
                    │  │  Redis   │  │  Firebase    │   │
                    │  │ (캐시)    │  │  FCM (푸시)  │   │
                    │  └──────────┘  └──────────────┘   │
                    │                                    │
                    │  ┌──────────┐  ┌──────────────┐   │
                    │  │  Kafka   │  │  AWS S3      │   │
                    │  │ (메시지)  │  │ (파일 스토리지)│   │
                    │  └──────────┘  └──────────────┘   │
                    └────────────────────────────────────┘
```

---

## 2. 기술 스택

| 구분 | 기술 | 버전/비고 |
|------|------|-----------|
| **Backend** | Spring Boot, Java, Spring Security, Spring Data JPA | 3.2.5, Java 21 |
| **Frontend** | React, TypeScript, Vite, React Router | 18, TS 5.6, Vite 5, RR 6 |
| **DB** | H2 (로컬), MariaDB (dev/prod) | H2 인메모리, RDS |
| **인프라** | Docker, Docker Compose, Nginx | 컨테이너 기반 배포 |
| **인증** | JWT (사용자 + 관리자 분리) | BCrypt, Stateless |
| **캐시** | Redis | 환율, 세션 (예정) |
| **큐** | Kafka | 메신저/비동기 (예정) |
| **파일** | AWS S3 / 로컬 스토리지 | 로컬: `./uploads` |
| **푸시** | Firebase FCM | 모바일 알림 (예정) |
| **메일** | Spring Mail (SMTP) | Gmail 또는 환경변수 |
| **다국어** | 한국어, 러시아어, 영어 | Accept-Language 헤더 기반 |

---

## 3. 백엔드 패키지 구조

```
back/src/main/java/com/ruxpress/
├── common/                    # 공통 모듈
│   ├── controller/            # FaviconController, HealthController
│   ├── dto/                   # ApiResponse, PageResponse
│   ├── entity/                # BaseEntity (공통 필드), Attachment
│   ├── exception/             # ErrorCode, BusinessException, GlobalExceptionHandler
│   └── util/                  # JwtUtil, IDGenerateUtil
├── config/                    # 설정
│   ├── SecurityConfig         # CSRF off, JWT 필터, 역할별 경로
│   ├── JwtAuthenticationFilter # Bearer 토큰 파싱·검증
│   ├── JwtTokenProvider       # 관리자 JWT 생성/검증
│   ├── WebConfig              # CORS (localhost:3000, :80)
│   ├── JpaConfig              # JPA Auditing, UTC
│   ├── MailConfig             # JavaMailSender
│   ├── MessageConfig          # Locale (ko/ru/en)
│   └── DataInitializer        # 초기 데이터
└── domain/                    # 도메인별 패키지
    ├── user/                  # 회원 (회원가입, 로그인, 마이페이지)
    ├── purchase/              # 구매 요청
    ├── inquiry/               # 1:1 문의
    ├── notice/                # 공지사항
    ├── exchange/              # 환율
    ├── notification/          # 알림
    ├── balance/               # 잔액 (지갑, 원장)
    ├── banktransfer/          # 계좌이체 (에스크로)
    ├── admin/                 # 관리자
    ├── transaction/           # 거래내역 (TODO)
    └── example/               # 연결 테스트
```

각 도메인은 `controller/`, `service/`, `repository/`, `entity/`, `dto/` 구조.

---

## 4. 프론트엔드 구조

```
front/src/
├── api/                       # API 클라이언트 (balance, bankTransfer, adminPurchase 등)
├── components/
│   ├── layouts/               # UserLayout, AdminLayout
│   ├── ui/                    # shadcn 스타일 (button, dialog, table 등)
│   └── figma/                 # ImageWithFallback
├── hooks/
│   ├── balance/useBalance     # 잔액 훅 (폴링, 이벤트)
│   ├── exchange/useExchangeRate # 환율 훅
│   ├── purchase/usePurchase   # 구매 요청 훅
│   └── useTranslation         # 다국어 훅
├── pages/
│   ├── user/                  # Home, Login, Signup, MyPage, PurchaseRequest, BankTransfer, Notice, Inquiry 등
│   └── admin/                 # Dashboard, Users, Purchases, Notices, Inquiries, BankTransfers, ExchangeRate, Settings 등
├── i18n/                      # ko.json, en.json, ru.json
├── types/                     # TypeScript 타입 정의
├── utils/                     # api, constants, format, stringUtils
└── styles/                    # theme, tailwind, fonts
```

---

## 5. 주요 흐름

### 사용자 요청 처리

1. 클라이언트(React) → Nginx(:80) → Spring Boot(:8080)
2. `JwtAuthenticationFilter`에서 Bearer 토큰 검증
3. Controller → Service → Repository → DB (JPA/MariaDB)
4. 필요 시 메일 발송(SMTP), 파일 업로드(S3/로컬)

### 인증 구조

- **사용자**: `JwtUtil` — 회원가입/로그인 시 JWT 발급, `Authorization: Bearer` 헤더
- **관리자**: `JwtTokenProvider` — 별도 관리자 로그인, `JwtAuthenticationFilter`에서 역할 검증
- `SecurityConfig`에서 `/api/v1/admin/**` 경로는 관리자 인증 필수, 나머지는 설정에 따라 분리

### 환율/잔액

- 환율: 외부 API → `exchange_rates` 테이블, Redis 캐시 (예정)
- 잔액: `user_wallets` + `wallet_ledger_entries` 원장, 입금 확정 시 적립, 구매 시 차감

---

## 6. 인프라

| 구분 | 기술 | 용도 |
|------|------|------|
| 컨테이너 | Docker Compose | backend(:8080) + frontend(:80) |
| DB | H2 (로컬) / MariaDB (dev/prod) | 영구 데이터 |
| 캐시 | Redis | 환율, 세션 |
| 큐 | Kafka | 메신저/비동기 처리 |
| 스토리지 | AWS S3 / 로컬(`./uploads`) | 파일 업로드 |
| 푸시 | Firebase FCM | 모바일 알림 |
| 메일 | SMTP (Gmail) | 인증 코드, 비밀번호 재설정 |

### 프로필별 환경

| 프로필 | 용도 | DB |
|--------|------|-----|
| local (기본) | 로컬 개발 | H2 in-memory |
| dev | 개발 서버 | MariaDB (환경변수) |
| prod | 운영 서버 | MariaDB (환경변수) |
| docker | Docker 환경 | MariaDB (Docker 내부 호스트) |
