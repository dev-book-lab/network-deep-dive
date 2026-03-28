# OSI 7계층 vs TCP/IP 4계층 — 계층 모델이 왜 필요한가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 계층 모델이 없다면 네트워크 프로그래밍에서 어떤 문제가 생기는가?
- OSI 7계층과 TCP/IP 4계층은 각각 어떤 계층이 어떤 일을 담당하는가?
- 패킷이 송신자에서 수신자까지 전달될 때 각 계층에서 무슨 일이 일어나는가?
- 캡슐화(Encapsulation)와 역캡슐화(Decapsulation)는 구체적으로 무엇을 붙이고 떼는가?
- Spring MVC가 받는 HTTP 요청은 계층 모델에서 어느 위치에 있는가?
- "L4 로드 밸런서", "L7 스위치" 같은 표현에서 L은 무엇을 의미하는가?

---

## 🔍 왜 이 개념이 중요한가

### 계층 없이 네트워크를 만들면 무슨 일이 생기는가

```
계층 없는 세계의 문제:

시나리오: 이더넷 위에서 HTTP 통신하는 코드를 짠다고 가정

  1. HTTP 개발자가 해야 할 일:
     - 이더넷 프레임 포맷을 직접 구성
     - MAC 주소를 어떻게 알아낼지 직접 구현 (ARP)
     - IP 라우팅 로직 직접 구현
     - TCP 재전송, ACK, 순서 보장 직접 구현
     - 그 위에서야 비로소 HTTP 헤더 파싱

  2. Wi-Fi로 교체하면?
     - 이더넷 프레임 → Wi-Fi 프레임으로 전체 재작성
     - HTTP 코드를 수정해야 함

  3. TCP 대신 UDP를 쓰고 싶다면?
     - HTTP 코드를 다시 수정해야 함

계층 모델의 해결책:
  "나는 내 계층의 일만 한다. 아래 계층은 신경 쓰지 않는다."

  HTTP는 TCP 소켓에 데이터를 쓰기만 하면 된다
  TCP는 IP 패킷으로 보내기만 하면 된다
  IP는 이더넷 프레임에 담기만 하면 된다
  이더넷은 전기 신호로 내보내기만 하면 된다

  → Wi-Fi로 교체해도 HTTP 코드는 한 줄도 안 바뀐다
  → HTTP를 HTTP/2로 바꿔도 TCP 코드는 안 바뀐다
```

---

## 🔬 내부 동작 원리

### 1. OSI 7계층 구조

```
OSI 7계층 모델 (Open Systems Interconnection):

┌─────────────────────────────────────────────────────────────┐
│  계층   │  이름           │  하는 일         │  프로토콜 예시      │
├─────────────────────────────────────────────────────────────┤
│   7    │  Application   │  사용자 데이터 처리 │ HTTP, SMTP, DNS │
│   6    │  Presentation  │  암호화, 인코딩    │ TLS, JPEG, ASCII│
│   5    │  Session       │  세션 관리, 동기화 │  RPC, NetBIOS    │
│   4    │  Transport     │  종단간 전달, 신뢰성│  TCP, UDP       │
│   3    │  Network       │  논리 주소, 라우팅 │  IP, ICMP, BGP   │
│   2    │  Data Link     │  물리 주소, 오류감지│  Ethernet, Wi-Fi│
│   1    │  Physical      │  비트 → 전기 신호  │  RJ45, 광섬유     │
└─────────────────────────────────────────────────────────────┘

현실에서 OSI 7계층이 깔끔하게 맞아떨어지지 않는 이유:
  - TLS는 6계층(Presentation)이지만 TCP 위에서 동작 → 4계층과 7계층 사이
  - DNS는 7계층이지만 UDP(4계층)를 사용
  - 현대 프로토콜은 경계가 뚜렷하지 않음

  → OSI는 개념 모델(Conceptual Model)
  → 실제 구현은 TCP/IP 4계층 모델을 사용
```

### 2. TCP/IP 4계층 구조

```
TCP/IP 4계층 모델 (실제 인터넷이 사용하는 모델):

┌──────────────────────────────────────────────────────────────────┐
│  계층  │  이름           │  담당 범위         │  헤더/프로토콜           │
├──────────────────────────────────────────────────────────────────┤
│   4   │  Application   │  사용자 데이터      │  HTTP 헤더, DNS 쿼리    │
│       │                │  (OSI 5,6,7 통합) │  FTP 명령, SMTP 메시지  │
├──────────────────────────────────────────────────────────────────┤
│   3   │  Transport     │  종단간(Port→Port)│  TCP 헤더 (20바이트)    │
│       │                │  신뢰성/비연결      │  UDP 헤더 (8바이트      │
├──────────────────────────────────────────────────────────────────┤
│   2   │  Internet      │  호스트간(IP→IP)   │  IP 헤더 (20바이트)     │
│       │                │  라우팅           │  ICMP, ARP            │
├──────────────────────────────────────────────────────────────────┤
│   1   │  Network       │  물리 전송         │  Ethernet 프레임       │
│       │  Access        │  (OSI 1,2 통합)   │  Wi-Fi 프레임          │
└──────────────────────────────────────────────────────────────────┘

OSI → TCP/IP 대응:
  OSI 7,6,5  →  TCP/IP Application (HTTP, TLS, DNS)
  OSI 4      →  TCP/IP Transport   (TCP, UDP)
  OSI 3      →  TCP/IP Internet    (IP, ICMP)
  OSI 2,1    →  TCP/IP Network Access (Ethernet, Wi-Fi)
```

### 3. 캡슐화: 송신 측에서 계층을 내려갈 때

```
Spring 애플리케이션이 HTTP 응답을 보내는 순간:

[Application Layer — Spring MVC]
  데이터: HTTP 응답 메시지
  ┌──────────────────────────────────────────┐
  │  HTTP/1.1 200 OK                         │
  │  Content-Type: application/json          │
  │  Content-Length: 27                      │
  │                                          │
  │  {"message": "hello world"}              │
  └──────────────────────────────────────────┘
  ↓ TCP 소켓으로 write()

[Transport Layer — TCP]
  TCP 헤더를 앞에 붙임 (Encapsulation)
  ┌────────────┬─────────────────────────────┐
  │ TCP Header │       HTTP 데이터             │
  │ (20 bytes) │                             │
  │ SrcPort    │  HTTP/1.1 200 OK ...        │
  │ DstPort    │  {"message": "hello world"} │
  │ Seq, ACK   │                             │
  └────────────┴─────────────────────────────┘
  ↓ IP 계층으로 전달

[Internet Layer — IP]
  IP 헤더를 앞에 붙임
  ┌───────────┬────────────┬─────────────────┐
  │ IP Header │ TCP Header │   HTTP 데이터     │
  │ (20 bytes)│ (20 bytes) │                 │
  │ SrcIP     │ SrcPort    │ HTTP/1.1 200 OK │
  │ DstIP     │ DstPort    │ {"message"...}  │
  │ TTL, Proto│ Seq, ACK   │                 │
  └───────────┴────────────┴─────────────────┘
  ↓ 네트워크 인터페이스로 전달

[Network Access Layer — Ethernet]
  Ethernet 헤더(앞) + FCS(뒤)를 붙임
  ┌──────────┬───────────┬────────────┬──────────┬─────┐
  │ Ethernet │ IP Header │ TCP Header │HTTP 데이터 │ FCS │
  │ Header   │ (20 bytes)│ (20 bytes) │          │ (4B)│
  │ DstMAC   │ SrcIP     │ SrcPort    │HTTP/1.1  │     │
  │ SrcMAC   │ DstIP     │ DstPort    │200 OK    │     │
  │ EtherType│ TTL       │ Seq, ACK   │{"msg"...}│     │
  └──────────┴───────────┴────────────┴──────────┴─────┘
  ↓ 전기 신호 / 광신호로 변환해서 케이블/무선으로 전송

각 계층이 붙이는 헤더 크기:
  Ethernet Header: 14 bytes (DstMAC 6 + SrcMAC 6 + EtherType 2)
  IP Header:       20 bytes (최소)
  TCP Header:      20 bytes (최소)
  ────────────────
  오버헤드 합계:   54 bytes

  실제 데이터 1바이트를 보내도 최소 54바이트의 헤더가 붙음
  → 작은 메시지를 자주 보내면 비효율 → Nagle 알고리즘의 배경
```

### 4. 역캡슐화: 수신 측에서 계층을 올라갈 때

```
클라이언트 브라우저가 응답을 받는 순간:

[Network Access Layer]
  수신: 전기 신호 → 비트 → Ethernet 프레임
  ┌──────────┬───────────┬────────────┬──────────┬─────┐
  │ Ethernet │ IP Header │ TCP Header │HTTP 데이터 │ FCS │
  └──────────┴───────────┴────────────┴──────────┴─────┘
  
  처리:
    DstMAC이 내 MAC 주소인지 확인 (아니면 버림)
    FCS로 프레임 오류 검사
    EtherType 확인 → 0x0800 (IPv4) → IP 계층으로 전달
    Ethernet 헤더 제거 (Decapsulation)

[Internet Layer — IP]
  수신: IP 패킷
  ┌───────────┬────────────┬──────────┐
  │ IP Header │ TCP Header │ HTTP 데이터│
  └───────────┴────────────┴──────────┘

  처리:
    DstIP가 내 IP인지 확인
    IP 헤더 체크섬 검증
    Protocol 필드 확인 → 6 (TCP) → TCP 계층으로 전달
    IP 헤더 제거

[Transport Layer — TCP]
  수신: TCP 세그먼트
  ┌────────────┬──────────┐
  │ TCP Header │ HTTP 데이터│
  └────────────┴──────────┘

  처리:
    DstPort 확인 → 어느 애플리케이션에 전달할지 결정
    Sequence Number로 순서 재조합
    ACK 전송
    TCP 헤더 제거 → 데이터를 소켓 수신 버퍼에 적재

[Application Layer — 브라우저]
  수신: HTTP 메시지 (순수 데이터)
  ┌──────────────────────────────────────────┐
  │  HTTP/1.1 200 OK                         │
  │  Content-Type: application/json          │
  │  {"message": "hello world"}              │
  └──────────────────────────────────────────┘
  → HTTP 헤더 파싱 → JSON 렌더링

각 계층의 핵심 역할:
  Network Access: "내가 받을 프레임인가?" (MAC 주소 필터)
  IP:             "이 패킷의 최종 목적지가 나인가?" (IP 주소 필터)
  TCP:            "이 데이터는 어느 프로세스에게?" (Port 필터, 순서 보장)
  Application:    "이 데이터의 의미는?" (HTTP/JSON 해석)
```

### 5. 중간 노드(라우터, 스위치)는 몇 계층까지 처리하는가

```
패킷이 거치는 장비별 처리 계층:

                    [Client]           [Router]         [Server]
                    L1~L7              L1~L3             L1~L7
                       │                 │                  │
Ethernet Frame ────────┼────────────────►│                  │
                       │       IP Packet │─────────────────►│
                       │                 │                  │

스위치 (L2 Switch):
  - Ethernet 프레임까지만 처리 (L1, L2)
  - MAC 주소 테이블을 보고 적절한 포트로 전달
  - IP 헤더를 열어보지 않음
  - 같은 네트워크(서브넷) 안에서의 전달 담당

라우터 (L3 Router):
  - IP 패킷까지 처리 (L1, L2, L3)
  - IP 헤더의 DstIP를 보고 라우팅 테이블에서 Next Hop 결정
  - 새 Ethernet 프레임으로 재포장해서 다음 구간으로 전달
  - 서로 다른 네트워크(서브넷) 간의 연결 담당

L4 로드 밸런서:
  - TCP 헤더까지 처리 (L1~L4)
  - DstPort를 보고 트래픽을 서버들에 분산
  - HTTP 내용(URL, 헤더)은 보지 않음
  - 빠르지만 정교한 라우팅 불가

L7 로드 밸런서 (예: Nginx):
  - HTTP 페이로드까지 처리 (L1~L7)
  - URL 경로, 헤더, 쿠키를 보고 라우팅 결정
  - "/api" → API 서버, "/static" → 파일 서버
  - 정교하지만 처리 비용이 큼

실제 통신 경로:
  Spring App → OS TCP/IP 스택 → NIC → L2 스위치 → L3 라우터 (여러 홉)
  → L7 로드 밸런서 → L3 라우터 → L2 스위치 → NIC → OS TCP/IP 스택 → DB
```

### 6. 각 계층의 데이터 단위 (PDU)

```
계층별 데이터 단위 이름:

  Application  →  Message (메시지)     : HTTP 요청/응답, DNS 쿼리
  Transport    →  Segment (세그먼트)   : TCP 세그먼트, UDP 데이터그램
  Internet     →  Packet  (패킷)       : IP 패킷
  Network Access → Frame  (프레임)     : Ethernet 프레임
  Physical     →  Bit     (비트)       : 0과 1

혼용되는 경우:
  "패킷을 보낸다" → 일상 표현에서는 계층 구분 없이 패킷으로 통칭
  "세그먼트 손실" → TCP 계층 문제
  "프레임 손상"   → Ethernet 계층 문제 (FCS 오류)
  
  정확한 진단을 위해 계층을 구분해야 함:
    ping 실패    → IP 계층(L3) 문제 가능성
    telnet 실패  → TCP 계층(L4) 문제 가능성
    curl 실패    → Application 계층(L7) 문제 가능성
```

---

## 😱 흔한 실수

```
Before — 계층 개념 없이 네트워크 장애를 대하는 방법:

  "서버에 접속이 안 돼요"
    → 무작정 서버 재시작
    → 방화벽 전부 열기
    → 로그 전체를 grep
    → 원인 파악 없이 시간 소비

  "L4 로드 밸런서로 바꾸면 더 빠를까요?"
    → L4와 L7의 차이가 뭔지 모름
    → URL 기반 라우팅이 필요한데 L4로 구성해서 기능 미동작

  "포트 80과 443이 왜 달라요?"
    → 포트가 Transport Layer(L4) 개념이라는 것을 모름
    → IP 주소와 포트를 혼동
```

---

## ✨ 올바른 접근

```
After — 계층을 알고 나면 장애를 단계적으로 격리할 수 있다:

  "서버에 접속이 안 돼요"

  Step 1: L3(IP) 확인
    ping 10.0.0.1
    → 응답 없음: L3 문제 (라우팅, 방화벽 ICMP 차단)
    → 응답 있음: L3는 정상, L4 확인

  Step 2: L4(TCP) 확인
    telnet 10.0.0.1 8080
    → "Connection refused": 서버 프로세스가 포트를 열고 있지 않음
    → "Connection timed out": 방화벽이 TCP SYN 차단
    → 연결됨: L4 정상, L7 확인

  Step 3: L7(HTTP) 확인
    curl -v http://10.0.0.1:8080/health
    → HTTP 오류 코드: 애플리케이션 문제
    → TLS 오류: 인증서 문제

  계층을 알면 진단 명령어 3개로 원인을 격리할 수 있다
```

---

## 💻 실전 실험

### 실험 1: 계층별 헤더를 직접 캡처하고 분석하기

```bash
# 실험 환경: curl로 HTTP 요청을 보내면서 tcpdump로 각 계층을 캡처

# 터미널 1: 캡처 시작
sudo tcpdump -i any -w /tmp/layers.pcap 'host example.com' &

# 터미널 2: HTTP 요청
curl -s http://example.com > /dev/null

# 캡처 중지
sudo kill %1

# 캡처 파일 분석 (텍스트로 출력)
sudo tcpdump -r /tmp/layers.pcap -v -nn

# 출력 예시:
# 14:23:01.123456 IP 192.168.1.100.54321 > 93.184.216.34.80: Flags [S], seq 1234567890
#   └─ IP 계층: 192.168.1.100 → 93.184.216.34
#   └─ TCP 계층: 포트 54321 → 80
#   └─ Flags [S]: SYN 패킷

# 각 계층 필드를 더 자세히 보기
sudo tcpdump -r /tmp/layers.pcap -vvv -nn -e | head -50
# -e 옵션: Ethernet(L2) 헤더도 출력
# 출력 예시:
# 14:23:01.123456 aa:bb:cc:dd:ee:ff > 11:22:33:44:55:66, ethertype IPv4 (0x0800)
#   └─ Ethernet 계층: MAC aa:bb:cc:dd:ee:ff → 11:22:33:44:55:66
#   └─ EtherType 0x0800: IPv4 패킷임을 표시
```

### 실험 2: 계층별 오버헤드 측정

```bash
# IP 헤더 + TCP 헤더 크기 확인
# tcpdump에서 length 확인
sudo tcpdump -r /tmp/layers.pcap -nn -q | head -20
# length 값: TCP 페이로드 크기 (헤더 제외)

# ping으로 IP + ICMP 오버헤드 확인
ping -s 0 example.com   # 데이터 0바이트 + ICMP 헤더
# ICMP echo reply: 8 bytes 헤더
# IP 헤더: 20 bytes
# Ethernet: 14 bytes
# → 총 42 bytes 전송

ping -s 1 example.com   # 데이터 1바이트
# → 총 43 bytes 전송
# → 1바이트 데이터를 보내기 위해 42바이트 오버헤드 발생!
```

### 실험 3: Spring MVC 요청이 계층을 타고 오는 경로 추적

```bash
# Spring Boot 애플리케이션 실행 중 가정 (포트 8080)

# 요청이 도착하는 순간 각 계층에서 일어나는 일 캡처
sudo tcpdump -i lo -nn -v 'tcp port 8080' &

# 다른 터미널에서 요청
curl http://localhost:8080/api/hello

# 캡처 결과 분석:
# 1) SYN 패킷       → TCP 연결 수립 시작 (L4)
# 2) SYN-ACK        → 서버 수신 확인 (L4)
# 3) ACK            → 연결 수립 완료 (L4)
# 4) HTTP 요청 데이터 → GET /api/hello HTTP/1.1 (L7)
# 5) ACK            → 데이터 수신 확인 (L4)
# 6) HTTP 응답 데이터 → HTTP/1.1 200 OK {...} (L7)
# 7) FIN-ACK        → 연결 종료 시작 (L4)

# Spring은 4번(L7)에서 데이터를 처음 받음
# 1~3번 (TCP Handshake)은 OS가 처리 — Spring은 관여하지 않음
```

### 실험 4: 계층별 장애 시뮬레이션

```bash
# L3 장애 시뮬레이션: IP 경로 차단
sudo iptables -A OUTPUT -d 93.184.216.34 -j DROP
ping example.com       # → Request timeout (L3 차단)
curl http://example.com  # → curl: (28) Connection timed out

# L4 장애 시뮬레이션: 특정 포트 차단
sudo iptables -D OUTPUT -d 93.184.216.34 -j DROP  # 이전 규칙 제거
sudo iptables -A OUTPUT -d 93.184.216.34 -p tcp --dport 80 -j REJECT
ping example.com        # → 성공 (L3는 정상)
curl http://example.com # → curl: (7) Connection refused (L4 차단)
telnet example.com 80   # → Connection refused

# L7 장애 시뮬레이션: 잘못된 HTTP 헤더
sudo iptables -D OUTPUT -d 93.184.216.34 -p tcp --dport 80 -j REJECT
curl -H "Host: invalid" http://example.com  # → 400 Bad Request (L7 오류)
# TCP 연결은 성공하지만 HTTP 레벨에서 실패

# 정리
sudo iptables -F OUTPUT
```

---

## 📊 계층별 처리 비교

```
계층별 처리 속도와 정교함:

┌─────────────┬──────────────┬──────────────────────┬─────────────────────┐
│  장비/소프트   │  처리 계층     │  처리 속도             │  라우팅 근거           │
├─────────────┼──────────────┼──────────────────────┼─────────────────────┤
│ NIC         │ L1           │ 나노초 단위             │ 전기 신호             │
│ L2 스위치     │ L1~L2        │ 수십 마이크로초          │ MAC 주소 테이블       │
│ L3 라우터     │ L1~L3        │ 수백 마이크로초          │ 라우팅 테이블(IP)      │
│ L4 LB       │ L1~L4        │ 1ms 미만              │ IP:Port             │
│ L7 LB(Nginx)│ L1~L7        │ 수 ms                 │ URL, Header, Cookie│
│ Spring MVC  │ L7           │ 수십~수백 ms           │ @RequestMapping     │
└─────────────┴──────────────┴──────────────────────┴─────────────────────┘

계층이 높을수록:
  ✅ 더 정교한 라우팅/처리 가능
  ❌ 처리 비용 증가
  ❌ 지연시간 증가

실무 선택 기준:
  단순 트래픽 분산만 필요 → L4 로드 밸런서 (빠름)
  URL/헤더 기반 라우팅 필요 → L7 로드 밸런서 (정교함)
  SSL 종료 필요 → L7 (TLS는 L7에서 처리)
  WebSocket 지원 필요 → L7 (HTTP Upgrade 해석 필요)
```

---

## ⚖️ 트레이드오프

```
계층 모델의 장단점:

장점:
  ① 독립적 발전:
     HTTP/1.1 → HTTP/2 교체 시 TCP 코드 변경 불필요
     Ethernet → Wi-Fi 교체 시 IP 코드 변경 불필요

  ② 관심사 분리:
     Spring 개발자는 TCP 재전송 로직을 모르고 개발 가능
     TCP 스택 개발자는 HTTP 의미론을 모르고 개발 가능

  ③ 표준화:
     어느 OS, 어느 하드웨어든 같은 계층 모델을 구현하면 통신 가능

단점:
  ① 오버헤드:
     각 계층이 헤더를 붙임 → 54바이트 이상의 메타데이터
     작은 메시지에서 헤더 비율이 높아짐

  ② 경직성:
     엄격한 계층 분리는 때로 비효율
     QUIC(HTTP/3)은 Transport + Crypto를 L4에서 통합 → 계층 경계 허물기

  ③ 계층 간 정보 은닉:
     L7(HTTP)에서 L4(TCP)의 혼잡 상태를 직접 알기 어려움
     → QUIC이 애플리케이션 레이어에 혼잡 제어를 통합한 이유
```

---

## 📌 핵심 정리

```
계층 모델 핵심 요약:

왜 계층 모델이 필요한가:
  → 관심사 분리: 각 계층이 자기 역할만 담당
  → 독립적 발전: 한 계층 교체 시 다른 계층 영향 없음
  → 표준화: 다른 구현체끼리도 통신 가능

TCP/IP 4계층:
  Application  → HTTP, DNS (사용자 데이터, 의미)
  Transport    → TCP, UDP (종단간 전달, Port)
  Internet     → IP, ICMP (논리 주소, 라우팅)
  Network Access → Ethernet, Wi-Fi (물리 전송, MAC)

캡슐화/역캡슐화:
  송신: 각 계층이 헤더를 앞에 붙임 (아래로 내려가며)
  수신: 각 계층이 헤더를 떼고 위로 전달 (올라가며)
  오버헤드: Ethernet(14) + IP(20) + TCP(20) = 54 bytes

장비별 처리 계층:
  L2 스위치    → MAC 주소로 전달
  L3 라우터    → IP 주소로 라우팅
  L4 로드밸런서 → IP:Port로 분산
  L7 로드밸런서 → URL/Header로 라우팅

진단 명령어 계층 매핑:
  ping          → L3 (IP 도달 확인)
  telnet host port → L4 (TCP 포트 오픈 확인)
  curl -v       → L7 (HTTP 응답 확인)
  tcpdump       → L2~L4 (패킷 레벨 캡처)
```

---

## 🤔 생각해볼 문제

**Q1.** TLS(HTTPS)는 OSI 모델의 몇 계층인가? TCP/IP 4계층 모델에서는 어디에 위치하는가?

<details>
<summary>해설 보기</summary>

OSI 7계층 기준으로 TLS는 **6계층(Presentation Layer)** 에 해당합니다. 암호화/복호화, 인코딩 변환이 Presentation Layer의 역할이기 때문입니다.

그러나 TCP/IP 4계층 모델에서는 경계가 명확하지 않습니다. TLS는 TCP(Transport) 위에서 동작하지만, HTTP(Application) 아래에 위치합니다. 즉, **Transport와 Application 사이의 중간 계층**처럼 동작합니다.

실용적으로는 "Application Layer의 일부" 또는 "L4.5"처럼 부르기도 합니다.

이 모호함이 QUIC(HTTP/3)의 설계에 영향을 줬습니다. QUIC은 TLS 1.3을 프로토콜 내부에 통합해서 UDP 위에서 암호화와 신뢰성 전송을 동시에 처리합니다. 계층 경계를 의도적으로 허물어 더 나은 성능을 달성한 예입니다.

</details>

---

**Q2.** 같은 PC에서 실행 중인 두 프로세스가 localhost로 통신할 때 (예: Spring App → 로컬 MySQL), 패킷이 실제로 NIC(네트워크 인터페이스 카드)를 통과하는가?

<details>
<summary>해설 보기</summary>

아닙니다. `localhost(127.0.0.1)` 통신은 **loopback 인터페이스**를 사용합니다. loopback은 실제 NIC가 아니라 OS 커널 내부의 가상 인터페이스입니다.

통신 경로:
```
Spring App → OS TCP/IP 스택 → loopback(lo) → OS TCP/IP 스택 → MySQL
```

실제 이더넷 케이블이나 NIC 하드웨어를 전혀 거치지 않습니다. 단, **OS의 TCP/IP 스택은 동일하게 거칩니다**. 즉, TCP 헤더, IP 헤더는 붙었다 떼어집니다.

확인 방법:
```bash
sudo tcpdump -i lo port 3306   # loopback 인터페이스의 MySQL 트래픽
sudo tcpdump -i eth0 port 3306 # 실제 NIC → 아무것도 안 잡힘
```

이 사실의 실무적 의미: localhost 통신도 TCP 연결이기 때문에 Connection Pool, TIME_WAIT, CLOSE_WAIT 등 모든 TCP 문제가 동일하게 발생합니다.

</details>

---

**Q3.** 라우터는 L3까지만 처리한다고 했는데, 그렇다면 NAT(Network Address Translation)를 수행하는 공유기는 어느 계층을 처리하는가?

<details>
<summary>해설 보기</summary>

공유기(NAT 라우터)는 일반 L3 라우터와 달리 **L4(Transport Layer)까지 처리**합니다.

이유: NAT는 IP 주소(L3)뿐만 아니라 포트 번호(L4)도 변환하기 때문입니다. 이를 NAPT(Network Address and Port Translation) 또는 PAT(Port Address Translation)라고 부릅니다.

```
내부 192.168.1.100:54321 → 외부 1.2.3.4:80
       (사설 IP:Port)

공유기가 변환:
  Src IP: 192.168.1.100 → 공인 IP (예: 5.6.7.8)
  Src Port: 54321 → 새 포트 (예: 60001)

응답이 오면:
  Dst Port: 60001 → 찾아보기 → 원래 192.168.1.100:54321로 복원
```

포트 번호는 TCP 헤더(L4)에 있으므로, 공유기는 TCP 헤더를 열어보고 수정합니다. 따라서 NAT 라우터는 **L4까지 처리**하는 장비입니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: IP와 라우팅 ➡️](./02-ip-and-routing.md)**

</div>
