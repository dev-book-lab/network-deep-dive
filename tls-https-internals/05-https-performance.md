# HTTPS 성능 최적화 — Session Resumption과 OCSP Stapling

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Session ID 방식과 Session Ticket 방식의 차이와 각각의 장단점은?
- TLS 세션 재개 시 실제로 절약되는 RTT는 얼마인가?
- OCSP Stapling이 없을 때 TLS 핸드쉐이크에 어떤 지연이 추가되는가?
- `curl -w "%{time_connect} %{time_appconnect}"`으로 TLS 오버헤드를 어떻게 측정하는가?
- Nginx에서 `ssl_session_tickets`, `ssl_stapling` 설정이 각각 무엇을 하는가?
- TLS False Start는 무엇이고 어떤 위험이 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
TLS 성능이 사용자 경험에 미치는 영향:

실측 시나리오 (RTT=150ms, 해외 사용자):
  최적화 없음:
    TCP Handshake: 225ms (1.5 RTT)
    TLS 1.2 Handshake: 300ms (2 RTT)
    OCSP 조회: 150ms (1 RTT, 별도 서버)
    첫 HTTP 요청: 150ms
    합계: 825ms (첫 바이트까지)
  
  최적화 후:
    TCP: 225ms
    TLS 1.3: 150ms (1 RTT)
    OCSP Stapling: 0ms
    HTTP/2 + 0-RTT: 0ms (재연결 시)
    합계: 375ms → 55% 단축!

"성능 최적화" 순서:
  ① TLS 1.3 전환 (가장 큰 효과, 1 RTT 절약)
  ② OCSP Stapling (1 RTT 절약)
  ③ Session Resumption (재연결 시 1 RTT 절약)
  ④ Connection Pool / Keep-Alive (재연결 자체 감소)
  ⑤ HTTP/2 (멀티플렉싱, 헤더 압축)
```

---

## 😱 흔한 실수

```
Before — TLS 성능을 모를 때:

실수 1: Session Ticket 키를 재시작할 때마다 새로 생성
  기본 설정: Nginx 재시작 시 새 Session Ticket 키 생성
  → 재시작 후 모든 클라이언트 세션 무효
  → 재시작 직후 TLS 전체 핸드쉐이크 폭증
  → Session Ticket 키를 파일로 관리하고 주기적으로 교체 필요

실수 2: 수평 확장 시 Session ID 재개 불가
  Session ID: 서버 메모리에 저장
  서버 A에 연결 후 서버 B로 라우팅 → 세션 없음 → 전체 핸드쉐이크
  → Session Ticket: 서버 상태 없음 → 어느 서버에서도 재개 가능
  단, 모든 서버가 같은 Session Ticket 키를 공유해야 함

실수 3: TLS 오버헤드를 측정하지 않고 최적화
  "TLS가 느리다" → 근거 없는 주장
  → curl -w로 측정 후 실제 병목 확인
  → TCP가 느린지, TLS가 느린지, OCSP가 느린지 구분

실수 4: ssl_session_cache 없이 Session ID 사용
  ssl_session_cache shared:SSL:10m;  ← 없으면 Session ID 재개 불가
  ssl_session_timeout 1d;
  → 설정 없으면 매번 전체 핸드쉐이크
```

---

## ✨ 올바른 접근

```
After — TLS 성능을 알고 나면:

Nginx 완전한 TLS 성능 설정:
  # SSL 기본 설정
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:...;
  ssl_prefer_server_ciphers off;  # TLS 1.3에서는 off 권장
  
  # 세션 재개 (Session ID)
  ssl_session_cache shared:SSL:50m;  # 50MB 공유 캐시
  ssl_session_timeout 1d;            # 1일 유지
  
  # 세션 재개 (Session Ticket)
  ssl_session_tickets on;
  ssl_session_ticket_key /etc/nginx/ssl/ticket.key;  # 공유 키 파일
  
  # OCSP Stapling
  ssl_stapling on;
  ssl_stapling_verify on;
  ssl_trusted_certificate /etc/nginx/ssl/chain.pem;
  resolver 8.8.8.8 8.8.4.4 valid=300s;
  resolver_timeout 5s;
  
  # ECDH 곡선 (성능 최적화)
  ssl_ecdh_curve X25519:secp521r1:secp384r1;

Session Ticket 키 관리 (수평 확장):
  # 키 생성
  openssl rand 80 > /etc/nginx/ssl/ticket.key
  chmod 600 /etc/nginx/ssl/ticket.key
  
  # 모든 서버에 동일한 키 배포 (Ansible, Chef, Puppet)
  # 주기적 교체 (Forward Secrecy 유지):
  # 1. 새 키 생성 → 배포 → Nginx 재로드
  # 2. 구 키: 한동안 유지 (기존 세션 만료까지)

TLS 성능 측정 표준 방법:
  curl -w "DNS: %{time_namelookup}s\nTCP: %{time_connect}s\nTLS: %{time_appconnect}s\nTotal: %{time_total}s\n" \
       -o /dev/null -s https://your-domain.com
  
  TLS 오버헤드 = time_appconnect - time_connect
  TLS 1.2: 약 2 RTT
  TLS 1.3: 약 1 RTT
  세션 재개: 약 1 RTT (TLS 1.2) 또는 0 RTT
```

---

## 🔬 내부 동작 원리

### 1. Session ID 재개

```
Session ID 방식:

최초 TLS 핸드쉐이크 후:
  서버: Master Secret + 세션 정보를 서버 메모리에 저장
  서버: ServerHello에 Session ID 포함 (예: 0xABCD1234)
  클라이언트: Session ID 저장

재연결 시:
Client                                     Server
│                                               │
│  ClientHello                                  │
│    session_id: 0xABCD1234  ← 이전 Session ID   │
│  ────────────────────────────────────────────►│
│                                               │  서버: 메모리에서 0xABCD1234 조회
│                                               │  Master Secret 복원
│                                               │
│  ServerHello                                  │
│    session_id: 0xABCD1234 (동일)               │
│  ChangeCipherSpec                             │
│  Finished                                     │
│  ◄─────────────────────────────────────────── │
│                                               │
│  ChangeCipherSpec                             │
│  Finished                                     │
│  ────────────────────────────────────────────►│
│                                               │
│  [Application Data]                           │

RTT 절약: 1 RTT (총 1 RTT, 기존 2 RTT 대비)

Session ID의 한계:
  서버 메모리에 저장 → 수평 확장 어려움
  서버 재시작 시 모든 세션 소멸
  HAProxy/Nginx로 로드밸런싱 시: Sticky Session 없으면 재개 불가

Nginx Session Cache 설정:
  ssl_session_cache shared:SSL:50m;
  # shared: 여러 Worker 프로세스가 공유
  # SSL: 캐시 이름 (여러 개 가능)
  # 50m: 50MB (약 20만 세션 저장 가능)
  ssl_session_timeout 1d;  # 1일 유지
```

### 2. Session Ticket 재개 — Stateless

```
Session Ticket 방식:

최초 핸드쉐이크 후:
  서버: Session Ticket = Encrypt(ticket_key, 세션 정보)
  서버: TLS Session Ticket Extension으로 클라이언트에게 전송
  클라이언트: Ticket 저장 (서버에는 아무것도 저장 안 함!)

재연결 시:
Client                                     Server
│                                               │
│  ClientHello                                  │
│    session_ticket: [암호화된 Ticket]            │ ← Ticket 제출
│  ────────────────────────────────────────────►│
│                                               │  Decrypt(ticket_key, Ticket)
│                                               │  → 세션 정보 복원
│  ServerHello + ChangeCipherSpec + Finished    │
│  ◄─────────────────────────────────────────── │
│                                               │
│  ChangeCipherSpec + Finished                  │
│  ────────────────────────────────────────────►│

RTT 절약: 1 RTT (Session ID와 동일)

Session Ticket의 장점:
  서버 상태 없음 (Stateless) → 수평 확장 친화적
  어느 서버로 라우팅되어도 재개 가능 (같은 ticket_key 공유 시)

Session Ticket의 보안 고려사항:
  ticket_key가 유출되면 모든 Session Ticket 복호화 가능
  → ticket_key는 주기적 교체 필요 (1일~1주)
  → 교체 시 이전 키도 일정 기간 유지 (기존 Ticket 만료까지)
  
  Forward Secrecy와의 관계:
    ECDHE 세션이라도 Session Ticket이 있으면:
    ticket_key 유출 → 세션 복호화 → Forward Secrecy 깨짐
    → ticket_key 관리가 ECDHE 개인키만큼 중요
  
  TLS 1.3의 PSK:
    Session Ticket 개념을 PSK(Pre-Shared Key)로 통합
    더 강력한 키 파생 (HKDF)
    AEAD로 Ticket 자체를 암호화

Nginx Session Ticket 설정:
  ssl_session_tickets on;  # 기본값
  ssl_session_ticket_key first.key;   # 현재 키 (암호화)
  ssl_session_ticket_key second.key;  # 이전 키 (복호화 전용)
  # → 키 교체 시 서비스 중단 없음
```

### 3. OCSP Stapling 상세

```
OCSP 미사용 시 문제:
  클라이언트가 TLS 핸드쉐이크 후 OCSP 서버에 별도 연결
  → OCSP RTT: 50~200ms (별도 서버, 전 세계에서 다름)
  → 사용자: 페이지 로딩이 느려보임

OCSP Soft-Fail:
  일부 클라이언트: OCSP 서버 접속 불가 시 연결 허용 (Soft-Fail)
  → 보안 약화 (폐기된 인증서도 연결 가능)

OCSP Stapling 동작 흐름:

[백그라운드 (주기적으로 서버가 실행)]
서버 → OCSP 서버: "내 인증서 상태는?"
OCSP 서버 → 서버: 서명된 OCSP 응답 (Good/Revoked) + 유효 기간
서버: 응답을 메모리에 캐시 (보통 1시간 유효)

[클라이언트 연결 시]
서버 → 클라이언트: Certificate + OCSP Response (Stapled)
클라이언트:
  OCSP 응답의 CA 서명 검증 (신뢰할 수 있는 응답인지)
  응답이 "Good" 확인
  OCSP 서버에 별도 연결 불필요!

OCSP 응답 구조:
  responseStatus: successful
  cert_status: good (또는 revoked, unknown)
  thisUpdate: 2026-03-28T00:00:00Z
  nextUpdate: 2026-03-28T12:00:00Z  ← 이 시각 전까지 유효
  signature: [CA의 서명]

Nginx 동작:
  ssl_stapling on;
  ssl_stapling_verify on;     # OCSP 응답의 CA 서명 검증
  resolver 8.8.8.8;           # OCSP 서버 도메인 조회용 DNS
  ssl_trusted_certificate /etc/nginx/ssl/chain.pem;  # CA 인증서 (응답 검증용)
  
  # Nginx가 백그라운드에서 OCSP 서버에 주기적으로 조회
  # 응답이 만료되기 전에 자동 갱신
```

### 4. TLS False Start

```
TLS False Start (RFC 7918):
  TLS 1.2 핸드쉐이크 중 Finished를 기다리지 않고 Application Data 전송

일반 TLS 1.2:
  Client → ServerHello... Finished (서버)
  Client ← ChangeCipherSpec + Finished
  Client → Application Data (HTTP 요청)

  2 RTT 후에야 HTTP 요청 가능

TLS False Start:
  Client ← ChangeCipherSpec + Finished (서버)
  Client → ChangeCipherSpec + Finished + Application Data (동시!)
  
  1.5 RTT에서 HTTP 요청 가능 (0.5 RTT 절약)

조건:
  Forward Secrecy 필수 (ECDHE 또는 DHE)
  AEAD Cipher Suite 필수 (AES-GCM, ChaCha20)
  → TLS 1.2에서만 의미 있음 (TLS 1.3은 기본 1 RTT)

위험:
  Finished 검증 전에 데이터 전송
  → 이론적으로 Finished 실패 시 이미 전송한 데이터 노출 위험
  → 실제로는 ECDHE 세션이므로 위험 낮음

Chrome, Firefox: TLS False Start 기본 활성화 (조건 충족 시)
Spring RestTemplate: 클라이언트 구현에 따라 다름

TLS 1.3 맥락:
  TLS 1.3은 이미 1 RTT → False Start 불필요
  0-RTT가 더 강력한 최적화
```

### 5. 전체 최적화 측정 방법

```
TLS 성능 측정 체크리스트:

1. 기본 측정:
  curl -w "namelookup=%{time_namelookup}s connect=%{time_connect}s appconnect=%{time_appconnect}s total=%{time_total}s\n" \
       -o /dev/null -s https://your-domain.com

2. TLS 오버헤드:
  TLS_overhead = time_appconnect - time_connect
  TLS 1.2: ≈ 2 × RTT
  TLS 1.3: ≈ 1 × RTT
  세션 재개: ≈ 1 × RTT (TLS 1.2) 또는 0

3. OCSP 오버헤드 확인:
  openssl s_client -connect host:443 -status 2>/dev/null | \
    grep "OCSP Response"
  응답 없으면: OCSP Stapling 미설정 → 클라이언트가 직접 조회

4. 세션 재개 확인:
  # 두 번 연결 후 재개 여부
  openssl s_client -connect host:443 -reconnect 2>/dev/null | \
    grep "Reused"
  # "Reused, TLSv1.2/3" 확인

5. SSL Labs 테스트:
  https://www.ssllabs.com/ssltest/
  → 종합 보안/성능 점수 확인
  → Session Resumption, OCSP Stapling 활성화 여부 확인

6. 부하 테스트 시 TLS 비용:
  wrk -t4 -c100 -d30s --latency https://your-domain.com
  # P50, P95, P99 레이턴시에서 TLS 오버헤드 비중 확인
```

---

## 💻 실전 실험

### 실험 1: TLS 핸드쉐이크 시간 측정

```bash
# TLS 버전별 핸드쉐이크 시간 비교
for version in --tlsv1.2 --tlsv1.3; do
  echo "=== $version ==="
  for i in 1 2 3; do
    curl -w "connect=%{time_connect}s appconnect=%{time_appconnect}s\n" \
         $version -o /dev/null -s https://example.com
  done
  echo ""
done

# 세션 재개 효과 (두 번째 연결이 빠른지 확인)
echo "=== 세션 재개 테스트 ==="
for i in 1 2 3 4 5; do
  curl -w "appconnect=%{time_appconnect}s\n" -o /dev/null -s https://example.com
done
# 첫 연결: 전체 핸드쉐이크
# 이후 연결: 재개 (더 빠름)
```

### 실험 2: OCSP Stapling 확인

```bash
# OCSP Stapling 있는 서버 확인
openssl s_client -connect cloudflare.com:443 -status 2>/dev/null | \
  grep -A 20 "OCSP Response"
# Response Status: successful (0x0)
# Cert Status: good

# OCSP Stapling 없는 서버 확인
openssl s_client -connect old-server.com:443 -status 2>/dev/null | \
  grep "OCSP"
# No OCSP response received → Stapling 미설정

# Nginx에서 Stapling 활성화 후 확인
curl -v https://your-server.com 2>&1 | grep -i "ocsp"
```

### 실험 3: Session Ticket 동작 확인

```bash
# Session Ticket 저장 및 재사용
openssl s_client -connect example.com:443 -tls1_3 \
  -sess_out /tmp/session.pem 2>/dev/null | grep "Session-ID\|TLS session ticket"

# Ticket으로 재연결
openssl s_client -connect example.com:443 -tls1_3 \
  -sess_in /tmp/session.pem 2>/dev/null | grep "Reused\|Session-ID"
# "Reused, TLSv1.3" 표시 → 세션 재개 성공

# Session ID 방식 테스트 (TLS 1.2)
openssl s_client -connect example.com:443 -tls1_2 \
  -reconnect 2>/dev/null | grep "Reused"
```

### 실험 4: 전체 최적화 비교

```bash
# 최적화 전후 비교 스크립트
measure_tls() {
  local host=$1
  echo "=== $host ==="
  curl -w "DNS=%{time_namelookup}s TCP=%{time_connect}s TLS=%{time_appconnect}s Total=%{time_total}s\n" \
       -o /dev/null -s "https://$host"
}

measure_tls "old-server.com"      # 최적화 안 된 서버
measure_tls "optimized-server.com" # OCSP Stapling, TLS 1.3 설정된 서버
measure_tls "cloudflare.com"      # 대규모 최적화 참고

# 병렬 연결 시 TLS 핸드쉐이크 부하
ab -n 1000 -c 100 -s 60 https://your-server.com/
# Requests per second, Connection Times 확인
```

---

## 📊 성능/비용 비교

```
각 최적화 기법의 효과 (RTT=100ms 기준):

┌───────────────────────────────────────────────────────────────────────┐
│  최적화 기법                │  절약 시간   │  구현 난이도   │  비고            │
├───────────────────────────────────────────────────────────────────────┤
│  TLS 1.3 전환             │  100ms     │  낮음        │  가장 효과적       │
│  OCSP Stapling           │  100ms     │  낮음        │  필수 설정        │
│  Session ID 재개          │  100ms     │  낮음        │  단일 서버        │
│  Session Ticket 재개      │  100ms     │  중간        │  분산 서버        │
│  0-RTT (TLS 1.3)         │  100ms     │  중간        │  보안 주의        │
│  HTTP/2 Keep-Alive       │  150ms     │  낮음        │  연결 자체 재사용   │
│  OCSP Must-Staple        │  0ms (보안) │  중간        │  폐기 강제        │
└───────────────────────────────────────────────────────────────────────┘

Session Cache 크기 계산:
  ssl_session_cache shared:SSL:50m
  세션당 약 256 bytes
  50MB → 약 200,000 세션 동시 저장 가능
  
  1000 동시 접속 × 1일 세션 = 86,400 세션/일
  10MB 캐시로 충분

OCSP Stapling 캐시 유효 기간:
  Let's Encrypt: nextUpdate까지 약 12시간
  Nginx: nextUpdate 전에 자동 갱신
  캐시 미스: Nginx가 OCSP 서버에 조회 (백그라운드)
  → 클라이언트는 영향 없음
```

---

## ⚖️ 트레이드오프

```
Session Ticket vs Session ID:

Session ID (서버 메모리):
  장점: 클라이언트가 세션 정보 저장 안 해도 됨
       ticket_key 관리 불필요
  단점: 서버 메모리 사용
       수평 확장 시 Sticky Session 필요 (또는 공유 캐시)

Session Ticket:
  장점: 서버 상태 없음 → 어느 서버로든 재개 가능
       수평 확장 자유로움
  단점: ticket_key 관리 필요 (주기적 교체)
       ticket_key 유출 시 Forward Secrecy 깨짐

TLS 1.3 PSK (권장):
  Session Ticket 개념을 통합
  더 강력한 키 파생 (HKDF)
  0-RTT Early Data 지원

Session 재개 Forward Secrecy:
  세션 재개는 이전 세션의 Master Secret 재사용
  → 이 세션 키가 유출되면 재개된 세션도 위험
  → 세션 유효 기간을 짧게 (1일 이하) 유지
  → TLS 1.3 PSK는 세션마다 새 키 파생으로 개선

OCSP Must-Staple 트레이드오프:
  장점: 폐기된 인증서 강제 거부 (Soft-Fail 방지)
  단점: Nginx OCSP Stapling 실패 시 → 연결 거부
       서버 설정 오류 = 서비스 다운
  → 설정이 안정된 후 점진적 도입 권장
```

---

## 📌 핵심 정리

```
HTTPS 성능 최적화 핵심 요약:

세션 재개 (1 RTT 절약):
  Session ID: 서버 메모리, 단일 서버에 적합
  Session Ticket: 서버 상태 없음, 수평 확장에 적합
  TLS 1.3 PSK: Session Ticket 통합, 더 강력
  재개 시: 전체 핸드쉐이크 2 RTT → 재개 1 RTT

OCSP Stapling (1 RTT 절약):
  서버가 OCSP 응답을 미리 캐시 → 클라이언트 직접 조회 불필요
  Nginx: ssl_stapling on; ssl_stapling_verify on;
  확인: openssl s_client -status -connect host:443

Session Ticket 보안:
  ticket_key 주기적 교체 (Forward Secrecy 유지)
  수평 확장 시 모든 서버에 동일 키 배포
  만료된 키도 일정 기간 복호화용으로 유지

TLS 성능 측정:
  curl -w "%{time_connect} %{time_appconnect}": TCP vs TLS 시간
  TLS 오버헤드 = time_appconnect - time_connect
  SSL Labs 점수: A+ 목표

최적화 우선순위:
  ① TLS 1.3 (가장 큰 효과)
  ② OCSP Stapling (필수)
  ③ Session Ticket (수평 확장)
  ④ HTTP/2 Keep-Alive (연결 재사용)
  ⑤ 0-RTT (선택적, 보안 주의)
```

---

## 🤔 생각해볼 문제

**Q1.** Nginx 서버를 재시작했더니 세션 재개가 되지 않는다. Session ID와 Session Ticket 각각의 경우에 어떤 이유이고 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**Session ID 경우:**
Session ID는 서버 메모리에 저장됩니다. Nginx 재시작 시 메모리가 초기화되므로 모든 Session ID가 사라집니다.

→ 재시작 후 클라이언트가 이전 Session ID를 제출해도 서버에 없음 → 전체 핸드쉐이크

**해결:**
- Worker 프로세스 재로드(`nginx -s reload`)는 Session Cache를 유지 (메모리 공유)
- 완전 재시작 후 세션 손실은 불가피 → Session Ticket으로 전환 권장

---

**Session Ticket 경우:**
Session Ticket은 서버가 ticket_key로 암호화한 토큰입니다. Nginx 재시작 시 기본으로 새 ticket_key를 생성합니다.

→ 클라이언트가 이전 ticket_key로 암호화된 Ticket을 제출 → 서버가 복호화 실패 → 전체 핸드쉐이크

**해결:**
```nginx
# ticket_key를 파일로 관리 (재시작해도 유지)
ssl_session_ticket_key /etc/nginx/ssl/ticket1.key;  # 현재 키
ssl_session_ticket_key /etc/nginx/ssl/ticket2.key;  # 이전 키

# 키 생성 (한 번만)
openssl rand 80 > /etc/nginx/ssl/ticket1.key
chmod 600 /etc/nginx/ssl/ticket1.key
```

재시작 후에도 같은 키 파일 사용 → 기존 Ticket 복호화 가능.

</details>

---

**Q2.** 10대의 서버가 Session Ticket을 사용하는데, 모두 다른 ticket_key를 사용하면 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

**문제 상황:**

```
사용자 → LB → 서버 A (ticket_key_A)
  → Session Ticket 발급 (ticket_key_A로 암호화)

다음 요청:
사용자 → LB → 서버 B (ticket_key_B) ← 다른 서버!
  → Ticket 제출 → ticket_key_B로 복호화 시도 → 실패!
  → 전체 핸드쉐이크 필요
```

**결과:**
- Session Ticket의 수평 확장 이점이 사라짐 (= Session ID와 같은 문제)
- 재개 성공률 = 1/10 (같은 서버로 라우팅될 확률)
- 나머지 9/10 요청은 전체 핸드쉐이크

**해결 방법:**

1. **공유 ticket_key (권장):**
```bash
# 1. 공통 키 생성
openssl rand 80 > /etc/ssl/ticket.key

# 2. 모든 서버에 배포 (Ansible)
ansible all -m copy -a "src=/etc/ssl/ticket.key dest=/etc/ssl/ticket.key mode=0600"

# 3. Nginx 재로드
ansible all -m command -a "nginx -s reload"
```

2. **Sticky Session (차선책):**
LB에서 클라이언트 IP 또는 쿠키 기반으로 항상 같은 서버로 라우팅

3. **키 교체 절차 (주기적으로):**
```
# 무중단 교체:
# 1. 새 키를 ticket1.key, 기존 키를 ticket2.key로
# 2. Nginx: ticket1.key (암호화) + ticket2.key (복호화만)
# 3. 기존 Ticket은 ticket2.key로 복호화 가능
# 4. 충분한 시간 후 ticket2.key 제거
```

</details>

---

**Q3.** OCSP Stapling이 활성화됐는데 Nginx 재시작 직후에는 OCSP 응답이 없다. 이때 클라이언트는 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**재시작 직후 상황:**

Nginx가 재시작되면 OCSP 응답 캐시가 초기화됩니다. OCSP 서버에 첫 조회를 보내기 전까지는 Stapled OCSP 응답이 없습니다 (일반적으로 수 초~수십 초 소요).

**클라이언트의 동작 (브라우저별 차이):**

1. **Soft-Fail (대부분의 브라우저):**
   - Stapled OCSP 없음 → 클라이언트가 직접 OCSP 서버 조회 시도
   - OCSP 서버 조회 성공 → "Good" → 연결 허용 (하지만 지연 추가)
   - OCSP 서버 조회 실패 → Soft-Fail → 연결 허용 (보안 약화)

2. **OCSP Must-Staple (인증서에 플래그 있는 경우):**
   - Stapled OCSP 없으면 → 연결 거부!
   - 재시작 직후 서비스 중단 가능

**대응 방법:**

```nginx
# Nginx 재로드 (재시작 대신) 권장 → 캐시 유지
nginx -s reload  # Worker 프로세스 교체, 캐시는 shared memory에 유지

# 재시작이 불가피한 경우:
# 1. 재시작 전 OCSP 응답 미리 가져오기
# 2. 또는 트래픽을 다른 서버로 잠시 전환

# Nginx 1.19.0+: ssl_stapling_responder로 직접 OCSP 서버 지정
ssl_stapling_responder http://r3.o.lencr.org/;
# → DNS 조회 없이 직접 접속으로 첫 응답 더 빠름
```

**재시작 vs 재로드:**
- `systemctl restart nginx` → 완전 재시작, 캐시 초기화, OCSP 응답 없음 (수 초)
- `nginx -s reload` → Worker 교체, shared memory(Session Cache + OCSP) 유지 ← 권장

</details>

---

<div align="center">

**[⬅️ 이전: 인증서와 PKI](./04-certificate-pki.md)** | **[홈으로 🏠](../README.md)** | **[다음: mTLS ➡️](./06-mtls-zero-trust.md)**

</div>
