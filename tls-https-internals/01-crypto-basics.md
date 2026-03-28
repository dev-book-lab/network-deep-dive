# 암호화 기초 — 대칭키/비대칭키/해시가 각각 어디에 쓰이는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 대칭키 암호화는 왜 빠르고, 어떤 문제가 있는가?
- 비대칭키는 키 교환 문제를 어떻게 해결하는가?
- HTTPS에서 비대칭키와 대칭키가 각각 어떤 역할을 하는가?
- 해시 함수는 암호화와 어떻게 다르고 어디에 쓰이는가?
- MAC(Message Authentication Code)과 디지털 서명의 차이는?
- AES-256이 "안전하다"는 말의 실제 의미는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"TLS 인증서를 발급받았는데 왜 개인키를 별도로 보관해야 하나요?":
  개인키가 노출되면:
    공격자가 서버 인증서를 도용 → 피싱 사이트
    과거 트래픽을 복호화 가능 (Forward Secrecy 없을 때)
  → 개인키는 HSM(Hardware Security Module) 또는 제한된 경로에만 저장

"비밀번호를 DB에 저장할 때 왜 Hash를 써야 하나요?":
  암호화(대칭/비대칭): 복호화 가능 → DB 유출 시 원문 노출
  해시: 단방향, 복호화 불가 → DB 유출해도 원문 알 수 없음
  단, MD5/SHA-1 해시는 Rainbow Table 공격에 취약
  → bcrypt, Argon2, PBKDF2 (Salt + 반복 해시) 필요

Spring Security 비밀번호:
  PasswordEncoder encoder = new BCryptPasswordEncoder();
  String hashed = encoder.encode("password123");
  // "$2a$10$..." → 복호화 불가, 검증만 가능

"JWT 토큰에는 서명이 들어가는데 이게 암호화인가요?":
  JWT: 서명 (디지털 서명 or HMAC) — 무결성 보장
  암호화 아님 → payload는 Base64로 누구나 읽을 수 있음
  → 민감 정보를 JWT payload에 넣으면 안 됨
```

---

## 😱 흔한 실수

```
Before — 암호화 기초를 모를 때:

실수 1: 해시 = 암호화로 오해
  "비밀번호를 SHA-256으로 암호화했어요"
  → 해시는 암호화가 아님 (단방향, 복호화 없음)
  → SHA-256 해시는 무차별 대입 공격에 취약 (빠르기 때문)
  → 비밀번호: bcrypt (slow hash) 사용 필수

실수 2: Base64 = 암호화로 오해
  Base64 인코딩: 바이너리를 텍스트로 변환
  → 인코딩은 누구나 디코딩 가능 (비밀 없음)
  → 절대 보안이 아님

실수 3: 개인키와 공개키 혼동
  "개인키로 데이터를 암호화하면 안전하지 않나요?"
  → 개인키로 암호화 = 서명 (누구나 공개키로 검증 가능)
  → 공개키로 암호화 = 기밀성 (개인키 소유자만 복호화)
  → 목적에 따라 다름

실수 4: TLS = HTTPS = 완전한 보안으로 오해
  TLS: 전송 중 암호화 (in-transit)
  저장 시 암호화: 별도 적용 필요 (at-rest)
  서버 코드 버그: TLS로 방어 불가
  → TLS는 중간자 공격 방어에 특화
```

---

## ✨ 올바른 접근

```
After — 암호화 기초를 알고 나면:

용도별 올바른 알고리즘 선택:

  비밀번호 저장:
    bcrypt(cost=12) 또는 Argon2id
    Spring: BCryptPasswordEncoder, Argon2PasswordEncoder
    절대 금지: MD5, SHA-1, 단순 SHA-256 (빠른 해시)

  데이터 무결성 검증:
    SHA-256, SHA-3 (해시)
    HMAC-SHA256 (키 있는 MAC) → 메시지 변조 방지

  API 서명 (AWS SigV4, JWT):
    HMAC-SHA256 (대칭키 서명) 또는 RSA/ECDSA (비대칭키 서명)
    JWT HS256 = HMAC, RS256 = RSA, ES256 = ECDSA

  데이터 암호화 (저장):
    AES-256-GCM (인증된 암호화 → 무결성도 보장)
    Spring: JCE, Bouncy Castle
    
  키 교환:
    ECDH, ECDHE (TLS에서 사용)
    RSA 키 교환은 TLS 1.3에서 제거됨

  데이터 전송:
    TLS 1.3 (ECDHE + AES-256-GCM + SHA-384)
    → 현재 권장 최선

Spring JCA 예시:
  // AES-256-GCM 암호화
  SecretKey key = KeyGenerator.getInstance("AES")
      .generateKey();  // 256-bit key
  Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
  cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
  byte[] encrypted = cipher.doFinal(plaintext);
```

---

## 🔬 내부 동작 원리

### 1. 대칭키 암호화 — 빠르지만 키 공유 문제

```
대칭키 암호화:
  같은 키로 암호화 + 복호화

AES (Advanced Encryption Standard):
  블록 암호 (128 bits = 16 bytes 단위로 처리)
  키 길이: 128, 192, 256 bits
  
  평문 → [AES 암호화 (키)] → 암호문
  암호문 → [AES 복호화 (같은 키)] → 평문

AES 운용 모드:
  ECB (Electronic Codebook): 각 블록 독립 암호화
    → 같은 평문 → 같은 암호문 → 패턴 노출 (사용 금지!)
  
  CBC (Cipher Block Chaining): 이전 블록과 XOR
    → 같은 평문이라도 이전 블록에 따라 다른 암호문
    → IV(Initialization Vector) 필요
    → 무결성 검증 없음 → Padding Oracle 공격 취약
  
  GCM (Galois/Counter Mode): CTR 모드 + 인증 태그
    → 암호화 + 무결성 동시 보장 (Authenticated Encryption)
    → TLS 1.3에서 기본 사용
    → AES-256-GCM: 현재 표준

대칭키의 문제 — 키 공유:
  Alice와 Bob이 안전하게 통신하려면:
  1. 같은 키를 공유해야 함
  2. 처음 만날 때 어떻게 키를 전달하는가?
  
  물리적 전달: 비현실적 (인터넷 통신)
  암호화해서 전달: 그 암호화 키는 어떻게 전달하는가? (순환)
  → 키 교환 문제 (Key Exchange Problem)

왜 빠른가:
  AES: 하드웨어 가속 지원 (AES-NI 명령어)
  1코어에서 수 GB/s 처리 가능
  → 대용량 데이터 암호화에 적합
```

### 2. 비대칭키 암호화 — 키 교환 문제 해결

```
비대칭키 (공개키 암호화):
  공개키(Public Key): 누구에게나 공개
  개인키(Private Key): 절대 비밀, 소유자만 보유
  
  두 키의 수학적 관계:
    공개키로 암호화 → 개인키로만 복호화 (기밀성)
    개인키로 서명 → 공개키로 검증 (인증/무결성)

RSA (Rivest-Shamir-Adleman):
  큰 수의 소인수분해가 어렵다는 수학적 원리
  
  키 생성:
    p, q = 큰 소수 (각 1024~2048 bits)
    n = p × q (공개)
    e = 공개지수 (보통 65537)
    d = 개인지수 (p, q로 계산, 비밀)
    공개키: (n, e), 개인키: (n, d)
  
  암호화: C = M^e mod n
  복호화: M = C^d mod n
  
  문제:
    느림: 수학 연산 비용 높음
    크기: 최소 2048 bits (RSA-4096이 더 안전)
    → 대용량 데이터 직접 암호화에 부적합

ECDH (Elliptic Curve Diffie-Hellman):
  타원 곡선 위의 이산 로그 문제 기반
  RSA-2048과 같은 보안 수준 = ECC-256 (키 8배 작음)
  → 모바일, IoT, TLS에서 선호

Diffie-Hellman 키 교환 원리:
  (공개 채널에서 대칭 비밀키를 생성)

  공개 파라미터: g, p (모두 알고 있음)
  
  Alice:
    a = 개인값 (비밀)
    A = g^a mod p (공개)
  
  Bob:
    b = 개인값 (비밀)
    B = g^b mod p (공개)
  
  A와 B 교환 (도청자도 볼 수 있음)
  
  Alice 계산: B^a mod p = g^(ab) mod p
  Bob 계산:   A^b mod p = g^(ab) mod p
  
  → 둘 다 g^(ab) mod p를 공유 (대칭키)
  → 도청자: A, B, g, p 알아도 a, b 모르면 g^(ab) 계산 불가
  (이산 로그 문제의 어려움)
  
  ECDHE: 타원 곡선 기반 DH, 더 짧은 키로 같은 보안
```

### 3. 해시 함수 — 단방향 지문

```
해시 함수의 특성:
  임의 길이 입력 → 고정 길이 출력 (SHA-256: 항상 256 bits)
  단방향: 출력으로 입력 복구 불가
  결정적: 같은 입력 → 항상 같은 출력
  충돌 저항: 다른 입력이 같은 출력을 갖기 매우 어려움
  눈사태 효과: 입력 1bit 변경 → 출력 완전히 달라짐

  SHA-256("Hello") = 185f8db32921b...7b7f1fba4d (64 hex chars)
  SHA-256("Hello!")= 334d016f755cd...4c26f8ac14 (완전히 다름!)

주요 알고리즘:
  MD5:    128 bits — 충돌 발견됨 (보안용 사용 금지)
  SHA-1:  160 bits — 충돌 발견됨 (보안용 사용 금지)
  SHA-256: 256 bits — 현재 표준
  SHA-3:  256/512 bits — SHA-2의 대안 (Keccak 기반)
  bcrypt:  느린 해시 — 비밀번호 전용 (의도적으로 느림)
  Argon2:  메모리 집약적 — 현재 비밀번호 표준

해시의 사용처:
  1. 무결성 검증:
     파일 다운로드 후: sha256sum file.tar.gz 비교
     Git 커밋 ID: 파일 트리의 SHA-1 해시
  
  2. 비밀번호 저장:
     DB에 hash(password) 저장
     로그인 시: hash(input) == stored_hash 비교
  
  3. 디지털 서명:
     서명 대상: 파일 전체가 아닌 해시(파일)
     → 비대칭키 연산을 파일 크기와 무관하게 고정
  
  4. HMAC:
     키 있는 해시 → 메시지 인증

Rainbow Table 공격:
  미리 계산된 "값 → 해시" 테이블
  해시 값만 있으면 원래 값 조회 가능
  
  방어: Salt 추가
    hash(password) = 취약
    hash(random_salt + password) = 안전
    salt는 DB에 공개 저장, 공격자는 각 salt마다 새 테이블 필요
    bcrypt는 자동으로 salt 포함
```

### 4. MAC과 디지털 서명

```
HMAC (Hash-based Message Authentication Code):
  키 있는 해시: MAC = HMAC(key, message)
  
  목적:
    무결성: 메시지가 변조되지 않았음
    인증:   이 MAC을 생성할 수 있는 키 소유자가 보냄
  
  특성:
    대칭키 기반 (같은 키로 생성 + 검증)
    빠름 (해시 연산)
    부인 불가 불가 (같은 키 소유자가 여럿이면 누가 만들었는지 모름)
  
  사용처:
    JWT HS256: Header.Payload에 HMAC-SHA256
    API 서명: AWS SigV4
    TLS MAC: 레코드 무결성 (GCM에서는 AEAD로 통합)

디지털 서명:
  비대칭키 기반
  서명: Signature = Sign(개인키, hash(message))
  검증: Verify(공개키, hash(message), Signature)
  
  목적:
    무결성: 서명 검증으로 변조 확인
    인증:   개인키 소유자가 서명했음
    부인 불가: 개인키는 소유자만 가짐 → 서명 사실 부인 불가
  
  알고리즘:
    RSA-PKCS1, RSA-PSS: RSA 기반
    ECDSA: 타원 곡선 기반 (더 짧은 서명, TLS/JWT에서 선호)
    EdDSA (Ed25519): 빠르고 안전, SSH 키에서 많이 사용

HMAC vs 디지털 서명 비교:
┌────────────────────────────────────────────────────────────────────┐
│  항목          │  HMAC                  │  디지털 서명                 │
├────────────────────────────────────────────────────────────────────┤
│  키           │  대칭 (같은 키 공유)       │  비대칭 (개인/공개키)          │
│  속도          │  빠름                   │  느림                      │
│  부인 불가      │  없음 (양쪽 키 동일)       │  있음 (개인키 고유)           │
│  공개 검증      │  불가 (키 필요)           │  가능 (공개키로)             │
│  사용          │  JWT HS256, API 서명    │  TLS 인증서, JWT RS256      │
└────────────────────────────────────────────────────────────────────┘

TLS에서 암호화 기법이 결합되는 방식:
  1. 비대칭키 (ECDHE):    대칭 세션키 안전하게 교환
  2. 디지털 서명 (ECDSA): 서버 인증서 검증
  3. 대칭키 (AES-256-GCM): 실제 데이터 암호화 (빠름)
  4. 해시 (SHA-384):       무결성 + PRF
  
  → 각각의 장점만 조합:
    비대칭키의 키 교환 보안 + 대칭키의 속도
```

### 5. 암호화 강도 이해

```
키 길이와 보안 강도:

  보안 강도 = 전수 대입 공격에 필요한 연산 수 (2^N)

┌───────────────────────────────────────────────────────────────────┐
│  알고리즘        │  키/파라미터     │  보안 강도    │  비고               │
├───────────────────────────────────────────────────────────────────┤
│  DES           │  56 bits      │  2^56       │  56시간에 크래킹      │
│  3DES          │  112 bits     │  2^112      │  레거시, 느림         │
│  AES-128       │  128 bits     │  2^128      │  안전               │
│  AES-256       │  256 bits     │  2^256      │  양자컴퓨터 대비       │
│  RSA-1024      │  1024 bits    │  ~2^80      │  사용 금지           │
│  RSA-2048      │  2048 bits    │  ~2^112     │  최소 권장           │
│  RSA-4096      │  4096 bits    │  ~2^140     │  고보안              │
│  ECC-256       │  256 bits     │  ~2^128     │  RSA-3072 동등      │
│  ECC-384       │  384 bits     │  ~2^192     │  고보안              │
└───────────────────────────────────────────────────────────────────┘

"AES-256이 안전하다"의 의미:
  2^256 연산 = 우주의 원자 수(10^80)보다 많음
  현재 기술로 전수 대입: 우주 나이(138억 년)의 수조 배 필요
  → 실용적으로 뚫을 수 없음

양자 컴퓨터의 위협:
  Grover 알고리즘: 대칭키 강도를 절반으로 감소
    AES-256 → 양자 기준 128 bits 보안 (여전히 안전)
    AES-128 → 양자 기준 64 bits 보안 (취약 가능)
  
  Shor 알고리즘: RSA, ECC 기반 공개키 암호화 무력화
    → 양자 컴퓨터가 실용화되면 RSA, ECDH, ECDSA 취약
  
  Post-Quantum Cryptography (PQC):
    NIST 표준화: CRYSTALS-Kyber (키 교환), CRYSTALS-Dilithium (서명)
    격자 기반 문제 → 양자 컴퓨터로도 어려움
    TLS 1.3에 PQC 알고리즘 통합 진행 중
```

---

## 💻 실전 실험

### 실험 1: openssl로 암호화 직접 해보기

```bash
# AES-256-CBC 암호화/복호화
echo "Hello, Cryptography!" > plain.txt
openssl enc -aes-256-cbc -salt -in plain.txt -out encrypted.bin -k "mypassword"
openssl enc -aes-256-cbc -d -in encrypted.bin -out decrypted.txt -k "mypassword"
diff plain.txt decrypted.txt  # 동일 확인

# AES-256-GCM (인증된 암호화)
openssl enc -aes-256-gcm -salt -in plain.txt -out encrypted_gcm.bin -k "mypassword"
# GCM은 인증 태그도 생성 → 변조 감지 가능

# 키 파일로 암호화 (비밀번호 대신)
openssl rand -hex 32 > aes.key  # 256 bits 랜덤 키
openssl enc -aes-256-cbc -in plain.txt -out encrypted.bin -kfile aes.key
```

### 실험 2: 해시 함수 특성 관찰

```bash
# SHA-256 해시
echo -n "Hello" | sha256sum
echo -n "Hello!" | sha256sum  # 1글자 차이 → 완전히 다른 해시

# 파일 무결성 검증
sha256sum /etc/hosts > hosts.sha256
sha256sum -c hosts.sha256  # OK
echo " " >> /etc/hosts  # 변조
sha256sum -c hosts.sha256  # FAILED

# bcrypt (느린 해시) vs SHA-256 (빠른 해시) 속도 비교
time echo -n "password" | sha256sum  # 즉각 (빠름 → 무차별 대입 위험)
# bcrypt는 의도적으로 느림 (약 100ms) → 초당 10번 시도만 가능
python3 -c "import bcrypt, time; t=time.time(); bcrypt.hashpw(b'password', bcrypt.gensalt(12)); print(f'{time.time()-t:.3f}s')"
```

### 실험 3: 디지털 서명 실습

```bash
# RSA 키 쌍 생성
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

# 파일 서명
echo "중요한 계약 내용" > contract.txt
openssl dgst -sha256 -sign private.pem -out signature.bin contract.txt

# 서명 검증
openssl dgst -sha256 -verify public.pem -signature signature.bin contract.txt
# Verified OK

# 파일 변조 후 검증
echo " 추가" >> contract.txt
openssl dgst -sha256 -verify public.pem -signature signature.bin contract.txt
# Verification Failure (변조 감지!)

# 키 크기 비교 (RSA-2048 vs ECC-256)
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out rsa2048.pem 2>/dev/null
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out ec256.pem 2>/dev/null
wc -c rsa2048.pem ec256.pem
# RSA-2048: ~1700 bytes, ECC-256: ~200 bytes (8배 차이)
```

### 실험 4: HMAC vs 디지털 서명 비교

```bash
# HMAC-SHA256 생성 (대칭키)
KEY="my-secret-key"
MESSAGE="user_id=123&action=purchase"
HMAC=$(echo -n "$MESSAGE" | openssl mac -digest SHA256 -macopt key:"$KEY" HMAC)
echo "HMAC: $HMAC"

# 검증 (같은 키로)
HMAC2=$(echo -n "$MESSAGE" | openssl mac -digest SHA256 -macopt key:"$KEY" HMAC)
[ "$HMAC" = "$HMAC2" ] && echo "검증 성공" || echo "변조 감지"

# 속도 비교: HMAC vs RSA 서명
time for i in $(seq 1 1000); do
  echo -n "data" | openssl mac -digest SHA256 -macopt key:"$KEY" HMAC
done 2>&1 | tail -1
# HMAC: 빠름

time for i in $(seq 1 100); do
  echo -n "data" | openssl dgst -sha256 -sign private.pem
done 2>&1 | tail -1
# RSA 서명: 더 느림 (하지만 부인 불가)
```

---

## 📊 성능/비용 비교

```
암호화 알고리즘별 성능 비교 (단일 코어 기준):

┌────────────────────────────────────────────────────────────────────┐
│  알고리즘           │  처리량            │  용도                        │
├────────────────────────────────────────────────────────────────────┤
│  AES-128-GCM      │  4 GB/s (AES-NI) │  데이터 암호화                 │
│  AES-256-GCM      │  3.5 GB/s        │  데이터 암호화 (더 안전)         │
│  ChaCha20-Poly1305│  1.5 GB/s        │  AES-NI 없는 기기             │
│  HMAC-SHA256      │  1 GB/s          │  메시지 인증                   │
│  SHA-256          │  700 MB/s        │  해시                        │
│  RSA-2048 서명     │  ~2000회/s        │  인증서, JWT                 │
│  RSA-2048 검증     │  ~80000회/s       │  검증 (서명보다 빠름)           │
│  ECDSA-256 서명    │  ~15000회/s       │  TLS, JWT                  │
│  ECDSA-256 검증    │  ~7000회/s        │  TLS                       │
│  ECDH-256         │  ~15000회/s       │  키 교환                     │
└────────────────────────────────────────────────────────────────────┘

TLS 핸드쉐이크 CPU 비용 (대략):
  서버 측: ECDHE(키 교환) + ECDSA(서명) ≈ 0.1ms
  클라이언트 측: ECDHE + 서명 검증 ≈ 0.1ms
  
  1000 TLS 핸드쉐이크/s: 약 100ms CPU 시간
  데이터 암호화/복호화(AES-GCM): 수 GB/s → 병목 없음
  → 핸드쉐이크가 TLS의 주요 CPU 비용
```

---

## ⚖️ 트레이드오프

```
보안 vs 성능:

AES-128 vs AES-256:
  AES-128: 더 빠름, 충분히 안전
  AES-256: 약간 느림, 양자 컴퓨터 대비 여유
  현실적 위협 모델에서는 AES-128도 충분
  Google, TLS: 두 가지 모두 사용

RSA vs ECC:
  RSA-2048: 폭넓은 지원, 구현 많음
             단점: 큰 키, 느린 서명, 느린 복호화
  ECC-256:  같은 보안 수준, 8배 작은 키, 3~4배 빠른 서명
             지원: 대부분의 최신 시스템
  → TLS 1.3: ECC 필수, RSA 선택

Forward Secrecy(전방 비밀성):
  RSA 키 교환: 서버 개인키 유출 시 과거 트래픽 복호화 가능
  ECDHE 키 교환: 세션별 임시 키 → 개인키 유출해도 과거 안전
  → TLS 1.3에서 RSA 키 교환 완전 제거, ECDHE 필수

알고리즘 수명:
  암호화는 시간이 지나면 약해짐 (컴퓨터 성능 향상)
  MD5, SHA-1: 이미 사용 금지
  RSA-1024: 사용 금지
  → 주기적 업그레이드 필요 (암호 민첩성, Crypto Agility)
```

---

## 📌 핵심 정리

```
암호화 기초 핵심 요약:

대칭키 암호화:
  같은 키로 암호화/복호화
  빠름 (AES-NI: GB/s)
  문제: 키를 어떻게 안전하게 공유?
  사용: 실제 데이터 암호화 (TLS 세션)

비대칭키 암호화:
  공개키로 암호화, 개인키로 복호화 (기밀성)
  개인키로 서명, 공개키로 검증 (인증/무결성)
  느림 → 대용량 데이터 직접 암호화 부적합
  사용: 키 교환 (ECDHE), 인증서 서명 (ECDSA)

해시 함수:
  단방향 (복호화 불가)
  무결성 검증, 비밀번호 저장
  비밀번호: 반드시 bcrypt/Argon2 (느린 해시)
  데이터 무결성: SHA-256

MAC vs 디지털 서명:
  HMAC: 대칭키, 빠름, 부인 불가 없음 (JWT HS256)
  서명: 비대칭키, 느림, 부인 불가 (TLS 인증서, JWT RS256)

TLS의 조합:
  ECDHE (키 교환) + ECDSA (인증) + AES-256-GCM (암호화) + SHA-384 (해시)
  = 각 기법의 장점만 결합

현재 권장:
  대칭: AES-256-GCM 또는 ChaCha20-Poly1305
  비대칭: ECC P-256 (ECDH/ECDSA) 또는 RSA-2048
  해시: SHA-256 이상
  비밀번호: bcrypt(12) 또는 Argon2id
```

---

## 🤔 생각해볼 문제

**Q1.** JWT를 HS256(HMAC)과 RS256(RSA) 중 어느 것으로 서명해야 하는가? 각각 언제 적합한가?

<details>
<summary>해설 보기</summary>

**HS256 (HMAC-SHA256):**
- 같은 비밀키로 서명 + 검증
- 토큰을 생성하는 서버와 검증하는 서버가 같거나, 같은 비밀키를 안전하게 공유할 수 있는 경우

**적합한 경우:**
- 단일 서비스 (Auth 서버 = Resource 서버)
- 내부 마이크로서비스 간 통신 (같은 비밀키 공유)

**문제:**
- 비밀키를 공유해야 함 → 여러 서비스에 키 배포 관리 복잡
- 키가 유출되면 모든 서비스 토큰 위조 가능

---

**RS256 (RSA-SHA256) / ES256 (ECDSA-SHA256):**
- 개인키로 서명(Auth 서버만 소유), 공개키로 검증(누구나 가능)

**적합한 경우:**
- OAuth 2.0 / OIDC (Auth Server가 별도로 존재)
- 여러 마이크로서비스가 토큰 검증 (공개키를 배포하면 됨)
- 타사 클라이언트가 검증해야 하는 경우
- JWKS(JSON Web Key Set) endpoint로 공개키 자동 배포 가능

**실무 권장:**
```yaml
# Keycloak, Auth0 등 OAuth 서버: RS256 또는 ES256 (권장)
# 단순 단일 서비스: HS256 가능하지만 키 관리 주의

# Spring Security + JWT 설정
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://auth.example.com/.well-known/jwks.json
          # 공개키 자동 갱신, RS256/ES256 검증
```

</details>

---

**Q2.** "HTTPS를 쓰면 안전하다"는 말의 정확한 의미와 한계는?

<details>
<summary>해설 보기</summary>

**HTTPS(TLS)가 보장하는 것:**
1. **기밀성(Confidentiality):** 전송 중 데이터를 제3자가 읽을 수 없음 (AES 암호화)
2. **무결성(Integrity):** 전송 중 데이터가 변조되지 않음 (GCM 인증 태그)
3. **서버 인증(Authentication):** 접속한 서버가 진짜 해당 도메인임 (인증서 검증)

**HTTPS가 보장하지 않는 것:**
1. **서버 코드 보안:** SQL Injection, XSS, 로직 버그는 TLS로 방어 불가
2. **클라이언트 보안:** 악성코드, 키로거는 암호화 전에 데이터 탈취 가능
3. **저장 데이터 보안:** DB의 데이터는 별도 암호화 필요 (at-rest)
4. **DoS 방어:** TLS가 연결 폭주를 막지 못함
5. **서버 신원 이상의 보장:** 인증서는 도메인 소유를 증명할 뿐, 서비스 신뢰성은 무관

**중요한 한계 - CA 신뢰:**
- 악의적인 CA가 위조 인증서를 발급하면 HTTPS도 무력화
- 실제 사례: DigiNotar 해킹(2011) → 구글, 모질라 등 위조 인증서 발급
- Certificate Transparency(CT)로 완화: 모든 인증서 공개 로그에 기록

**결론:** "HTTPS = 전송 구간 보안"이지 "전체 시스템 보안"이 아닙니다. 서버 보안, DB 보안, 코드 보안이 함께 필요합니다.

</details>

---

**Q3.** AES-256으로 암호화한 데이터가 유출됐다. 공격자가 복호화할 수 있는 현실적인 방법은 무엇인가?

<details>
<summary>해설 보기</summary>

AES-256 자체를 "깨는" 것은 현실적으로 불가능합니다. 하지만 다른 공격 벡터가 존재합니다.

**1. 키 탈취 (가장 현실적)**
- 소스 코드에 하드코딩된 키 (`String key = "my-secret-key"`)
- 환경 변수나 설정 파일에서 유출
- 메모리 덤프로 실행 중인 키 추출

**2. 알고리즘 구현 취약점**
- 잘못된 운용 모드 (ECB: 패턴 노출)
- 정적 IV 사용 → 같은 키+IV로 암호화하면 패턴 노출
- Padding Oracle 공격 (CBC 모드 + 잘못된 오류 처리)

**3. 약한 키 파생**
```python
# 취약한 방식
key = hashlib.sha256(b"password").digest()  # 예측 가능

# 올바른 방식
key = hashlib.pbkdf2_hmac('sha256', password, salt, iterations=600000)
```

**4. 사이드 채널 공격**
- 암호화 연산 시간 측정으로 키 추론 (Timing Attack)
- 전력 소비 분석 (하드웨어 공격)

**방어:**
```java
// 키를 환경 변수나 HSM에서 로드
String key = System.getenv("ENCRYPTION_KEY");
// 또는 AWS KMS, HashiCorp Vault 사용

// 적절한 운용 모드
Cipher.getInstance("AES/GCM/NoPadding");  // GCM = 안전

// 랜덤 IV 사용
byte[] iv = new byte[12];
new SecureRandom().nextBytes(iv);
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: TLS 1.2 핸드쉐이크 ➡️](./02-tls12-handshake.md)**

</div>
