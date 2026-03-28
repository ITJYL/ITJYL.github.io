---
title: "05. 개발 환경 세팅"
date: 2026-02-28
categories: [Ruxpress Project, 기획·설계]
order: 5
tags: [개발환경, Docker, Spring Boot, React, H2, MariaDB]
---

# 05. 개발 환경 세팅

로컬에서 ruxpress 프로젝트를 실행하기 위한 절차를 정리합니다.

---

## 1. 필요 도구

| 도구 | 버전 | 용도 |
|------|------|------|
| JDK | 21+ | Spring Boot 백엔드 |
| Node.js | 18+ | React 프론트엔드 |
| Docker & Docker Compose | 최신 | 컨테이너 배포 |
| Git | 최신 | 소스 관리 |

---

## 2. 저장소 클론

```bash
git clone https://github.com/ruxpress0228/ruxpress.git
cd ruxpress
```

---

## 3. Docker로 실행 (권장)

가장 간단한 방법입니다. Docker Compose가 백엔드·프론트를 한 번에 올립니다.

### docker-compose.yml

```yaml
services:
  backend:
    build:
      context: ./back
      dockerfile: Dockerfile
    container_name: ruxpress-back
    ports:
      - "8080:8080"
    env_file:
      - ./back/.env
    restart: unless-stopped

  frontend:
    build:
      context: ./front
      dockerfile: Dockerfile
    container_name: ruxpress-front
    ports:
      - "80:80"
    depends_on:
      - backend
    restart: unless-stopped
```

### 실행

```bash
docker compose up -d

# 또는 스크립트 사용
./script/manage.sh start       # Linux/Mac
script\manage.bat start        # Windows
```

### 접속 URL

| 서비스 | URL |
|--------|-----|
| 프론트엔드 | http://localhost |
| 백엔드 API | http://localhost:8080 |
| Health Check | http://localhost:8080/api/health |
| H2 Console | http://localhost:8080/h2-console |
| 연결 테스트 | http://localhost/example |

---

## 4. 로컬 직접 실행 (Docker 없이)

### 4.1 백엔드 (Spring Boot)

```bash
cd back

# 환경 변수 설정 (.env 파일 또는 export)
# JWT_SECRET, JWT_EXPIRATION 등 필수
# SMTP 없으면 인증 코드가 서버 로그에만 출력됨

# 실행 (기본 local 프로필 → H2 인메모리)
./mvnw spring-boot:run
```

`local` 프로필은 H2 인메모리 DB를 사용하므로 별도 DB 설치가 필요 없습니다. 서버 재시작 시 데이터가 초기화됩니다.

### 4.2 프론트엔드 (React)

```bash
cd front

npm install
npm run dev
# → http://localhost:3000 (Vite dev server)
```

---

## 5. 프로필별 설정

### application.yml (공통)

```yaml
server:
  port: 8080

spring:
  application:
    name: ruxpress-back
  profiles:
    active: local
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 60MB
  messages:
    basename: messages/messages
    encoding: UTF-8

app:
  frontend:
    base-url: ${FRONTEND_BASE_URL:http://localhost}
  jwt:
    secret: ${JWT_SECRET:}
    expiration: ${JWT_EXPIRATION:}
```

### application-local.yml (로컬 개발)

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:ruxpress;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
  h2:
    console:
      enabled: true
      path: /h2-console
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: create-drop
    show-sql: true

storage:
  type: local
  local:
    upload-dir: ./uploads

logging:
  level:
    com.ruxpress: DEBUG
```

### 프로필 요약

| 프로필 | 용도 | DB | DDL |
|--------|------|-----|-----|
| local | 로컬 개발 | H2 in-memory | create-drop |
| dev | 개발 서버 | MariaDB (환경변수) | update |
| prod | 운영 서버 | MariaDB (환경변수) | validate |
| docker | Docker 환경 | MariaDB (내부 호스트) | update |

---

## 6. 환경 변수

### 필수

| 변수 | 설명 | 예시 |
|------|------|------|
| JWT_SECRET | JWT 서명 키 (32자 이상) | `my-secret-key-...` |
| JWT_EXPIRATION | 토큰 만료(ms) | `86400000` (24시간) |

### 선택 (SMTP — 없으면 로그에만 코드 출력)

| 변수 | 설명 |
|------|------|
| MAIL_HOST | SMTP 호스트 (예: smtp.gmail.com) |
| MAIL_PORT | 포트 (예: 587) |
| MAIL_USERNAME | 계정 |
| MAIL_PASSWORD | 앱 비밀번호 |

### dev/prod용

| 변수 | 설명 |
|------|------|
| DB_URL | JDBC URL |
| DB_USERNAME | DB 계정 |
| DB_PASSWORD | DB 비밀번호 |

---

## 7. 운영 스크립트 (`script/`)

| 스크립트 | 설명 |
|----------|------|
| init.sh | 서버 최초 환경 설정 (패키지, clone, 이미지 빌드) |
| manage.sh / manage.bat | 서비스 관리 (start/stop/restart) |
| rebuild.sh / rebuild.bat | 코드 변경 후 재배포 (중지→빌드→시작) |

---

## 8. API 테스트

```bash
# Health Check
curl http://localhost:8080/api/health

# 다국어 인사
curl http://localhost:8080/api/v1/examples/hello
curl -H "Accept-Language: ru" http://localhost:8080/api/v1/examples/hello
curl -H "Accept-Language: en" http://localhost:8080/api/v1/examples/hello

# 서버 시간
curl http://localhost:8080/api/v1/examples/time

# Echo
curl -X POST http://localhost:8080/api/v1/examples/echo \
  -H "Content-Type: application/json" \
  -d '{"message":"hello"}'
```

---

## 9. CORS 설정

`WebConfig.java`에서 개발용 CORS를 허용합니다.

- 허용 Origin: `http://localhost:3000`, `http://localhost:80`, `http://localhost`
- 허용 Method: GET, POST, PUT, PATCH, DELETE, OPTIONS
- Credentials: true

---

## 10. 다국어

한국어(ko), 러시아어(ru), 영어(en) 3개 언어 지원.

- **백엔드**: `Accept-Language` 헤더 → `messages_ko.properties`, `messages_ru.properties`, `messages_en.properties`
- **프론트**: `i18n/ko.json`, `ru.json`, `en.json` — 헤더 언어 전환 버튼
