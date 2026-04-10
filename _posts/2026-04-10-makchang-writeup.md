# [Dreamhack] 막창 좋아하세요? Write-up

- **Date:** 2026-04-10
- **Category:** Web Hacking
- **Tags:** #PHP #Type_Casting #Scientific_Notation #Length_Constraint

## 1. Problem Description
"🍖 막창 좋아하세요? 🍖"
불판 온도를 입력받아 특정 조건을 만족하면 플래그를 출력하는 PHP 기반 웹 문제임. 소스코드에 플래그 출력 조건이 명시되어 있음.

## 2. Vulnerability Analysis

### A. Strict Length Constraint
- **Logic:** `if (strlen($temp) <= 4)`
- **Problem:** 입력값 `temp`의 길이가 단 4자 이하로 매우 제한적임.

### B. Float Comparison Logic
- **Logic:** `if ((float)$temp > 1000000000000000)`
- **Goal:** 4자 이하의 문자열을 `float`으로 형변환했을 때 10^15 (1,000,000,000,000,000)보다 커야 함.
- 일반적인 숫자 표기법(예: `9999`)으로는 절대 이 수치를 넘을 수 없음.

### C. PHP Scientific Notation Support
- PHP는 문자열을 숫자로 변환할 때 지수 표기법(Scientific Notation)인 `e`를 지원함.
- `1e16`은 10의 16승을 의미하며, 문자열 길이는 4자이지만 값은 10^16이 되어 조건을 우회할 수 있음.

## 3. Exploit Path

1. **Payload Selection:** 문자열 길이가 4자이면서 10^15보다 큰 값을 찾음. 
   - `1e16` (길이 4, 값 10^16)
   - `9e15` (길이 4, 값 9*10^15)
2. **Execution:** URL 파라미터로 페이로드를 전달함.
   - `?temp=1e16`
3. **Flag Acquisition:** 서버가 조건을 만족하는 것으로 판단하여 숨겨진 플래그를 출력함.

## 4. Lesson Learned
- PHP의 유연한 타입 변환(Loose Typing)은 개발자의 의도와 다른 결과를 초래할 수 있음.
- 지수 표기법(`e`, `E`)이나 16진수 표기법(`0x`) 등 다양한 숫자 표현 방식이 필터링 우회에 사용될 수 있음을 명심해야 함.

## 5. Flag
`DH{makchang_makchang_makchang_makchang_makchang}`
