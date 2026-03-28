# DNS 조회 흐름 — Resolver, Root NS, TLD NS, Authoritative NS

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `api.example.com`을 입력했을 때 IP 주소를 얻기까지 어떤 서버를 거치는가?
- Recursive Resolver와 Iterative Resolver의 차이는 무엇인가?
- Root NS, TLD NS, Authoritative NS는 각각 어떤 역할을 하는가?
- `dig +trace example.com`으로 무엇을 확인할 수 있는가?
- JVM(Spring)의 DNS 캐시 TTL 설정이 왜 중요한가?
- DNS 조회 실패 시 어떤 순서로 진단해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"배포 후 일부 서버에서만 새 IP로 요청이 간다":
  원인: DNS TTL + JVM DNS 캐시의 이중 캐싱
  JVM 기본값: 네트워크 주소 캐시 영구 (보안 설정 시)
  → DNS TTL이 만료돼도 JVM이 이전 IP 계속 사용
  → Spring 서비스 재시작 전까지 이전 서버로 요청

"DNS 조회가 실패해서 서비스 전체가 다운됐다":
  외부 DNS 서버 의존 → DNS 서버 장애 = 서비스 장애
  → 로컬 캐싱 Resolver 필수 (systemd-resolved, dnsmasq)
  → 내부 서비스 도메인은 CoreDNS(K8s) 또는 Route53 Private Hosted Zone

실무 연결 고리:
  Docker/K8s:   컨테이너마다 /etc/resolv.conf → CoreDNS
  Spring Cloud: Ribbon/Feign의 DNS 캐시 설정
  마이크로서비스: 서비스 디스커버리 = DNS 기반 (K8s Service)
  AWS:          Route53 Resolver, Private Hosted Zone
  
"k8s에서 my-service.default.svc.cluster.local을 어떻게 찾나?":
  CoreDNS가 cluster.local 존을 관리
  내부 요청 → CoreDNS → 해당 Service의 ClusterIP 반환
```

---

## 😱 흔한 실수

```
Before — DNS 조회 흐름을 모를 때:

실수 1: JVM DNS 캐시 무시
  DNS TTL=60초로 낮춰도 JVM이 무기한 캐시
  # JVM 기본 설정 (보안 매니저 활성화 시):
  networkaddress.cache.ttl = -1  (영구 캐시!)
  → 해결: JVM 옵션 또는 $JAVA_HOME/conf/security/java.security 수정
    networkaddress.cache.ttl=30
    networkaddress.cache.negative.ttl=10

실수 2: DNS 조회 실패를 서버 문제로 오해
  "curl: Could not resolve host: api.example.com"
  → 서버가 다운됐다고 판단
  → 실제: /etc/resolv.conf의 nameserver가 잘못됨
  진단 순서: dig @8.8.8.8 api.example.com → 외부 DNS는 정상
             dig api.example.com → 로컬 Resolver 문제

실수 3: CNAME을 Zone Apex에 사용
  example.com CNAME cdn.provider.com → 불가!
  (Zone Apex = 루트 도메인 = @)
  → CNAME은 서브도메인에만 사용 가능
  → Zone Apex: A 레코드 또는 ALIAS/ANAME 레코드 사용

실수 4: TTL을 높게 유지한 채 DNS 변경
  TTL=86400(24시간) 상태에서 IP 변경
  → 전 세계 캐시가 24시간 동안 이전 IP 사용
  → 해결: 변경 24시간 전에 TTL을 300(5분)으로 낮춤
```

---

## ✨ 올바른 접근

```
After — DNS 조회 흐름을 알면:

JVM DNS 캐시 올바른 설정:
  # $JAVA_HOME/conf/security/java.security
  networkaddress.cache.ttl=30        # 30초 후 재조회
  networkaddress.cache.negative.ttl=10 # 실패한 조회 10초 캐시

  # 또는 JVM 시작 옵션
  -Dsun.net.inetaddr.ttl=30
  -Dsun.net.inetaddr.negative.ttl=10

  # Spring Boot application.properties
  # (직접 설정 없음, JVM 설정 사용)

DNS 장애 대비 다중 Resolver:
  /etc/resolv.conf:
    nameserver 10.0.0.2    # 내부 캐싱 Resolver (1순위)
    nameserver 8.8.8.8     # Google DNS (백업)
    nameserver 1.1.1.1     # Cloudflare DNS (백업)
    options timeout:2 attempts:3

DNS 전파 전략 (서비스 마이그레이션):
  T-24h: TTL을 86400 → 300으로 낮춤
  T-0:   IP 변경 (최대 300초 내 전파)
  T+1h:  안정화 확인 후 TTL 다시 높임

진단 워크플로:
  # 1. 외부 DNS 정상 확인
  dig @8.8.8.8 api.example.com
  
  # 2. 전체 조회 경로 추적
  dig +trace api.example.com
  
  # 3. 로컬 Resolver 확인
  cat /etc/resolv.conf
  systemctl status systemd-resolved
  
  # 4. 현재 캐시 확인 (재귀 없이)
  dig +norecurse api.example.com @127.0.0.53
```

---

## 🔬 내부 동작 원리

### 1. DNS 계층 구조

```
DNS 이름 공간 계층:

.                          ← Root (최상위, "점")
├── com.                   ← TLD (Top-Level Domain)
│   ├── example.com.       ← Second-Level Domain
│   │   ├── api.example.com.   ← Subdomain
│   │   └── www.example.com.
│   └── google.com.
├── net.
├── org.
├── kr.                    ← ccTLD (Country Code)
│   └── co.kr.
└── (1000+ TLD)

각 계층을 담당하는 Name Server:
  Root NS:        .(루트) 존 관리 → 13개 루트 서버 클러스터
  TLD NS:         .com, .net, .org 존 관리 (Verisign 등)
  Authoritative NS: example.com 존 관리 (Route53, Cloudflare 등)

FQDN (Fully Qualified Domain Name):
  api.example.com. (끝의 점이 Root를 의미)
  실제로는 api.example.com.을 조회하지만 편의상 끝 점 생략
```

### 2. 완전한 DNS 조회 흐름

```
브라우저에서 "api.example.com" 입력 시:

Step 1: 로컬 캐시 확인
┌─────────────────────────────────────────────────────────┐
│  브라우저 DNS 캐시 확인                                     │
│  chrome://net-internals/#dns 또는 about:networking       │
│  → 없으면 다음 단계                                         │
│                                                         │
│  OS DNS 캐시 확인                                         │
│  (Windows: ipconfig /displaydns)                        │
│  (Linux: systemd-resolved 캐시)                          │
│  → 없으면 다음 단계                                         │
│                                                         │
│  /etc/hosts 파일 확인                                     │
│  127.0.0.1  localhost                                   │
│  → 없으면 네트워크 DNS 조회                                  │
└─────────────────────────────────────────────────────────┘

Step 2: Stub Resolver → Recursive Resolver
┌──────────────────────────────────────────────────────────────┐
│  Stub Resolver (앱/OS 내장):                                   │
│    /etc/resolv.conf의 nameserver로 쿼리 전송                    │
│    nameserver 192.168.1.1 (공유기 또는 ISP DNS)                │
│                                                              │
│  Recursive Resolver (캐싱 Resolver):                          │
│    ISP DNS, 8.8.8.8, 1.1.1.1 등                              │
│    클라이언트를 대신해 전체 조회 수행                                │
│    결과를 TTL 동안 캐시                                          │
└──────────────────────────────────────────────────────────────┘

Step 3: Recursive Resolver → Root NS
  Recursive Resolver: "api.example.com의 IP는?"
  (캐시에 없는 경우)
  
  Root NS 주소는 Resolver에 하드코딩됨 (Root Hints 파일)
  13개 Root NS 클러스터:
    a.root-servers.net. (VeriSign)
    b.root-servers.net. (USC-ISI)
    ...
    m.root-servers.net. (WIDE Project)
  
  Resolver → Root NS:
    "api.example.com의 IP는?"
  
  Root NS → Resolver:
    "모르지만 .com을 담당하는 TLD NS는 여기야"
    com.  NS  a.gtld-servers.net.
    com.  NS  b.gtld-servers.net.
    ...
    a.gtld-servers.net.  A  192.5.6.30  (Glue Record)

Step 4: Recursive Resolver → TLD NS (.com)
  Resolver → TLD NS (a.gtld-servers.net.):
    "api.example.com의 IP는?"
  
  TLD NS → Resolver:
    "example.com의 Authoritative NS는 여기야"
    example.com.  NS  ns1.example.com.
    example.com.  NS  ns2.example.com.
    ns1.example.com.  A  205.251.196.1  (Glue Record)

Step 5: Recursive Resolver → Authoritative NS
  Resolver → Authoritative NS (ns1.example.com.):
    "api.example.com의 IP는?"
  
  Authoritative NS → Resolver:
    "여기 있어!"
    api.example.com.  A  203.0.113.10  TTL=300
  
  Resolver: 결과를 TTL(300초) 동안 캐시

Step 6: Resolver → Stub Resolver → 앱
  앱: IP 주소 203.0.113.10 받음
  JVM: 자체 캐시에 저장 (networkaddress.cache.ttl 설정값 동안)

전체 흐름 다이어그램:
  앱 → Stub Resolver → Recursive Resolver
                              │
                    ┌─────────┼──────────┐
                    ↓         ↓          ↓
                 Root NS    TLD NS   Authoritative NS
                 (.)        (.com)   (example.com)
                    │         │          │
                    └─────────┼──────────┘
                              ↓
                        최종 IP 반환
```

### 3. Recursive vs Iterative 조회

```
Recursive 조회 (클라이언트 관점):
  클라이언트 → Recursive Resolver: "api.example.com?"
  Recursive Resolver: 알아서 다 조회해서 최종 답만 반환
  클라이언트: 중간 과정 모름, 최종 IP만 받음

Iterative 조회 (Resolver 내부 동작):
  Resolver → Root NS: "api.example.com?" → 위임(Referral)
  Resolver → TLD NS:  "api.example.com?" → 위임(Referral)
  Resolver → Auth NS: "api.example.com?" → 최종 답

  각 단계는 Iterative: "모르면 이쪽으로 가봐"
  Recursive Resolver가 이 Iterative 과정을 수행
  클라이언트는 Recursive 방식으로 사용

왜 Recursive Resolver가 필요한가:
  모든 클라이언트가 직접 Iterative 조회 시:
  → Root NS에 엄청난 부하 (13개 서버가 전 세계 쿼리 처리)
  → 각 클라이언트가 중복 조회 (캐시 없음)
  
  Recursive Resolver의 역할:
  → 클라이언트 대신 Iterative 조회
  → TTL 동안 캐시 → 다음 쿼리는 캐시에서 즉시 응답
  → ISP당 1개 Resolver가 수백만 클라이언트 캐시 공유
```

### 4. Glue Record — 닭과 달걀 문제 해결

```
문제: example.com의 NS = ns1.example.com
      ns1.example.com의 IP를 알려면 example.com에 물어봐야 함
      example.com을 알려면 ns1.example.com에 물어봐야 함
      → 순환 참조!

Glue Record (Glue 레코드):
  TLD NS에 함께 저장하는 NS 서버의 A 레코드

TLD NS에서 example.com 조회 시 응답:
  example.com.  NS  ns1.example.com.
  example.com.  NS  ns2.example.com.
  ns1.example.com.  A  205.251.196.1  ← Glue Record!
  ns2.example.com.  A  205.251.197.1  ← Glue Record!

→ Resolver가 ns1.example.com의 IP를 즉시 알 수 있음
→ 순환 참조 해결

도메인 등록 시 요구사항:
  도메인 등록 시 NS 서버 등록 + Glue Record 등록
  Route53: 자동으로 Glue Record 처리
  직접 운영 시: 레지스트라에 NS 서버 IP 함께 등록 필요
```

### 5. JVM DNS 캐시 — 실무 핵심

```
JVM의 DNS 캐싱 레이어:
  OS DNS 캐시와 별도로 JVM 자체 캐시 운영
  java.net.InetAddress가 DNS 조회 결과 캐싱

기본값 (JVM 버전/설정에 따라 다름):
  보안 매니저 없음: OS DNS TTL 존중 (기본 30초)
  보안 매니저 활성화: networkaddress.cache.ttl = -1 (영구!)

영구 캐시의 문제:
  블루-그린 배포 시:
  Old Server: 203.0.113.10
  New Server: 203.0.113.20
  DNS 변경: api.example.com → 203.0.113.20
  
  JVM (보안 매니저, networkaddress.cache.ttl=-1):
  → 여전히 203.0.113.10으로 요청
  → 재시작 전까지 이전 서버로 계속 트래픽!

올바른 JVM DNS 설정:
  방법 1: Java Security 파일
    # $JAVA_HOME/conf/security/java.security
    networkaddress.cache.ttl=30
    networkaddress.cache.negative.ttl=10

  방법 2: JVM 시작 인수
    java -Dsun.net.inetaddr.ttl=30 \
         -Dsun.net.inetaddr.negative.ttl=10 \
         -jar app.jar

  방법 3: 코드에서 설정 (Spring Boot)
    @SpringBootApplication
    public class App {
        public static void main(String[] args) {
            java.security.Security.setProperty(
                "networkaddress.cache.ttl", "30");
            SpringApplication.run(App.class, args);
        }
    }

Spring Cloud / Ribbon:
  ribbon.ServerListRefreshInterval=30000  # 30초마다 서버 목록 갱신
  → Eureka 기반 서비스 디스커버리와 연동

K8s에서의 DNS:
  CoreDNS: K8s 내부 DNS 서버
  Pod의 /etc/resolv.conf:
    nameserver 10.96.0.10  (CoreDNS ClusterIP)
    search default.svc.cluster.local svc.cluster.local cluster.local
    options ndots:5

  ndots:5 의미:
    "점이 5개 미만이면 search 도메인을 앞에 붙여서 시도"
    my-service → my-service.default.svc.cluster.local 시도
    → CoreDNS가 응답 (해당 Service의 ClusterIP)
```

---

## 💻 실전 실험

### 실험 1: dig +trace로 전체 DNS 조회 경로 추적

```bash
# 전체 DNS 조회 경로 추적 (Root → TLD → Auth → 최종)
dig +trace +additional api.example.com

# 출력 예:
# ; <<>> DiG 9.18.0 <<>> +trace +additional api.example.com
# .  518400  IN  NS  a.root-servers.net.   ← Root NS 목록
# ...
# com.  172800  IN  NS  a.gtld-servers.net. ← .com TLD NS
# ...
# example.com.  172800  IN  NS  ns1.example.com. ← Auth NS
# ...
# api.example.com.  300  IN  A  203.0.113.10 ← 최종 응답

# TTL 값 확인 (두 번째 컬럼)
dig api.example.com | grep -E "^api|ANSWER"
# api.example.com.  295  IN  A  203.0.113.10
# → 295초 남음 (원래 300초, 5초 경과)

# 다시 조회하면 TTL 감소 확인
sleep 5
dig api.example.com | grep "^api"
# api.example.com.  290  IN  A  203.0.113.10 → 290초로 감소
```

### 실험 2: 특정 DNS 서버 직접 조회

```bash
# 외부 DNS 서버에 직접 조회
dig @8.8.8.8 api.example.com A  # Google DNS
dig @1.1.1.1 api.example.com A  # Cloudflare DNS

# Authoritative NS에 직접 조회 (캐시 없음, 가장 신선한 정보)
# 1. NS 서버 찾기
dig example.com NS
# example.com.  3600  IN  NS  ns1.example.com.

# 2. Authoritative NS에 직접 조회
dig @ns1.example.com api.example.com A
# 응답에 "aa" 플래그 = Authoritative Answer

# 로컬 Resolver에서 캐시 재귀 없이 조회
dig +norecurse api.example.com @127.0.0.53
# ANSWER 있으면: 캐시 히트
# ANSWER 없으면: 캐시 미스 (AUTHORITY 섹션에 위임 정보)
```

### 실험 3: DNS 응답 구조 분석

```bash
# 상세 응답 분석
dig +multiline api.example.com ANY

# 플래그 분석
dig api.example.com +short  # IP만 출력
dig api.example.com +noall +answer  # ANSWER 섹션만
dig api.example.com +stats  # 통계 포함

# 응답 플래그 이해:
# QR: 응답 (Query Response)
# AA: Authoritative Answer (권한 있는 응답)
# TC: Truncated (메시지가 잘림, UDP 512 bytes 제한)
# RD: Recursion Desired (재귀 요청)
# RA: Recursion Available (재귀 가능)

# UDP vs TCP DNS
dig api.example.com +tcp  # TCP 강제 (512 bytes 초과 시 자동)
```

### 실험 4: JVM DNS 캐시 동작 확인

```bash
# JVM DNS 캐시 확인 방법 (arthas 사용)
java -jar arthas-boot.jar
# ognl "@java.net.InetAddress@getCachedAddresses()"

# 직접 확인 (Java 코드)
cat > TestDnsCache.java << 'EOF'
import java.net.InetAddress;
import java.lang.reflect.Field;
import java.util.concurrent.ConcurrentHashMap;

public class TestDnsCache {
    public static void main(String[] args) throws Exception {
        // DNS 캐시 TTL 설정
        java.security.Security.setProperty("networkaddress.cache.ttl", "5");
        
        System.out.println("First lookup: " + InetAddress.getByName("github.com"));
        Thread.sleep(3000);
        System.out.println("After 3s: " + InetAddress.getByName("github.com"));
        Thread.sleep(3000);
        System.out.println("After 6s (TTL expired): " + InetAddress.getByName("github.com"));
    }
}
EOF
javac TestDnsCache.java && java TestDnsCache

# K8s CoreDNS 확인
kubectl get pod -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns | tail -20
kubectl exec -it some-pod -- cat /etc/resolv.conf
```

---

## 📊 성능/비용 비교

```
DNS 조회 비용:

캐시 없는 완전 조회 (콜드 스타트):
  Recursive Resolver → Root NS: 50~200ms (지리적 거리)
  Recursive Resolver → TLD NS:  20~100ms
  Recursive Resolver → Auth NS: 10~50ms
  총: 80~350ms (첫 번째 조회)

캐시 히트 (Recursive Resolver):
  Resolver → 클라이언트: 1~5ms (같은 ISP/VPC)
  → 99% 이상의 DNS 조회가 여기서 처리됨

DNS 조회 레이어별 캐시:
  브라우저 캐시: 수십ms 이하 (메모리)
  OS 캐시:      1~2ms (메모리)
  로컬 Resolver: 1~5ms (로컬 네트워크)
  ISP Resolver:  1~20ms (ISP 네트워크)
  Root → Auth:  80~350ms (한 번만 발생)

TTL 설정별 트레이드오프:
  TTL=60초:  매 분마다 Resolver 조회 → Authoritative NS 부하
  TTL=300초: 5분 캐시 → 변경 시 최대 5분 전파 지연
  TTL=3600초: 1시간 캐시 → 변경 시 최대 1시간 전파 지연
  TTL=86400초: 24시간 캐시 → 변경 전 미리 낮춰야 함
```

---

## ⚖️ 트레이드오프

```
TTL 설정:

낮은 TTL (60~300초):
  장점: 빠른 DNS 변경 전파, 빠른 장애 복구
  단점: Authoritative NS에 더 많은 쿼리
        Resolver 캐시 미스 증가 → 조회 지연 가능

높은 TTL (3600~86400초):
  장점: Authoritative NS 부하 감소, 캐시 히트율 높음
  단점: DNS 변경이 느리게 전파
        장애 상황에서 롤백이 오래 걸림

권장 전략:
  평상시:    TTL=3600 (1시간) — 안정적 서비스
  마이그레이션 24시간 전: TTL=300 (5분)으로 낮춤
  마이그레이션 완료 후:  다시 TTL=3600으로 높임

Public vs Private DNS:
  Public DNS (Route53 Public):
    인터넷 전체에서 조회 가능
    TTL 낮으면 Route53 쿼리 비용 증가 ($0.40/100만 쿼리)
  
  Private DNS (Route53 Private Hosted Zone):
    VPC 내부에서만 조회 가능
    내부 서비스 도메인 (internal.example.com)
    외부 노출 없음, 보안 강화

Recursive Resolver 선택:
  8.8.8.8/1.1.1.1: 빠르지만 쿼리 로그 (프라이버시 이슈)
  자체 운영:       완전 제어, 설정 복잡
  ISP DNS:        느릴 수 있음, 신뢰도 낮음
  → 중요 인프라: Route53 Resolver 또는 Unbound 자체 운영
```

---

## 📌 핵심 정리

```
DNS 조회 흐름 핵심 요약:

계층 구조:
  Root NS (.) → TLD NS (.com) → Authoritative NS (example.com)
  각 NS는 "모르면 이 서버로"(위임) 방식으로 동작

Recursive vs Iterative:
  Recursive: 클라이언트 요청, Resolver가 알아서 조회
  Iterative: Resolver 내부 동작, 각 NS에 차례로 질의

조회 순서:
  브라우저 캐시 → OS 캐시 → /etc/hosts → Recursive Resolver
  → (캐시 없으면) Root → TLD → Authoritative NS

Glue Record:
  TLD NS에 NS 서버의 A 레코드 함께 저장
  순환 참조 해결

JVM DNS 캐시 (핵심!):
  networkaddress.cache.ttl=30 (기본값 영구 가능)
  보안 매니저 활성화 시 -1 (영구) → 반드시 수정
  K8s: ndots:5, search 도메인 자동 붙임

TTL 전략:
  평상시: 3600초
  변경 전: 300초로 낮춤 (24시간 전)
  변경 후: 안정화 확인 후 다시 높임

진단:
  dig +trace: 전체 조회 경로 확인
  dig @8.8.8.8: 외부 DNS로 직접 조회
  dig +norecurse: 캐시 상태 확인
  cat /etc/resolv.conf: 로컬 Resolver 확인
```

---

## 🤔 생각해볼 문제

**Q1.** K8s Pod 내부에서 `my-service` 라는 이름으로 다른 Service에 접근한다. DNS 조회는 어떤 순서로 이루어지는가?

<details>
<summary>해설 보기</summary>

**K8s Pod의 `/etc/resolv.conf`:**
```
nameserver 10.96.0.10        # CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

**`my-service` 조회 순서 (ndots:5):**

`my-service`는 점이 0개 < 5개이므로 search 도메인을 앞에 붙여 시도합니다.

1. `my-service.default.svc.cluster.local.` → CoreDNS 조회
   - `default` 네임스페이스의 `my-service` Service가 있으면 → ClusterIP 반환 ✅
   - 없으면 NXDOMAIN → 다음 시도

2. `my-service.svc.cluster.local.` → NXDOMAIN
3. `my-service.cluster.local.` → NXDOMAIN
4. `my-service.` (절대 이름) → 외부 DNS 조회

**실무 영향:**
- 같은 네임스페이스의 서비스: `my-service`로 충분
- 다른 네임스페이스: `my-service.other-namespace`로 접근 → `my-service.other-namespace.svc.cluster.local`로 확장됨
- 외부 도메인(`api.example.com`): 5단계 탐색 후 외부 DNS로 나감 → 최대 4번의 불필요한 CoreDNS 쿼리 발생
  - ndots:1로 설정하거나 FQDN 사용(`api.example.com.` 끝에 점)으로 최적화 가능

</details>

---

**Q2.** Root NS는 전 세계에 13개밖에 없는데 어떻게 수십억 개의 DNS 쿼리를 처리하는가?

<details>
<summary>해설 보기</summary>

**13개 클러스터, 수천 개의 물리 서버:**

"13개 Root NS"는 13개의 IP 주소 클러스터입니다. 실제로는 Anycast 기술을 사용하여 수백~수천 개의 물리 서버가 전 세계에 분산됩니다.

**Anycast:**
```
같은 IP 주소 (예: 198.41.0.4)를 여러 서버가 공유
BGP 라우팅: 클라이언트에서 가장 가까운 서버로 자동 라우팅
서울 → 가장 가까운 a.root-servers.net 서버
뉴욕 → 다른 a.root-servers.net 서버
```

**캐싱이 핵심:**
- Root NS의 응답을 Recursive Resolver가 TTL(518400초 = 6일) 동안 캐시
- 같은 Resolver를 사용하는 수백만 명이 같은 캐시 공유
- 실제로 Root NS에 도달하는 쿼리는 전체의 1% 미만

**통계:** a.root-servers.net은 초당 수백만 쿼리를 처리하지만, 대부분은 Resolver 캐시에서 처리되어 Root까지 오지 않습니다. "Root Zone Amplification"이 DDoS 공격에 쓰이는 이유이기도 합니다.

</details>

---

**Q3.** DNS 조회 결과가 `dig @8.8.8.8`과 `dig @1.1.1.1`에서 다른 IP를 반환한다면 무슨 이유인가?

<details>
<summary>해설 보기</summary>

**같은 도메인에서 다른 IP를 반환하는 정상적인 이유:**

1. **GeoDNS / 지역 기반 라우팅:**
   Cloudflare(1.1.1.1)는 미국 서버, Google(8.8.8.8)은 다른 지역 서버
   GeoDNS가 Resolver의 위치에 따라 다른 IP 반환
   → 가장 가까운 CDN 서버로 유도

2. **DNS 로드밸런싱 (Round-Robin):**
   도메인에 여러 A 레코드가 있고, 각 Resolver가 다른 순서로 받음
   ```
   example.com.  A  203.0.113.10
   example.com.  A  203.0.113.11
   ```
   → 8.8.8.8에서는 .10을 먼저, 1.1.1.1에서는 .11을 먼저 받을 수 있음

3. **캐시 시점 차이:**
   두 Resolver가 캐시한 시점이 달라서 배포 중간에 조회
   → TTL이 만료된 Resolver는 새 IP, 미만료된 Resolver는 이전 IP

4. **문제가 있는 경우:**
   DNS 스푸핑(DNS Cache Poisoning): 한쪽 Resolver 캐시가 변조됨
   → DNSSEC으로 방어

**진단:**
```bash
dig +trace @8.8.8.8 example.com  # 전체 경로 확인
dig +trace @1.1.1.1 example.com  # 비교
# Authoritative NS에서 같은 응답 → 정상 (GeoDNS 또는 Round-Robin)
# Authoritative NS에서 다른 응답 → 비정상 (조사 필요)
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: DNS 레코드 완전 가이드 ➡️](./02-dns-records.md)**

</div>
