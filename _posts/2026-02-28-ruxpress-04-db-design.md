---
title: "04. DB 설계"
date: 2026-02-28
categories: [Ruxpress Project, 기획·설계]
order: 4
tags: [DB, MariaDB, ERD, JPA]
---

# 04. DB 설계

MariaDB + JPA 엔티티 기준 핵심 테이블을 정리합니다. 로컬은 H2, dev/prod는 MariaDB(AWS RDS) 사용.

---

## 1. 테이블 목록

| 테이블명 | 엔티티 | 설명 |
|----------|--------|------|
| users | User | 회원 |
| verifications | Verification | 이메일/휴대폰 인증 코드 |
| user_devices | — (DDL만) | 사용자 기기 |
| user_social_accounts | — (DDL만) | SNS 계정 연동 |
| purchase_requests | PurchaseRequest | 구매 요청 |
| inquiries | Inquiry | 1:1 문의 |
| inquiry_replies | InquiryReply | 문의 답변 |
| attachments | Attachment | 첨부파일 |
| notices | Notice | 공지사항 |
| notifications | Notification | 알림 |
| exchange_rates | ExchangeRate | 환율 정보 |
| admins | Admin | 관리자 |
| system_settings | SystemSetting | 시스템 설정 (수수료율 등) |
| settlement_accounts | SettlementAccount | 정산 계좌 |
| transfer_ledger_entries | TransferLedgerEntry | 이체 원장 |
| user_wallets | UserWallet | 사용자 지갑 (가용 잔액) |
| wallet_ledger_entries | WalletLedgerEntry | 지갑 원장 (충전/차감) |

공통 상위 클래스: **BaseEntity** — `id`, `createdAt`, `updatedAt`, `deletedAt`

---

## 2. 핵심 테이블 상세

### users (회원)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| email | VARCHAR(255) UNIQUE | 이메일 |
| password_hash | VARCHAR(255) | BCrypt 암호화 |
| phone | VARCHAR(20) | 휴대폰 |
| nickname | VARCHAR(50) | 닉네임 |
| profile_image_url | VARCHAR(500) |  |
| status | ENUM(ACTIVE, SUSPENDED, WITHDRAWN) | 상태 |
| email_verified | BOOLEAN |  |
| phone_verified | BOOLEAN |  |
| signup_type | ENUM(EMAIL, PHONE, GOOGLE) |  |
| timezone | VARCHAR(50) | 기본 Asia/Seoul |
| last_login_at | DATETIME |  |
| created_at, updated_at, deleted_at | DATETIME |  |

### verifications (인증 코드)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| type | ENUM(EMAIL, PHONE) |  |
| target | VARCHAR(255) | 이메일 또는 번호 |
| code | VARCHAR(10) | 6자리 |
| is_verified | BOOLEAN |  |
| attempt_count | INT | 최대 5회 |
| expires_at | DATETIME | 10분 유효 |

### purchase_requests (구매 요청)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | BIGINT FK |  |
| request_number | VARCHAR | 고유 번호 |
| product_name | VARCHAR | 상품명 |
| product_urls | JSON | 다중 URL |
| quantity | INT |  |
| options | JSON | 옵션(색상/사이즈 등) |
| price_rub | DECIMAL | 루블 가격 |
| price_krw | DECIMAL | 원화 가격 |
| exchange_rate | DECIMAL | 적용 환율 |
| fee_rate | DECIMAL | 수수료율 |
| fee_amount | DECIMAL | 수수료 금액 |
| total_amount_krw | DECIMAL | 총액(원화) |
| status | ENUM(DRAFT, SUBMITTED, ...) | 상태 |
| assigned_admin_id | BIGINT | 담당 관리자 |
| admin_memo | TEXT | 관리자 메모 |

### inquiries (문의)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | BIGINT FK |  |
| category | ENUM(ORDER, DELIVERY, PAYMENT, OTHER) |  |
| title | VARCHAR |  |
| content | TEXT |  |
| status | ENUM(WAITING, ANSWERED, CLOSED) |  |

### inquiry_replies (문의 답변)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| inquiry_id | BIGINT FK |  |
| admin_id | BIGINT FK |  |
| content | TEXT |  |
| is_read | BOOLEAN |  |

### notices (공지사항)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| admin_id | BIGINT FK | 작성자 |
| title | VARCHAR |  |
| content | TEXT |  |
| is_pinned | BOOLEAN | 상단 고정 |
| status | ENUM(ACTIVE, HIDDEN) |  |
| view_count | INT |  |

### exchange_rates (환율)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| currency_pair | VARCHAR | 예: RUB_KRW |
| rate | DECIMAL |  |
| source | ENUM(API, MANUAL) |  |
| fetched_at | DATETIME |  |

### user_wallets (지갑)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | BIGINT UK | 회원 1:1 |
| balance | DECIMAL(18,2) | 가용 잔액(KRW) |
| version | INT | 낙관적 락 |

### wallet_ledger_entries (지갑 원장)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | BIGINT |  |
| entry_type | VARCHAR(50) | CREDIT_BANK_DEPOSIT 등 |
| amount | DECIMAL(18,2) |  |
| currency | VARCHAR(3) | 기본 KRW |
| transfer_ledger_entry_id | BIGINT UK | 입금 확정 멱등 키 |
| purchase_request_id | BIGINT | 구매 멱등 키 (entry_type과 복합 UK) |
| transfer_refund_entry_id | BIGINT UK | 환불 멱등 키 |
| memo | VARCHAR(500) |  |

### settlement_accounts (정산 계좌)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| bank_name | VARCHAR |  |
| account_number | VARCHAR |  |
| account_holder | VARCHAR |  |
| is_active | BOOLEAN |  |

### transfer_ledger_entries (이체 원장)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | BIGINT FK |  |
| settlement_account_id | BIGINT FK |  |
| entry_type | ENUM(DEPOSIT, ESCROW_HOLD, SETTLEMENT, REFUND) |  |
| status | ENUM(PENDING, CONFIRMED, CANCELLED, ...) |  |
| amount | DECIMAL |  |
| currency | VARCHAR(3) |  |
| parent_entry_id | BIGINT | 부모 원장 (환불/정산용) |

### admins (관리자)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| username | VARCHAR UK |  |
| password_hash | VARCHAR |  |
| name | VARCHAR |  |
| role | ENUM(SUPER_ADMIN, ADMIN, AGENT) | 역할 |
| status | ENUM(ACTIVE, SUSPENDED) |  |

---

## 3. 주요 관계

- `users` 1 — N `purchase_requests`
- `users` 1 — N `inquiries`
- `inquiries` 1 — N `inquiry_replies`
- `users` 1 — 1 `user_wallets`
- `users` 1 — N `wallet_ledger_entries`
- `users` 1 — N `transfer_ledger_entries`
- `transfer_ledger_entries` 자기 참조 (parent_entry_id)
- `admins` 1 — N `notices`
- `admins` 1 — N `inquiry_replies`

---

## 4. Enum 목록

| Enum | 값 |
|------|-----|
| UserStatus | ACTIVE, SUSPENDED, WITHDRAWN |
| SignupType | EMAIL, PHONE, GOOGLE |
| PurchaseRequestStatus | DRAFT, SUBMITTED, RECEIVED, IN_PROGRESS, COMPLETED, CANCELLED, REJECTED |
| InquiryStatus | WAITING, ANSWERED, CLOSED |
| InquiryCategory | ORDER, DELIVERY, PAYMENT, OTHER |
| NoticeStatus | ACTIVE, HIDDEN |
| ExchangeRateSource | API, MANUAL |
| WalletLedgerEntryType | CREDIT_BANK_DEPOSIT, CREDIT_CARD, CREDIT_PURCHASE_REFUND, DEBIT_PURCHASE, DEBIT_BANK_REFUND |
| TransferLedgerEntryType | DEPOSIT, ESCROW_HOLD, SETTLEMENT, REFUND |
| TransferLedgerStatus | PENDING, CONFIRMED, CANCELLED |
| AdminRole | SUPER_ADMIN, ADMIN, AGENT |
| AdminStatus | ACTIVE, SUSPENDED |
