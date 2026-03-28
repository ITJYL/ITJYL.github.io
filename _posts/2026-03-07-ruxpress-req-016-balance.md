---
title: "REQ-016 잔액표시 시스템 구현 정리"
date: 2026-03-07
categories: [Ruxpress Project, REQ-016 잔액]
order: 0
tags: [Spring Boot, React, 잔액, 지갑, 원장, JWT]
---

# REQ-016 잔액표시 시스템 구현 정리

요구사항 **REQ-016 잔액 표시 시스템**의 구현 내용을 커밋 기준으로 정리합니다.  
브랜치: `feature/REQ-016-Balance-check`, 커밋: `[feat] REQ-016 잔액표시 시스템`

---

## 1. 요구사항 대응

| 요구사항 | 구현 여부 | 비고 |
|----------|-----------|------|
| **REQ-016-01** 잔액 충전 | ✅ | 계좌이체 입금 확정 시 자동 적립 (`creditForBankDeposit`) |
| **REQ-016-02** 잔액 표시 | ✅ | 메인 홈·구매 요청 폼·이체 페이지에 실시간 잔액 표시 |
| **REQ-016-03** 잔액 사용 내역 | ✅ | 원장(Ledger) 페이지네이션 API, 충전/차감 내역 조회 |

추가 구현:
- 구매 요청 제출 시 **잔액 차감** (잔액 부족 시 제출 불가)
- 구매 취소/거절 시 **잔액 환급**
- 이체 환불 시 **잔액 차감** (입금 타입만)
- **멱등성** 보장 (중복 요청 시 이중 처리 방지)
- **낙관적 락** (`@Version`) + 동시 요청 충돌 처리

---

## 2. 전체 흐름

### 잔액 충전 (입금 확정)

```
사용자: 이체 신청 → 관리자: 입금 확정
  → BankTransferService.confirmDeposit()
    → BalanceService.creditForBankDeposit(userId, amount, transferLedgerEntryId)
      → UserWallet.credit(amount)   // 잔액 증가
      → WalletLedgerEntry 저장      // 원장 기록 (CREDIT_BANK_DEPOSIT)
```

### 잔액 차감 (구매 제출)

```
사용자: 구매 요청 제출 (status=SUBMITTED)
  → PurchaseService.createPurchaseRequest()
    → BalanceService.debitForPurchase(userId, totalAmountKrw, purchaseRequestId)
      → UserWallet.debit(amount)    // 잔액 차감 (부족 시 예외)
      → WalletLedgerEntry 저장      // 원장 기록 (DEBIT_PURCHASE)
```

### 잔액 환급 (구매 취소/거절)

```
관리자: 구매 상태 CANCELLED 또는 REJECTED로 변경
  → PurchaseService.updatePurchaseRequestStatus()
    → BalanceService.creditForPurchaseRefund(userId, totalAmountKrw, purchaseRequestId)
      → UserWallet.credit(amount)   // 잔액 복원
      → WalletLedgerEntry 저장      // 원장 기록 (CREDIT_PURCHASE_REFUND)
```

---

## 3. 백엔드 구조

### 3.1 DB 테이블 (DDL)

**user_wallets** — 사용자별 가용 잔액

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | BIGINT (UNIQUE) | 회원 ID |
| balance | DECIMAL(18,2) | 가용 잔액 (KRW) |
| version | INT | 낙관적 락 (`@Version`) |
| created_at, updated_at | DATETIME |  |

**wallet_ledger_entries** — 충전/차감 감사 원장

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | BIGINT | 회원 ID |
| entry_type | VARCHAR(50) | CREDIT_BANK_DEPOSIT, DEBIT_PURCHASE 등 |
| amount | DECIMAL(18,2) | 금액 (양수) |
| currency | VARCHAR(3) | 기본 KRW |
| transfer_ledger_entry_id | BIGINT (UK) | 입금 확정 원장 ID (멱등 키) |
| purchase_request_id | BIGINT | 구매 요청 ID (entry_type과 복합 UK) |
| transfer_refund_entry_id | BIGINT (UK) | 이체 환불 원장 ID (멱등 키) |
| memo | VARCHAR(500) |  |
| created_at | DATETIME |  |

멱등성은 **Unique Key**로 보장합니다. 같은 `transfer_ledger_entry_id`나 `(purchase_request_id, entry_type)` 조합이 이미 있으면 무시하거나 예외로 처리합니다.

### 3.2 엔티티

**UserWallet**

- `credit(amount)`: 잔액 증가
- `debit(amount)`: 잔액 차감, 부족 시 `INSUFFICIENT_BALANCE` 예외
- `@Version`: 동시 수정 시 `ObjectOptimisticLockingFailureException` 발생

**WalletLedgerEntry**

- 팩토리 메서드로 생성: `creditBankDeposit()`, `debitPurchase()`, `creditPurchaseRefund()`, `debitBankRefund()`
- 타입: `CREDIT_BANK_DEPOSIT`, `CREDIT_CARD`, `CREDIT_PURCHASE_REFUND`, `DEBIT_PURCHASE`, `DEBIT_BANK_REFUND`

### 3.3 API

| Method | Path | 설명 |
|--------|------|------|
| GET | `/api/v1/balances/me` | 내 잔액 조회 |
| GET | `/api/v1/balances/me/ledger?page=&size=` | 내 원장(충전/차감 내역) 페이지네이션 |
| GET | `/api/v1/admin/purchases` | 관리자 구매 요청 목록 |
| GET | `/api/v1/admin/purchases/:id` | 관리자 구매 요청 상세 |
| PATCH | `/api/v1/admin/purchases/:id/status` | 관리자 구매 상태 변경 (취소/거절 시 환급) |

### 3.4 BalanceService 핵심 로직

```java
// 입금 확정 → 지갑 적립
creditForBankDeposit(userId, amount, transferLedgerEntryId)

// 구매 제출 → 잔액 차감
debitForPurchase(userId, amount, purchaseRequestId)

// 구매 취소/거절 → 환급
creditForPurchaseRefund(userId, amount, purchaseRequestId)

// 이체 환불 → 잔액 차감
debitForBankRefund(userId, amount, transferRefundEntryId)
```

모든 메서드에서:
1. **멱등 체크**: 이미 처리된 ID면 no-op (중복 차감/적립 방지)
2. **지갑 조회/생성**: `getOrCreateWallet()` — 없으면 0원 지갑 자동 생성
3. **잔액 변경**: `wallet.credit()` 또는 `wallet.debit()` (부족 시 예외)
4. **원장 저장**: `WalletLedgerEntry` 기록 (UK 위반 시 `CONCURRENT_UPDATE` 예외)

### 3.5 연동: PurchaseService

- 구매 요청 생성 시 status가 `SUBMITTED`이면 → `balanceService.debitForPurchase()` 호출
- 관리자가 상태를 `CANCELLED`/`REJECTED`로 변경 시 → `balanceService.creditForPurchaseRefund()` 호출

### 3.6 연동: BankTransferService

- 입금 확정 시 entry 타입이 `DEPOSIT`이면 → `balanceService.creditForBankDeposit()` 호출
- 환불 시 부모 entry 타입이 `DEPOSIT`이면 → `balanceService.debitForBankRefund()` 호출
- `ESCROW_HOLD` 타입은 지갑 연동 없음

### 3.7 ErrorCode 추가

| 코드 | 상태 | 설명 |
|------|------|------|
| INSUFFICIENT_BALANCE | 400 | 잔액 부족 |
| PURCHASE_NOT_FOUND | 404 | 구매 요청 없음 |
| INVALID_PURCHASE_STATE | 400 | 구매 상태 전환 불가 |

---

## 4. 프론트엔드 구조

### 4.1 API 연동 (`api/balance.ts`)

- `getMyBalance()`: `GET /v1/balances/me` → `BalanceResponse { balance }`
- `getMyWalletLedger(page, size)`: `GET /v1/balances/me/ledger` → 페이지네이션된 원장 목록

### 4.2 커스텀 훅 (`useBalance`)

- `balance`: 현재 잔액 (null이면 비로그인)
- `loading`: 로딩 중 여부
- `refetch()`: 강제 갱신 (구매 제출·입금 후 호출)
- **10초 주기 폴링** + 탭 복귀 시 자동 갱신 + `BALANCE_CHANGE_EVENT`로 전역 알림

### 4.3 잔액 표시 위치

**Home (메인)**
- 로그인 시 잔액 카드 표시: 원화(₩) + 환율 적용한 루블(₽) 환산 금액

**PurchaseRequestForm (구매 요청 폼)**
- 예상 총액 아래에 "사용 가능 잔액" 표시
- **잔액 부족 시**: 빨간색 경고 + 제출 버튼 비활성화 + 부족 메시지

**BankTransfer (이체 페이지)**
- 상단 헤더 옆에 현재 잔액 표시
- 입금 신청 완료 후 `refetchBalance()` 호출로 즉시 갱신

### 4.4 다국어 (i18n)

ko, en, ru 3개 언어에 잔액 관련 키 추가:
- `balance.available` (사용 가능 잔액)
- `balance.insufficient` (잔액 부족)
- `balance.insufficientDetail` (잔액 ₩{balance}, 필요 금액 ₩{total})
- `balance.rubEquivalent` (≈ {amount} ₽)

---

## 5. 테스트 코드

### BalanceServiceTest (13개 테스트)

- 입금 적립 정상, 멱등 무시, 지갑 자동 생성
- 구매 차감 정상, 멱등 무시, 잔액 부족 예외
- 구매 환급 정상, 멱등 무시, 차감 이력 없으면 무시
- 이체 환불 차감 정상, 멱등 무시, 잔액 부족 예외
- 잔액 조회 (지갑 없으면 0)

### BankTransferServiceTest 추가 (5개 테스트)

- 입금 확정 시 DEPOSIT 타입만 지갑 적립 확인
- 입금 확정 시 ESCROW_HOLD 타입은 지갑 미적립 확인
- 환불 시 부모가 DEPOSIT이면 지갑 차감 확인
- 환불 시 부모가 ESCROW_HOLD이면 지갑 미차감 확인

---

## 6. 구현 범위 요약

| 구현됨 | 미구현 (추후) |
|--------|-------------|
| 입금 확정 시 자동 충전 | 카드 충전 |
| 구매 제출 시 잔액 차감 | 내역 다운로드 (CSV, PDF) |
| 구매 취소/거절 시 환급 | 잔액 부족 시 알림 (푸시) |
| 이체 환불 시 차감 | 관리자 수동 잔액 조정 |
| 잔액·원장 API (조회, 페이지네이션) | |
| 메인/구매폼/이체 페이지 잔액 표시 | |
| 잔액 부족 시 제출 차단 | |
| 멱등성 + 낙관적 락 | |
| 다국어 (ko, en, ru) | |
| 단위 테스트 (18개) | |
