# 인증서와 PKI — X.509 구조와 CA 체인 검증

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- X.509 인증서는 어떤 필드로 구성되고 각각의 의미는?
- Root CA → Intermediate CA → End-Entity 체인을 어떻게 검증하는가?
- Self-Signed 인증서가 실무에서 갖는 한계는?
- OCSP와 CRL은 어떻게 인증서 폐기 여부를 확인하는가?
- OCSP Stapling은 왜 필요하고 어떻게 동작하는가?
- `openssl x509 -text`로 인증서를 분석할 때 무엇을 확인해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"HTTPS 인증서 오류가 났어요":
  NET::ERR_CERT_AUTHORITY_INVALID    → Root CA 신뢰 안 됨
  NET::ERR_CERT_DATE_INVALID         → 인증서 만료
  NET::ERR_CERT_COMMON_NAME_INVALID  → 도메인 불일치
  
  각 오류를 정확히 이해해야 올바른 해결 가능
  → "Self-Signed라서" "Intermediate CA가 없어서" "SAN이 없어서"

인증서 체인 불완전 문제:
  서버 설정 시 중간 CA 인증서 누락
  → 일부 클라이언트(cURL, 구형 Android): 오류
  → Chrome: Root CA 캐시로 자동 해결 (위험 신호!)
  → 진단: openssl s_client -connect host:443 | grep "Certificate chain"

인증서 만료 모니터링 (중요!):
  Let's Encrypt: 90일마다 갱신 필요
  만료 시: 서비스 완전 중단 (HTTPS 오류)
  → 자동 갱신 설정 + 모니터링 알람 필수
  
  curl: 만료 날짜 확인
  openssl s_client -connect host:443 | openssl x509 -noout -dates
```

---

## 😱 흔한 실수

```
Before — PKI를 모를 때:

실수 1: Intermediate CA 인증서 서버에 포함 안 함
  서버 인증서만 설정
  → 브라우저: Root CA까지 체인 구성 시도
  → cURL, API 클라이언트: "unable to get local issuer certificate"
  → Nginx: ssl_certificate에 체인 인증서 포함 필요
    cat domain.crt intermediate.crt > fullchain.pem

실수 2: SAN(Subject Alternative Name) 없는 인증서
  CN(Common Name)만 있는 인증서
  → Chrome 58+: SAN 없으면 인증서 오류
  → Let's Encrypt, 모든 현대 CA: SAN 자동 포함

실수 3: 인증서 만료 모니터링 없음
  갑자기 "SSL certificate expired" 알람
  → 프로덕션 서비스 다운
  → certbot auto-renew + cron 또는 Kubernetes cert-manager

실수 4: Self-Signed 인증서로 프로덕션 사용 시도
  → 모든 클라이언트에서 인증서 오류
  → 브라우저: "Your connection is not private"
  → API 클라이언트: verify=False (위험!) 설정 유혹
```

---

## ✨ 올바른 접근

```
After — PKI를 알고 나면:

올바른 인증서 설정 (Nginx):
  # 전체 체인 파일 (End-Entity + Intermediate CA 순서)
  ssl_certificate /etc/nginx/ssl/fullchain.pem;   # 체인 포함
  ssl_certificate_key /etc/nginx/ssl/privkey.pem; # 개인키

  # fullchain.pem 생성:
  cat domain.crt intermediate.crt > fullchain.pem
  # (Root CA는 포함하지 않음 — 클라이언트에 내장됨)

체인 확인:
  openssl s_client -connect your-domain.com:443 2>/dev/null | \
    grep -E "Certificate chain|s:|i:|verify"

인증서 만료 자동화 (Let's Encrypt + certbot):
  certbot certonly --webroot -d example.com -d www.example.com
  
  # 자동 갱신 (cron)
  0 0,12 * * * root certbot renew --quiet
  
  # 갱신 후 Nginx 재로드 (deploy hook)
  # /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
  #!/bin/bash
  systemctl reload nginx

인증서 만료 모니터링:
  # 만료 날짜 확인
  openssl x509 -noout -dates -in /etc/nginx/ssl/cert.pem
  # notAfter=Mar 28 10:00:00 2026 GMT

  # Prometheus + blackbox_exporter
  # ssl_cert_expiry 메트릭으로 만료 임박 알람
```

---

## 🔬 내부 동작 원리

### 1. X.509 인증서 구조

```
X.509 인증서 필드:

openssl x509 -text -in cert.pem 출력 예시:
─────────────────────────────────────────────────────────────────────
Certificate:
    Data:
        Version: 3 (0x2)                         ← X.509 v3
        Serial Number:                            ← CA가 발급한 고유 번호
            04:00:00:00:00:01:1e:44:a5:e4:05
    Signature Algorithm: sha256WithRSAEncryption  ← 서명 알고리즘
        Issuer: C=US, O=Let's Encrypt, CN=R3     ← 발급한 CA
        Validity
            Not Before: Jan 01 00:00:00 2026 GMT ← 유효 시작
            Not After : Apr 01 00:00:00 2026 GMT ← 유효 종료 (90일)
        Subject: CN=api.example.com               ← 이 인증서의 주체
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey  ← ECC 공개키
            EC Public Key: (256 bit)
                pub:
                    04:68:1f:7b:...              ← 실제 공개키
        X509v3 extensions:                        ← 확장 필드
            X509v3 Subject Alternative Name:      ← SAN (필수!)
                DNS:api.example.com, DNS:www.api.example.com
            X509v3 Key Usage: critical
                Digital Signature                 ← 서명 용도
            X509v3 Extended Key Usage:
                TLS Web Server Authentication     ← 서버 인증 용도
            X509v3 Basic Constraints: critical
                CA:FALSE                          ← CA 아님 (End-Entity)
            Authority Information Access:
                OCSP - URI:http://r3.o.lencr.org  ← 폐기 확인 URL
                CA Issuers - URI:http://r3.i.lencr.org/... ← 상위 CA 인증서
            X509v3 CRL Distribution Points:
                URI:http://r3.c.lencr.org/...    ← 폐기 목록 URL
    Signature Algorithm: sha256WithRSAEncryption
        72:ec:a2:...                              ← CA의 서명
─────────────────────────────────────────────────────────────────────

각 필드의 의미:
  Serial Number:  폐기 시 CRL에서 검색하는 고유 식별자
  Issuer:         이 인증서에 서명한 CA
  Subject:        인증서 소유자 (도메인)
  SAN:            실제로 검증되는 도메인 목록 (CN 대신 사용)
  Basic Constraints CA:FALSE: 이 인증서로 다른 인증서 발급 불가
  OCSP URI:       실시간 폐기 확인 URL
  CA Issuers URI: 상위 CA 인증서 다운로드 URL
```

### 2. CA 체인 — 신뢰의 계층

```
PKI 신뢰 계층:

Level 0 (Root CA):
  자기 서명 (Self-Signed) 인증서
  브라우저/OS에 사전 내장 (약 100~150개)
  오프라인 보관 (개인키 유출 시 인터넷 전체 위험)
  
  예: ISRG Root X1 (Let's Encrypt의 Root)
      DigiCert Global Root CA
      GlobalSign Root CA

Level 1 (Intermediate CA):
  Root CA가 서명한 인증서
  실제 서버 인증서 발급 담당
  Root CA의 개인키 보호를 위한 완충재
  교차 서명: 다른 Root CA가 Intermediate CA에 서명 가능
  
  예: Let's Encrypt R3
      DigiCert SHA2 Secure Server CA

Level 2 (End-Entity / Leaf Certificate):
  실제 서버에 설치되는 인증서
  도메인을 증명 (SAN 필드)
  CA:FALSE (다른 인증서 발급 불가)

체인 검증 과정:
  1. 서버로부터 [end-entity cert, intermediate cert] 수신
  2. end-entity cert의 Issuer = intermediate cert의 Subject 확인
  3. intermediate cert으로 end-entity cert의 서명 검증
  4. intermediate cert의 Issuer = root CA의 Subject 확인
  5. root CA 인증서를 OS/브라우저 신뢰 저장소에서 조회
  6. root CA로 intermediate cert의 서명 검증
  7. 현재 시간이 Validity 범위 내인지 확인
  8. 도메인이 SAN에 포함되는지 확인

교차 서명의 중요성:
  ISRG Root X1 (Let's Encrypt 신규 Root):
    2021년 이전 Android: 신뢰하지 않음
    IdenTrust (오래된 Root)가 ISRG Root X1에 교차 서명
    → 구형 Android에서도 Let's Encrypt 인증서 신뢰
  
  2021년 9월: IdenTrust 교차 서명 만료
  → Android 7.1 이하: Let's Encrypt 인증서 오류
  → 해결: 구형 Android 업데이트 또는 다른 CA 사용
```

### 3. 인증서 유효성 검증 유형

```
DV (Domain Validation):
  도메인 소유권만 확인
  자동화 가능 (ACME 프로토콜 — Let's Encrypt)
  발급 시간: 수 분
  주소창: 자물쇠 표시
  사용: 일반 웹사이트, API 서버

OV (Organization Validation):
  도메인 + 기업 실재 확인
  수동 심사 필요
  발급 시간: 수 일
  주소창: 자물쇠 + 기업명 (일부 브라우저)
  사용: 기업 사이트

EV (Extended Validation):
  도메인 + 기업 심층 심사
  법적 실체 확인 필요
  발급 시간: 수 주
  주소창: 과거에 초록 막대 (현재 Chrome은 구분 없음)
  사용: 금융, 정부

Wildcard 인증서:
  *.example.com → sub1.example.com, sub2.example.com 모두 커버
  sub.sub.example.com은 커버 안 됨 (1단계 서브도메인만)
  SAN 기반 멀티 도메인 인증서와 조합 가능

SAN (Subject Alternative Name):
  api.example.com, www.example.com, example.com
  여러 도메인을 하나의 인증서로 (Subject Alternative Name 목록)
  Let's Encrypt에서 최대 100개 SAN 지원
```

### 4. 인증서 폐기 — OCSP와 CRL

```
인증서 폐기가 필요한 경우:
  개인키 유출 (서버 해킹)
  직원 퇴사 (클라이언트 인증서)
  도메인 이전

CRL (Certificate Revocation List):
  CA가 주기적으로 발행하는 폐기된 Serial Number 목록
  
  문제:
  파일 크기가 클 수 있음 (수 MB)
  최신 상태가 아닐 수 있음 (발행 주기 = 수 시간~수 일)
  클라이언트가 전체 파일 다운로드 필요

OCSP (Online Certificate Status Protocol):
  개별 인증서 상태를 실시간으로 조회
  
  클라이언트 → OCSP 서버: "Serial Number 12345의 상태는?"
  OCSP 서버 → 클라이언트: "Good / Revoked / Unknown"
  
  문제:
  OCSP 서버 응답 지연이 TLS 핸드쉐이크 지연에 추가
  RTT=50ms + OCSP RTT=100ms = 150ms 추가 지연
  OCSP 서버 다운 → Soft-Fail(연결 허용) vs Hard-Fail(거부)
  프라이버시: OCSP 서버가 어느 사이트에 접속하는지 알게 됨

OCSP Stapling:
  해결책: 서버가 OCSP 응답을 미리 받아서 TLS 핸드쉐이크에 포함
  
  서버 동작:
    주기적(수 시간마다) OCSP 서버에 자신의 상태 조회
    CA의 서명이 담긴 OCSP 응답 캐시
  
  클라이언트 연결 시:
    Certificate + OCSP 응답(stapled)을 함께 전달
    클라이언트: OCSP 서버에 별도 연결 없이 폐기 확인
    → 추가 RTT 없음!

  Nginx 설정:
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8;  # OCSP 서버 조회에 사용할 DNS
    ssl_trusted_certificate /etc/nginx/ssl/chain.pem;  # CA 체인
  
  확인:
    openssl s_client -connect your-domain.com:443 -status 2>/dev/null | \
      grep -A 5 "OCSP Response"
    # "Response Status: successful" 확인

OCSP Must-Staple:
  인증서에 "OCSP Stapling 필수" 플래그 포함
  클라이언트: Stapled OCSP 없으면 연결 거부
  → 더 강력한 보안 (OCSP 없이 폐기된 인증서 사용 불가)
  → 하지만 서버 설정 오류 시 서비스 다운 위험
```

---

## 💻 실전 실험

### 실험 1: 인증서 파싱 및 분석

```bash
# 서버 인증서 가져오기
openssl s_client -connect github.com:443 2>/dev/null | \
  openssl x509 -noout -text | head -60

# 핵심 정보만 추출
openssl s_client -connect github.com:443 2>/dev/null | \
  openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# SAN(Subject Alternative Name) 확인
openssl s_client -connect github.com:443 2>/dev/null | \
  openssl x509 -noout -ext subjectAltName
# DNS:github.com, DNS:www.github.com

# 만료일 확인
openssl s_client -connect github.com:443 2>/dev/null | \
  openssl x509 -noout -enddate
# notAfter=...
```

### 실험 2: 체인 검증 상세 확인

```bash
# 인증서 체인 전체 확인
openssl s_client -connect github.com:443 -showcerts 2>/dev/null | \
  grep -E "BEGIN CERTIFICATE|subject=|issuer="

# 체인 검증 과정
openssl s_client -connect github.com:443 2>/dev/null | \
  grep -E "Verify return|Certificate chain"
# Verify return code: 0 (ok) → 체인 검증 성공

# 잘못된 체인 (Intermediate CA 없는 경우):
# Verify return code: 21 (unable to verify the first certificate)

# 체인 파일 직접 검증
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt \
  -untrusted intermediate.pem server.pem
```

### 실험 3: OCSP 상태 확인

```bash
# OCSP Stapling 확인
openssl s_client -connect github.com:443 -status 2>/dev/null | \
  grep -A 10 "OCSP Response"

# 직접 OCSP 조회
# 1. 인증서 저장
openssl s_client -connect github.com:443 -showcerts 2>/dev/null | \
  awk '/BEGIN CERT/{f=1} f{print} /END CERT/{f=0;exit}' > server.pem

# 2. 발급 CA 인증서 저장 (두 번째 인증서)
openssl s_client -connect github.com:443 -showcerts 2>/dev/null | \
  awk '/BEGIN CERT/{f++;if(f==2){print}} f==2' > issuer.pem

# 3. OCSP URI 추출
OCSP_URL=$(openssl x509 -noout -ocsp_uri -in server.pem)
echo "OCSP URL: $OCSP_URL"

# 4. OCSP 쿼리
openssl ocsp -issuer issuer.pem -cert server.pem \
  -url "$OCSP_URL" -resp_text 2>/dev/null | grep -E "Cert Status|This Update"
# Cert Status: good
```

### 실험 4: Self-Signed 인증서 생성 및 한계 확인

```bash
# Self-Signed 인증서 생성
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
  -days 365 -nodes \
  -subj "/CN=my-api.local" \
  -addext "subjectAltName=DNS:my-api.local,IP:127.0.0.1"

# 한계 확인: 브라우저/curl에서 오류
curl https://my-api.local  # SSL certificate problem: self-signed certificate
curl -k https://my-api.local  # -k: 검증 무시 (위험!)

# Python requests에서
# requests.get(url, verify=False)  # 위험!
# requests.get(url, verify='cert.pem')  # 올바른 방법 (CA로 지정)

# 자체 CA로 인증서 발급 (내부 환경):
# 1. Root CA 생성
openssl genrsa -out ca.key 4096
openssl req -new -x509 -key ca.key -out ca.crt -days 3650 -subj "/CN=My Root CA"

# 2. 서버 인증서 서명
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj "/CN=my-api.internal"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 365 -extfile <(echo "subjectAltName=DNS:my-api.internal")

# 3. 클라이언트: ca.crt를 신뢰 저장소에 추가 (CA로 검증)
curl --cacert ca.crt https://my-api.internal  # 오류 없음
```

---

## 📊 성능/비용 비교

```
인증서 옵션별 비교:

┌──────────────────────────────────────────────────────────────────────┐
│  항목            │  DV          │  OV           │  EV                 │
├──────────────────────────────────────────────────────────────────────┤
│  발급 시간        │  수 분        │  수 일         │  수 주               │
│  비용            │  무료(LE)     │  수만원/년      │  수십만원/년           │
│  심사 범위        │  도메인만      │  기업 실재      │  심층 심사             │
│  자동화          │  가능         │  어려움         │  불가                │
│  신뢰 수준        │  낮음         │  중간          │  높음                │
│  일반 용도        │  대부분        │  기업 사이트    │  금융, 정부            │
└──────────────────────────────────────────────────────────────────────┘

OCSP Stapling 효과:
  OCSP 미사용:    OCSP 서버 RTT 추가 없음 (Soft-Fail)
                  단, 폐기 확인 미실시 → 보안 약화
  
  OCSP (Stapling 없음): OCSP 서버 RTT 추가 (~100ms)
  OCSP Stapling:        0 RTT 추가 (서버가 미리 캐시)
  
  대규모 서비스에서 OCSP Stapling 필수:
  사용자 수 × OCSP RTT = 엄청난 지연 누적
```

---

## ⚖️ 트레이드오프

```
인증서 유효 기간:

짧은 유효 기간 (90일 — Let's Encrypt):
  장점: 개인키 유출 시 피해 범위 제한
        폐기 필요성 감소 (자주 교체)
  단점: 자동 갱신 필수 (수동으로 불가능)
        갱신 실패 시 서비스 중단

긴 유효 기간 (1~2년):
  장점: 갱신 관리 부담 낮음
  단점: 폐기 시 기간 동안 취약점 유지
  CA/Browser Forum: 최대 398일로 제한 (2020년)

Wildcard vs SAN 인증서:
  Wildcard (*.example.com):
    장점: 서브도메인 자유롭게 추가
    단점: 개인키 여러 서버 공유 (유출 위험)
          Let's Encrypt: DNS 챌린지만 가능
  
  SAN 인증서 (api.example.com, www.example.com):
    장점: 각 도메인 독립 인증서 가능
    단점: 도메인 추가마다 재발급

Self-Signed vs CA 발급:
  Self-Signed:
    장점: 즉시 생성, 비용 없음
    단점: 모든 클라이언트에서 오류
    사용: 개발 환경, 내부 서비스 (자체 CA 필요)
  
  CA 발급 (Let's Encrypt):
    장점: 모든 클라이언트 신뢰
    단점: 자동 갱신 인프라 필요
    사용: 프로덕션
```

---

## 📌 핵심 정리

```
인증서와 PKI 핵심 요약:

X.509 인증서 핵심 필드:
  Subject/Issuer: 소유자와 발급 CA
  Validity:       유효 기간
  SAN:            인증 도메인 목록 (CN 대신 사용)
  Public Key:     서버 공개키
  Signature:      CA의 서명
  Basic Constraints: CA 여부 (End-Entity: CA:FALSE)

CA 체인:
  Root CA → Intermediate CA → End-Entity
  Root: OS/브라우저에 내장
  Intermediate: 서버와 함께 전송 필수
  검증: 서명 체인 + 유효 기간 + SAN 도메인 확인

폐기 확인:
  CRL: 폐기 목록 파일 (크고 오래됨)
  OCSP: 실시간 단일 조회 (RTT 추가)
  OCSP Stapling: 서버가 미리 OCSP 응답 캐시 → 0 RTT 추가

Self-Signed 한계:
  브라우저/클라이언트가 신뢰 안 함
  내부 서비스: 자체 Root CA 생성 후 배포
  프로덕션: Let's Encrypt 또는 상용 CA

실무 필수:
  Intermediate CA 인증서를 fullchain.pem에 포함
  OCSP Stapling 활성화 (ssl_stapling on)
  인증서 만료 모니터링 (30일, 7일 알람)
  자동 갱신 설정 (certbot renew)

진단 명령어:
  openssl x509 -text -in cert.pem     인증서 상세 파싱
  openssl s_client -connect host:443  체인 + OCSP 확인
  openssl verify -CAfile ca.crt cert.pem 체인 검증
```

---

## 🤔 생각해볼 문제

**Q1.** Certificate Transparency(CT)란 무엇이고, 왜 PKI 보안에 중요한가?

<details>
<summary>해설 보기</summary>

**Certificate Transparency (CT, RFC 6962):**

모든 공개 TLS 인증서를 공개 불변 로그(Append-only Log)에 기록하는 시스템입니다.

**왜 필요한가:**

DigiNotar 사건(2011): 해킹당한 CA가 google.com을 포함한 수백 개의 위조 인증서 발급. 수 개월 동안 발견되지 않음.

CT 이전: CA가 몰래 위조 인증서를 발급해도 발견하기 어려움.
CT 이후: 모든 인증서가 공개 로그에 기록됨 → 도메인 소유자가 자신의 도메인에 발급된 인증서를 모니터링 가능.

**동작 방식:**
```
CA → 인증서 발급 → CT Log 서버에 제출 → SCT(Signed Certificate Timestamp) 수신
   → SCT를 인증서에 포함 → 서버가 클라이언트에게 전달
클라이언트 → SCT 검증 → 이 인증서가 CT 로그에 기록됐음 확인
```

**실무 영향:**
- Chrome: TLS 인증서에 SCT 필수 (2018년~). SCT 없으면 "Not Secure" 경고.
- 도메인 소유자: crt.sh (Certificate Transparency 검색)에서 자신의 도메인 인증서 모니터링 가능.

```bash
# crt.sh에서 도메인 인증서 조회
curl "https://crt.sh/?q=%.example.com&output=json" | jq '.[].name_value'
# → 자신 모르는 서브도메인에 발급된 인증서 발견 가능
```

</details>

---

**Q2.** Intermediate CA 없이 Root CA가 직접 서버 인증서에 서명하지 않는 이유는?

<details>
<summary>해설 보기</summary>

**Root CA가 직접 서명하지 않는 이유:**

1. **오프라인 보안:** Root CA 개인키는 오프라인 HSM(Hardware Security Module)에 보관. 직접 서명하려면 Root CA를 온라인으로 가져와야 함 → 개인키 노출 위험.

2. **침해 복구:** Intermediate CA가 해킹되면 해당 Intermediate CA만 폐기하면 됨. 새 Intermediate CA로 재발급.  Root CA가 침해되면 해당 Root CA를 신뢰하는 모든 인증서가 무효 → 수억 개 장치의 신뢰 저장소 업데이트 필요 → 사실상 불가능.

3. **운영 유연성:** 용도별로 다른 Intermediate CA 운영 가능:
   - "Let's Encrypt E1" (ECC 기반)
   - "Let's Encrypt R3" (RSA 기반)
   - 지역별, 서비스별 분리

4. **감사와 통제:** Intermediate CA의 발급 로그를 Root CA와 독립적으로 감사 가능.

**비유:** Root CA = 공증 기관장의 마스터 도장 (금고에 보관), Intermediate CA = 실무 직원의 도장 (매일 사용). 마스터 도장이 훼손되면 전체 공증 체계 붕괴, 직원 도장이 훼손되면 해당 직원만 교체.

</details>

---

**Q3.** 인증서가 만료됐는데 OCSP 응답이 "Good"이라면 클라이언트는 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**검증 순서:**
TLS 핸드쉐이크에서 인증서 검증은 **여러 조건을 동시에** 확인합니다. 모든 조건이 충족되어야 유효합니다.

**만료 인증서 + OCSP "Good"의 경우:**
1. 유효 기간 확인 → `Not After < 현재 시각` → **즉시 실패**
2. OCSP 상태가 "Good"이더라도 이미 만료 오류 발생 → OCSP 확인 의미 없음

```
클라이언트 처리:
  Step 1: Validity.NotAfter 확인
  if (현재시각 > NotAfter) → ERR_CERT_DATE_INVALID → 연결 거부
  (OCSP 확인도 하지 않음)
```

**오류 메시지:**
```
ERR_CERT_DATE_INVALID
"Your connection is not private"
"NET::ERR_CERT_DATE_INVALID"
```

**OCSP "Good"이 무의미한 이유:**
OCSP "Good"은 "이 인증서가 폐기되지 않았음"을 의미하는 것이지 "이 인증서가 유효함"을 의미하지 않습니다. 만료는 폐기와 다릅니다.

**실무에서 만료 직전 조치:**
```bash
# 남은 유효 기간 확인
openssl x509 -noout -checkend 86400 -in cert.pem  # 24시간 내 만료?
echo $?  # 0: 아직 유효, 1: 24시간 내 만료

# 자동 갱신 (certbot)
certbot renew --pre-hook "systemctl stop nginx" \
              --post-hook "systemctl start nginx"
```

</details>

---

<div align="center">

**[⬅️ 이전: TLS 1.3](./03-tls13-improvements.md)** | **[홈으로 🏠](../README.md)** | **[다음: HTTPS 성능 최적화 ➡️](./05-https-performance.md)**

</div>
