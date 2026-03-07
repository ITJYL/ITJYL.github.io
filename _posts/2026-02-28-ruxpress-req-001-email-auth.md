---
title: "REQ-001 회원 관리 (이메일 인증) 구현 정리"
date: 2026-02-28
categories: [Ruxpress Project]
subcategory: 구현
order: 0
tags: [Spring Boot, React, 회원가입, 로그인, 이메일 인증, JWT]
---

한러 구매대행 프로젝트 요구사항 명세서 **REQ-001 회원 관리 시스템** 중 **이메일 인증 기반 회원가입·로그인**만 구현한 내용을 소스 기준으로 정리했습니다. (휴대폰 인증, SNS 로그인은 미구현)

---

## 1. 요구사항 대응 (REQ-001)

| 요구사항 | 구현 여부 | 비고 |
|----------|-----------|------|
| **REQ-001-01** 이메일 기반 회원가입 | ✅ | 이메일 중복 확인, 이메일 인증(verification), 비밀번호 암호화 저장 |
| REQ-001-02 휴대폰 인증 회원가입 | ❌ | UI 탭만 있고 백엔드 미연동 |
| REQ-001-03 SNS 간편 로그인 | ❌ | 버튼만 노출 |
| REQ-001-04 로그인 보안 | △ | JWT·세션 없음(Stateless), 새 기기 알림·자동 로그아웃 미구현 |

---

## 2. 전체 흐름

### 회원가입 (이메일)

1. 사용자가 **이메일 입력** → **인증 메일 발송** 요청  
2. 백엔드: 6자리 숫자 코드 생성, 10분 유효, `verifications` 테이블 저장 + SMTP로 메일 발송 (SMTP 미설정 시 로그에만 출력)  
3. 사용자가 **인증번호 입력** → **인증 확인** 요청  
4. 백엔드: 코드·만료·시도 횟수 검증 후 해당 레코드 `isVerified = true`  
5. 사용자가 **비밀번호, 닉네임** 입력 후 **회원가입** 요청  
6. 백엔드: “이메일 인증 완료 + 30분 이내” 조건 확인, 이메일 중복 확인, 비밀번호 BCrypt 암호화 후 `users` 테이블 저장  

### 로그인

1. 사용자가 **이메일·비밀번호** 입력 후 로그인 요청  
2. 백엔드: 이메일로 회원 조회, 비밀번호 일치 확인, `lastLoginAt` 갱신, JWT 발급  
3. 프론트: 응답의 `token`을 `localStorage`에 저장, 이후 API 요청 시 `Authorization: Bearer {token}` 헤더로 전달  

---

## 3. 백엔드 구조 (Spring Boot)

### 3.1 API 엔드포인트

| Method | Path | 설명 |
|--------|------|------|
| POST | `/api/v1/users/email/send-verification` | 이메일 인증 코드 발송 |
| POST | `/api/v1/users/email/verify` | 인증 코드 검증 |
| POST | `/api/v1/users/signup` | 이메일 인증 후 회원가입 |
| POST | `/api/v1/users/login` | 이메일/비밀번호 로그인 (JWT 반환) |

### 3.2 엔티티

**User** (`users` 테이블)

- `id`, `email`(unique), `password_hash`, `nickname`, `phone`, `profile_image_url`
- `status` (ACTIVE, SUSPENDED, WITHDRAWN), `email_verified`, `phone_verified`
- `signup_type` (EMAIL, PHONE, GOOGLE), `timezone`, `last_login_at`, `created_at`, `updated_at` 등  

이메일 가입 시: `signupType=EMAIL`, `emailVerified=true`, `passwordHash`는 BCrypt.

**Verification** (`verifications` 테이블)

- `type` (EMAIL, PHONE), `target`(이메일 또는 번호), `code`(6자리), `isVerified`, `attemptCount`, `expiresAt`, `createdAt`  
인증 코드 검증 시 만료·시도 횟수(최대 5회) 체크 후 `isVerified` 갱신.

### 3.3 서비스·설정 요약

- **EmailVerificationService**: 인증 코드 생성(6자리 숫자), 10분 유효, 저장; 코드 검증 시 만료·시도 횟수·일치 여부 확인.  
- **EmailService**: SMTP로 인증 메일 발송. `spring.mail.password` 미설정 시 실제 발송 없이 로그에만 코드 출력 (로컬/개발용).  
- **UserService**:  
  - `signupWithEmail`: “이메일 인증 완료 + 30분 이내” 검증, 이메일 중복 확인, 닉네임·비밀번호 검증, BCrypt 인코딩 후 User 저장.  
  - `login`: 이메일·비밀번호 검증, 계정 상태(ACTIVE) 확인, `lastLoginAt` 갱신, JWT 생성 후 `LoginResponse`(token, userId, email, nickname) 반환.  
- **SecurityConfig**: Stateless, CSRF 비활성화, `PasswordEncoder`는 BCrypt. (인증 경로는 현재 permitAll, JWT 필터는 별도 적용 전제)  
- **JwtUtil**: `app.jwt.secret`, `app.jwt.expiration` 기반 HMAC-SHA 토큰 생성·파싱, `userId` 추출.  
- **MailConfig**: `Environment`에서 `spring.mail.*` 읽어 `JavaMailSender` 빈 생성 (host, port, username, password trim 처리).

### 3.4 DTO·검증

- **EmailVerificationRequest**: `email` (필수, @Email).  
- **EmailVerifyRequest**: `email`, `code` (6자리 숫자).  
- **EmailSignupRequest**: `email`, `password`(8자 이상, 영문+숫자+특수문자), `nickname`.  
- **LoginRequest**: `email`, `password`.  
- **LoginResponse**: `token`, `userId`, `email`, `nickname`.

예외는 `BusinessException` + `ErrorCode`(인증 만료, 시도 초과, 이메일 중복, 권한 등)로 통일.

---

## 4. 프론트엔드 구조 (React)

### 4.1 회원가입 (Signup)

- **이메일 탭**만 실제 연동:  
  - 이메일 입력 → “인증” 버튼 → `POST /v1/users/email/send-verification`  
  - 6자리 인증번호 입력 → “확인” → `POST /v1/users/email/verify`  
  - 인증 성공 시 비밀번호·비밀번호 확인·닉네임 입력 후 “가입하기” → `POST /v1/users/signup`  
- 휴대폰 탭: 인증/가입 API 미연동 (목업만).  
- Google 가입 버튼: UI만 존재.

### 4.2 로그인 (Login)

- 이메일·비밀번호 입력 → `POST /v1/users/login`  
- 응답 `data.token`을 `localStorage`(예: `STORAGE_KEYS.TOKEN`)에 저장 후 메인(/)으로 이동.  
- `api` 유틸: 요청 시 `Authorization: Bearer {token}` 자동 첨부.

### 4.3 기타

- 공통 API 베이스 URL·헤더(`api.ts`), 토스트(success/error)로 메시지 표시.  
- 로그인 후 인증이 필요한 API는 서버에서 JWT 검증 후 사용 (현재 백엔드는 permitAll이라 추후 필터 적용 필요).

---

## 5. 구현 범위 요약

- **구현됨**: 이메일 인증 코드 발송/검증, 이메일 인증 후 회원가입(이메일 중복 확인, 비밀번호 암호화), 이메일/비밀번호 로그인(JWT 발급·클라이언트 저장).  
- **미구현**: 휴대폰 인증, SNS 로그인, 비밀번호 찾기, JWT 인증 필터·인가 경로, 새 기기 로그인 알림, 세션 타임아웃 등.

이 문서는 **REQ-001** 중 이메일 인증 기반 회원가입·로그인 구현 내용을 소스 기준으로 정리한 것입니다. 휴대폰·SNS는 추후 동일 패턴으로 확장할 수 있습니다.
