---
title: "03. API 설계"
date: 2026-02-28
categories: [Ruxpress Project, 기획·설계]
order: 3
tags: [API, REST, Spring Boot, JWT]
---

# 03. API 설계

ruxpress 백엔드의 REST API 목록을 컨트롤러 기준으로 정리합니다.

---

## 공통 규칙

- **Base URL**: `/api` (Nginx 프록시: `:80` → `:8080`)
- **인증**: JWT Bearer 토큰 (`Authorization: Bearer {token}`)
- **요청/응답**: JSON (`Content-Type: application/json`)
- **다국어**: `Accept-Language` 헤더 (ko, ru, en)
- **공통 응답 구조**: `ApiResponse<T>` — `{ code, message, data }`
- **페이지네이션**: `PageResponse<T>` — `{ content, totalElements, totalPages, page, size }`

---

## 사용자 API

### 회원 (`/api/v1/users`)

| Method | Path | 설명 |
|--------|------|------|
| POST | /email/send-verification | 이메일 인증 코드 발송 |
| POST | /email/verify | 인증 코드 검증 |
| POST | /signup | 이메일 인증 후 회원가입 |
| POST | /login | 이메일/비밀번호 로그인 (JWT 반환) |
| GET | /me | 내 정보 조회 |
| POST | /me/password | 비밀번호 변경 |
| POST | /password/forgot | 비밀번호 찾기 (재설정 메일 발송) |
| POST | /password/reset | 비밀번호 재설정 |

### 구매 요청 (`/api/v1/purchases`)

| Method | Path | 설명 |
|--------|------|------|
| POST | / | 구매 요청 등록 (SUBMITTED 시 잔액 차감) |
| GET | / | 내 구매 요청 목록 (페이지네이션) |
| GET | /recent | 최근 구매 요청 3건 |

### 1:1 문의 (`/api/v1/inquiries`)

| Method | Path | 설명 |
|--------|------|------|
| POST | / | 문의 등록 (multipart, 파일 첨부) |
| GET | / | 내 문의 목록 |
| GET | /{id} | 문의 상세 |
| GET | /attachments/{attachmentId}/download | 첨부파일 다운로드 |

### 공지사항 (`/api/v1/notices`)

| Method | Path | 설명 |
|--------|------|------|
| GET | / | 공지 목록 |
| GET | /{id} | 공지 상세 |

### 환율 (`/api/v1/exchange-rates`)

| Method | Path | 설명 |
|--------|------|------|
| GET | /current | 현재 환율 조회 |
| GET | / | 환율 목록 (히스토리) |
| POST | /fetch | 외부 API에서 환율 가져오기 |
| POST | /manual | 관리자 수동 환율 입력 |

### 잔액 (`/api/v1/balances`)

| Method | Path | 설명 |
|--------|------|------|
| GET | /me | 내 잔액 조회 |
| GET | /me/ledger | 내 원장 (충전/차감 내역, 페이지네이션) |

### 계좌이체 (`/api/v1/bank-transfers`)

| Method | Path | 설명 |
|--------|------|------|
| GET | /settlement-accounts | 정산 계좌 목록 |
| POST | /deposit-reports | 입금 신고 |
| GET | / | 내 이체 내역 목록 |
| GET | /{id} | 이체 상세 |
| GET | /{id}/receipt | 영수증 조회 |

### 기타

| Method | Path | 설명 |
|--------|------|------|
| GET | /api/health | 헬스 체크 |
| GET | /api/v1/examples/hello | 다국어 인사 테스트 |
| GET | /api/v1/examples/time | 서버 시간 |
| POST | /api/v1/examples/echo | 에코 테스트 |

---

## 관리자 API (`/api/v1/admin/*`)

### 관리자 인증

| Method | Path | 설명 |
|--------|------|------|
| POST | /auth/login | 관리자 로그인 (별도 JWT) |

### 회원 관리 (`/admin/users`)

| Method | Path | 설명 |
|--------|------|------|
| GET | / | 회원 목록 (검색/필터) |
| GET | /{id} | 회원 상세 |
| PATCH | /{id}/status | 회원 상태 변경 (정상/정지/탈퇴) |
| GET | /stats | 회원 통계 |

### 구매 관리 (`/admin/purchases`)

| Method | Path | 설명 |
|--------|------|------|
| GET | / | 구매 요청 목록 (상태 필터) |
| GET | /{id} | 구매 요청 상세 |
| PATCH | /{id}/status | 상태 변경 (취소/거절 시 잔액 환급) |

### 문의 관리 (`/admin/inquiries`)

| Method | Path | 설명 |
|--------|------|------|
| GET | / | 문의 목록 |
| GET | /{id} | 문의 상세 |
| POST | /{id}/replies | 답변 작성 |
| PUT | /{id}/replies/{replyId} | 답변 수정 |
| DELETE | /{id}/replies/{replyId} | 답변 삭제 |
| PATCH | /{id}/status | 문의 상태 변경 |

### 공지 관리 (`/admin/notices`)

| Method | Path | 설명 |
|--------|------|------|
| GET | / | 공지 목록 |
| POST | / | 공지 작성 |
| PUT | /{id} | 공지 수정 |
| DELETE | /{id} | 공지 삭제 |
| PATCH | /{id}/pin | 중요 공지 고정/해제 |

### 이체 관리 (`/admin/bank-transfers`)

| Method | Path | 설명 |
|--------|------|------|
| GET | / | 이체 내역 목록 |
| GET | /{id} | 이체 상세 |
| POST | /{id}/confirm | 입금 확정 (DEPOSIT이면 지갑 적립) |
| POST | /{parentId}/settlement | 정산 처리 |
| POST | /{parentId}/refund | 환불 처리 (DEPOSIT이면 지갑 차감) |
| POST | /{id}/cancel | 이체 취소 |

### 정산 계좌 관리 (`/admin/settlement-accounts`)

| Method | Path | 설명 |
|--------|------|------|
| GET | / | 정산 계좌 목록 |
| GET | /{id} | 계좌 상세 |
| POST | / | 계좌 등록 |
| PUT | /{id} | 계좌 수정 |
| DELETE | /{id} | 계좌 삭제 |

### 환율/설정/통계

| Method | Path | 설명 |
|--------|------|------|
| GET | /settings/fee-rate | 수수료율 조회 |
| PUT | /settings/fee-rate | 수수료율 변경 |
| GET | /settings/templates | 답변 템플릿 목록 |
| POST | /settings/templates | 템플릿 등록 |
| PUT | /settings/templates/{id} | 템플릿 수정 |
| DELETE | /settings/templates/{id} | 템플릿 삭제 |
| GET | /stats/dashboard | 대시보드 통계 |

### 관리자 계정 (`/admin/admins`)

| Method | Path | 설명 |
|--------|------|------|
| GET | / | 관리자 목록 |
| POST | / | 관리자 생성 |
| PUT | /{id} | 관리자 수정 |
| PATCH | /{id}/status | 관리자 상태 변경 |

### 웹훅

| Method | Path | 설명 |
|--------|------|------|
| POST | /webhooks/bank-incoming | 은행 입금 웹훅 |

---

## 에러 코드 (ErrorCode)

| 코드 | HTTP | 설명 |
|------|------|------|
| INVALID_INPUT | 400 | 입력값 오류 |
| INVALID_VERIFICATION_CODE | 400 | 인증번호 불일치 |
| VERIFICATION_EXPIRED | 400 | 인증번호 만료 |
| DUPLICATE_EMAIL | 409 | 이메일 중복 |
| UNAUTHORIZED | 401 | 인증 필요 |
| FORBIDDEN | 403 | 권한 없음 |
| INSUFFICIENT_BALANCE | 400 | 잔액 부족 |
| CONCURRENT_UPDATE | 409 | 동시 수정 충돌 |
| PURCHASE_NOT_FOUND | 404 | 구매 요청 없음 |
| INVALID_PURCHASE_STATE | 400 | 구매 상태 전환 불가 |
