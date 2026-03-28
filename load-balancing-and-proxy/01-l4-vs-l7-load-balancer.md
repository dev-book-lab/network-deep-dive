# L4 vs L7 로드 밸런서 — 계층별 분산 방식과 한계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- L4 로드 밸런서는 어떤 정보를 보고 어떻게 트래픽을 분산하는가?
- L7 로드 밸런서가 L4보다 할 수 있는 일은 무엇인가?
- DSR(Direct Server Return)은 왜 응답 지연을 줄이는가?
- NAT 방식 L4 LB의 병목은 어디서 발생하는가?
- AWS NLB(L4)와 ALB(L7) 중 어떤 기준으로 선택하는가?
- L4 LB에서 WebSocket이 잘 동작하는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"WebSocket 연결이 ALB에서 90초마다 끊긴다":
  ALB: HTTP 연결을 기반으로 하는 L7 LB
  ALB 기본 idle timeout = 60초
  WebSocket은 HTTP Upgrade 후 지속 연결
  → ALB가 idle로 판단하고 연결 종료
  → 해결: ALB idle_timeout 증가 또는 WebSocket heartbeat
  
  NLB(L4) 사용 시: TCP 레벨에서 처리 → idle timeout 더 긺

"100만 동시 연결을 처리하는 LB":
  L7 LB(Nginx): 각 연결마다 HTTP 파싱 → CPU/메모리 많이 사용
  L4 LB(NLB): TCP 헤더만 보고 패킷 포워딩 → 더 적은 리소스
  → 순수 대용량 TCP: NLB
  → URL 라우팅, 인증, TLS 종단: ALB

선택 기준:
  gRPC 서비스: L4 NLB (HTTP/2 프로토콜 그대로 통과)
  마이크로서비스 API: L7 ALB (URL 기반 라우팅)
  데이터베이스 프록시: L4 NLB (TCP 그대로)
  정적 파일 서빙: L7 ALB (캐싱, 압축)
```

---

## 😱 흔한 실수

```
Before — LB 계층을 모를 때:

실수 1: L7 ALB를 gRPC에 그냥 사용
  gRPC: HTTP/2 기반, 단일 연결에 여러 스트림
  ALB: HTTP/1.1 또는 HTTP/2 지원하지만
       gRPC 프로토콜 특성 상 스트리밍에서 문제 가능
  → gRPC 전용 설정 필요 (Protocol = gRPC)
  → 또는 NLB + Envoy 조합

실수 2: L4 LB에서 IP 기반 Sticky Session 기대
  L4 LB: IP:Port → 백엔드 결정 (기본 라운드로빈)
  IP Hash 방식: 소스 IP → 같은 백엔드로
  문제: NAT 뒤 클라이언트 → 같은 소스 IP → 한 서버에 몰림
  → L7 쿠키 기반 Session Affinity가 더 균등

실수 3: L4 LB에서 SSL 인증서 설치
  L4 LB: TLS 파싱 안 함 → 인증서 설치 불가
  → TLS 종단(Termination)은 L7 LB 또는 백엔드 서버
  → L4 LB에서 TLS 패스스루(Pass-through)만 가능

실수 4: 헬스 체크 포트를 서비스 포트와 분리 안 함
  L4 LB: TCP 포트 연결로 헬스 체크
  서비스 포트 8080이 응답해도 앱이 실제로 안 동작할 수 있음
  → L7 LB: HTTP 200 응답 확인 → 더 정확한 헬스 체크
```

---

## ✨ 올바른 접근

```
After — LB 계층을 알면:

서비스 유형별 LB 선택:
  REST API (HTTP/HTTPS):
    ALB (L7): URL 기반 라우팅, 인증, 헤더 조작
  
  gRPC:
    ALB (L7 + gRPC): AWS ALB는 gRPC 지원
    또는 NLB (L4) + Nginx/Envoy (L7) 내부 처리
  
  WebSocket:
    ALB (L7): idle_timeout 증가 필요
    NLB (L4): TCP 그대로 통과, 더 단순
  
  데이터베이스/Cache:
    NLB (L4): MySQL, Redis, MongoDB → TCP 그대로
  
  고성능 TCP (게임, 스트리밍):
    NLB (L4): 낮은 지연, 높은 처리량

AWS ALB 설정 (URL 기반 라우팅):
  리스너 규칙:
    /api/users*  → UserService Target Group
    /api/orders* → OrderService Target Group
    /api/products* → ProductService Target Group
    기본값       → FrontendService Target Group

AWS NLB 설정:
  리스너: TCP 443 → Target Group (TLS 패스스루)
  또는 TLS 443 → Target Group (NLB에서 TLS 종단)
  헬스 체크: TCP 연결 또는 HTTP (TLS 종단 시)
```

---

## 🔬 내부 동작 원리

### 1. L4 로드 밸런서 — TCP/IP 레벨 분산

```
L4 LB 동작 (NAT 방식):

클라이언트: 203.0.113.100
LB IP:      10.0.0.1 (VIP: Virtual IP)
서버 A:     10.0.1.10:8080
서버 B:     10.0.1.11:8080

클라이언트 → LB:
  SRC: 203.0.113.100:54321
  DST: 10.0.0.1:443 (VIP)

LB → 서버 A (NAT):
  SRC: 10.0.0.1:54321      ← Destination NAT (DNAT)
  DST: 10.0.1.10:8080      ← 패킷 목적지 변경

서버 A → LB:
  SRC: 10.0.1.10:8080
  DST: 10.0.0.1:54321

LB → 클라이언트 (역NAT):
  SRC: 10.0.0.1:443        ← 클라이언트는 서버 A IP를 모름
  DST: 203.0.113.100:54321

L4 LB가 보는 정보:
  - 소스 IP:Port
  - 목적지 IP:Port (VIP)
  - TCP 플래그 (SYN, ACK, FIN...)
  → HTTP 헤더, URL, 쿠키: 보지 않음 (파싱 안 함)

분산 알고리즘:
  라운드로빈: 순서대로 서버 선택
  Least Connections: 현재 연결 수 가장 적은 서버
  IP Hash: 소스 IP 해시 → 같은 IP는 같은 서버
  Random: 무작위 선택

L4 LB의 장점:
  빠름: IP 헤더만 보고 결정 → CPU 부하 낮음
  프로토콜 무관: TCP/UDP 모두 처리
  TLS 그대로 통과: 백엔드에서 TLS 처리
  
L4 LB의 한계:
  URL, 쿠키, 헤더 기반 라우팅 불가
  HTTP 레벨 헬스 체크 어려움 (TCP 연결만 확인)
  WebSocket: 지원하지만 L7 기능 없음
```

### 2. L7 로드 밸런서 — HTTP 레벨 분산

```
L7 LB 동작:

클라이언트 ──TLS+HTTP──▶ L7 LB ──HTTP/gRPC──▶ 백엔드

L7 LB가 하는 일:
  1. TLS 종단 (클라이언트와 TLS 핸드쉐이크)
  2. HTTP 요청 파싱 (메서드, URL, 헤더, 쿠키)
  3. 라우팅 결정
  4. 백엔드에 새 HTTP 요청 전송
  → 두 개의 독립 연결 유지

L7 LB가 볼 수 있는 정보:
  HTTP 메서드: GET, POST, PUT...
  URL 경로: /api/users, /api/orders...
  HTTP 헤더: Host, Authorization, User-Agent...
  쿠키: session_id, preferences...
  요청 바디 (일부): Content-Type 등

L7 라우팅 예시:
  요청: GET /api/users/123 HTTP/1.1
        Host: api.example.com
        X-Version: v2
  
  규칙 1: /api/users/* → UserService 그룹
  규칙 2: X-Version: v2 → v2 버전 그룹
  규칙 3: /api/orders/* → OrderService 그룹
  → UserService v2 그룹으로 라우팅

L7 LB만 가능한 기능:
  ① URL 기반 라우팅
  ② 헤더 기반 라우팅 (A/B 테스트, 버전 관리)
  ③ 쿠키 기반 Sticky Session
  ④ HTTP 레벨 헬스 체크 (200 OK 확인)
  ⑤ TLS 종단 + 내부 평문 통신
  ⑥ HTTP 헤더 추가/제거 (X-Forwarded-For 등)
  ⑦ 요청/응답 변환
  ⑧ WAF(Web Application Firewall) 통합
  ⑨ 액세스 로그 (URL, 응답 시간 포함)
  ⑩ 압축(gzip), 캐싱

연결 흐름 비교:
  L4: 클라이언트↔LB 연결 = 서버↔LB 연결 (TCP 레벨 그대로)
  L7: 클라이언트↔LB 연결 ≠ LB↔서버 연결 (완전히 분리)
      → 클라이언트가 느려도 서버는 빠르게 응답 후 연결 끊기 가능
```

### 3. DSR(Direct Server Return)

```
문제: NAT LB의 응답 트래픽도 LB를 거침

일반 NAT LB 흐름:
  클라이언트 → LB → 서버
  서버 → LB → 클라이언트  ← 응답도 LB 통과!
  
  웹 서비스: 요청 1KB, 응답 1MB (비대칭)
  LB: 들어오는 1KB + 나가는 1MB 처리
  → 응답 트래픽이 LB 병목

DSR (Direct Server Return):
  클라이언트 → LB → 서버 (요청만 LB 통과)
  서버 → 클라이언트 (응답은 LB 우회, 서버가 직접!)
  
  어떻게 가능한가:
    서버에 VIP(LB의 공인 IP)를 루프백 인터페이스에 설정
    클라이언트는 VIP로 요청 → LB가 서버로 전달 (DST MAC 변경)
    서버: "이 패킷의 목적지(VIP)는 내 루프백 IP" → 응답은 직접 전송
    클라이언트: 응답 SRC = VIP → LB에서 온 것처럼 보임

  DSR 장점:
    응답 트래픽 LB 우회 → LB 처리량 80~90% 감소
    대역폭 집약적인 서비스에 필수 (동영상 스트리밍 등)
  
  DSR 한계:
    L4에서만 사용 가능 (MAC 주소 조작 기반)
    서버와 LB가 같은 L2 네트워크에 있어야 함
    서버 설정 필요 (루프백에 VIP 추가)
    NAT 없음 → 서버가 클라이언트 IP 직접 볼 수 있음

AWS NLB + DSR (Proxy Protocol):
  NLB: 클라이언트 IP를 그대로 서버에 전달 가능
  서버: 클라이언트 IP를 직접 확인 가능
  (ALB: X-Forwarded-For 헤더로 전달)
```

### 4. 헬스 체크 비교

```
L4 헬스 체크 (TCP 연결):
  LB → 서버: TCP SYN
  서버 → LB: TCP SYN-ACK (연결 가능)
  LB: "서버 살아있음"
  
  문제:
  서버 포트 응답 O, 하지만 앱 로직 다운
  DB 연결 실패, 메모리 부족 등 → TCP 연결은 OK
  → L4 헬스 체크는 "포트가 열려있는가"만 확인

L7 헬스 체크 (HTTP 응답):
  LB → 서버: GET /health HTTP/1.1
  서버 → LB: HTTP 200 OK {"status": "healthy"}
  LB: "서버 정상"
  
  더 정확한 헬스 체크:
  Spring Boot Actuator:
    /actuator/health → {"status": "UP", "components": {...}}
    DB 연결, 외부 서비스 상태까지 확인
  
  커스텀 헬스 체크:
    @GetMapping("/health")
    public ResponseEntity<Map<String,String>> health() {
        boolean dbOk = checkDatabase();
        boolean cacheOk = checkRedis();
        if (dbOk && cacheOk) {
            return ResponseEntity.ok(Map.of("status", "healthy"));
        }
        return ResponseEntity.status(503)
            .body(Map.of("status", "unhealthy"));
    }
    
    → LB: 503 응답 시 해당 서버로 트래픽 차단

헬스 체크 설정 (AWS ALB):
  프로토콜: HTTP
  경로: /actuator/health
  정상 코드: 200
  간격: 30초
  비정상 임계값: 3회 (90초 내 3번 실패 시 제외)
  정상 임계값: 2회 (60초 내 2번 성공 시 복구)
```

### 5. 알고리즘 상세

```
Least Connections (최소 연결):
  현재 활성 연결 수가 가장 적은 서버 선택
  
  서버 A: 100 연결
  서버 B: 50 연결  ← 새 요청 여기로
  서버 C: 75 연결
  
  적합: 처리 시간이 다양한 요청 (DB 쿼리 등)
  부적합: 연결 수 != 처리 부하인 경우

Weighted Round Robin:
  서버별 가중치로 분산
  서버 A (weight=3): 3번
  서버 B (weight=1): 1번
  → A:A:A:B:A:A:A:B... 순서
  
  적합: 서버 사양이 다를 때 (큰 서버에 더 많은 트래픽)

IP Hash:
  소스 IP → 해시 → 서버 선택
  같은 IP → 항상 같은 서버
  
  문제: NAT 뒤 여러 클라이언트 → 같은 IP → 쏠림
  → CGN(Carrier Grade NAT): 수천 명이 같은 IP

Consistent Hashing:
  서버가 추가/제거될 때 영향받는 요청 최소화
  해시 링에 서버와 요청 배치
  서버 추가: 해당 서버 담당 범위만 재배분
  분산 캐시(Memcached, Redis Cluster)에서 사용
```

---

## 💻 실전 실험

### 실험 1: L4 vs L7 동작 차이 확인

```bash
# L4 NLB 동작 확인 (소스 IP 보존)
# (서버 측에서 접속 IP 확인)
curl http://nlb.example.com/myip
# 실제 클라이언트 IP 반환 (X-Forwarded-For 없이)

# L7 ALB 동작 확인 (X-Forwarded-For 헤더)
curl -v http://alb.example.com/myip 2>&1 | grep "X-Forwarded"
# X-Forwarded-For: 203.0.113.100 (실제 클라이언트 IP)
# X-Forwarded-Proto: https (원래 프로토콜)
# X-Forwarded-Port: 443

# Nginx에서 실제 클라이언트 IP 읽기
set_real_ip_from  10.0.0.0/8;  # LB의 IP 범위
real_ip_header    X-Forwarded-For;
# $remote_addr이 실제 클라이언트 IP로 설정됨
```

### 실험 2: 헬스 체크 시뮬레이션

```bash
# Spring Boot Actuator 헬스 체크
curl http://localhost:8080/actuator/health
# {"status":"UP","components":{"db":{"status":"UP"},"redis":{"status":"UP"}}}

# 헬스 체크 엔드포인트 응답 시간 측정
curl -w "Response: %{time_total}s Status: %{http_code}\n" \
     -o /dev/null -s http://localhost:8080/actuator/health

# 헬스 체크가 응답하는 동안 서버가 503 반환하는 경우
# → LB가 해당 서버 제외 확인
for i in $(seq 1 10); do
  curl -o /dev/null -s -w "%{http_code} " http://alb.example.com/api/test
done
# 정상 서버들만 200 반환 확인
```

### 실험 3: URL 기반 라우팅 (L7)

```bash
# ALB URL 라우팅 테스트
curl http://alb.example.com/api/users/123
# → UserService로 라우팅 확인

curl http://alb.example.com/api/orders/456
# → OrderService로 라우팅 확인

# 헤더 기반 라우팅 (A/B 테스트)
curl -H "X-AB-Test: B" http://alb.example.com/api/feature
# → B 버전 서버로 라우팅

# Nginx로 URL 기반 라우팅
cat > nginx-l7.conf << 'EOF'
upstream users { server user-service:8080; }
upstream orders { server order-service:8080; }

server {
    location /api/users/ { proxy_pass http://users; }
    location /api/orders/ { proxy_pass http://orders; }
    location / { return 404; }
}
EOF
```

### 실험 4: 연결 분산 확인

```bash
# 각 백엔드 서버의 연결 수 확인 (Nginx 상태 페이지)
curl http://nginx/nginx_status
# Active connections: 150
# server accepts handled requests
#  1000 1000 5000
# Reading: 10 Writing: 50 Waiting: 90

# 백엔드별 요청 분산 확인 (로그 분석)
tail -f /var/log/nginx/access.log | grep "upstream_addr"
# upstream_addr: 10.0.1.10:8080 (서버 A)
# upstream_addr: 10.0.1.11:8080 (서버 B)
# 분산 비율 확인
awk '{print $NF}' /var/log/nginx/access.log | sort | uniq -c
```

---

## 📊 성능/비용 비교

```
L4 vs L7 성능 비교:

┌─────────────────────────────────────────────────────────────────────┐
│  항목             │  L4 (NLB)          │  L7 (ALB)                   │
├─────────────────────────────────────────────────────────────────────┤
│  처리량           │  수백만 rps          │  수십만 rps                   │
│  지연 추가         │  ~0.1ms            │  1~10ms (파싱 비용)           │
│  CPU 사용량       │  매우 낮음            │  높음 (HTTP 파싱)             │
│  TLS 종단         │  가능 (별도 설정)      │  기본 지원                    │
│  라우팅 기준        │  IP:Port만          │  URL, 헤더, 쿠키 등           │
│  헬스 체크         │  TCP 연결            │  HTTP 200 응답              │
│  로깅             │  TCP 레벨            │  HTTP 레벨 (URL, 상태)       │
│  AWS 비용         │  더 저렴             │  약간 더 비쌈                 │
└─────────────────────────────────────────────────────────────────────┘

AWS 비용 예시 (us-east-1):
  NLB: $0.008/LCU/h + $0.0225/h = ~$25/월 (기본)
  ALB: $0.008/LCU/h + $0.0225/h = ~$25/월 (기본)
  → 기본 비용은 비슷, LCU(Load Balancer Capacity Unit) 사용량이 다름
  → NLB: 네트워크 처리량 기반 / ALB: 요청 수 + 처리량 기반
```

---

## ⚖️ 트레이드오프

```
L4 선택 시나리오:
  ✅ 낮은 지연 필요 (게임, 실시간 앱)
  ✅ 비HTTP 프로토콜 (MySQL, Redis, SMTP)
  ✅ 고성능 TCP (수백만 rps)
  ✅ gRPC with connection-level load balancing
  ✅ 클라이언트 IP 직접 확인 필요
  ❌ URL 기반 라우팅 필요 → ALB 사용

L7 선택 시나리오:
  ✅ URL/헤더 기반 마이크로서비스 라우팅
  ✅ TLS 종단 + 내부 평문 (단순 인증서 관리)
  ✅ 쿠키 기반 Sticky Session
  ✅ A/B 테스트, 카나리 배포
  ✅ WAF, 인증, 요청 변환
  ✅ 상세 HTTP 액세스 로그
  ❌ 매우 낮은 지연 필요 → NLB 고려

혼합 아키텍처 (흔한 패턴):
  인터넷 → NLB (L4) → Nginx (L7 프록시) → 백엔드
  
  NLB: 외부 인터넷 → 내부 Nginx
  Nginx: URL 라우팅, 캐싱, 압축, 로깅 (L7)
  → 두 계층의 장점 결합
```

---

## 📌 핵심 정리

```
L4 vs L7 로드 밸런서 핵심 요약:

L4 (Transport Layer):
  보는 것: IP, Port, TCP 플래그만
  동작: DNAT로 패킷 목적지 변경 → 서버로 전달
  장점: 빠름, 낮은 CPU, 프로토콜 무관
  단점: URL/헤더 라우팅 불가, HTTP 헬스 체크 어려움
  예: AWS NLB, HAProxy (TCP 모드)

L7 (Application Layer):
  보는 것: HTTP 메서드, URL, 헤더, 쿠키
  동작: 두 개의 독립 TCP 연결 (클라이언트↔LB, LB↔서버)
  장점: 세밀한 라우팅, TLS 종단, 헬스 체크 정확
  단점: HTTP 파싱 비용, 지연 추가
  예: AWS ALB, Nginx, Envoy

DSR (Direct Server Return):
  응답 트래픽이 LB를 우회
  LB 병목 완화 (응답이 요청보다 큰 서비스)
  L4에서만 사용 가능, L2 네트워크 제약

헬스 체크:
  L4: TCP 연결 → 포트 열림 여부만
  L7: HTTP 200 → 앱 로직 정상 여부 확인
  Spring Boot: /actuator/health 엔드포인트 활용

선택 기준:
  비HTTP 프로토콜, 낮은 지연 → L4 (NLB)
  URL 라우팅, TLS 종단, WAF → L7 (ALB)
  혼합 필요 → NLB + Nginx(L7)
```

---

## 🤔 생각해볼 문제

**Q1.** gRPC 서비스에 L7 ALB를 사용할 때 어떤 문제가 발생하고 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**gRPC의 특성:**
gRPC는 HTTP/2 기반으로, 하나의 TCP 연결에 여러 RPC 스트림을 멀티플렉싱합니다.

**L7 ALB(또는 L4 LB)의 문제:**

**L4 수준에서 로드밸런싱 시:**
- 클라이언트가 LB에 하나의 TCP 연결 → LB가 하나의 백엔드 서버로 매핑
- 그 연결에 있는 모든 gRPC 요청이 같은 서버로 → 불균등 분산

```
클라이언트 → LB → 서버 A (HTTP/2 연결 1개, 100 RPC)
             LB → 서버 B (연결 없음, 0 RPC)
```

**해결 방법:**

1. **Proxy Protocol 레벨에서 gRPC 인식 (L7 gRPC 로드밸런싱):**
```nginx
# Nginx gRPC 로드밸런싱
upstream grpc_backend {
    server 10.0.1.10:9090;
    server 10.0.1.11:9090;
}

server {
    listen 443 ssl http2;
    location / {
        grpc_pass grpc://grpc_backend;
    }
}
```
→ Nginx가 gRPC 스트림 레벨에서 분산 (RPC 단위로 다른 서버)

2. **Envoy/Istio 사이드카:**
- 클라이언트 사이드에 Envoy 프록시 배포
- Envoy가 gRPC 메서드 레벨에서 로드밸런싱

3. **AWS ALB gRPC 지원:**
- ALB의 프로토콜 버전을 "gRPC"로 설정
- gRPC 메서드 기반 라우팅 가능

**실무 권장:**
- 소규모: Nginx gRPC 프록시
- 쿠버네티스: Istio/Linkerd 서비스 메시
- AWS: ALB + gRPC 지원 활성화

</details>

---

**Q2.** ALB가 클라이언트와 백엔드 사이에서 "두 개의 독립 TCP 연결"을 유지한다는 것이 실제로 어떤 이점을 제공하는가?

<details>
<summary>해설 보기</summary>

**두 개의 독립 연결:**
```
클라이언트 ──[연결 A]── ALB ──[연결 B]── 백엔드
```

**이점 1 — 느린 클라이언트로부터 백엔드 보호:**
- 클라이언트가 2G 네트워크 → 응답을 천천히 받음
- 백엔드: ALB에게 빠르게 응답하고 연결 해제 → 다음 요청 처리
- ALB: 백엔드 응답을 버퍼링 → 느린 클라이언트에게 천천히 전달
- 백엔드 연결 점유 시간 = 실제 처리 시간 (클라이언트 속도 무관)

**이점 2 — Connection Pool:**
- ALB → 백엔드: Keep-Alive로 연결 재사용
- 클라이언트: 매번 새 연결 열어도 ALB가 기존 백엔드 연결 재사용
- 백엔드 TLS 핸드쉐이크 비용 절감

**이점 3 — 프로토콜 변환:**
- 클라이언트 → ALB: HTTPS (TLS + HTTP/1.1)
- ALB → 백엔드: HTTP (평문, 내부 네트워크)
- 백엔드가 TLS 처리 없이 빠르게 응답

**이점 4 — 요청 재시도:**
- 백엔드 서버 응답 없음 → ALB가 다른 서버에 재시도
- 클라이언트는 재시도 인식 못함 (투명한 재시도)

**단점:**
- 지연: 두 연결 관리 오버헤드
- 메모리: 두 연결 상태 유지
- 복잡성: 클라이언트 IP가 백엔드에 직접 전달 안 됨 (X-Forwarded-For 필요)

</details>

---

**Q3.** 하나의 서버에 배포된 서비스가 CPU 100%로 포화 상태인데 헬스 체크는 200을 반환한다. LB는 계속 트래픽을 보낸다. 어떻게 이를 방지하는가?

<details>
<summary>해설 보기</summary>

**문제:** 서버가 과부하 상태이지만 헬스 체크 엔드포인트는 가볍게 처리되어 200을 반환.

**방법 1 — CPU 사용률 헬스 체크 통합:**
```java
@GetMapping("/health")
public ResponseEntity<Map<String,Object>> health() {
    OperatingSystemMXBean osBean = ManagementFactory
        .getPlatformMXBean(OperatingSystemMXBean.class);
    double cpuLoad = osBean.getSystemCpuLoad();
    
    if (cpuLoad > 0.95) {  // CPU 95% 이상
        return ResponseEntity.status(503)
            .body(Map.of("status", "overloaded", "cpu", cpuLoad));
    }
    return ResponseEntity.ok(Map.of("status", "healthy", "cpu", cpuLoad));
}
```

**방법 2 — Actuator + 커스텀 HealthIndicator:**
```java
@Component
public class CpuHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        double cpu = getCpuLoad();
        if (cpu > 0.9) {
            return Health.down()
                .withDetail("cpu", cpu)
                .withDetail("message", "CPU overloaded")
                .build();
        }
        return Health.up().withDetail("cpu", cpu).build();
    }
}
```

**방법 3 — 큐 길이/스레드 풀 기반:**
```java
// 스레드 풀 포화 감지
ThreadPoolExecutor executor = (ThreadPoolExecutor) threadPool;
int queueSize = executor.getQueue().size();
if (queueSize > 1000) {
    return ResponseEntity.status(503).body("Queue full");
}
```

**방법 4 — 서킷 브레이커 + LB 통합:**
Resilience4j 서킷 브레이커가 열리면 → `/health` 엔드포인트에서 503 반환 → LB가 트래픽 차단

**Best Practice:**
헬스 체크를 단순 "살아있는가"가 아닌 "요청을 처리할 수 있는가"로 설계. Spring Boot의 `management.endpoint.health.show-details=always` 설정으로 세부 정보 노출.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: 리버스 프록시와 Nginx ➡️](./02-reverse-proxy-nginx.md)**

</div>
