---
title: "[Dreamhack] Overheat Write-up"
date: 2026-04-10
categories: [CTF, Web]
tags: [SSRF, Bypass]
---

# [Dreamhack] Overheat Write-up

## 1. 개요
- 요청 카운트(`current`)가 `max`(220,913) 도달 시 플래그 출력.
- 외부 IP 접속 시 2초 지연 발생으로 일반적인 방법으론 불가능.

## 2. 코드 분석
- **지연 로직:** `127.0.0.1` 외 IP는 `before_request`에서 `time.sleep(2)` 수행.
- **SSRF:** `/send`에서 `X-Forwarded-For` 헤더를 통해 대상 IP 및 경로 제어 가능.
- **재귀 호출:** 서버 응답 헤더를 다시 전달하므로 자기 자신을 호출하는 재귀 구조 생성 가능.

## 3. 공격 전략
- `X-Forwarded-For`에 `127.0.0.1:5000/send?run=true&` 주입.
- 로컬 루프백을 통해 2초 지연 없이 카운트 폭증 유도.
- 타임아웃 전까지 한 번의 요청으로 다수의 카운트 획득.

## 4. 익스플로잇 및 결과
- 서버 상태 확인 결과 이미 카운트가 목표치를 상회하여 플래그 노출됨.
- **Flag:** `SP{L31_y0Uu_d0wN_J4m3s_n0rR1S}`

## 5. 결론
- 로컬 환경과 외부 환경의 처리 차이를 이용한 속도 제한 우회.
- SSRF를 통한 내부 재귀 호출의 위험성 확인.
