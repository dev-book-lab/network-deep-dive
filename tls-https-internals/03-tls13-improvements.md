# TLS 1.3 — 1-RTT 핸드쉐이크와 Forward Secrecy

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- TLS 1.3이 2-RTT에서 1-RTT로 줄인 핵심 방법은 무엇인가?
- TLS 1.3에서 삭제된 알고리즘과 기능은 무엇이고 왜 삭제됐는가?
- Forward Secrecy가 왜 TLS 1.3에서 필수가 됐는가?
- 0-RTT Early Data는 어떻게 동작하고 왜 위험한가?
- `curl --tlsv1.3 -v`로 확인할 수 있는 TLS 1.3 고유 특징은?
- TLS 1.2 → TLS 1.3 마이그레이션 시 주의할 점은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
TLS 1.3이 성능에 미치는 영향:

RTT=100ms 환경에서 HTTPS 요청 비용:
  TCP: 150ms
  TLS 1.2: +200ms (2 RTT)
  TLS 1.3: +100ms (1 RTT)
  
  총 연결 수립:
  TCP + TLS 1.2: 350ms
  TCP + TLS 1.3: 250ms → 30% 단축!

  HTTPS를 처음 요청하는 사용자에게 직접 체감됨

보안 관점:
  TLS 1.2: 취약 알고리즘 선택 가능 → 설정 실수 = 취약점
  TLS 1.3: 취약 알고리즘 원천 제거 → 설정 단순화

실무 도입 현황 (2026 기준):
  대부분의 브라우저: TLS 1.3 기본 지원
  Let's Encrypt: TLS 1.3 기본 제공
  Spring Boot 내장 Tomcat 10+: TLS 1.3 지원
  Nginx 1.13+: TLS 1.3 지원
  
  도입 시 고려:
  구형 클라이언트 (Java 8 초기 버전, 구형 Android):
  TLS 1.3 미지원 → TLS 1.2 폴백 필요
  → TLS 1.2와 1.3 동시 활성화 권장
```

---

## 😱 흔한 실수

```
Before — TLS 1.3을 모를 때:

실수 1: TLS 1.3에서 ChangeCipherSpec 메시지를 기대
  TLS 1.2: ChangeCipherSpec 메시지가 명시적으로 존재
  TLS 1.3: ChangeCipherSpec 없음 (호환성용 더미만 존재)
  → 방화벽이 ChangeCipherSpec을 기대하고 TLS 1.3 차단
  → 특정 레거시 방화벽/DPI 장비에서 TLS 1.3 연결 실패

실수 2: 0-RTT를 POST 요청에 사용
  0-RTT Early Data = 재전송 공격(Replay Attack) 위험
  GET /api/products → 재전송해도 문제없음 (안전)
  POST /api/payment → 재전송 = 중복 결제 (위험!)
  → 서버가 0-RTT 요청의 멱등성을 확인하거나 거부해야 함

실수 3: TLS 1.3으로 업그레이드 후 성능 저하
  TLS 1.3의 암호화 범위 확장:
  서버 인증서도 암호화됨 (TLS 1.2에서는 평문)
  → 일부 DPI(Deep Packet Inspection) 장비가 처리 못함
  → 기업 방화벽에서 차단 → 타임아웃 → 느려 보임

실수 4: TLS 1.3에서 사용 불가한 Cipher Suite 설정
  TLS 1.3: Cipher Suite가 고정됨 (설정 없이 자동 선택)
  TLS_AES_256_GCM_SHA384, TLS_AES_128_GCM_SHA256, TLS_CHACHA20_POLY1305_SHA256
  → TLS 1.2용 Cipher Suite 설정이 TLS 1.3에 영향 없음
```

---

## ✨ 올바른 접근

```
After — TLS 1.3을 알고 나면:

Spring Boot TLS 1.3 설정:
  server.ssl.protocol=TLS
  server.ssl.enabled-protocols=TLSv1.3,TLSv1.2   # 둘 다 활성화
  
  # TLS 1.3 Cipher Suite는 자동 (설정 불필요)
  # TLS 1.2용 Cipher Suite만 필요:
  server.ssl.ciphers=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,...

Nginx TLS 1.3:
  ssl_protocols TLSv1.2 TLSv1.3;  # 둘 다
  ssl_early_data on;               # 0-RTT 활성화 (주의!)
  proxy_set_header Early-Data $ssl_early_data;  # 앱에 전달
  # 앱에서 Early-Data 헤더 확인 후 비멱등 요청 거부

0-RTT 안전한 사용:
  # Spring에서 0-RTT 요청 감지 및 거부
  @Component
  public class EarlyDataFilter implements Filter {
      @Override
      public void doFilter(ServletRequest req, ...) {
          String earlyData = ((HttpServletRequest) req).getHeader("Early-Data");
          if ("1".equals(earlyData) && isNonIdempotent(request)) {
              response.setStatus(425);  // 425 Too Early
              return;
          }
          chain.doFilter(req, res, chain);
      }
  }

TLS 1.3 활성화 확인:
  curl -v --tlsv1.3 https://your-server.com 2>&1 | grep -E "TLSv1.3|Cipher"
  # TLSv1.3 (OUT), TLS handshake, Client hello (1):
  # SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
```

---

## 🔬 내부 동작 원리

### 1. TLS 1.2 → TLS 1.3: 무엇이 달라졌나

```
TLS 1.3에서 제거된 것:
  ① 취약 알고리즘:
     RSA 키 교환 (Forward Secrecy 없음)
     DH (DHE) 정적 키 교환
     RC4, DES, 3DES, MD5, SHA-1
     CBC 모드 암호 (Padding Oracle 취약)
     EXPORT 등급 암호화
     NULL 암호화
     압축 (CRIME 공격)
  
  ② 핸드쉐이크 단계:
     ChangeCipherSpec (암시적으로 처리)
     ServerKeyExchange (ClientHello에 통합)
     ServerHelloDone (불필요해짐)
     ClientKeyExchange (ClientHello에 통합)
  
  ③ 기능:
     Session ID 재개 (Session Ticket으로 대체, PSK로 통합)
     Renegotiation (재협상, 보안 문제)

TLS 1.3에서 추가된 것:
  ① 1-RTT 핸드쉐이크 (기본)
  ② 0-RTT Early Data (옵션)
  ③ 필수 Forward Secrecy (ECDHE/DHE 필수)
  ④ Encrypted Extensions (더 많은 정보 암호화)
  ⑤ Certificate 암호화 (TLS 1.2에서는 평문!)
  ⑥ PSK (Pre-Shared Key) 세션 재개
  ⑦ HKDF 기반 키 파생 (더 강력)
```

### 2. TLS 1.3 핸드쉐이크 — 1-RTT 달성 방법

```
TLS 1.2: 2 RTT의 이유
  RTT 1: ClientHello → ServerHello + Cert + ServerKeyExchange
  서버 공개키를 받은 후에야 클라이언트가 키 교환 가능
  RTT 2: ClientKeyExchange → Finished

TLS 1.3: 1 RTT로 단축
  핵심: ClientHello에 KeyShare(ECDHE 공개키)를 미리 포함!

Client                                          Server
│                                                    │
│  ① ClientHello                                     │
│    + key_share (X25519 공개키 미리 포함)               │
│    + supported_versions (TLS 1.3)                  │
│    + signature_algorithms                          │
│  ─────────────────────────────────────────────────►│
│                                                    │  [서버 동작]
│                                                    │  ECDHE 개인키 생성
│                                                    │  서버 ECDH 공개키 생성
│                                                    │  Pre-Master Secret 계산
│                                                    │  핸드쉐이크 키 파생
│                                                    │  Certificate 암호화
│                                                    │  CertificateVerify 계산
│                                                    │  Finished 계산
│                                                    │
│  ② ServerHello                                     │
│    + key_share (서버 X25519 공개키)                   │
│  ③ {EncryptedExtensions}     [암호화 시작!]           │
│  ④ {Certificate}             [인증서도 암호화!]        │
│  ⑤ {CertificateVerify}                             │
│  ⑥ {Finished}                                      │
│  ◄───────────────────────────────────────────────  │
│                                                    │
│  [클라이언트 동작]                                     │
│  서버 ECDH 공개키로 Pre-Master Secret 계산              │
│  키 파생 → 서버 인증서 복호화 → 검증                      │
│                                                    │
│  ⑦ {Finished}                                      │
│  ─────────────────────────────────────────────────►│
│                                                    │
│  [이미 ⑥ 이후부터 Application Data 전송 가능!]          │

총 RTT: 1 RTT (ServerHello부터 암호화, Application Data는 ⑦과 함께)

왜 ClientHello에 KeyShare를 미리 보낼 수 있나:
  클라이언트가 서버가 지원하는 곡선(X25519, P-256 등)을 추측
  supported_groups 확장에 선호 목록 포함
  key_share에 가장 가능성 높은 곡선의 공개키 포함
  
  서버가 다른 곡선을 원하면:
  HelloRetryRequest 전송 → 클라이언트가 재시도
  → 이 경우 2 RTT가 됨 (실제론 드묾)
```

### 3. TLS 1.3 키 파생 — HKDF

```
TLS 1.2 키 파생: PRF(Master Secret, label, seed)
TLS 1.3 키 파생: HKDF (HMAC-based Key Derivation Function, RFC 5869)

HKDF 두 단계:
  Extract: HKDF-Extract(salt, IKM) → PRK (Pseudo-Random Key)
  Expand:  HKDF-Expand(PRK, info, L) → OKM (Output Key Material)

TLS 1.3 키 스케줄:
  Early Secret = HKDF-Extract(0, PSK or 0)
  ↓ 0-RTT 키 파생
  Handshake Secret = HKDF-Extract(Early Secret derived, ECDHE)
  ↓ 핸드쉐이크 암호화 키 파생
  Master Secret = HKDF-Extract(Handshake Secret derived, 0)
  ↓ Application Data 암호화 키 파생

핸드쉐이크부터 암호화:
  ServerHello 이후 즉시 암호화 시작!
  TLS 1.2: ChangeCipherSpec 전까지 평문
  TLS 1.3: ServerHello 직후부터 암호화 (EncryptedExtensions부터)
  
  → 인증서, 확장 정보가 암호화됨
  → 중간자가 서버 인증서 내용을 볼 수 없음 (도메인은 SNI로 여전히 노출)
```

### 4. Forward Secrecy 보장

```
Forward Secrecy (전방 비밀성, FS) 또는 Perfect Forward Secrecy (PFS):
  현재 세션 키가 미래에 유출된 장기 개인키로 복호화될 수 없음

TLS 1.2 RSA 키 교환 (Forward Secrecy 없음):
  Pre-Master Secret = RSA-encrypt(서버 공개키, 무작위)
  서버: 개인키로 복호화
  
  공격자가 과거 트래픽을 저장해두고 나중에 서버 개인키 확보 시:
  → Pre-Master Secret 복호화 → Master Secret → 세션 키 → 평문!
  
  "녹음했다가 나중에 복호화" 공격 가능

TLS 1.3 ECDHE (Forward Secrecy 필수):
  클라이언트 ECDHE: 임시 키 쌍 생성 (세션마다 다름)
  서버 ECDHE:       임시 키 쌍 생성 (세션마다 다름)
  
  Pre-Master Secret = ECDH(서버임시공개키, 클라이언트임시개인키)
                    = ECDH(클라이언트임시공개키, 서버임시개인키)
  
  핸드쉐이크 후 임시 개인키 삭제!
  
  공격자가 서버 장기 개인키 확보해도:
  → 임시 ECDHE 개인키는 이미 삭제됨
  → Pre-Master Secret 계산 불가
  → 과거 세션 복호화 불가 → Forward Secrecy!

왜 TLS 1.3에서 Forward Secrecy가 필수인가:
  TLS 1.3: RSA 키 교환 제거
  ECDHE만 허용 → 모든 세션이 Forward Secrecy 보장
  → "안전한 설정인지 확인" 부담 없음
```

### 5. 0-RTT Early Data

```
0-RTT 동작 (세션 재개 시):

첫 연결:
  일반 1-RTT 핸드쉐이크
  서버: PSK (Pre-Shared Key) = Session Ticket 발급
  클라이언트: PSK 저장 (세션 티켓)

재연결 (0-RTT):
Client                                          Server
│                                                    │
│  ClientHello                                       │
│    + pre_shared_key (PSK = 이전 세션 티켓)            │
│    + early_data (0-RTT 지원)                        │
│                                                    │
│  [Early Data — 0-RTT!]                             │
│  GET /api/products HTTP/1.1  ← 핸드쉐이크 전 데이터     │
│  ─────────────────────────────────────────────────►│
│                                                    │
│  ServerHello + Finished...                         │
│  ◄───────────────────────────────────────────────  │
│                                                    │
│  Finished                                          │
│  ─────────────────────────────────────────────────►│

효과: 데이터가 첫 번째 패킷과 함께 전송 → 0 추가 RTT

0-RTT 암호화 키:
  early_traffic_secret = HKDF(Early Secret, "c e traffic", CH_hash)
  → PSK 기반 (이전 연결의 Master Secret에서 파생)
  → ECDHE 포함 안 됨! (아직 ECDHE 교환 전)
  → PSK가 유출되면 Early Data 복호화 가능 (단, 재전송 공격이 더 현실적)

0-RTT Replay Attack 위험:
  공격자가 0-RTT 패킷 캡처 → 여러 서버에 재전송
  서버: 같은 요청을 여러 번 처리
  
  안전한 경우: GET /products (멱등)
  위험한 경우: POST /payment (중복 결제!)
  
  방어 방법:
  ① 서버가 0-RTT 요청 비멱등성 확인 후 거부 (425 Too Early)
  ② Anti-replay 데이터베이스 (전송된 nonce 추적)
     → 규모 확장 어려움, 분산 환경에서 복잡
  ③ 0-RTT 완전 비활성화 (보수적 선택)
  
  Google의 선택:
    search.google.com: 0-RTT 사용 (GET, 멱등)
    google.com/pay: 0-RTT 비활성화
```

---

## 💻 실전 실험

### 실험 1: TLS 1.3 핸드쉐이크 확인

```bash
# TLS 1.3 연결 확인
curl -v --tlsv1.3 https://cloudflare.com 2>&1 | grep -E "TLSv|Cipher|SSL"
# SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384

# openssl로 TLS 1.3 핸드쉐이크 추적
openssl s_client -connect cloudflare.com:443 -tls1_3 -msg 2>&1 | \
  grep -E "Hello|Certificate|Finished|Extension"

# TLS 1.2 vs TLS 1.3 RTT 비교
for version in --tlsv1.2 --tlsv1.3; do
  printf "$version: "
  curl -w "connect=%{time_connect}s appconnect=%{time_appconnect}s\n" \
       $version -o /dev/null -s https://example.com
done
# TLS 1.3이 appconnect 시간이 더 짧음 확인
```

### 실험 2: 0-RTT 동작 확인

```bash
# 세션 재개 확인 (Session Ticket)
openssl s_client -connect example.com:443 -tls1_3 \
  -sess_out /tmp/session.pem 2>&1 | grep -E "Session|Ticket|Resumption"

# PSK로 재연결
openssl s_client -connect example.com:443 -tls1_3 \
  -sess_in /tmp/session.pem 2>&1 | grep -E "Reused\|resumed\|Session"
# "Reused, TLSv1.3" 또는 "Session-ID" 확인

# 0-RTT Early Data (지원하는 서버에서)
echo "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n" | \
  openssl s_client -connect example.com:443 -tls1_3 \
  -sess_in /tmp/session.pem -early_data /dev/stdin 2>&1 | \
  grep -E "Early data|0-RTT"
```

### 실험 3: Forward Secrecy 검증

```bash
# TLS 1.2 RSA 키 교환 (Forward Secrecy 없음)
openssl s_client -connect example.com:443 -tls1_2 \
  -cipher "RSA" 2>&1 | grep -E "Cipher|Forward"

# TLS 1.2 ECDHE (Forward Secrecy 있음)
openssl s_client -connect example.com:443 -tls1_2 \
  -cipher "ECDHE" 2>&1 | grep "Cipher"
# ECDHE-RSA-AES256-GCM-SHA384 → Forward Secrecy

# TLS 1.3 (항상 Forward Secrecy)
openssl s_client -connect example.com:443 -tls1_3 2>&1 | \
  grep "Cipher"
# TLS_AES_256_GCM_SHA384 → 항상 ECDHE 포함

# SSL Labs로 Forward Secrecy 등급 확인
# https://www.ssllabs.com/ssltest/ → "Forward Secrecy" 섹션
```

### 실험 4: TLS 1.3 암호화 범위 확인

```bash
# TLS 1.2: Certificate가 평문 (Wireshark에서 확인)
sudo tcpdump -nn -i any 'host target.com and port 443' -w /tmp/tls12.pcap &
curl --tlsv1.2 https://target.com -o /dev/null
# Wireshark: tls.handshake.type == 11 → Certificate 내용 보임

# TLS 1.3: Certificate가 암호화
sudo tcpdump -nn -i any 'host target.com and port 443' -w /tmp/tls13.pcap &
curl --tlsv1.3 https://target.com -o /dev/null
# Wireshark: ServerHello 이후 모든 것이 Application Data (암호화)
# → Certificate 내용을 SSLKEYLOGFILE 없이는 볼 수 없음
```

---

## 📊 성능/비용 비교

```
TLS 1.2 vs TLS 1.3 상세 비교:

┌─────────────────────────────────────────────────────────────────────┐
│  항목              │  TLS 1.2             │  TLS 1.3                 │
├─────────────────────────────────────────────────────────────────────┤
│  핸드쉐이크 RTT      │  2 RTT               │  1 RTT                   │
│  세션 재개          │  1 RTT               │  0 RTT (옵션)             │
│  Forward Secrecy  │  선택 (ECDHE 필요)     │  필수                     │
│  암호화 범위         │  Finished부터         │  ServerHello 직후         │
│  인증서 암호화        │  없음 (평문)           │  있음 (암호화)             │
│  지원 Cipher 수     │  많음 (취약 포함)       │  5개만 (모두 안전)          │
│  설정 복잡도         │  높음 (Cipher 선택)    │  낮음 (자동)               │
└─────────────────────────────────────────────────────────────────────┘

RTT별 성능 개선:
  RTT=1ms  (LAN):   1ms 단축  (미미)
  RTT=30ms (국내):  30ms 단축 (체감 가능)
  RTT=150ms (해외): 150ms 단축 (큰 차이!)

TLS 1.3 CPU 비용:
  ECDHE: TLS 1.2와 유사
  암호화 범위 증가: 약간의 추가 암호화 비용
  전체적으로 TLS 1.2와 유사하거나 약간 더 빠름

0-RTT 절감 효과:
  세션 재개 + 0-RTT:
  이전: 1 RTT (세션 재개) + 0.5 RTT (데이터) = 1.5 RTT
  0-RTT: 0 RTT + 0 RTT = 0 RTT (데이터 즉시 전송)
  → 고지연 환경에서 큰 효과 (모바일, 해외 서버)
```

---

## ⚖️ 트레이드오프

```
TLS 1.3 도입 트레이드오프:

장점:
  ① 더 빠른 핸드쉐이크 (1 RTT)
  ② 필수 Forward Secrecy
  ③ 더 강력한 기본 보안 (취약 알고리즘 제거)
  ④ 인증서 암호화 (프라이버시 향상)
  ⑤ 단순화된 Cipher Suite 설정

단점:
  ① 레거시 클라이언트 미지원:
     Java 8 초기 버전, 구형 Android, iOS < 12
     → TLS 1.2 폴백 필요
  
  ② DPI 호환성:
     일부 방화벽/IDS가 TLS 1.3 핸드쉐이크 차단
     → 기업 환경에서 문제 발생 가능
  
  ③ 0-RTT 보안 위험:
     Replay Attack 가능 → 신중한 설정 필요
     → GET 등 멱등 요청에만 허용, 또는 완전 비활성화

  ④ 인증서 암호화의 부작용:
     기업 네트워크의 SSL Inspection 어려움
     → 일부 기업 환경에서 차단 후 TLS 1.2로 다운그레이드

0-RTT 활성화 결정:
  절대 비활성화: 금융, 결제, 의료 (재전송 공격 위험)
  조건부 활성화: GET 등 안전한 요청만, 425 Too Early 처리
  자유롭게 활성화: 캐시 히트 예상, 읽기 전용 서비스
```

---

## 📌 핵심 정리

```
TLS 1.3 핵심 요약:

1 RTT 달성 방법:
  ClientHello에 key_share (ECDHE 공개키) 미리 포함
  서버: 첫 응답부터 암호화된 Certificate + Finished 전송
  클라이언트: 서버 응답 받는 즉시 Application Data 가능

제거된 것:
  RSA 키 교환, 정적 DH, RC4, DES/3DES, MD5/SHA-1, CBC 취약 모드
  ChangeCipherSpec (암묵적 처리), 핸드쉐이크 재협상, 압축

추가된 것:
  필수 Forward Secrecy (ECDHE), 인증서 암호화
  0-RTT Early Data (PSK 기반, Replay 위험 있음)
  HKDF 기반 키 파생 (더 강력)
  PSK 세션 재개 (Session ID/Ticket 통합)

Forward Secrecy:
  임시 ECDHE 키 → 세션 후 삭제
  서버 장기 개인키 유출해도 과거 세션 복호화 불가
  TLS 1.3에서 100% 보장 (선택 불가)

0-RTT 사용 지침:
  멱등 GET: 사용 가능
  POST/PUT/DELETE: 서버에서 425 반환 (거부)
  결제/의료: 0-RTT 완전 비활성화

실무 설정:
  protocols: TLSv1.2, TLSv1.3 (둘 다)
  TLS 1.3 Cipher Suite: 자동 (설정 불필요)
  TLS 1.2 Cipher Suite: ECDHE만 허용 (RSA 키 교환 금지)
```

---

## 🤔 생각해볼 문제

**Q1.** TLS 1.3 핸드쉐이크에서 클라이언트가 `key_share` 확장에 포함하는 ECDHE 공개키를 서버가 거부하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

서버가 클라이언트의 key_share에 포함된 그룹(예: X25519)을 지원하지 않는 경우:

**HelloRetryRequest (HRR):**
```
Client: ClientHello + key_share[X25519]
Server: HelloRetryRequest + supported_groups[P-256]
Client: ClientHello(재전송) + key_share[P-256]
Server: ServerHello + key_share[P-256] + Certificate + Finished
Client: Finished
```

→ 2 RTT가 됩니다 (1 RTT 목표를 달성하지 못함).

**이를 최소화하는 방법:**
- 클라이언트가 key_share에 여러 그룹의 공개키를 미리 포함 (다만 크기 증가)
- 또는 이전 연결에서 서버가 선호하는 그룹을 기억해두고 다음 연결에 우선 사용

**실제 동작:**
대부분의 현대 브라우저/서버는 X25519를 모두 지원하므로 HRR는 드뭅니다. 실제로 Chrome은 X25519를 첫 번째 key_share로 보내며, 모던 서버는 거의 항상 이를 수락합니다.

</details>

---

**Q2.** TLS 1.3에서 인증서가 암호화되는데, 이것이 인증서 핀닝(Certificate Pinning)에 어떤 영향을 미치는가?

<details>
<summary>해설 보기</summary>

**인증서 핀닝(Certificate Pinning):**
클라이언트가 서버 인증서(또는 공개키)를 미리 저장하고, TLS 연결 시 받은 인증서와 비교하는 보안 기법.

**TLS 1.3 암호화의 영향:**

인증서가 암호화되더라도 클라이언트는 TLS 핸드쉐이크 완료 후 서버 인증서를 복호화하여 확인합니다. 핀닝 검증은 복호화 후에 이루어지므로 **기능적으로는 영향 없음**.

**달라지는 것:**
- 네트워크 중간자(방화벽, 패킷 캡처)가 핸드쉐이크 패킷에서 인증서를 직접 읽을 수 없음
- 하지만 클라이언트-서버 간 핀닝 검증 자체는 동일하게 동작

**SSL Inspection에 미치는 영향:**
기업 방화벽이 TLS를 검사(SSL Inspection)하려면:
- TLS 1.2: Certificate가 평문 → 어느 사이트에 접속하는지 확인 쉬움
- TLS 1.3: Certificate 암호화 → SSL Inspection이 더 어려움
→ 일부 기업 보안 장비가 TLS 1.3을 차단하고 TLS 1.2로 다운그레이드하는 이유

**모바일 앱 핀닝:**
```kotlin
// Android (OkHttp)
CertificatePinner pinner = new CertificatePinner.Builder()
    .add("api.example.com", "sha256/...")  // 공개키 핀
    .build();
// → TLS 1.3에서도 동일하게 동작
```

</details>

---

**Q3.** TLS 1.3을 도입했는데 특정 API 클라이언트에서만 연결이 실패한다. 어떻게 진단하는가?

<details>
<summary>해설 보기</summary>

**진단 순서:**

**1. 클라이언트가 TLS 1.3을 지원하는지 확인**
```bash
# Java 버전 확인 (Java 11+: TLS 1.3 지원)
java -version

# Python 버전 (Python 3.7+: TLS 1.3 지원)
python3 -c "import ssl; print(ssl.OPENSSL_VERSION)"
```

**2. 서버 로그에서 TLS Alert 코드 확인**
```bash
# Nginx 에러 로그
grep "SSL_do_handshake\|SSL: error\|handshake" /var/log/nginx/error.log

# 자주 보이는 오류:
# protocol version → TLS 1.3 미지원 클라이언트
# handshake failure → Cipher Suite 불일치
# unknown ca → 인증서 체인 불완전
```

**3. 특정 버전 강제 시도**
```bash
# 클라이언트 측 (curl로 시뮬레이션)
curl -v --tlsv1.2 https://your-server.com  # TLS 1.2로 우회 성공 여부
curl -v --tlsv1.3 https://your-server.com  # TLS 1.3 실패 확인
```

**4. 서버 설정으로 TLS 1.2 폴백 확인**
```nginx
# Nginx: TLS 1.2도 허용
ssl_protocols TLSv1.2 TLSv1.3;  # 둘 다 허용하면 자동 폴백
```

**5. 방화벽/DPI 장비 확인**
일부 구형 방화벽이 TLS 1.3 형식의 패킷을 파싱하지 못하고 차단. tcpdump로 TCP RST 또는 TLS Alert 발생 여부 확인.

**실제로 많이 발생하는 원인:**
- Java 8u251 이전: `jdk.tls.server.protocols` 설정 필요
- .NET Framework 4.5: TLS 1.3 미지원 (4.8 이상 필요)
- 구형 Android(< API 29): TLS 1.3 미지원

</details>

---

<div align="center">

**[⬅️ 이전: TLS 1.2 핸드쉐이크](./02-tls12-handshake.md)** | **[홈으로 🏠](../README.md)** | **[다음: 인증서와 PKI ➡️](./04-certificate-pki.md)**

</div>
