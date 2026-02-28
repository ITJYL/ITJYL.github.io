---
title: "02. 시스템 아키텍처"
date: 2026-02-28
categories: [Ruxpress Project]
tags: [아키텍처, AWS, Spring Boot, React]
---

# 02. 시스템 아키텍처

전체 시스템 구성과 데이터 흐름을 정리합니다.

---

## 전체 구성도

(노션/ draw.io 등에서 만든 구성도를 이미지로 넣거나, 아래처럼 텍스트로 요약해 두세요.)

```
[클라이언트]
  - 웹 (React)
  - 모바일 (네이티브 + 웹뷰)

        ↓ HTTPS

[백엔드] Spring Boot (AWS EC2/ECS 등)
  - REST API
  - 세션/인증

        ↓

[데이터/인프라]
  - Maria DB (RDS)
  - Redis (캐시/세션)
  - Kafka (메시지)
  - S3 (파일)
  - Firebase FCM (푸시)
```

---

## 주요 흐름

### 사용자 요청 처리

1. 클라이언트(웹/앱) → API Gateway 또는 Spring Boot
2. 인증/세션 확인 (Redis)
3. 비즈니스 로직 수행, DB 조회/저장 (Maria DB)
4. 필요 시 파일 업로드 (S3), 푸시 발송 (FCM), 메시지 발행 (Kafka)

### 환율/캐싱

- 환율 정보: Redis 캐싱, 주기적 갱신

---

## 인프라 (예시)

| 구분 | 기술 | 용도 |
|------|------|------|
| DB | Maria DB (AWS RDS) | 영구 데이터 |
| 캐시 | Redis | 환율, 세션 |
| 큐 | Kafka | 메신저/비동기 처리 |
| 스토리지 | AWS S3 | 파일 업로드 |
| 푸시 | Firebase FCM | 모바일 알림 |

---

**이전 글:** 01. 기능 명세 (MVP)  
**다음 글:** 03. API 설계
