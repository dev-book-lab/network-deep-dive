# NAT와 포트 포워딩 — 사설 IP가 인터넷에 나가는 방법

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 왜 사설 IP(192.168.x.x, 10.x.x.x)로는 직접 인터넷에 나갈 수 없는가?
- NAPT(Network Address and Port Translation)는 어떻게 여러 기기가 하나의 공인 IP를 공유하는가?
- NAT 테이블은 어떻게 응답 패킷을 올바른 내부 호스트로 되돌려 보내는가?
- 포트 포워딩(인바운드 NAT)과 SNAT(아웃바운드 NAT)의 차이는?
- Docker의 `-p 8080:80` 옵션이 내부적으로 어떻게 동작하는가?
- NAT가 TCP 연결에 미치는 영향과 Connection Tracking이란?

---

## 🔍 왜 이 개념이 중요한가

### NAT를 모르면 Docker, Kubernetes, 클라우드 네트워킹이 블랙박스가 된다

```
NAT를 모를 때 겪는 문제:

  상황 1: Docker 컨테이너 포트 노출
    docker run -p 8080:80 nginx
    → "왜 호스트의 8080으로 접근하면 컨테이너의 80에 연결되죠?"
    → NAT를 모르면 설명 불가

  상황 2: 외부에서 서버가 보이지 않음
    집에서 서버 실행 → 공인 IP로 접근 안 됨
    → 공유기의 포트 포워딩이 없기 때문
    → NAT를 모르면 왜 안 되는지 모름

  상황 3: Cloud NAT, VPC 설정
    AWS EC2: 인스턴스에 공인 IP 없이 인터넷 나가기
    → NAT Gateway 필요
    → 왜 필요한지, 어떻게 동작하는지 모름

  상황 4: Connection Tracking 문제
    방화벽에서 나가는 트래픽만 허용했는데
    → 응답도 허용해야 하는가?
    → Connection Tracking을 알면: 응답은 자동으로 허용됨
```

---

## 🔬 내부 동작 원리

### 1. 사설 IP와 공인 IP

```
IP 주소 공간 분류:

공인 IP (Public IP):
  인터넷에서 유일한 주소
  라우팅 가능 (전 세계 라우터가 경로를 알고 있음)
  ISP가 할당

사설 IP (Private IP) — RFC 1918:
  ┌──────────────────────────────────────────────────────┐
  │  범위                    │  클래스   │  주소 수          │
  ├──────────────────────────────────────────────────────┤
  │  10.0.0.0/8             │  A      │  16,777,216      │
  │  172.16.0.0/12          │  B      │  1,048,576       │
  │  192.168.0.0/16         │  C      │  65,536          │
  └──────────────────────────────────────────────────────┘
  인터넷에서 라우팅 불가 (라우터가 경로를 모름)
  조직 내부에서만 사용
  여러 조직이 같은 주소 범위를 독립적으로 사용 가능

사설 IP가 인터넷에 나갈 수 없는 이유:
  인터넷 라우터: 192.168.x.x로 오는 패킷 → 어디로 보내야 하는지 모름
  인터넷 라우터의 라우팅 테이블에 192.168.0.0/16 경로 없음
  → 응답 패킷이 집으로 돌아올 방법이 없음

  해결책: NAT
  사설 IP를 공인 IP로 변환해서 인터넷에 나가게 함
```

### 2. SNAT (Source NAT) — 아웃바운드

```
NAPT (Network Address and Port Translation) 동작:

환경:
  내부 PC A:  192.168.1.10
  내부 PC B:  192.168.1.11
  공유기:     내부 192.168.1.1 / 외부(공인) 1.2.3.4
  외부 서버:  8.8.8.8:53 (Google DNS)

PC A가 Google DNS에 쿼리 전송:
─────────────────────────────────────────────────────────────────────
Step 1: PC A → 공유기
  패킷: SrcIP=192.168.1.10, SrcPort=54321, DstIP=8.8.8.8, DstPort=53

Step 2: 공유기가 NAT 변환 (SNAT)
  원본:    SrcIP=192.168.1.10, SrcPort=54321
  변환 후: SrcIP=1.2.3.4,      SrcPort=60001  ← 공인 IP + 새 포트

  NAT 테이블에 매핑 기록:
  ┌─────────────────────────────────────────────────────────────────┐
  │  내부 IP:Port          │  외부 IP:Port    │  목적지 IP:Port         │
  ├─────────────────────────────────────────────────────────────────┤
  │  192.168.1.10:54321   │  1.2.3.4:60001  │  8.8.8.8:53           │
  └─────────────────────────────────────────────────────────────────┘

Step 3: 변환된 패킷 → 인터넷 전송
  패킷: SrcIP=1.2.3.4, SrcPort=60001, DstIP=8.8.8.8, DstPort=53
  → 외부 서버는 1.2.3.4:60001에서 요청이 왔다고 인식

Step 4: 외부 서버 응답
  패킷: SrcIP=8.8.8.8, SrcPort=53, DstIP=1.2.3.4, DstPort=60001

Step 5: 공유기가 NAT 역변환
  DstIP:Port = 1.2.3.4:60001 → NAT 테이블 조회 → 192.168.1.10:54321
  변환 후: DstIP=192.168.1.10, DstPort=54321

Step 6: PC A에게 전달
  패킷: SrcIP=8.8.8.8, SrcPort=53, DstIP=192.168.1.10, DstPort=54321
─────────────────────────────────────────────────────────────────────

PC B가 동시에 같은 서버에 요청:
  NAT 테이블 추가:
  192.168.1.11:54321 → 1.2.3.4:60002 (다른 포트 할당)
  
  외부에서는 1.2.3.4:60001과 1.2.3.4:60002가 다른 연결
  → 같은 공인 IP로 수천 개의 내부 호스트가 동시 접속 가능
  → 포트 범위 최대 65535개 → 동시 외부 연결 최대 ~65535개
```

### 3. 포트 포워딩 (DNAT) — 인바운드

```
DNAT (Destination NAT) — 외부에서 내부로:

목적: 외부 인터넷에서 내부 사설 IP 서버에 접근 가능하게 함

환경:
  공유기 공인 IP: 1.2.3.4
  내부 웹서버:    192.168.1.100:8080
  포트 포워딩 설정: 외부 1.2.3.4:80 → 내부 192.168.1.100:8080

외부 클라이언트(5.6.7.8)가 내 서버에 접근:
─────────────────────────────────────────────────────────────────────
Step 1: 클라이언트 → 공유기
  패킷: SrcIP=5.6.7.8, SrcPort=12345, DstIP=1.2.3.4, DstPort=80

Step 2: 공유기가 DNAT 변환
  DstIP=1.2.3.4, DstPort=80 → 포트 포워딩 규칙 조회
  변환 후: DstIP=192.168.1.100, DstPort=8080

Step 3: 내부 서버로 전달
  패킷: SrcIP=5.6.7.8, SrcPort=12345, DstIP=192.168.1.100, DstPort=8080

Step 4: 서버 응답
  패킷: SrcIP=192.168.1.100, SrcPort=8080, DstIP=5.6.7.8, DstPort=12345

Step 5: 공유기가 역변환 (응답 패킷)
  SrcIP=192.168.1.100, SrcPort=8080 → 1.2.3.4:80로 변환
  패킷: SrcIP=1.2.3.4, SrcPort=80, DstIP=5.6.7.8, DstPort=12345
─────────────────────────────────────────────────────────────────────

포트 포워딩 규칙:
  외부포트  →  내부 IP:포트
  80       →  192.168.1.100:8080
  443      →  192.168.1.100:8443
  3306     →  192.168.1.200:3306  (DB 직접 노출 — 보안 위험!)
```

### 4. iptables로 NAT 구현 (Linux)

```
Linux iptables NAT 체인:

패킷 흐름과 iptables 체인:

외부 → 내부 (인바운드 DNAT):
  외부 패킷 도착
    → PREROUTING 체인 (DNAT 적용: 목적지 IP:Port 변환)
    → 라우팅 결정
    → FORWARD 체인 (패킷 통과 허용 여부)
    → 내부 호스트로 전달

내부 → 외부 (아웃바운드 SNAT):
  내부 패킷 생성
    → FORWARD 체인
    → POSTROUTING 체인 (SNAT/MASQUERADE: 출발지 IP:Port 변환)
    → 외부로 전송

iptables 규칙 확인:
  # NAT 규칙 확인
  sudo iptables -t nat -L -n -v

  # 출력 예시 (Docker 실행 중):
  Chain PREROUTING (policy ACCEPT)
  target  prot  opt  in    out   source      destination
  DOCKER  all   --   any   any   anywhere    anywhere     ADDRTYPE match dst-type LOCAL
  
  Chain POSTROUTING (policy ACCEPT)
  target      prot  opt  in    out        source          destination
  MASQUERADE  all   --   any   !docker0   172.17.0.0/16   anywhere

  Chain DOCKER (2 references)
  target      prot  opt  in       out    source    destination
  DNAT        tcp   --   !docker0 any    anywhere  anywhere  tcp dpt:8080 to:172.17.0.2:80

해석:
  MASQUERADE: 172.17.0.0/16(Docker 컨테이너)에서 나가는 트래픽의 SrcIP를
              호스트의 공인 IP로 변환 (동적 SNAT)
  
  DNAT: 호스트의 8080 포트로 오는 TCP를 172.17.0.2:80으로 전달
        → docker run -p 8080:80이 이 규칙 생성
```

### 5. Docker 포트 매핑의 실제 동작

```
docker run -p 8080:80 nginx

이 명령이 만드는 것:

1. 컨테이너 생성:
   컨테이너 IP: 172.17.0.2 (Docker bridge 네트워크)
   컨테이너 내부 포트: 80 (nginx가 리슨)

2. iptables DNAT 규칙 생성:
   호스트의 모든 IP:8080으로 오는 TCP → 172.17.0.2:80으로 전달
   
   # 실제 규칙 (docker inspect로 확인):
   -A DOCKER ! -i docker0 -p tcp -m tcp --dport 8080 -j DNAT
             --to-destination 172.17.0.2:80

3. iptables MASQUERADE 규칙 (이미 존재):
   172.17.0.0/16에서 나가는 트래픽 → 호스트 IP로 SNAT

패킷 흐름 (외부 클라이언트 → 컨테이너):
  클라이언트:5.6.7.8:12345 → 호스트:1.2.3.4:8080
  DNAT 적용 → 172.17.0.2:80
  컨테이너가 응답: 172.17.0.2:80 → 5.6.7.8:12345
  MASQUERADE 적용 → 1.2.3.4:8080 → 5.6.7.8:12345

컨테이너에서 외부로 나가는 패킷:
  172.17.0.2:랜덤포트 → 8.8.8.8:53
  MASQUERADE → 1.2.3.4:랜덤포트 → 8.8.8.8:53

NAT 테이블 확인 (Connection Tracking):
  sudo conntrack -L
  # tcp 6 114 ESTABLISHED src=5.6.7.8 dst=1.2.3.4 sport=12345 dport=8080
  #   [UNREPLIED] src=172.17.0.2 dst=5.6.7.8 sport=80 dport=12345 ...
```

### 6. Connection Tracking (Conntrack)

```
Connection Tracking:
  NAT의 핵심 메커니즘
  각 연결의 상태를 추적해서 응답 패킷이 자동으로 역변환됨

Conntrack 테이블 구조:
  ┌─────────────────────────────────────────────────────────────────┐
  │  원본 방향      │  응답 방향            │  상태         │  TTL        │
  ├─────────────────────────────────────────────────────────────────┤
  │  src=192.168.1.10:54321 → dst=8.8.8.8:53                        │
  │  [변환됨: src=1.2.3.4:60001]         │  ESTABLISHED │  118s       │
  └─────────────────────────────────────────────────────────────────┘

Connection Tracking 상태:
  NEW:         첫 번째 패킷 (SYN) — 연결 추적 시작
  ESTABLISHED: 양방향 트래픽 확인 — 연결 성립
  RELATED:     기존 연결과 관련된 새 연결 (FTP 데이터 채널 등)
  INVALID:     유효하지 않은 패킷 — 드롭
  TIME_WAIT:   연결 종료 후 대기

방화벽과 Connection Tracking의 관계:
  iptables 규칙:
  -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
  -A INPUT -m state --state NEW -p tcp --dport 80 -j ACCEPT
  -A INPUT -j DROP

  의미:
  - ESTABLISHED: 이미 허용된 연결의 응답 패킷 → 자동 허용
  - NEW, port 80: 새 HTTP 연결 → 허용
  - 그 외: 모두 차단

  → 인바운드 80을 허용하면 아웃바운드 응답은 별도 규칙 없이 허용됨
  → Conntrack이 응답을 ESTABLISHED로 인식하기 때문

Conntrack 테이블 용량:
  cat /proc/sys/net/netfilter/nf_conntrack_max
  # 기본: 65536 (시스템마다 다름)
  
  고트래픽 환경에서 Conntrack 테이블이 가득 차면:
  → 새 연결 생성 불가 → "table full, dropping packet" 커널 로그
  → 증상: 간헐적 연결 실패
  → 해결: nf_conntrack_max 증가 또는 타임아웃 감소
```

---

## 😱 흔한 실수

```
Before — NAT를 모를 때:

실수 1: "서버에 공인 IP를 달았는데 외부에서 접근이 안 돼요"
  → 실제로는 공유기 뒤에 있는 사설 IP
  → 공인 IP처럼 보이는 것은 공유기의 IP
  → 포트 포워딩 설정 필요한데 설정 안 함

실수 2: Conntrack 테이블 고갈
  고트래픽 서버에서 간헐적 연결 실패
  → dmesg: "nf_conntrack: table full, dropping packet"
  → NAT를 모르면 원인 파악 불가
  → 해결: conntrack max 증가 또는 NAT 없이 Direct Routing 사용

실수 3: Docker 컨테이너에서 서버 IP를 잘못 로깅
  컨테이너 내부에서 접속 IP 로깅
  → SNAT로 인해 실제 클라이언트 IP 대신 172.17.0.1이 찍힘
  → X-Forwarded-For 헤더나 Proxy Protocol 필요
```

---

## ✨ 올바른 접근

```
After — NAT를 알고 나면:

클라이언트 IP 보존:
  Docker/Nginx 앞에서 X-Forwarded-For 설정:
  nginx: proxy_set_header X-Real-IP $remote_addr;
  
  Spring:
  @RequestHeader("X-Real-IP") String clientIp
  또는 HttpServletRequest.getRemoteAddr() (프록시 미사용 시)

Conntrack 튜닝:
  # 현재 사용량 확인
  cat /proc/sys/net/netfilter/nf_conntrack_count
  cat /proc/sys/net/netfilter/nf_conntrack_max
  
  # 증가 (영구 설정)
  echo 'net.netfilter.nf_conntrack_max = 131072' >> /etc/sysctl.conf
  sysctl -p

  # 단기 TCP Conntrack 타임아웃 (TIME_WAIT 줄이기)
  sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30

Docker 실제 IP 확인:
  # 실제 클라이언트 IP가 필요한 경우 host 네트워크 사용
  docker run --network=host nginx
  # 단, 컨테이너가 호스트 포트 네임스페이스 공유 → 격리 없음
```

---

## 💻 실전 실험

### 실험 1: iptables NAT 규칙 분석

```bash
# NAT 테이블 전체 확인
sudo iptables -t nat -L -n -v --line-numbers

# Docker 실행 후 변화 확인
docker run -d -p 8080:80 --name test-nginx nginx
sudo iptables -t nat -L -n -v --line-numbers

# DOCKER 체인의 DNAT 규칙 확인
sudo iptables -t nat -L DOCKER -n -v
# 출력: DNAT tcp -- !docker0 * 0.0.0.0/0 0.0.0.0/0 tcp dpt:8080 to:172.17.0.x:80

# POSTROUTING의 MASQUERADE 규칙
sudo iptables -t nat -L POSTROUTING -n -v
# 출력: MASQUERADE all -- * !docker0 172.17.0.0/16 0.0.0.0/0

# 정리
docker rm -f test-nginx
```

### 실험 2: Connection Tracking 실시간 관찰

```bash
# conntrack 도구 설치
sudo apt install conntrack -y

# 현재 연결 추적 테이블 확인
sudo conntrack -L

# 실시간 모니터링
sudo conntrack -E  # 이벤트 스트림

# 다른 터미널에서 연결 생성
curl http://example.com

# conntrack에서 확인되는 항목:
#     [NEW] tcp 6 120 SYN_SENT src=192.168.1.100 dst=93.184.216.34 ...
# [UPDATE] tcp 6 60 SYN_RECV ...
# [UPDATE] tcp 6 432 ESTABLISHED ...

# Docker 포트 포워딩의 conntrack 확인
docker run -d -p 8080:80 nginx
curl http://localhost:8080
sudo conntrack -L | grep 8080
# tcp ESTABLISHED src=127.0.0.1 dst=172.17.0.x sport=xxxxx dport=80

docker rm -f $(docker ps -q)
```

### 실험 3: SNAT 동작 추적

```bash
# Docker 컨테이너에서 외부로 나가는 SNAT 추적

# 컨테이너 실행
docker run -it --rm alpine sh

# 컨테이너 내부에서 (컨테이너 터미널):
ip addr show  # 컨테이너 IP 확인 (172.17.x.x)
curl -s https://ifconfig.me  # 외부에서 보이는 IP (호스트의 공인 IP)
# → 컨테이너 IP(172.17.x.x)가 아닌 호스트 IP가 출력됨 = SNAT 동작 확인

# 호스트에서 MASQUERADE 동작 확인
sudo tcpdump -i docker0 -nn &  # docker0 브릿지 인터페이스
# → 컨테이너의 172.17.x.x IP 패킷이 보임

sudo tcpdump -i eth0 -nn 'host ifconfig.me' &
# → eth0에서는 호스트 IP가 SrcIP로 보임 = SNAT 완료
```

### 실험 4: Conntrack 고갈 시뮬레이션

```bash
# 현재 conntrack 설정 확인
cat /proc/sys/net/netfilter/nf_conntrack_max
cat /proc/sys/net/netfilter/nf_conntrack_count

# conntrack 테이블을 모두 보기
sudo conntrack -L | wc -l

# 테이블이 가득 찼을 때의 커널 메시지
dmesg | grep conntrack
# 고갈 시: "nf_conntrack: table full, dropping packet"

# 임시로 max 늘리기
sudo sysctl -w net.netfilter.nf_conntrack_max=262144

# TIME_WAIT conntrack 확인 (TCP 종료 후 남는 항목)
sudo conntrack -L | grep TIME_WAIT | wc -l

# TIME_WAIT conntrack 타임아웃 줄이기
sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
```

---

## 📊 NAT 성능 비교

```
NAT 처리 비용:

항목별 처리:
  패킷당 NAT 변환: 수 마이크로초 (커널 공간에서 처리)
  Conntrack 조회: 해시 테이블 O(1)
  iptables 규칙 매칭: 규칙 수에 선형 O(n)

실제 성능:
  소규모 NAT (수십 개 규칙): 무시할 수준
  Docker 환경 (수백 개 규칙): 1~2% 오버헤드
  Kubernetes kube-proxy iptables 모드 (수천 개 규칙):
    → 서비스 수 증가 시 O(n) 문제 → ipvs 모드로 전환 권장
    → ipvs: 해시 테이블 기반 O(1)

Conntrack 테이블 메모리:
  각 항목: 약 300 bytes
  65536 항목: 약 20 MB
  131072 항목: 약 40 MB

고성능 NAT 대안:
  DPDK (Data Plane Development Kit): 커널 우회
  XDP (eXpress Data Path): 커널 내 but iptables 우회
  → 초당 수백만 패킷 처리 가능
```

---

## ⚖️ 트레이드오프

```
NAT의 장단점:

장점:
  ① 공인 IP 절약:
     수천 개의 사설 IP 기기가 1개의 공인 IP 공유
  
  ② 보안:
     내부 IP가 외부에 노출되지 않음
     포트 포워딩 없이는 외부에서 내부 접근 불가

단점:
  ① End-to-End 연결 파괴:
     P2P 애플리케이션 어려움 (BitTorrent, VoIP)
     외부에서 먼저 연결 시작 불가
     → STUN/TURN/ICE 같은 NAT 우회 기술 필요 (WebRTC)
  
  ② 클라이언트 IP 숨김:
     서버 입장에서 실제 클라이언트 IP를 알기 어려움
     → X-Forwarded-For, PROXY Protocol로 해결
  
  ③ Conntrack 테이블 고갈:
     고트래픽 환경에서 메모리/CPU 부하
     → Direct Routing, IPv6로 NAT 자체를 없애는 방향

IPv6와 NAT:
  IPv6: 주소 공간이 충분 (2^128)
  → 모든 기기가 공인 IPv6 주소를 가질 수 있음
  → NAT 불필요 → End-to-End 연결 복원
  → 하지만 보안상 방화벽은 여전히 필요
```

---

## 📌 핵심 정리

```
NAT와 포트 포워딩 핵심 요약:

사설 IP가 인터넷에 나가는 방법 (SNAT):
  사설 IP:Port → 공인 IP:Port 변환 (MASQUERADE)
  NAT 테이블에 매핑 기록
  응답 패킷이 오면 역변환해서 내부로 전달
  포트 번호로 여러 내부 호스트 구분 (NAPT)

외부에서 내부로 들어오는 방법 (DNAT/포트 포워딩):
  공인 IP:외부포트 → 사설 IP:내부포트
  docker run -p 8080:80 → iptables DNAT 규칙 생성
  iptables -t nat -L DOCKER로 규칙 확인

Connection Tracking (Conntrack):
  각 연결의 상태를 추적 (NEW → ESTABLISHED → TIME_WAIT)
  응답 패킷을 ESTABLISHED로 인식 → 별도 방화벽 규칙 없이 허용
  테이블 고갈 시: "nf_conntrack: table full" → 연결 실패

주요 명령어:
  iptables -t nat -L -n -v    → NAT 규칙 확인
  conntrack -L                → 현재 연결 추적 테이블
  conntrack -E                → 실시간 이벤트 모니터링
  sysctl net.netfilter.nf_conntrack_max → Conntrack 최대 항목 수
```

---

## 🤔 생각해볼 문제

**Q1.** 집에서 Spring Boot 서버(포트 8080)를 실행했다. 친구에게 내 공인 IP를 알려줘도 친구가 접근하지 못한다. 왜 그런가? 해결하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

집의 인터넷 환경은 대부분 공유기(NAT) 뒤에 있습니다. 친구가 접근하려는 공인 IP는 **공유기의 IP**이지, 내 PC의 IP(192.168.x.x)가 아닙니다.

인터넷에서 온 패킷이 공유기에 도달하면, 공유기는 이 패킷을 어느 내부 기기로 전달해야 할지 모릅니다. 포트 포워딩 규칙이 없기 때문입니다.

해결 방법:
1. **포트 포워딩 설정**: 공유기 관리 페이지(보통 192.168.1.1)에서 포트 포워딩 설정
   - 외부 포트: 8080
   - 내부 IP: 내 PC의 사설 IP (예: 192.168.1.100)
   - 내부 포트: 8080

2. **외부 공인 IP 확인**: `curl ifconfig.me`

3. **ISP가 포트 차단하는 경우**: 일부 ISP는 80, 443 포트를 차단합니다. 8080 같은 다른 포트를 사용하세요.

4. **더 나은 방법**: ngrok 같은 터널링 서비스 사용 (포트 포워딩 설정 없이 외부 접근 가능)

</details>

---

**Q2.** Docker 컨테이너 내부에서 `echo $REMOTE_ADDR`로 찍힌 IP가 `172.17.0.1`이다. 실제 클라이언트 IP를 얻으려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

`172.17.0.1`은 Docker bridge 네트워크의 게이트웨이 IP입니다. SNAT(MASQUERADE)으로 인해 컨테이너에서 보이는 클라이언트 IP가 docker0 브릿지 IP로 변환된 것입니다.

**방법 1: X-Forwarded-For 헤더 활용**
Nginx를 앞에 두고 실제 IP를 헤더에 추가합니다.
```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```
Spring에서: `request.getHeader("X-Real-IP")`

**방법 2: Docker host 네트워크 모드**
```bash
docker run --network=host spring-app
```
컨테이너가 호스트의 네트워크 네임스페이스를 공유하여 NAT 없이 직접 통신합니다. 단, 포트 격리가 없어지므로 컨테이너 포트가 곧 호스트 포트가 됩니다.

**방법 3: PROXY Protocol (Nginx + Spring)**
TCP 레벨에서 클라이언트 IP를 전달하는 프로토콜입니다. HTTP 헤더보다 신뢰성 높습니다.

**방법 4: Kubernetes에서는 externalTrafficPolicy: Local**
NodePort/LoadBalancer에서 SNAT를 끄고 실제 클라이언트 IP를 유지합니다.

</details>

---

**Q3.** AWS EC2 인스턴스에 공인 IP가 없는 상태에서 인터넷에 나가려면 NAT Gateway가 필요하다. NAT Gateway 없이 인터넷에 나갈 수 있는 방법이 있는가?

<details>
<summary>해설 보기</summary>

예, 몇 가지 방법이 있습니다.

1. **공인 IP 할당**: EC2 인스턴스에 직접 Elastic IP를 할당하면 NAT 없이 인터넷 통신이 가능합니다.

2. **Egress-Only Internet Gateway (IPv6)**: IPv6 주소를 사용하면 인터넷 게이트웨이를 통해 외부로만 나갈 수 있습니다. IPv6는 공인 주소이므로 NAT가 불필요합니다.

3. **VPC Endpoint**: S3, DynamoDB, SQS 같은 AWS 서비스에는 인터넷을 통하지 않고 AWS 내부 네트워크로 통신하는 VPC Endpoint를 사용할 수 있습니다.

4. **NAT Instance**: NAT Gateway 대신 EC2 인스턴스를 NAT 역할로 직접 구성합니다. 비용은 저렴하지만 관리가 필요합니다.

트레이드오프: NAT Gateway는 관리형 서비스로 고가용성이 보장되지만 비용이 있습니다. 공인 IP 직접 할당은 간단하지만 보안상 외부에 IP가 노출됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Ethernet과 ARP](./03-ethernet-and-arp.md)** | **[홈으로 🏠](../README.md)** | **[다음: 네트워크 진단 도구 ➡️](./05-network-diagnostic-tools.md)**

</div>
