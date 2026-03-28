# TLS 1.2 핸드쉐이크 — ClientHello부터 Finished까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- TLS 1.2 핸드쉐이크의 각 단계에서 무슨 정보가 교환되는가?
- Cipher Suite는 어떻게 구성되고, 각 부분의 의미는?
- ClientHello의 지원 목록과 ServerHello의 선택이 어떻게 협상되는가?
- Pre-Master Secret, Master Secret, Session Key는 각각 무엇인가?
- ChangeCipherSpec과 Finished 메시지의 목적은?
- `openssl s_client -tls1_2 -msg`로 핸드쉐이크를 어떻게 추적하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
TLS 1.2 핸드쉐이크를 알아야 하는 이유:
  현재도 많은 서버/클라이언트가 TLS 1.2 사용
  TLS 1.3을 이해하려면 1.2의 문제점을 먼저 알아야 함
  장애 진단:
    "HTTPS 연결이 안 돼요" → Cipher Suite 불일치?
    "일부 클라이언트만 SSL 오류" → 버전/Cipher Suite 지원 차이

  Cipher Suite 협상 실패 시나리오:
    클라이언트: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 지원
    서버: TLS_RSA_WITH_3DES_EDE_CBC_SHA 만 지원
    → 공통 Cipher Suite 없음 → 핸드쉐이크 실패 → alert: handshake_failure

  TLS 버전 강제 설정:
    # application.properties (Spring Boot)
    server.ssl.protocol=TLS
    server.ssl.enabled-protocols=TLSv1.2,TLSv1.3
    server.ssl.ciphers=TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,...
    
    잘못 설정하면:
    → 보안 취약 Cipher Suite 허용 → 다운그레이드 공격
    → 너무 제한하면 일부 클라이언트 접속 불가
```

---

## 😱 흔한 실수

```
Before — TLS 핸드쉐이크를 모를 때:

실수 1: 약한 Cipher Suite 허용
  SSL_RSA_WITH_RC4_128_MD5   ← RC4 취약 (2015년 금지)
  SSL_RSA_WITH_3DES_EDE_CBC  ← SWEET32 공격 취약
  모든 Cipher Suite 허용 → 다운그레이드 공격에 취약

실수 2: TLS 버전 제한 안 함
  SSLv3 허용 → POODLE 공격 (2014)
  TLS 1.0 허용 → BEAST 공격
  현재 권장: TLS 1.2 최소, TLS 1.3 선호

실수 3: 인증서 체인 불완전 전송
  Intermediate CA 인증서를 서버에 포함하지 않음
  → 일부 클라이언트: 인증서 체인 검증 실패
  → "SSL certificate problem: unable to get local issuer certificate"

실수 4: 핸드쉐이크 타임아웃 설정 미스
  TLS 핸드쉐이크 = 2 RTT (TLS 1.2)
  RTT=100ms → 핸드쉐이크 = 200ms
  connectTimeout = 150ms → 핸드쉐이크 중 타임아웃!
  → connectTimeout > 2 RTT로 설정 필요
```

---

## ✨ 올바른 접근

```
After — TLS 1.2 핸드쉐이크를 알면:

권장 Cipher Suite 설정 (TLS 1.2):
  TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 ← 최선 (ECC 인증서)
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384   ← 최선 (RSA 인증서)
  TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
  TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
  
  금지: RSA 키 교환 (DHE/ECDHE 없음), RC4, 3DES, NULL, EXPORT

Nginx TLS 1.2 설정:
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:...;
  ssl_prefer_server_ciphers on;
  ssl_ecdh_curve X25519:secp384r1;  # ECDH 곡선 선택

핸드쉐이크 타임아웃:
  connectTimeout > 2 RTT (TLS 1.2 기준)
  appconnectTimeout > 2.5 RTT (TCP+TLS 합산)
  # curl 측정
  curl -w "%{time_connect} %{time_appconnect}\n" https://example.com -o /dev/null -s
```

---

## 🔬 내부 동작 원리

### 1. TLS 1.2 전체 핸드쉐이크 흐름

```
Client                                          Server
│                                                    │
│  ① ClientHello                                     │
│  ─────────────────────────────────────────────────►│
│                                                    │
│  ② ServerHello                                     │
│  ◄───────────────────────────────────────────────  │
│  ③ Certificate (서버 인증서 체인)                     │
│  ◄───────────────────────────────────────────────  │
│  ④ ServerKeyExchange (ECDHE 경우)                   │
│  ◄───────────────────────────────────────────────  │
│  ⑤ ServerHelloDone                                 │
│  ◄───────────────────────────────────────────────  │
│                                                    │
│  ⑥ ClientKeyExchange                               │
│  ─────────────────────────────────────────────────►│
│  ⑦ ChangeCipherSpec                               │
│  ─────────────────────────────────────────────────►│
│  ⑧ Finished (암호화됨)                               │
│  ─────────────────────────────────────────────────►│
│                                                    │
│  ⑨ ChangeCipherSpec                                │
│  ◄───────────────────────────────────────────────  │
│  ⑩ Finished (암호화됨)                               │
│  ◄───────────────────────────────────────────────  │
│                                                    │
│  [이제 암호화된 데이터 전송 가능]                         │
│  ─────────────────────────────────────────────────►│

RTT 비용:
  ① ClientHello 전송 ~ ⑤ ServerHelloDone 수신: 1 RTT
  ⑥ ClientKeyExchange 전송 ~ ⑩ Finished 수신: 1 RTT
  총: 2 RTT → TLS 1.2의 가장 큰 단점
```

### 2. 각 메시지 상세

```
① ClientHello:
  TLS Version: 1.2 (0x0303)
  Client Random: 32 bytes 무작위 (시간 + 난수)
  Session ID: 이전 세션 재개 시 사용 (없으면 빈 값)
  Cipher Suites: 클라이언트 지원 목록
    [TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
     TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
     TLS_RSA_WITH_AES_256_CBC_SHA256, ...]
  Compression: [null] (압축은 CRIME 취약점으로 비활성)
  Extensions:
    server_name: "api.example.com" (SNI — 어느 사이트?)
    signature_algorithms: [ecdsa_secp256r1_sha256, rsa_pss_rsae_sha256, ...]
    supported_groups: [x25519, secp256r1, secp384r1]
    session_ticket: (세션 티켓 지원 광고)

② ServerHello:
  TLS Version: 1.2
  Server Random: 32 bytes 무작위 (독립적)
  Session ID: 새 세션 ID (재개 시 클라이언트 Session ID 반환)
  Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (하나 선택)
  Compression: null

Cipher Suite 해석:
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  ─── ───── ─── ─── ─────── ─── ───────
  |   |     |   |     |      |   └─ PRF/MAC 해시 알고리즘
  |   |     |   |     |      └──── GCM (운용 모드)
  |   |     |   |     └──────────── AES-256 (대칭 암호)
  |   |     |   └──────────────── WITH (구분자)
  |   |     └──────────────────── RSA (서버 인증서 타입)
  |   └────────────────────────── ECDHE (키 교환)
  └────────────────────────────── TLS

③ Certificate:
  서버 인증서 체인 전송 (DER 인코딩)
  [End-Entity 인증서, Intermediate CA 인증서, (Root CA 선택)]
  클라이언트: 체인 검증 + 서버 이름 일치 확인

④ ServerKeyExchange (ECDHE 경우):
  ECDH 서버 공개키 (ephemeral) 전송
  서버 개인키로 서명 (클라이언트가 검증)
  내용: 선택한 EC 곡선 + 서버 ECDH 공개키 + 서명

⑤ ServerHelloDone:
  서버 메시지 완료 신호 (빈 메시지)

⑥ ClientKeyExchange:
  ECDHE의 경우: 클라이언트 ECDH 공개키 전송
  → 서버와 클라이언트 각자 Pre-Master Secret 계산 가능
  
  Pre-Master Secret = ECDH(서버 공개키, 클라이언트 개인키)
                    = ECDH(클라이언트 공개키, 서버 개인키)
                    (DH의 수학적 특성)

세션 키 생성:
  Master Secret = PRF(Pre-Master Secret,
                       "master secret",
                       ClientRandom + ServerRandom)
  → 48 bytes
  
  4개의 키 파생:
  key_block = PRF(Master Secret, "key expansion",
                  ServerRandom + ClientRandom)
  
  분리:
    client_write_MAC_key: 클라이언트→서버 MAC 키
    server_write_MAC_key: 서버→클라이언트 MAC 키
    client_write_key:     클라이언트→서버 암호화 키
    server_write_key:     서버→클라이언트 암호화 키
    client_write_IV:      클라이언트→서버 IV
    server_write_IV:      서버→클라이언트 IV

⑦ ChangeCipherSpec:
  "이제부터 협상된 암호를 사용하겠습니다"
  → 이전: 평문 (핸드쉐이크 메시지)
  → 이후: 암호화 (위에서 파생한 키로)

⑧ Finished:
  암호화된 첫 번째 메시지
  내용: verify_data = PRF(Master Secret, "client finished",
                           Hash(모든 이전 핸드쉐이크 메시지))
  → 핸드쉐이크 메시지가 변조되지 않았음을 검증
  → 양쪽이 같은 Master Secret을 가지고 있음을 확인

⑨⑩ 서버의 ChangeCipherSpec + Finished:
  서버도 같은 과정으로 검증
  Finished 메시지에 "server finished" 포함
```

### 3. RSA 키 교환 (TLS 1.2에서 선택 사항)

```
RSA 키 교환 (Forward Secrecy 없음):
  ④ ServerKeyExchange 없음 (RSA 인증서 자체가 공개키)
  
  ⑥ ClientKeyExchange:
    Pre-Master Secret (48 bytes) 클라이언트가 생성
    서버 공개키로 암호화해서 전송
    서버: 개인키로 복호화 → Pre-Master Secret 획득

  문제 — Forward Secrecy 없음:
    서버 개인키 유출 시:
    → 모든 과거 录음된 TLS 세션의 Pre-Master Secret 복호화 가능
    → Master Secret → 세션 키 → 평문 데이터 복호화
    
    NSA가 트래픽을 녹음해두고 나중에 개인키 확보 시:
    → 모든 과거 통신 복호화 가능

ECDHE vs RSA 키 교환 비교:
  RSA:  서버 공개키가 키 교환에 직접 사용 → 개인키 유출 = 모든 세션 위험
  ECDHE: 임시(ephemeral) 키 쌍 생성 → 세션마다 다른 키 → 개인키 유출해도 과거 세션 안전

TLS 1.3: RSA 키 교환 완전 제거, ECDHE 필수화
```

### 4. Record Layer — 실제 데이터 전송

```
TLS Record 구조:
  Content Type (1 byte): Handshake(22), ChangeCipherSpec(20), 
                         Alert(21), Application Data(23)
  Protocol Version (2 bytes): 0x0303 (TLS 1.2)
  Length (2 bytes): 최대 16384 bytes
  Payload: 핸드쉐이크 메시지 또는 암호화된 데이터

암호화된 Application Data Record:
  [Record Header (5 bytes)]
  [Encrypted Data (= AES-GCM(plaintext, write_key, write_IV))]
  [Authentication Tag (16 bytes, GCM)]
  
  AES-GCM:
    Nonce = IV XOR 시퀀스번호 (재전송 공격 방지)
    Auth Tag: 복호화 전 무결성 확인 → 변조 즉시 감지

TLS Alert:
  심각도: Warning(1), Fatal(2)
  내용:
    close_notify: 정상 종료 알림
    handshake_failure: 공통 Cipher Suite 없음
    certificate_unknown: 인증서 검증 실패
    bad_record_mac: 무결성 검증 실패 (공격 또는 오류)
    decrypt_error: 복호화 실패
    protocol_version: 버전 불일치
    access_denied: 클라이언트 인증 실패 (mTLS)
```

---

## 💻 실전 실험

### 실험 1: openssl s_client로 핸드쉐이크 추적

```bash
# TLS 1.2 핸드쉐이크 전체 추적
openssl s_client -connect api.example.com:443 -tls1_2 -msg 2>&1 | head -60

# 출력:
# >>> TLS 1.2 Handshake [length 0120], ClientHello
# <<< TLS 1.2 Handshake [length 004d], ServerHello
# <<< TLS 1.2 Handshake [length 12a4], Certificate
# <<< TLS 1.2 Handshake [length 010d], ServerKeyExchange
# <<< TLS 1.2 Handshake [length 0004], ServerHelloDone
# >>> TLS 1.2 Handshake [length 0046], ClientKeyExchange
# >>> TLS 1.2 ChangeCipherSpec [length 0001]
# >>> TLS 1.2 Handshake [length 0030], Finished
# <<< TLS 1.2 ChangeCipherSpec [length 0001]
# <<< TLS 1.2 Handshake [length 0030], Finished

# Cipher Suite 확인
openssl s_client -connect api.example.com:443 2>&1 | grep -E "Cipher|Protocol|TLS"
# Protocol: TLSv1.2
# Cipher: ECDHE-RSA-AES256-GCM-SHA384
```

### 실험 2: 서버의 지원 Cipher Suite 스캔

```bash
# nmap으로 TLS 지원 Cipher Suite 스캔
nmap --script ssl-enum-ciphers -p 443 api.example.com 2>/dev/null | \
  grep -E "TLSv|cipher|strength"

# testssl.sh로 상세 분석
# docker run --rm -ti drwetter/testssl.sh api.example.com
# → 취약한 Cipher Suite, 프로토콜 버전 감지

# 특정 Cipher Suite 강제 시도
openssl s_client -connect api.example.com:443 \
  -cipher "ECDHE-RSA-AES256-GCM-SHA384" 2>&1 | grep -E "Cipher|error"

# 약한 Cipher Suite가 허용되는지 확인
openssl s_client -connect api.example.com:443 \
  -cipher "DES-CBC3-SHA" 2>&1 | grep -E "Cipher|error|alert"
# alert handshake failure → 차단됨 (좋음)
```

### 실험 3: 핸드쉐이크 타이밍 측정

```bash
# TLS 핸드쉐이크 시간 측정
curl -w "TCP Connect: %{time_connect}s\nTLS Handshake: %{time_appconnect}s\nTotal: %{time_total}s\n" \
     --tlsv1.2 -o /dev/null -s https://api.example.com

# 출력:
# TCP Connect: 0.052s       ← TCP 3-Way Handshake
# TLS Handshake: 0.205s    ← TCP + TLS 1.2 (2 RTT)
# Total: 0.327s

# TLS만의 시간 = TLS Handshake - TCP Connect = 0.153s = 약 2 RTT
# RTT ≈ 0.153 / 2 = 76ms

# TLS 1.3과 비교
curl -w "TLS: %{time_appconnect}s\n" --tlsv1.3 -o /dev/null -s https://api.example.com
# TLS 1.3: 약 1 RTT → 더 빠름
```

### 실험 4: Wireshark로 핸드쉐이크 분석

```bash
# TLS 세션 키 파일 생성 (Wireshark 복호화용)
export SSLKEYLOGFILE=/tmp/tls-keys.txt
curl -s https://api.example.com -o /dev/null

# Wireshark 분석:
# 1. Edit → Preferences → Protocols → TLS
# 2. (Pre)-Master-Secret log filename: /tmp/tls-keys.txt
# 3. Display Filter: tls.handshake
# 4. 각 핸드쉐이크 메시지 클릭 → 상세 필드 확인

# tshark로 Cipher Suite 추출
tshark -r capture.pcap -Y "tls.handshake.type == 1" \
  -T fields -e tls.handshake.ciphersuite 2>/dev/null | head -5
```

---

## 📊 성능/비용 비교

```
TLS 핸드쉐이크 RTT 비용:

환경별 TLS 1.2 핸드쉐이크 비용:
  LAN (RTT=1ms):   2 RTT = 2ms (무시 가능)
  국내 (RTT=30ms): 2 RTT = 60ms (체감 가능)
  해외 (RTT=150ms):2 RTT = 300ms (현저히 느림)

연결 재사용 효과 (Session Resumption):
  최초 연결:    2 RTT (전체 핸드쉐이크)
  Session ID:  1 RTT (서버 메모리에서 재개)
  Session Ticket: 1 RTT (서버 상태 없음)
  → 재연결 시 50% 단축

TLS vs TCP-only 오버헤드:
  TCP Handshake:     1.5 RTT
  TLS 1.2 추가:      2 RTT
  TLS 1.3 추가:      1 RTT (개선!)
  
  실제 비율:
  데이터 요청 10ms, RTT=50ms:
  TCP only: 75ms + 10ms = 85ms
  TCP+TLS1.2: 75ms + 100ms + 10ms = 185ms (2.2배)
  TCP+TLS1.3: 75ms + 50ms + 10ms = 135ms (1.6배)
```

---

## ⚖️ 트레이드오프

```
Cipher Suite 선택:

ECDHE vs RSA 키 교환:
  ECDHE: Forward Secrecy ✅, 약간 느림 (ECDH 연산)
  RSA:   Forward Secrecy ❌, 빠름 (키 생성 없음)
  → 보안 > 성능: ECDHE 필수

AES-128 vs AES-256:
  AES-128-GCM: 빠름, 충분히 안전
  AES-256-GCM: 약간 느림, 양자 컴퓨터 대비 여유
  → 대부분 환경에서 AES-128-GCM으로 충분

SHA-256 vs SHA-384:
  SHA-256: 빠름, PRF로 사용
  SHA-384: AES-256과 같은 보안 수준 맞춤
  → AES-256-GCM과 함께: SHA-384 권장

TLS 1.2 vs TLS 1.3:
  TLS 1.2: 광범위 지원 (레거시 포함), 2 RTT
  TLS 1.3: 최신 보안, 1 RTT, 취약 알고리즘 제거
  → 새 서비스: TLS 1.3 우선 + TLS 1.2 폴백
  → TLS 1.0/1.1: 반드시 비활성화 (PCI-DSS 2018 요구)

기업 환경 고려:
  일부 레거시 클라이언트(구형 Java, .NET)가 최신 Cipher Suite 미지원
  → 지원 범위와 보안 수준의 균형 필요
  → caniuse.com 같은 TLS 지원 확인 도구 활용
```

---

## 📌 핵심 정리

```
TLS 1.2 핸드쉐이크 핵심 요약:

7단계 (RSA 키 교환 시 ServerKeyExchange 없음):
  ① ClientHello: 버전, 난수, Cipher Suite 목록, SNI
  ② ServerHello: Cipher Suite 선택, 서버 난수
  ③ Certificate: 서버 인증서 체인
  ④ ServerKeyExchange: ECDHE 공개키 + 서명 (ECDHE만)
  ⑤ ServerHelloDone: 서버 메시지 완료
  ⑥ ClientKeyExchange: 클라이언트 ECDHE 공개키
  ⑦⑧ ChangeCipherSpec + Finished (클라이언트)
  ⑨⑩ ChangeCipherSpec + Finished (서버)

비용: 2 RTT

키 파생:
  Pre-Master Secret → (PRF) → Master Secret → (PRF) → 4개 세션 키
  클라이언트/서버 각각 독립적으로 계산 (동일 결과)

Cipher Suite 형식:
  TLS_[키교환]_[인증]_WITH_[대칭암호]_[해시]
  권장: ECDHE + ECDSA/RSA + AES-256-GCM + SHA-384

TLS 1.2의 한계:
  2 RTT → TLS 1.3에서 1 RTT로 개선
  RSA 키 교환 허용 → Forward Secrecy 없음
  취약 알고리즘 (RC4, 3DES, NULL) 허용 → 비활성화 필요

진단 명령어:
  openssl s_client -connect host:443 -tls1_2 -msg
  curl -w "%{time_appconnect}" --tlsv1.2
  nmap --script ssl-enum-ciphers -p 443
```

---

## 🤔 생각해볼 문제

**Q1.** TLS 핸드쉐이크에서 `Client Random`과 `Server Random` 두 개의 난수가 모두 필요한 이유는 무엇인가?

<details>
<summary>해설 보기</summary>

**하나의 난수만 사용했을 때의 문제:**

만약 서버 난수만 사용한다면, 공격자가 서버 난수를 제어할 수 있으면 세션 키를 예측할 수 있습니다. 반대로 클라이언트 난수만 사용한다면 클라이언트 제어 가능.

**두 개의 난수 조합의 이점:**

```
Master Secret = PRF(Pre-Master Secret, "master secret",
                    ClientRandom || ServerRandom)
```

1. **세션 고유성 보장**: 같은 서버-클라이언트 쌍이라도 매 세션마다 다른 난수 → 다른 Master Secret → 다른 세션 키

2. **공격 방지**: 
   - 클라이언트 Random을 공격자가 예측하더라도 서버 Random으로 인해 예측 불가
   - 반대도 마찬가지 → 양쪽 모두 예측할 수 없어야 함

3. **재전송 공격 방지**: 이전 세션의 Pre-Master Secret이 같아도 난수가 다르면 다른 세션 키 → 이전 세션 키로 새 세션 공격 불가

4. **Replay Attack 방지**: 시간 정보(Server Random의 첫 4바이트는 현재 시각)로 오래된 핸드쉐이크 재전송 감지 (TLS 1.3에서는 제거됨)

</details>

---

**Q2.** TLS 핸드쉐이크 중 `Finished` 메시지는 왜 핸드쉐이크 끝에 있고, 어떻게 전체 핸드쉐이크의 무결성을 보장하는가?

<details>
<summary>해설 보기</summary>

**Finished 메시지의 역할:**

```
Finished.verify_data = PRF(Master Secret, "client finished",
                            Hash(모든 이전 핸드쉐이크 메시지))
```

**보장하는 것:**

1. **핸드쉐이크 메시지 무결성**: 모든 이전 핸드쉐이크 메시지의 해시를 포함. 중간자가 ClientHello나 ServerHello를 변조했다면 Hash 값이 달라짐 → Finished 불일치 → 즉시 감지

2. **같은 Master Secret 확인**: Finished는 Master Secret으로 서명. 서버와 클라이언트가 다른 Pre-Master Secret을 가지면 다른 Master Secret → Finished 불일치 → 핸드쉐이크 실패

3. **암호화 전환 확인**: ChangeCipherSpec 후 첫 암호화 메시지. 암호화가 올바르게 설정됐는지 확인.

**공격 시나리오:**
```
공격자가 ServerHello의 Cipher Suite를 더 약한 것으로 교체 시도:
→ 클라이언트와 서버의 핸드쉐이크 메시지 Hash가 달라짐
→ 서버의 Finished.verify_data ≠ 클라이언트가 기대하는 값
→ 핸드쉐이크 실패 → 공격 탐지
```

이것이 TLS가 다운그레이드 공격에 강한 이유입니다.

</details>

---

**Q3.** SNI(Server Name Indication)가 없다면 하나의 IP에서 여러 HTTPS 사이트를 호스팅할 때 어떤 문제가 생기는가?

<details>
<summary>해설 보기</summary>

**문제의 근원:**

HTTP에서는 `Host` 헤더로 가상 호스팅이 가능하지만, TLS는 HTTP보다 먼저 협상됩니다. 서버는 TLS 핸드쉐이크(인증서 전송) 시점에 클라이언트가 어느 도메인으로 접속하는지 알 수 없습니다.

**SNI 없는 환경의 문제:**
- IP당 하나의 인증서만 제공 가능
- 여러 도메인 → 여러 IP 필요 (IP 낭비)
- CDN, 공유 호스팅에서 수백만 도메인 = 수백만 IP 필요

**SNI 해결책:**
```
ClientHello Extension:
  server_name: "api.example.com"
```
→ 서버가 TLS 핸드쉐이크 중에 어느 도메인인지 알고 해당 인증서 선택

**SNI의 프라이버시 문제:**
SNI는 평문으로 전송됩니다! 즉, TLS 암호화 이전에 접속 도메인이 노출됩니다.
→ ISP나 중간자가 어느 사이트에 접속하는지 알 수 있음

**해결: ESNI/ECH (Encrypted Client Hello)**
- ClientHello 자체를 암호화
- 서버의 공개키(DNS TXT 레코드에 게시)로 암호화
- TLS 1.3 + HTTP/3 환경에서 사용 가능
- Cloudflare, Firefox 지원 중 (2024년 기준)

</details>

---

<div align="center">

**[⬅️ 이전: 암호화 기초](./01-crypto-basics.md)** | **[홈으로 🏠](../README.md)** | **[다음: TLS 1.3 ➡️](./03-tls13-improvements.md)**

</div>
