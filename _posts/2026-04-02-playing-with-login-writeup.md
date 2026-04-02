# [Dreamhack] playing-with-login Write-up

- **Date:** 2026-04-02
- **Category:** Web Hacking
- **Tags:** #Authentication_Bypass #Unicode_Collation #HPP #Database_Truncation

## 1. Problem Description
"Building and upgrading my own web service..."
기존의 `v1` 서비스를 `v2`로 업그레이드하는 과정에서 발생하는 로직 및 데이터 저장소 간의 불일치를 공략하는 문제입니다.

## 2. Vulnerability Analysis

### A. Dual Storage Inconsistency
- **v1 (In-memory Dictionary):** Python의 딕셔너리를 사용하여 사용자 정보를 관리합니다. 문자열을 바이너리 수준에서 엄격하게 비교하므로 `'admin' != 'admın'` 입니다.
- **v2 (MariaDB):** 관계형 데이터베이스를 사용하며, `username` 필드의 Collation 규칙(예: `utf8mb4_unicode_ci`)에 따라 유니코드 유사 문자를 동일하게 취급할 수 있습니다. 특히 `ı` (Dotless i)가 `i`로 매칭되는 특성이 있습니다.

### B. Logic Bleeding (v2 wraps v1)
- `/v2/request-password-change` 로직을 보면, DB에서 `admin` 계정을 찾아 토큰을 생성하지만, 알림 배달은 요청 파라미터로 받은 `username` (변조된 이름)의 인박스로 보냅니다.
- `inbox` 시스템은 `v1`과 `v2`가 공유하는 전역 변수이므로, `v1`에서 만든 변조된 계정의 인박스로 `admin`의 링크를 가로챌 수 있습니다.

### C. Fake Error (Honey-pot Error Page)
- `v2` 기능들은 로직이 성공적으로 수행된 뒤에도 `abort(501)`을 명시적으로 호출하여 사용자에게 "구현되지 않음" 에러를 보여줌으로써 공격자를 기만합니다.

## 3. Exploit Path

1. **Account Creation (v1):** `/v1/signup`에서 `admın` (Dotless i) 계정을 생성합니다. Python 딕셔너리에는 `admin`과 별개의 키로 저장됩니다.
2. **Link Hijacking (v2):** `/v2/request-password-change`에 `username=admın`으로 POST 요청을 보냅니다.
   - DB는 `admın`을 `admin`으로 인식하여 비밀번호 변경 토큰을 생성합니다.
   - 서버는 생성된 링크를 `inbox['admın']`으로 배달합니다.
3. **Password Update:** `admın` 인박스에서 가로챈 `/v2/change-password/<token>` 링크에 POST 요청을 보내 `admin`의 비밀번호를 변경합니다. (501 에러는 무시)
4. **Flag Acquisition:** 변경된 비밀번호로 `admin` 로그인을 수행한 뒤, `/v1/mypage`에 접속하여 초기화 시 저장된 플래그를 획득합니다.

## 4. Lesson Learned
- 서로 다른 데이터 처리 엔진(Dictionary vs DB)이 공존할 때 발생하는 **Semantic Gap**은 치명적인 인증 우회 취약점으로 이어질 수 있습니다.
- 사용자 입력값에 대한 정규화(Normalization)는 반드시 모든 계층에서 일관되게 적용되어야 합니다.

## 5. Flag
`DH{C011473_0N_UN4M3:DSmONcCKnLEtrSUuZZ/PUg==}`
