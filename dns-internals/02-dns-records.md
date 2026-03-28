# DNS 레코드 완전 가이드 — A, CNAME, MX, TXT, SRV

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- A, AAAA, CNAME, MX, TXT, SRV, PTR 레코드의 정확한 용도와 형식은?
- CNAME이 Zone Apex에서 사용 불가능한 이유와 대안은?
- MX 레코드의 우선순위(Priority)는 어떻게 동작하는가?
- TXT 레코드로 SPF, DKIM, DMARC를 어떻게 설정하는가?
- SRV 레코드는 어떻게 서비스 위치를 광고하는가?
- `dig -t MX`, `dig -t TXT` 등으로 각 레코드를 어떻게 조회하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"메일이 스팸으로 분류되는 이유":
  SPF 레코드 없음 → 발신 서버 인증 안 됨 → 스팸 처리
  DKIM 서명 없음 → 메일 변조 여부 확인 불가
  DMARC 없음 → 도메인 사칭 메일 차단 정책 없음
  
  → TXT 레코드 3종 세트 필수:
    SPF: "이 IP가 우리 도메인 메일 서버야"
    DKIM: "이 공개키로 메일 서명 검증해줘"
    DMARC: "SPF/DKIM 실패 시 이렇게 처리해줘"

"www.example.com과 example.com을 같은 서버로 보내고 싶다":
  잘못된 방법:
    example.com  CNAME  www.example.com → Zone Apex에서 불가!
  
  올바른 방법:
    example.com  A  203.0.113.10  (직접 IP 지정)
    www.example.com  CNAME  example.com  (또는 같은 A 레코드)
    
    또는 Route53 Alias, Cloudflare CNAME Flattening 사용

"K8s에서 gRPC 서비스 디스커버리":
  SRV 레코드로 포트와 프로토콜 정보 광고
  _grpclb._tcp.my-service.namespace.svc.cluster.local
  → gRPC 클라이언트가 자동으로 백엔드 서버 목록 조회
```

---

## 😱 흔한 실수

```
Before — DNS 레코드를 모를 때:

실수 1: CNAME 체인이 너무 길어짐
  www → alias1 → alias2 → alias3 → 최종 A 레코드
  각 CNAME 조회마다 추가 DNS 쿼리 필요
  → 최대 8단계 제한 (무한 루프 방지)
  → CNAME은 1~2단계로 유지

실수 2: MX 레코드에 IP 직접 지정
  example.com  MX  10  203.0.113.5  (IP)
  → RFC 위반! MX 레코드는 호스트명만 가능
  example.com  MX  10  mail.example.com  (올바름)
  mail.example.com  A  203.0.113.5

실수 3: PTR 레코드 설정 없이 메일 발송
  PTR (역방향 DNS): IP → 도메인명
  메일 서버가 PTR 없으면 스팸 필터 강화
  → 203.0.113.10.in-addr.arpa  PTR  mail.example.com
  → ISP/클라우드에 PTR 레코드 설정 요청 필요

실수 4: TXT 레코드 여러 개 값 문법 오류
  잘못: example.com TXT "v=spf1 include:_spf.google.com" "-all"
  올바름: example.com TXT "v=spf1 include:_spf.google.com -all"
  → 여러 문자열은 합쳐서 해석, 255자 초과 시 분할 필요
```

---

## ✨ 올바른 접근

```
After — DNS 레코드를 알면:

Zone 파일 기본 구조:
  $ORIGIN example.com.    ; 기준 도메인
  $TTL 3600               ; 기본 TTL
  
  ; SOA (Start of Authority) — 존 메타데이터
  @  IN  SOA  ns1.example.com.  hostmaster.example.com. (
              2026032801  ; Serial (날짜+버전)
              3600        ; Refresh
              900         ; Retry
              604800      ; Expire
              300         ; Negative TTL
  )
  
  ; NS 레코드 — 권한 있는 Name Server
  @  IN  NS  ns1.example.com.
  @  IN  NS  ns2.example.com.
  
  ; A 레코드
  @       IN  A      203.0.113.10   ; example.com
  www     IN  A      203.0.113.10
  api     IN  A      203.0.113.20
  
  ; CNAME (서브도메인만!)
  docs    IN  CNAME  api.example.com.
  
  ; MX
  @  IN  MX  10  mail1.example.com.
  @  IN  MX  20  mail2.example.com.  ; 백업
  
  ; TXT (SPF)
  @  IN  TXT  "v=spf1 include:_spf.google.com ~all"
  
  ; DMARC
  _dmarc  IN  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"

이메일 인프라 TXT 체크리스트:
  dig TXT example.com      → SPF 확인
  dig TXT _dmarc.example.com → DMARC 확인
  dig TXT default._domainkey.example.com → DKIM 확인
  mxtoolbox.com/supertool.aspx → 종합 점검
```

---

## 🔬 내부 동작 원리

### 1. 핵심 레코드 완전 분석

```
A 레코드 (Address):
  호스트명 → IPv4 주소 매핑
  형식: name  TTL  IN  A  IPv4
  
  예시:
  api.example.com.  300  IN  A  203.0.113.10
  api.example.com.  300  IN  A  203.0.113.11  ← 복수 A (라운드로빈)
  
  하나의 이름에 여러 A 레코드 → DNS 레벨 로드밸런싱
  (단, 클라이언트 캐시로 인해 불균등 — 04 문서 참조)

AAAA 레코드 (Quad-A):
  호스트명 → IPv6 주소 매핑
  형식: name  TTL  IN  AAAA  IPv6
  
  예시:
  api.example.com.  300  IN  AAAA  2001:db8::1
  
  A와 AAAA 동시 보유 → 듀얼 스택
  클라이언트: Happy Eyeballs 알고리즘으로 빠른 쪽 선택

CNAME 레코드 (Canonical Name):
  별칭 → 정식 이름 매핑
  형식: alias  TTL  IN  CNAME  canonical
  
  예시:
  www.example.com.  300  IN  CNAME  example.com.
  docs.example.com. 300  IN  CNAME  api.example.com.
  static.example.com. 300 IN CNAME cdn.provider.net.
  
  CNAME 조회 흐름:
    www.example.com 요청
    → CNAME: example.com (별칭, 다시 조회)
    → A: 203.0.113.10 (최종 응답)
    → 클라이언트: www도, example.com도, 203.0.113.10도 모두 반환
  
  Zone Apex에서 CNAME 불가 이유:
    Zone Apex(@)는 SOA와 NS 레코드를 반드시 가져야 함
    CNAME은 "다른 이름으로 대체"를 의미
    → Zone Apex가 CNAME이면 SOA/NS 레코드도 "사라짐" → RFC 위반
    
    대안:
    Route53 Alias 레코드: ELB, CloudFront에 Zone Apex 연결
    Cloudflare CNAME Flattening: CNAME을 A로 자동 변환
    직접 A 레코드: 동일 IP 중복 설정

MX 레코드 (Mail Exchange):
  도메인 → 메일 서버 매핑
  형식: domain  TTL  IN  MX  priority  hostname
  
  예시:
  example.com.  3600  IN  MX  10  aspmx.l.google.com.  ← 1순위
  example.com.  3600  IN  MX  20  alt1.aspmx.l.google.com. ← 2순위
  example.com.  3600  IN  MX  30  alt2.aspmx.l.google.com. ← 3순위
  
  Priority 값: 낮을수록 우선 (10 < 20 < 30)
  메일 서버 장애 시 Priority 높은 서버로 자동 폴백
  
  중요: MX 레코드의 값은 반드시 A 레코드를 가진 호스트명
        IP 주소 직접 지정 → RFC 위반 → 일부 메일 서버 거부

TXT 레코드 (Text):
  임의 텍스트 데이터 저장
  형식: name  TTL  IN  TXT  "text"
  
  용도: SPF, DKIM, DMARC, 도메인 소유권 확인, 기타
  최대 길이: 단일 문자열 255 bytes
  255자 초과 시 여러 문자열로 분할:
  "v=spf1 include:_spf.google.com include:mailgun.org " "include:sendgrid.net -all"
  → DNS 서버가 합쳐서 해석

SRV 레코드 (Service):
  서비스 위치(호스트 + 포트) 광고
  형식: _service._proto.name  TTL  IN  SRV  priority  weight  port  target
  
  예시:
  _https._tcp.example.com.  300  IN  SRV  10  100  443  api.example.com.
  _grpc._tcp.my-service.default.svc.cluster.local.  SRV  0  50  9090  pod1.example.
  
  필드:
    priority: 낮을수록 우선 (MX와 같은 원리)
    weight: 같은 priority에서 가중치 (로드밸런싱)
    port: 서비스 포트
    target: 실제 서버 호스트명
  
  사용처:
    VoIP (SIP): _sip._tcp.example.com
    XMPP: _xmpp-server._tcp.example.com
    gRPC load balancing: _grpclb._tcp.service
    K8s Headless Service: 각 Pod의 SRV 레코드 자동 생성

PTR 레코드 (Pointer):
  IPv4 → 호스트명 역방향 조회
  형식: reversed-IP.in-addr.arpa.  TTL  IN  PTR  hostname
  
  예시:
  IP: 203.0.113.10
  PTR 이름: 10.113.0.203.in-addr.arpa.
  PTR 값: mail.example.com.
  
  조회: dig -x 203.0.113.10
  
  메일 서버에 PTR 필수:
  수신 메일 서버가 발신 IP의 PTR 확인
  PTR 없거나 forward-confirmed 아니면 스팸 처리
  (forward-confirmed: PTR → 호스트명 → A → 원래 IP 일치)
```

### 2. 이메일 보안 TXT 레코드 3종 세트

```
SPF (Sender Policy Framework):
  "우리 도메인 메일은 이 서버에서만 보냄"
  
  example.com.  TXT  "v=spf1 [메커니즘] [수정자]"
  
  메커니즘:
    ip4:203.0.113.0/24  → 이 IP 범위 허용
    include:_spf.google.com → Google의 SPF 목록 포함
    a:mail.example.com  → 해당 A 레코드 허용
    mx:example.com      → MX 서버 허용
  
  수정자 (최종 규칙):
    -all  → 나머지 모두 거부 (Fail)
    ~all  → 나머지 소프트 실패 (SoftFail, 스팸 처리)
    +all  → 모두 허용 (의미 없음)
    ?all  → Neutral (비권장)
  
  예시:
  "v=spf1 include:_spf.google.com include:sendgrid.net ip4:10.0.0.0/8 -all"
  → Google, SendGrid, 내부망에서 보낸 메일만 허용, 나머지 거부

DKIM (DomainKeys Identified Mail):
  메일에 비대칭키 서명 → 변조 여부 확인
  
  서버: 메일 본문/헤더를 개인키로 서명 → DKIM-Signature 헤더 추가
  수신자: TXT 레코드의 공개키로 서명 검증
  
  selector._domainkey.example.com  TXT  "v=DKIM1; k=rsa; p=[공개키]"
  
  예시:
  default._domainkey.example.com.  TXT
    "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC..."
  
  DKIM-Signature 헤더 (메일에 포함):
    v=1; a=rsa-sha256; c=relaxed/relaxed;
    d=example.com; s=default;
    h=from:to:subject:date;
    bh=[본문 해시]; b=[서명]

DMARC (Domain-based Message Authentication, Reporting & Conformance):
  SPF와 DKIM 결과를 기반으로 처리 정책 정의
  
  _dmarc.example.com.  TXT  "v=DMARC1; [태그들]"
  
  주요 태그:
    p=none/quarantine/reject  → 정책 (없음/격리/거부)
    rua=mailto:dmarc-reports@example.com → 집계 보고 수신 주소
    ruf=mailto:forensics@example.com → 포렌식 보고 수신 주소
    pct=100  → 적용 비율 (100% = 전체)
    adkim=s  → DKIM 정렬 엄격도 (strict/relaxed)
    aspf=s   → SPF 정렬 엄격도
  
  예시:
  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com; pct=100"
  → SPF/DKIM 모두 실패하면 메일 거부 + 보고서 전송
  
  도입 전략:
    1단계: p=none (모니터링만, 차단 없음)
    2단계: p=quarantine (스팸함으로)
    3단계: p=reject (완전 차단) — 안정화 후
```

### 3. SOA 레코드 — 존의 메타데이터

```
SOA (Start of Authority):
  해당 DNS 존의 관리 정보

example.com.  IN  SOA  ns1.example.com.  hostmaster.example.com. (
    2026032801  ; Serial: 날짜(YYYYMMDD) + 버전(01)
                ; 변경마다 증가 → Secondary NS가 갱신 여부 확인
    3600        ; Refresh: Secondary NS가 Primary에서 갱신하는 주기(초)
    900         ; Retry: Refresh 실패 시 재시도 주기(초)
    604800      ; Expire: Primary 연결 안 되면 Secondary가 포기하는 시간(초)
    300         ; Minimum TTL: Negative 응답(NXDOMAIN)의 TTL
)

Primary/Secondary NS:
  Primary NS: 존 파일 원본 보유, 직접 수정 가능
  Secondary NS: Primary에서 Zone Transfer로 복제
  → 이중화로 Primary 장애 시에도 DNS 서비스 유지

Zone Transfer (AXFR):
  Primary → Secondary: 전체 존 파일 전송
  IXFR: 변경분만 전송 (Incremental Zone Transfer)
  
  보안 주의: AXFR을 허용하면 외부에서 모든 레코드 조회 가능
  → 특정 IP (Secondary NS)만 AXFR 허용
  allow-transfer { 203.0.113.2; };
```

---

## 💻 실전 실험

### 실험 1: 각 레코드 타입 조회

```bash
# A 레코드
dig A google.com +short
# 142.250.x.x

# AAAA 레코드
dig AAAA google.com +short
# 2404:6800:4004:xxx::xxx

# MX 레코드 (우선순위 포함)
dig MX gmail.com
# gmail.com. MX 5 gmail-smtp-in.l.google.com.
# gmail.com. MX 10 alt1.gmail-smtp-in.l.google.com.

# TXT 레코드 (SPF 확인)
dig TXT gmail.com +short
# "v=spf1 redirect=_spf.google.com"

# DKIM 확인 (selector: google)
dig TXT google._domainkey.gmail.com +short

# DMARC 확인
dig TXT _dmarc.gmail.com +short
# "v=DMARC1; p=none; rua=..."

# SRV 레코드
dig SRV _https._tcp.google.com
# (SRV 없으면 NXDOMAIN)

# PTR (역방향 조회)
dig -x 8.8.8.8 +short
# dns.google.

# CNAME 체인 추적
dig CNAME www.github.com
dig A www.github.com  # CNAME 포함한 최종 A 레코드
```

### 실험 2: 이메일 인프라 점검

```bash
# SPF 확인
dig TXT example.com | grep "v=spf1"

# DMARC 확인
dig TXT _dmarc.example.com

# DKIM 확인 (selector를 알아야 함)
# selector는 메일 헤더의 DKIM-Signature에서 s= 값
dig TXT default._domainkey.example.com

# 종합 이메일 설정 점검
# mxtoolbox.com 또는 mail-tester.com 사용

# 내 도메인 SPF 유효성 검사
dig TXT my-domain.com +short | grep spf
# 10개 이상의 DNS lookup이 있으면 SPF 실패 가능 (10 lookup 제한)
```

### 실험 3: CNAME 조회 과정

```bash
# CNAME 체인 확인
dig CNAME www.example.com
# www.example.com. 300 IN CNAME example.com.

# 최종 A 레코드까지 자동 해결
dig A www.example.com
# www.example.com. 300 IN CNAME example.com.
# example.com. 300 IN A 203.0.113.10

# Zone Apex에서 CNAME 시도 (RFC 위반 확인)
# example.com 자체에 CNAME 설정하면 안 됨
# Route53 Alias 레코드 조회 (실제로는 A로 반환됨)
dig A example.com
# → Route53 Alias는 CNAME처럼 동작하지만 A로 반환
```

### 실험 4: SRV 레코드로 서비스 디스커버리

```bash
# K8s Headless Service SRV 조회
# (K8s 클러스터 내부에서)
dig SRV my-service.default.svc.cluster.local
# _my-port._tcp.my-service.default.svc.cluster.local.
#   SRV 0 50 80 pod-ip-10-0-0-1.my-service.default.svc.cluster.local.

# gRPC 서비스 디스커버리
dig SRV _grpclb._tcp.my-grpc-service.default.svc.cluster.local

# XMPP 서버 찾기
dig SRV _xmpp-server._tcp.gmail.com
# _xmpp-server._tcp.gmail.com. SRV 5 0 5269 xmpp-server.l.google.com.
```

---

## 📊 성능/비용 비교

```
레코드 타입별 DNS 조회 횟수:

A 레코드 직접:
  api.example.com → A → 1번 조회
  
CNAME 1단계:
  www.example.com → CNAME → example.com → A → 2번 조회
  
CNAME 체인:
  alias1 → CNAME → alias2 → CNAME → example.com → A → 3번 조회
  각 단계마다 추가 DNS 쿼리 → 지연 증가
  최대 8단계 (무한 루프 방지)

MX 조회:
  메일 발송 시:
  1. example.com MX 조회
  2. MX 호스트명 A 조회
  = 2번 조회

SPF 조회 비용 (10 lookup 제한):
  include: 사용마다 1 DNS 조회
  include:_spf.google.com → 내부에 또 include 있으면 추가 조회
  10개 초과 → SPF PermError (유효하지 않은 SPF)
  
  최적화:
  include를 많이 쓰는 대신 ip4/ip6으로 직접 IP 명시
  또는 SPF 레코드를 하나의 include로 통합

Route53 비용:
  Hosted Zone: $0.50/월
  쿼리: $0.40/백만 건 (Standard)
        $0.60/백만 건 (Latency 기반 라우팅)
  Alias 쿼리: 무료 (AWS 리소스 대상)
```

---

## ⚖️ 트레이드오프

```
A vs CNAME 선택:

A 레코드:
  장점: 빠름 (1번 조회), Zone Apex 가능
  단점: IP 변경 시 모든 A 레코드 업데이트 필요
  적합: Zone Apex, IP가 자주 변경 안 되는 서버

CNAME:
  장점: 한 곳에서 IP 관리 (canonical 이름만 변경)
  단점: 추가 DNS 조회, Zone Apex 불가
  적합: CDN, 외부 서비스 연동, IP가 자주 변경되는 경우

CDN 연동 패턴:
  static.example.com  CNAME  mycdn.cdn-provider.net.
  → CDN 내부에서 최적 IP 반환 (GeoDNS, Anycast)
  → IP 변경을 CDN이 알아서 처리

MX Priority 설계:
  단순 이중화:    MX 10 primary, MX 20 backup
  동등 부하분산:  MX 10 server1, MX 10 server2 (같은 priority)
  Google Workspace: MX 1 aspmx.l.google.com (여러 개 권장)

TTL 설정 레코드별 권장:
  A/AAAA:   300~3600초 (IP 변경 주기에 맞춤)
  CNAME:    3600초 (잘 변경 안 됨)
  MX:       3600~86400초 (메일 서버 자주 안 바뀜)
  TXT(SPF): 3600초 (변경 시 전파 고려)
  SOA:      86400초 (거의 변경 없음)
```

---

## 📌 핵심 정리

```
DNS 레코드 핵심 요약:

주요 레코드:
  A:     호스트명 → IPv4 (기본)
  AAAA:  호스트명 → IPv6
  CNAME: 별칭 → 정식 이름 (서브도메인만, Zone Apex 불가)
  MX:    도메인 → 메일 서버 (숫자 작을수록 우선)
  TXT:   임의 텍스트 (SPF/DKIM/DMARC/소유권 확인)
  SRV:   서비스 위치 (hostname + port, K8s/gRPC 등)
  PTR:   IP → 호스트명 역방향 (메일 서버 필수)
  SOA:   존 메타데이터 (시리얼, TTL 기본값)

이메일 보안 3종 세트:
  SPF: "이 서버가 우리 메일 서버" (발신 서버 인증)
  DKIM: "개인키로 서명, 공개키로 검증" (변조 방지)
  DMARC: "SPF/DKIM 실패 시 처리 정책" (도메인 보호)

Zone Apex 제약:
  @에 CNAME 불가 (SOA/NS와 충돌)
  대안: A 레코드, Route53 Alias, Cloudflare CNAME Flattening

MX 주의사항:
  MX 값은 호스트명만 (IP 직접 지정 금지)
  Priority 낮을수록 우선순위 높음

진단:
  dig -t [TYPE] domain  레코드 조회
  dig MX domain         MX 조회
  dig TXT _dmarc.domain DMARC 조회
  dig -x IP             PTR 역방향 조회
  mxtoolbox.com         이메일 설정 종합 점검
```

---

## 🤔 생각해볼 문제

**Q1.** `example.com`을 AWS ALB에 연결하고 싶다. ALB는 IP가 자주 바뀌는데 어떻게 Zone Apex에 설정하는가?

<details>
<summary>해설 보기</summary>

**문제:** Zone Apex(`example.com`)에 CNAME 불가, ALB는 IP가 자주 바뀜.

**Route53 Alias 레코드 (AWS 전용):**
```
example.com.  A  ALIAS  my-alb-123456.us-east-1.elb.amazonaws.com.
```
- DNS 응답: CNAME이 아닌 A 레코드로 반환 (IP를 직접 반환)
- Route53이 ALB의 현재 IP를 추적하고 자동 업데이트
- Alias 쿼리 비용 없음 (무료)
- Zone Apex 가능

**Cloudflare CNAME Flattening:**
```
example.com.  CNAME  my-alb.elb.amazonaws.com.
```
- Cloudflare가 내부적으로 CNAME을 A 레코드로 변환
- 클라이언트에게 A 레코드로 반환

**직접 IP 고정 (ALB는 IP가 바뀌어서 불가, 이론적):**
```
example.com.  A  54.x.x.x  (ALB IP)
```
- ALB IP가 바뀌면 수동 업데이트 필요 → 서비스 중단 위험

**권장:** AWS 환경에서는 Route53 Alias 레코드 사용. 다른 DNS 서비스라면 CNAME Flattening 지원 여부 확인.

</details>

---

**Q2.** SPF 레코드에 10 DNS Lookup 제한이 있다. `include`를 많이 사용하는 현재 설정에서 어떻게 최적화하는가?

<details>
<summary>해설 보기</summary>

**10 lookup 제한의 의미:**
SPF 검증 시 DNS 조회가 10번을 초과하면 PermError → SPF 검증 실패 → 스팸 처리

**Lookup을 소비하는 메커니즘:** `include`, `a`, `mx`, `exists`, `ptr`

**최적화 방법:**

1. **ip4/ip6으로 직접 명시 (Lookup 0회):**
```
v=spf1 ip4:203.0.113.0/24 include:_spf.google.com -all
```
고정 IP를 가진 발신 서버는 `include` 대신 `ip4`로 직접 지정.

2. **include 중첩 확인:**
```bash
# 현재 SPF의 실제 lookup 수 계산
dig TXT example.com +short | grep spf
# include 목록을 재귀적으로 확인
```

3. **SPF 플래트닝 서비스:**
`include`를 분석해서 최종 IP 목록으로 자동 변환
→ Mailhardener, AutoSPF 등 서비스 활용
→ 단, 발신 IP 변경 시 자동 업데이트 필요

4. **중복 include 제거:**
같은 IP를 두 개의 include가 포함하면 하나 제거

5. **DMARC로 SPF 의존도 낮추기:**
DKIM이 올바르게 설정되면 SPF 실패해도 DMARC 통과 가능 (or 조건).
SPF보다 DKIM에 더 의존하는 구조로 전환.

</details>

---

**Q3.** K8s Headless Service를 사용하면 일반 Service와 DNS 조회 결과가 어떻게 다른가?

<details>
<summary>해설 보기</summary>

**일반 Service (ClusterIP):**
```yaml
spec:
  clusterIP: 10.96.100.1  # 고정 가상 IP
```
- DNS 조회: `my-service.default.svc.cluster.local` → `10.96.100.1` (하나의 IP)
- kube-proxy가 이 IP를 Pod들의 실제 IP로 부하분산
- 클라이언트는 Pod IP를 알 수 없음

**Headless Service:**
```yaml
spec:
  clusterIP: None  # Headless
```
- DNS 조회: `my-service.default.svc.cluster.local` → Pod IP 목록 (A 레코드 여러 개)
  ```
  10.0.0.1  (pod1)
  10.0.0.2  (pod2)
  10.0.0.3  (pod3)
  ```
- SRV 레코드: `_http._tcp.my-service.default.svc.cluster.local` → 각 Pod의 SRV
- 클라이언트가 Pod IP를 직접 알고 자체 로드밸런싱 가능

**Headless Service가 필요한 경우:**
1. **StatefulSet:** 각 Pod에 안정적인 고유 이름 필요
   `web-0.my-service.default.svc.cluster.local`
2. **gRPC 로드밸런싱:** gRPC 클라이언트가 직접 연결 관리 (HTTP/2 연결 유지)
   일반 Service → 단일 IP → 부하분산이 TCP 레벨 (연결 단위)
   Headless → Pod IP 직접 → gRPC 클라이언트가 RPC 레벨 부하분산
3. **Kafka, Elasticsearch:** 클러스터 멤버 직접 발견

</details>

---

<div align="center">

**[⬅️ 이전: DNS 조회 흐름](./01-dns-resolution-flow.md)** | **[홈으로 🏠](../README.md)** | **[다음: DNS 캐싱과 전파 ➡️](./03-dns-caching-and-propagation.md)**

</div>
