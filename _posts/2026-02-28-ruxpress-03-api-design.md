---
title: "03. API 설계"
date: 2026-02-28
categories: [Ruxpress Project]
tags: [API, REST, Spring Boot]
---

# 03. API 설계

주요 REST API 목록과 규칙을 정리합니다.

---

## 공통 규칙

- **Base URL**: `https://api.example.com` (실제 도메인으로 변경)
- **인증**: JWT 또는 세션 쿠키 (선택 사항 적어 두기)
- **요청/응답**: JSON
- **에러 형식**: 공통 에러 코드/메시지 구조

---

## 회원

| Method | Path | 설명 |
|--------|------|------|
| POST | /api/auth/signup | 회원가입 |
| POST | /api/auth/login | 로그인 |
| POST | /api/auth/logout | 로그아웃 |
| GET | /api/users/me | 내 정보 조회 |
| PATCH | /api/users/me | 내 정보 수정 |

---

## 구매 요청

| Method | Path | 설명 |
|--------|------|------|
| POST | /api/purchase-requests | 구매 요청 등록 |
| GET | /api/purchase-requests | 목록 조회 |
| GET | /api/purchase-requests/:id | 상세 조회 |
| PATCH | /api/purchase-requests/:id | 상태 등 수정 |

---

## 문의

| Method | Path | 설명 |
|--------|------|------|
| POST | /api/inquiries | 문의 등록 |
| GET | /api/inquiries | 내 문의 목록 |
| GET | /api/inquiries/:id | 문의 상세 |

---

## 공지사항

| Method | Path | 설명 |
|--------|------|------|
| GET | /api/notices | 공지 목록 |
| GET | /api/notices/:id | 공지 상세 |

---

## 기타

| Method | Path | 설명 |
|--------|------|------|
| GET | /api/exchange-rates | 환율 조회 (캐시) |
|  |  |  |

---

**이전 글:** 02. 시스템 아키텍처  
**다음 글:** 04. DB 설계
