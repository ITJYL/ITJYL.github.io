---
title: "Swift Optional 완전 이해하기"
date: 2026-02-21
categories: [Swift]
tags: [Optional, iOS]
---

## Optional이 왜 필요할까?

Swift는 nil 안전성을 위해 Optional을 도입했다.

```swift
var name: String? = "자연"
print(name!)