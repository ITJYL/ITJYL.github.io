---
title: "05. 개발 환경 세팅"
date: 2026-02-28
categories: [Ruxpress Project]
tags: [개발환경, 로컬, Docker, Spring Boot, React]
---

# 05. 개발 환경 세팅

로컬에서 프로젝트를 실행하기 위한 절차를 정리합니다.

---

## 필요 도구

- [ ] JDK 17+ (Spring Boot)
- [ ] Node.js 18+ (React)
- [ ] Maria DB (로컬 또는 Docker)
- [ ] Redis (로컬 또는 Docker)
- [ ] (선택) Docker / Docker Compose
- [ ] (선택) Kafka 로컬 실행

---

## 백엔드 (Spring Boot)

```bash
# 저장소 클론 후
cd ruxpress-backend

# 설정
# application.yml 또는 .env 에 DB, Redis 등 설정

# 실행
./gradlew bootRun
# 또는
mvn spring-boot:run
```

### 환경 변수 예시

- `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`
- `REDIS_HOST`, `REDIS_PORT`
- `AWS_ACCESS_KEY`, `AWS_SECRET_KEY` (S3 사용 시)

---

## 프론트엔드 (React)

```bash
cd ruxpress-frontend

npm install
npm run dev
```

### 환경 변수 예시

- `VITE_API_BASE_URL` 또는 `REACT_APP_API_BASE_URL`

---

## DB / Redis (Docker 예시)

```bash
# Maria DB
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=local mariadb:latest

# Redis
docker run -d -p 6379:6379 redis:latest
```

---

## 브랜치/컨벤션 (팀 규칙 있으면 적기)

- main: 배포용
- develop: 개발 통합
- feature/기능명: 기능 개발

---

**이전 글:** 04. DB 설계  
**시리즈 목차:** [Ruxpress Project 카테고리](/categories/ruxpress-project/)에서 00~05 확인
