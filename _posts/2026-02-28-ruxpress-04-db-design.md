---
title: "04. DB 설계"
date: 2026-02-28
categories: [Ruxpress Project]
tags: [DB, Maria DB, ERD, 스키마]
---

# 04. DB 설계

Maria DB 기준 핵심 테이블과 관계를 정리합니다.

---

## ERD 개요

(노션/ ERD 도구 이미지를 붙이거나, 아래 테이블만 채워도 됩니다.)

---

## 핵심 테이블

### User (회원)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| email | VARCHAR |  |
| password_hash | VARCHAR |  |
| name | VARCHAR |  |
| created_at | DATETIME |  |
| updated_at | DATETIME |  |

---

### Purchase Request (구매 요청)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | FK |  |
| status | VARCHAR | 접수/처리중/완료 등 |
| ... |  | (노션에서 옮겨 적기) |

---

### Inquiry (문의)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | FK |  |
| title | VARCHAR |  |
| content | TEXT |  |
| created_at | DATETIME |  |

---

### Notice (공지사항)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| title | VARCHAR |  |
| content | TEXT |  |
| created_at | DATETIME |  |

---

### Notification (알림)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| user_id | FK |  |
| type | VARCHAR |  |
| read_at | DATETIME |  |

---

### Exchange Rate (환율)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | BIGINT PK |  |
| currency_pair | VARCHAR |  |
| rate | DECIMAL |  |
| fetched_at | DATETIME |  |

---

**이전 글:** 03. API 설계  
**다음 글:** 05. 개발 환경 세팅
