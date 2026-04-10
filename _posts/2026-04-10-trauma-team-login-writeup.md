# [Dreamhack] TRAUMA TEAM - LOGIN Write-up

- **Date:** 2026-04-10
- **Category:** Web Hacking
- **Tags:** #Authentication_Bypass #Brute_Force #Case_Normalization #Logic_Vulnerability

## 1. Problem Description
"TRAUMA TEAM INTERNATIONAL MEDICAL PORTAL"
로그인 서비스에서 관리자 계정을 탈취하여 특정 유저(`gloria martinez`)를 보험에 가입시키고 플래그를 획득하는 문제입니다.

## 2. Vulnerability Analysis

### A. Identifier Normalization Bypass
- **Filtering Logic:** `if user_input == "admin":` 구문으로 직접적인 `admin` 접근을 차단합니다.
- **Normalization:** 필터링 이후 `(user_input.upper()).lower()` 처리를 수행합니다.
- **Bypass:** `admIn`, `ADMIN` 등 대소문자가 섞인 입력값은 첫 번째 필터링을 통과하지만, 최종적으로는 `admin` 계정으로 정규화되어 로그인이 가능해집니다.

### B. Weak Administrative Credentials
- **Password Generation:** `ADMIN_PW = str(random.randint(0, 2000)).zfill(5)`
- **Vulnerability:** 관리자 비밀번호가 00000부터 02000 사이의 5자리 숫자로 고정되어 있습니다. 전체 경우의 수가 2,001개에 불과하여 브루트 포스(Brute Force) 공격에 매우 취약합니다.

### C. Administrative Privilege Abuse
- **Endpoint:** `/add_insured`
- **Logic:** `session.get('user') == 'admin'` 권한을 확인한 뒤, 지정된 유저의 `is_insured` 상태를 `True`로 변경합니다.
- **Impact:** 관리자 계정 탈취 후 이 기능을 이용해 목표 대상인 `gloria martinez`를 보험에 강제 등록시킬 수 있습니다.

## 3. Exploit Path

1. **Bypass & Brute Force:** `id=admIn`으로 고정하고 `pw=00000~02000` 범위를 탐색하여 관리자 비밀번호를 획득합니다. (본 문제에서는 `01106`으로 확인됨)
2. **Admin Login:** 획득한 자격 증명으로 로그인하여 관리자 세션을 확보합니다.
3. **Insurance Enrollment:** 관리자 메뉴 혹은 `/add_insured?target=gloria martinez` 직접 호출을 통해 대상 유저를 보험에 등록합니다.
4. **Flag Acquisition:** `gloria martinez` / `P@ssw0rd` 계정으로 로그인하여 대시보드 상단에 활성화된 플래그를 확인합니다.

## 4. Lesson Learned
- 블랙리스트 기반의 필터링은 입력값 정규화 로직이 뒤따를 경우 쉽게 무력화될 수 있습니다. (White-list 권장)
- 관리자 계정과 같은 핵심 자산에 대해 충분한 복잡성을 갖지 않는 무작위 비밀번호 생성 로직을 사용하는 것은 위험합니다.

## 5. Flag
`SP{Bu1_1tS_t0O_l41te_glorya}`
