---
title: "REQ-001 회원가입·로그인·마이페이지·비밀번호 변경/찾기 구현 정리"
date: 2026-03-28
categories: [Ruxpress Project, REQ-001 사용자 인증]
order: 1
tags: [Spring Boot, React, JWT, 회원가입, 로그인, 이메일인증, 비밀번호재설정, 마이페이지]
---

# REQ-001 회원 인증 시스템 구현 정리

요구사항 **REQ-001 사용자 인증 시스템**의 구현 내용을 커밋 기준으로 정리합니다.  
브랜치: `feature/REQ-016-Balance-check`, 커밋: `[feat] REQ-001 이메일 인증 후 회원가입,로그인,마이페이지 연동, 비밀번호 변경/찾기, 로그아웃`

---

## 1. 요구사항 대응

| 요구사항 | 구현 여부 | 비고 |
|----------|-----------|------|
| 이메일 인증 후 회원가입 | ✅ | 6자리 코드 발송 → 인증 후 가입 폼 활성화 |
| 회원가입 시 주소 입력 | ✅ | 우편번호(선택) + 기본주소(필수) + 상세주소(선택) |
| 이메일/비밀번호 로그인 | ✅ | JWT 발급, localStorage 에 토큰·유저 정보 저장 |
| 마이페이지 프로필 조회 | ✅ | `GET /v1/users/me` — 실서버 데이터 연동 |
| 비밀번호 변경 (로그인 상태) | ✅ | 현재 비밀번호 확인 후 변경 |
| 비밀번호 찾기 (비로그인) | ✅ | 이메일로 재설정 링크 발송 → 토큰 기반 변경 |
| 로그아웃 | ✅ | 클라이언트 세션 제거 + 헤더 즉시 반영 |

---

## 2. 전체 흐름

```
[회원가입]
  이메일 입력 → 인증코드 발송(SMTP) → 코드 입력·검증
  → 비밀번호·닉네임·주소 입력 → POST /v1/users/signup

[로그인]
  이메일·비밀번호 → POST /v1/users/login → JWT 발급
  → localStorage 저장 → 헤더 UI 갱신(CustomEvent)

[마이페이지]
  GET /v1/users/me (Bearer JWT) → 프로필·주소·가입일 표시
  → 비밀번호 변경 Dialog

[비밀번호 찾기]
  이메일 입력 → POST /v1/users/password/forgot
  → Verification(PASSWORD_RESET) 레코드 생성 → 재설정 링크 메일
  → 링크 클릭 → POST /v1/users/password/reset (토큰+새 비밀번호)

[로그아웃]
  localStorage 일괄 삭제 → CustomEvent 발행 → 헤더 로그인 버튼 전환
```

---

## 3. Backend 구현

### 3-1. 변경 파일 목록 (Backend)

| 파일 | 변경 내용 |
|------|-----------|
| `JwtUtil.java` | `resolveUserIdFromAuthorizationHeader()` 추가 — Authorization 헤더에서 일반 회원 userId 추출, 관리자 JWT(`role` 클레임 존재)는 null 반환 |
| `UserController.java` | `GET /me`, `POST /me/password`, `POST /password/forgot`, `POST /password/reset` 4개 엔드포인트 추가 |
| `ChangePasswordRequest.java` | 신규 DTO — `currentPassword`, `newPassword` (8자↑, 영문·숫자·특수문자 패턴 검증) |
| `ForgotPasswordRequest.java` | 신규 DTO — `email` (@Email 검증) |
| `ResetPasswordRequest.java` | 신규 DTO — `token`, `newPassword` (패턴 동일) |
| `EmailSignupRequest.java` | `addressPostalCode`, `addressLine1`(필수), `addressLine2` 필드 추가 |
| `UserResponse.java` | 주소 3개 필드 추가 반환 |
| `User.java` (Entity) | `addressPostalCode`, `addressLine1`, `addressLine2` 컬럼 추가, `changePasswordHash()` 메서드 추가 |
| `Verification.java` | `VerificationType.PASSWORD_RESET` 추가, `code` 컬럼 길이 10→64, `type` 길이 10→32 |
| `VerificationRepository.java` | `findTopByTypeAndCodeAndIsVerified…()`, `deleteByTypeAndTarget()` 쿼리 추가 |
| `EmailService.java` | `sendPasswordResetLink()` — 비밀번호 재설정 링크 메일 발송 |
| `PasswordResetService.java` | 신규 — 재설정 토큰 생성·검증·비밀번호 변경 전체 플로우 |
| `UserService.java` | `getProfileForCurrentUser()`, `changePassword()` 추가, `signupWithEmail()` 에 주소 필드 처리 추가 |
| `application.yml` | `app.frontend.base-url`, `app.mail.password-reset.subject` 설정 추가 |

### 3-2. API 엔드포인트

| Method | Path | 설명 | 인증 |
|--------|------|------|------|
| GET | `/v1/users/me` | 로그인 회원 프로필 조회 | Bearer JWT |
| POST | `/v1/users/me/password` | 비밀번호 변경 (현재 비밀번호 확인) | Bearer JWT |
| POST | `/v1/users/password/forgot` | 비밀번호 찾기 (재설정 링크 메일 발송) | 불필요 |
| POST | `/v1/users/password/reset` | 토큰으로 비밀번호 재설정 | 불필요 |

### 3-3. PasswordResetService 핵심 로직

**재설정 요청 (`requestPasswordReset`)**

```java
@Transactional
public void requestPasswordReset(String email) {
    String normalized = email.trim().toLowerCase();
    userRepository.findByEmail(normalized).ifPresent(user -> {
        if (user.getStatus() != UserStatus.ACTIVE) return;
        if (user.getPasswordHash() == null || user.getPasswordHash().isBlank()) return;

        // 기존 토큰 삭제 후 새 토큰 생성
        verificationRepository.deleteByTypeAndTarget(
            Verification.VerificationType.PASSWORD_RESET, normalized);

        String token = UUID.randomUUID().toString();
        Verification row = Verification.builder()
                .type(Verification.VerificationType.PASSWORD_RESET)
                .target(normalized)
                .code(token)
                .isVerified(false)
                .expiresAt(LocalDateTime.now().plusHours(1))
                .build();
        verificationRepository.save(row);

        String resetUrl = frontendBaseUrl + "/reset-password?token=" + token;
        emailService.sendPasswordResetLink(normalized, resetUrl);
    });
    // 미가입 이메일도 동일 응답 → 이메일 노출 방지
}
```

**설계 포인트**:
- 가입 여부와 관계없이 항상 **동일한 성공 메시지** 반환 — 이메일 존재 여부 노출 방지
- UUID 기반 토큰 → `Verification` 테이블에 `PASSWORD_RESET` 타입으로 저장
- 유효시간 **1시간**, 사용 완료 시 `isVerified = true` 처리
- SMTP 미설정 환경에서는 로그에 URL만 출력 (개발 편의)

**비밀번호 재설정 (`resetPassword`)**

```java
@Transactional
public void resetPassword(String token, String newPassword) {
    Verification verification = verificationRepository
            .findTopByTypeAndCodeAndIsVerifiedOrderByCreatedAtDesc(
                Verification.VerificationType.PASSWORD_RESET, token, false)
            .orElseThrow(() -> new BusinessException(
                ErrorCode.VERIFICATION_NOT_FOUND, "유효하지 않거나 이미 사용된 링크"));

    if (verification.isExpired()) {
        throw new BusinessException(ErrorCode.VERIFICATION_EXPIRED, "링크 만료");
    }

    User user = userRepository.findByEmail(verification.getTarget())
            .orElseThrow(() -> new BusinessException(ErrorCode.NOT_FOUND, "회원 없음"));

    user.changePasswordHash(passwordEncoder.encode(newPassword));
    verification.markVerified(); // 1회용 토큰 소모
}
```

### 3-4. JwtUtil — 일반 회원 / 관리자 분리

```java
public Long resolveUserIdFromAuthorizationHeader(String authorizationHeader) {
    // "Bearer xxxxx" 파싱
    String[] parts = authorizationHeader.trim().split("\\s+", 2);
    if (parts.length != 2 || !parts[0].equalsIgnoreCase("Bearer")) return null;

    Claims c = parseToken(parts[1].trim());
    if (c.get("role") != null) return null;   // 관리자 JWT 제외
    String sub = c.getSubject();
    return sub == null ? null : Long.parseLong(sub);
}
```

관리자 JWT에는 `role` 클레임이 포함되어 있어, 동일 토큰 저장소(`STORAGE_KEYS.TOKEN`)를 사용하더라도 **일반 회원 전용 API에서 관리자 토큰이 혼입되지 않도록** 필터링합니다.

### 3-5. User 엔티티 변경

```java
// 주소 필드 추가
@Column(name = "address_postal_code", length = 10)
private String addressPostalCode;

@Column(name = "address_line1", length = 255)
private String addressLine1;

@Column(name = "address_line2", length = 255)
private String addressLine2;

// 비밀번호 해시 변경 메서드
public void changePasswordHash(String passwordHash) {
    this.passwordHash = passwordHash;
}
```

`updateLastLogin()` 메서드를 제거하고 `changePasswordHash()`로 대체하여, 비밀번호 변경 시에만 사용하는 명확한 네이밍으로 변경했습니다.

### 3-6. Verification 엔티티 확장

```java
public enum VerificationType {
    EMAIL,
    PHONE,
    PASSWORD_RESET   // 신규 추가
}
```

`code` 컬럼 길이를 10→64로 확장하여 UUID 토큰을 저장할 수 있도록 했고, `type` 컬럼도 10→32로 확장하여 `PASSWORD_RESET` 문자열을 수용합니다.

---

## 4. Frontend 구현

### 4-1. 변경 파일 목록 (Frontend)

| 파일 | 변경 내용 |
|------|-----------|
| `ForgotPassword.tsx` | 신규 — 비밀번호 찾기 페이지 |
| `ResetPassword.tsx` | 신규 — URL 쿼리 토큰 기반 비밀번호 재설정 페이지 |
| `Signup.tsx` | 이메일 전용으로 리팩터링, 주소 입력 추가, Google/휴대폰 가입 제거 |
| `Login.tsx` | console.log 제거, Google 로그인 제거, `notifyUserAuthChange()` 호출 추가 |
| `MyPage.tsx` | 목업 데이터 → 실서버 API 연동, 비밀번호 변경 Dialog 추가 |
| `UserLayout.tsx` | 로그인 상태에 따라 헤더 UI 분기 (닉네임 표시 / 로그인 버튼) |
| `routes.ts` | `/forgot-password`, `/reset-password` 라우트 추가 |
| `api.ts` | `notifyUserAuthChange()`, `clearUserSession()` 유틸 함수 추가 |
| `constants.ts` | `USER_EMAIL`, `USER_NICKNAME` 키, `USER_AUTH_CHANGE_EVENT` 상수 추가 |
| `domain.ts` | `User` 타입에 주소 필드 3개 추가 |

### 4-2. 인증 상태 실시간 반영 (CustomEvent 패턴)

로그인/로그아웃 후 헤더의 사용자 정보가 즉시 갱신되도록 `CustomEvent`를 활용합니다.

```typescript
// constants.ts
export const USER_AUTH_CHANGE_EVENT = 'ruxpress:user-auth-change';

// api.ts
export function notifyUserAuthChange(): void {
  window.dispatchEvent(new Event(USER_AUTH_CHANGE_EVENT));
}

export function clearUserSession(): void {
  localStorage.removeItem(STORAGE_KEYS.TOKEN);
  localStorage.removeItem(STORAGE_KEYS.USER_ID);
  localStorage.removeItem(STORAGE_KEYS.USER_EMAIL);
  localStorage.removeItem(STORAGE_KEYS.USER_NICKNAME);
  notifyUserAuthChange();
}
```

`UserLayout.tsx`에서 이벤트를 구독하여 `authRevision` 상태를 갱신하면, `userToken`과 `userNickname`이 다시 평가되어 헤더가 즉시 전환됩니다.

```tsx
// UserLayout.tsx
useEffect(() => {
  const syncAuth = () => setAuthRevision((n) => n + 1);
  window.addEventListener(USER_AUTH_CHANGE_EVENT, syncAuth);
  return () => window.removeEventListener(USER_AUTH_CHANGE_EVENT, syncAuth);
}, []);

const userToken = localStorage.getItem(STORAGE_KEYS.TOKEN);
const userNickname = localStorage.getItem(STORAGE_KEYS.USER_NICKNAME);
```

헤더 렌더링 분기:

```tsx
{userToken ? (
  <div className="flex items-center gap-1">
    <span className="text-sm text-gray-700 truncate">{userNickname ?? "회원"}</span>
    <Link to="/mypage">마이페이지 아이콘</Link>
    <Button onClick={handleLogout}>로그아웃 아이콘</Button>
  </div>
) : (
  <Link to="/login">
    <Button variant="ghost" className="text-blue-600">로그인</Button>
  </Link>
)}
```

### 4-3. Signup 리팩터링

**제거된 기능**: 휴대폰 가입 탭, Google 가입 버튼 → **이메일 인증 단일 플로우**로 단순화

**추가된 필드**: 배송지 주소 (우편번호·기본주소·상세주소)

인증 전후 UI가 조건부 렌더링됩니다:
1. **인증 전**: 이메일 입력 + 인증번호 발송/확인
2. **인증 후**: 비밀번호·닉네임·배송지 입력 + 가입 버튼 활성화

### 4-4. MyPage — 실서버 연동

이전 버전은 `mockData.ts`의 `currentUser`를 참조했지만, 이 커밋에서 `GET /v1/users/me` API를 호출하여 실제 서버 데이터로 전환했습니다.

주요 변경:
- **프로필 로딩**: `useCallback` + `useEffect`로 마운트 시 API 호출
- **401 응답 처리**: 세션 제거 후 로그인 페이지 리다이렉트
- **비밀번호 변경**: Dialog로 현재 비밀번호 + 새 비밀번호 입력 → `POST /v1/users/me/password`
- **배송지 표시**: 주소 3개 필드를 공백으로 조인하여 MapPin 아이콘과 함께 출력
- **날짜 포맷**: `formatDateKo()`, `formatDateTimeKo()` 헬퍼로 한국어 날짜 출력

### 4-5. ForgotPassword / ResetPassword 페이지

| 페이지 | 경로 | 동작 |
|--------|------|------|
| `ForgotPassword` | `/forgot-password` | 이메일 입력 → `POST /v1/users/password/forgot` |
| `ResetPassword` | `/reset-password?token=xxx` | URL에서 토큰 추출 → 새 비밀번호 입력 → `POST /v1/users/password/reset` → 로그인 페이지로 이동 |

`ResetPassword`에서 토큰이 URL에 없으면 "링크가 올바르지 않습니다" 안내 메시지가 표시되고 입력 필드가 비활성화됩니다.

---

## 5. 설정 변경

### application.yml

```yaml
app:
  frontend:
    base-url: ${FRONTEND_BASE_URL:http://localhost}
  mail:
    password-reset:
      subject: ${MAIL_PASSWORD_RESET_SUBJECT:Ruxpress 비밀번호 재설정}
```

`FRONTEND_BASE_URL` 환경 변수로 재설정 링크의 도메인을 제어하며, 기본값은 `http://localhost`입니다.

### tsconfig.json

```json
{
  "include": ["src"],
  "exclude": ["src/**/*.test.ts", "src/**/*.spec.ts"]
}
```

테스트 파일을 컴파일 대상에서 제외하여 빌드 속도를 개선했습니다.

---

## 6. 커밋 통계

| 항목 | 수치 |
|------|------|
| 변경 파일 수 | 25개 |
| 추가 라인 | +991 |
| 삭제 라인 | -338 |
| 신규 파일 | 5개 (`ChangePasswordRequest`, `ForgotPasswordRequest`, `ResetPasswordRequest`, `PasswordResetService`, `ForgotPassword.tsx`, `ResetPassword.tsx`) |

---

## 7. 핵심 설계 결정 정리

| 결정 사항 | 이유 |
|-----------|------|
| 비밀번호 찾기 시 항상 동일 응답 | 이메일 존재 여부 노출 방지 (보안) |
| UUID 토큰 + Verification 테이블 재사용 | 별도 테이블 없이 `PASSWORD_RESET` 타입으로 확장 |
| 재설정 토큰 1시간 유효 + 1회용 | 토큰 탈취 리스크 최소화 |
| `role` 클레임으로 관리자/일반 JWT 분리 | 동일 토큰 키 사용 시에도 API 접근 권한 분리 가능 |
| CustomEvent 기반 인증 상태 동기화 | React 상태 관리 없이 localStorage 변경을 Layout에 즉시 반영 |
| 휴대폰/Google 가입 제거 | MVP 범위 축소, 이메일 인증 단일 플로우로 UX 단순화 |
| 비밀번호 패턴 검증 (`영문+숫자+특수문자 8자↑`) | Backend DTO 레벨 `@Pattern` + Frontend 길이 검증 이중 체크 |
