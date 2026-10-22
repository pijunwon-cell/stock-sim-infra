# 모의투자 플랫폼 인프라 구축·보안 검증

실시간 주식 모의투자 웹 플랫폼의 인프라를 설계·구축하고, 공격 시나리오로 보안 통제를 검증했다.

| | |
|---|---|
| 기간 | 2026.08.03 ~ 2026.08.25 (3주) |
| 인원 | 5명 (인프라 보안 12기) |
| 내 역할 | 인프라·네트워크 총괄, 웹 서버 구축 |
| 서비스 | 카카오/구글 소셜 로그인, 실시간 시세·호가, 지정가/시장가 주문, 포트폴리오·거래내역 |
| 서버 | 8대 (방화벽/VPN 게이트웨이, 웹, DB, 로그, 외부 API 연동 4대) |

## 1. 목표

- 서비스 단위로 서버 역할 분리
- Any-to-Any 비활성화, 사전 정의된 통신만 허용하는 화이트리스트 방화벽
- WAF, IDS/IPS, SIEM을 연계한 탐지·차단·로그 수집
- 설계한 통제를 공격 시뮬레이션과 방화벽 차단 테스트로 직접 검증

### 서비스 규모 가정

| 항목 | 수치 |
|---|---|
| 총 이용자 | 5,000명 |
| 일 평균 이용자 | 1,000~1,500명 |
| 평균 동시 접속 | 50~100명 |
| 피크 동시 접속 | 200–400명 (개장 직후 09:00–10:00) |
| 일 평균 트래픽 | 15~20GB |
| 피크 / 이벤트 트래픽 | 40~60Mbps / 100Mbps |

## 2. 일정

| 기간 | 내용 |
|---|---|
| 08.03 ~ 08.05 | 시장 동향·오픈 API 규제 조사, R&R·WBS·Ground Rule 작성 |
| 08.06 ~ 08.09 | OPNsense·WireGuard 설계, 네트워크 다이어그램, LEMP·Graylog 관제 설계, DB 스키마·인증 정책 |
| 08.10 ~ 08.16 | OPNsense·WireGuard 구축, Nginx·PHP-FPM·ModSecurity 설정, MariaDB 구축, Filebeat/Packetbeat·Graylog 연동 |
| 08.13 ~ 08.25 | 모의침투 테스트(SQLi, XSS, SSH 무차별 대입 등), 차단/탐지 로그 수집, 발표자료·시연 영상 |

## 3. 서버 구성

| 서버 | 역할 | 사양 | 소프트웨어 |
|---|---|---|---|
| OPNsense | 방화벽, VPN 게이트웨이 | 4코어 / 4GB / 64GB | OPNsense 26.7.1, WireGuard |
| Web | 웹 서비스 | 4코어 / 4GB / 64GB | Ubuntu 26.04, Nginx 1.28.3, PHP-FPM 8.5.4 |
| DB | 데이터베이스 | 4코어 / 6GB / 64GB | Ubuntu 26.04, MariaDB 11.8.6 |
| Log | 중앙 로그 | 6코어 / 8GB / 64GB | Ubuntu 26.04, Graylog 7.1.6, OpenSearch, MongoDB |
| Kakao API | 카카오 로그인 연동 | 4코어 / 4GB / 64GB | Nginx, FastAPI, Uvicorn, PyMySQL |
| Google API | 구글 로그인 연동 | 4코어 / 4GB / 64GB | PHP 8.5.4 내장 서버 |
| Naver API | 뉴스 검색 수집 | 4코어 / 4GB / 64GB | PHP 8.5.4 (CLI) |
| KIS API | 시세·시장데이터 수집 | 4코어 / 4GB / 64GB | Python 3.14 |

## 4. 네트워크 구성

```
Internet -> OPNsense (방화벽, NAT, WireGuard 게이트웨이)
         -> Web Server (DMZ): Nginx, PHP-FPM, ModSecurity
         -> 내부망 (WireGuard Hub & Spoke, 10.10.10.0/24)
              DB / Log / Kakao / Google / Naver / KIS
```

서버가 팀원별 물리 네트워크에 분산되어 있어, 구성도를 물리 구성(Underlay)과 VPN 구성(Overlay)으로 나눠 작성했다.

| 구역 | 용도 | 배치 |
|---|---|---|
| 외부망 | 인터넷 연결, 방화벽 게이트웨이 | OPNsense |
| DMZ | 외부에 공개되는 웹 서비스 | Web Server |
| 내부망 | 데이터 저장, 내부 처리, 로그 수집. 외부 직접 접근 불가 | DB, Log, API 서버 |

외부에서 열린 포트는 80(HTTPS로 리다이렉트), 443, 51820(WireGuard, 허용된 IP만)뿐이다.

## 5. 내가 한 일

**네트워크 아키텍처와 방화벽(NAT/ACL) 설계**
- 구간을 나눠 규칙을 정의했다: 사용자→방화벽, 방화벽→내부망, 내부망 간, 관리망, 로그망, 내부망→외부망
- 서버마다 필요한 통신만 허용하고 나머지는 차단했다 (아래 6장)
- 웹 포트(80, 443)에 IP당 60초 300연결 초과 시 1시간 자동 차단을 적용했다

**OPNsense 방화벽과 WireGuard VPN 구축**
- 팀원별로 분산된 서버를 WireGuard Hub & Spoke로 연결해 하나의 내부망(10.10.10.0/24)으로 구성했다
- 외부 포트 포워딩: 443, 80, 51820 (WireGuard)

**Nginx + PHP-FPM 웹 서버 구축**
- HTTPS 강제, HSTS 등 보안 헤더, `server_tokens off`
- 쿠키 HttpOnly / SameSite=Lax / Secure 적용
- 시세·뉴스 수집 스크립트는 Nginx location 규칙으로 외부에서 직접 실행하지 못하게 차단

**Suricata IDS/IPS 연동**
- OPNsense에 Suricata를 연동하고 SQL Injection·XSS 탐지 룰을 직접 작성

**ModSecurity WAF 적용**
- 웹 서버에 ModSecurity와 OWASP CRS를 적용해 네트워크(IPS)와 애플리케이션(WAF) 두 계층에서 방어

## 6. 방화벽 규칙

### 사용자 → OPNsense

| 출발지 | 포트 | 동작 | 사유 |
|---|---|---|---|
| Any | TCP 80, 443 | 허용 | IP당 60초 300연결 초과 시 1시간 자동 차단 |
| 허용된 IP | UDP 51820 | 허용 | 관리자·내부 서버 간 WireGuard |
| Any | 그 외 | 차단 | |

### OPNsense → 내부망

| 목적지 | 포트 | 동작 |
|---|---|---|
| Web Server | TCP 80 | 허용 (IDS/IPS 검사 통과 트래픽) |
| 그 외 서버 | ALL | 차단 (DB, API 직접 노출 방지) |

### 내부망 간

| 출발지 → 목적지 | 포트 | 사유 |
|---|---|---|
| Web → DB | 3306 | 회원 정보, 잔고, 거래 데이터 조회·갱신 |
| Kakao / Google API → DB | 3306 | 소셜 로그인 계정·세션 생성·조회 |
| KIS API → DB | 3306 | 시세·호가 적재 |
| Naver API → DB | 3306 | 뉴스 적재 |
| Web → Kakao API | 80 | 카카오 로그인 인증 처리 |
| Web → Google API | 8090 | 구글 로그인 인증 처리 |
| Web → KIS API | 8080 | 실시간 시세 스트림 |
| 그 외 | ALL | 차단 |

### 관리망

| 출발지 → 목적지 | 포트 | 동작 |
|---|---|---|
| 관리자 고정 IP → 전체 서버 | TCP 22 | 허용 |
| 관리자 고정 IP → OPNsense 관리 웹 | TCP 8888 (HTTPS) | 허용 |
| VPN 대역 → 전체 서버 | ICMP | 허용 (상태 확인) |
| 그 외 | ALL | 차단 |

### 로그망

| 출발지 → 목적지 | 포트 | 수집 대상 |
|---|---|---|
| 전체 서버 → Graylog | UDP 1514 | 방화벽 차단 기록, OS Syslog |
| 전체 서버 → Graylog | TCP 5044 | Nginx, PHP, MariaDB 로그 (Filebeat) |
| 그 외 | ALL | 차단 |

### 내부망 → 외부망

| 출발지 | 목적지 포트 | 사유 |
|---|---|---|
| Kakao / Google API | 443 | OAuth 토큰 발급 |
| Naver API | 443 | 뉴스 검색 API |
| Web | 80, 443 | OS 보안 패치 |
| KIS API | 29443, 31000 | 실시간 시세 웹소켓, 주문 API |
| DB, Log | 차단 | 데이터 유출 방지 |

### IP 차단 정책

| 대상 | 처리 |
|---|---|
| 허용 목록 외 모든 통신 | 기본 차단 (Any-to-Any 비활성화) |
| 60초 내 300회 이상 연결 시도 | DDoS 의심, 1시간 자동 차단 |
| SSH·관리자 페이지 로그인 반복 실패 | 자동 차단 (sshlockout) |
| Tor 출구 노드 | Suricata가 탐지 시 차단 |
| 알려진 악성 IP·C2 | abuse.ch, Spamhaus DROP 등 위협 정보 기반 차단 |

## 7. 계정·접근 정책 (팀 구성)

| 영역 | 내용 |
|---|---|
| 사용자 인증 | 소셜 로그인(OAuth 2.0) 전용, 비밀번호 수집 없음 |
| 구글 로그인 | state 파라미터와 세션 값 일치 확인 |
| 카카오 로그인 | 1회용 ticket 검증 후 쿠키 즉시 삭제 |
| 세션 | 로그인 시 세션 ID 재발급(고정 공격 방지), 로그아웃 즉시 파기, 15분 미동작 시 자동 로그아웃 |
| 리다이렉트 | 고정 redirect_uri 사용 (오픈 리다이렉트 방지) |
| 요청 위조 | 상태 변경 요청에 세션 기반 CSRF 토큰 검증 |
| 다중 기기 | 로그인 기기·IP 기록, 마이페이지에서 개별 기기 로그아웃 |
| 관리자 계정 | 12자 이상 복잡도, 90일 변경, 최근 5개 재사용 금지 |
| SSH | 지정 고정 IP만, ROOT 직접 로그인 금지, 5회 실패 시 1시간 차단 |
| 백오피스 | 허가된 IP·계정만 접근, 모든 관리자 작업을 DB와 Graylog에 기록 |
| DB 접근 | 서비스별 계정 분리와 최소 권한, VPN 사설 IP에만 바인딩, 비밀번호 16자 이상 |

## 8. 데이터 보호

- 전 구간 HTTPS 강제, HSTS, `server_tokens off`
- MariaDB InnoDB TDE: 사용자 계정, 거래내역, 잔고, 매매일지 테이블 암호화. 키 파일은 별도 관리, 백업 서버에도 같은 설정
- OAuth 식별값은 SHA-256 단방향 해싱(Salt 적용)
- 외부 API 키는 환경변수(.env)로 분리
- 거래내역 CSV: `=`, `+`, `-`, `@`로 시작하는 값 앞에 `'`를 붙여 수식 삽입 방지
- 쿼리는 파라미터 바인딩, 본인 데이터만 조회

## 9. WAF와 IDS/IPS

**WAF: ModSecurity + OWASP CRS** (Nginx 앞단, SecRuleEngine On)

| 항목 | 내용 |
|---|---|
| 탐지 방식 | Anomaly Scoring (점수 누적, 임계치 초과 시 차단) |
| SQL Injection | CRS 942 계열, 403 차단 |
| XSS | CRS 941 계열, 403 차단 |
| 임계 초과 | CRS 949110, 403 차단 |
| 정상 트래픽 | 회귀 테스트로 오탐 확인 |

**IDS/IPS: Suricata** (WAN, 내부망 인터페이스, IPS 모드)

| 룰셋 | 동작 |
|---|---|
| SQLi/XSS 자체 제작 룰 | Drop 7개, Alert 2개 |
| 웹 공격 대응 14종 (emerging-web_server, emerging-sql 등) | Alert (오탐 검증 후 전환 예정) |
| 알려진 악성 IP·C2 (abuse.ch 5종, botcc, compromised, drop, dshield, ciarmy) | Drop |
| Tor | Drop |
| C2 위협 인텔리전스 | Alert |

## 10. 로그·백업·점검

| 항목 | 내용 |
|---|---|
| 수집 | 방화벽(filterlog), Suricata, WAF, Nginx/PHP/MariaDB, API 서버, SSH 로그인 성공·실패, 시스템 오류, 네트워크 프로토콜(Packetbeat) |
| 저장 | Graylog 중앙 서버 (OpenSearch), 보관 29~30일, 만료 로그 자동 정리 |
| DB 백업 | 일 1회 (전체 30일, 중요 데이터 90일 보관), 월 1회 복구 테스트 |
| 보안 로그 | 일 1회 점검, 1년 이상 보관 |
| 애플리케이션 | 주 1회 백업 (4주 보관), 월 1회 복구 테스트 |
| 방화벽·로그 서버 | 주 1회 백업 (12주 보관), 월 1회 복구 테스트 |
| 전체 시스템 | 월 1회 (3개월 보관) |
| 보안 업데이트 | OS·애플리케이션·방화벽 월 1회 이상 |
| 정기 점검 | 일일(서비스·연동·리소스·침해 시도), 주간(차단 IP 검토, 토큰 유효성, 백업 무결성), 월간(패치, 계정 권한, DR 훈련, 모의 공격) |

## 11. 보안 검증

공격 시나리오 7개를 수행했고, 각 시나리오에서 설계한 통제가 동작함을 확인했다.

| 시나리오 | 검증한 통제 |
|---|---|
| SSH Brute Force | 로그인 실패 반복 시 IP 자동 차단 (sshlockout) |
| CSRF (주문 위조) | 세션 기반 CSRF 토큰 검증 |
| 매수·매도 동시 요청 (Race Condition) | 동시 요청 처리 방어 |
| IDOR (타인 데이터 접근) | 본인 데이터만 조회 |
| WAF SQLi / XSS | ModSecurity 탐지, HTTP 403 차단 |
| CSV 수식 인젝션 | 수식이 계산되지 않고 텍스트로 표시 |
| `/stream` 미인증 접근 | 미인증 상태에서 접근 거부 |

**방화벽 룰셋 차단 테스트**: 외부망에서 Web, DB, Log, Kakao, Google, KIS, Naver 서버로 직접 접근을 각각 시도해 Default Deny 동작과 방화벽 차단 로그를 확인했다.

## 12. 한계

- 서버가 팀원별 물리 네트워크에 분산되어 있어 네트워크 구성을 일관되게 설명하는 문서화가 필요했다.
- OAuth 연동 서버 중 운영용 스택(Nginx/FastAPI)과 개발용 스택(PHP 내장 서버)이 섞여 있어, 운영을 가정하면 스택 통일이 필요하다.
- Suricata 웹 공격 룰 14종은 오탐 검증 전이라 Alert로 두었다.

## 13. 사용 기술

OPNsense, WireGuard, Nginx, PHP-FPM, ModSecurity (OWASP CRS), Suricata, Graylog, OpenSearch, MariaDB, Filebeat, Packetbeat, Ubuntu Server

## 문서

- [인프라 구축 보고서](docs/인프라_구축_보고서.pdf)
- [수행 결과 보고서](docs/수행_결과_보고서.pdf)
