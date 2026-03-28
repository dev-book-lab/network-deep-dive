# TCP vs UDP — 구조 차이와 UDP를 선택하는 조건

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- TCP 헤더와 UDP 헤더의 구조 차이는 무엇이고, 각각 어떤 기능 차이로 이어지는가?
- UDP가 "신뢰성이 없다"는 말의 정확한 의미는?
- 게임, DNS, 스트리밍이 UDP를 선택하는 구체적인 이유는?
- QUIC은 UDP 위에서 어떻게 신뢰성을 구현하는가?
- DNS가 UDP를 쓰면서도 실제로는 신뢰성을 어떻게 보장하는가?
- "TCP를 쓰면 항상 신뢰성이 있다"는 명제는 완전히 참인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"왜 DNS는 UDP를 쓰나요?":
  모르면: "빠르기 때문이요" (불완전한 답)
  알면: "DNS 쿼리는 512바이트 이하의 단일 요청-응답
         Handshake 비용(1.5 RTT)이 쿼리 자체보다 비쌈
         손실 시 애플리케이션이 타임아웃 후 재시도 → 재전송 구현
         → UDP가 최적"

"HTTP/3은 왜 UDP 기반인가요?":
  TCP의 Head-of-Line Blocking:
    TCP 연결에서 패킷 1개 손실 → 이후 모든 스트림 차단
    HTTP/2에서 100개 스트림이 1개 패킷 손실에 모두 블로킹
  
  QUIC(UDP 기반):
    스트림별 독립 재전송
    패킷 1개 손실 → 해당 스트림만 영향
    다른 99개 스트림은 정상 전송

"UDP는 무조건 빠르고 TCP는 무조건 느리다?"
  잘못된 이해:
    UDP 자체의 속도 차이는 헤더 크기 정도
    성능 차이는 TCP의 혼잡 제어, Handshake, HOL Blocking에서 옴
  
  실제:
    같은 조건에서 재전송 없다면 UDP ≈ TCP 속도
    고손실 환경: UDP 애플리케이션 재전송 vs TCP 재전송 비교
    → 어느 것이 빠른지는 구현에 달림
```

---

## 😱 흔한 실수

```
Before — TCP/UDP 차이를 모를 때:

실수 1: "UDP는 신뢰성이 없으니 중요한 데이터엔 TCP만 쓴다"
  과도한 일반화
  DNS(중요 인프라), NTP(시간 동기화), DHCP 모두 UDP 사용
  → UDP 위에서 신뢰성을 구현하는 것이 가능
  → 필요한 만큼만 신뢰성을 구현해서 오버헤드 최소화

실수 2: "게임에 UDP를 쓰면 패킷 손실이 발생한다"
  사실이지만 의도적 선택
  게임에서 패킷 손실 > 패킷 지연 (Lag)
  100ms 지연된 위치 정보 < 최신 위치 정보 (손실해도 OK)
  → TCP의 재전송 대기가 오히려 더 나쁜 경험
  → UDP + 상태 기반 보정이 최선

실수 3: "QUIC은 UDP라 신뢰성이 없다"
  UDP 위에서 TCP보다 더 정교한 신뢰성 구현
  패킷 번호, ACK, 재전송, SACK, 스트림별 흐름 제어 모두 QUIC 레이어에서 처리
```

---

## ✨ 올바른 접근

```
After — 올바른 프로토콜 선택 기준:

프로토콜 선택 질문:
  Q1. 모든 데이터가 순서대로 정확히 전달되어야 하는가?
      예 → TCP 또는 QUIC
      아니오(게임 위치 업데이트, 스트리밍) → UDP

  Q2. 연결 수립 오버헤드가 성능 요구사항에 맞는가?
      Handshake = 1.5 RTT + TLS = 3 RTT (TLS 1.2)
      아니오(DNS, NTP, 초소형 쿼리) → UDP

  Q3. 패킷 손실 발생 시 어떻게 처리할 것인가?
      TCP: OS가 자동 처리
      UDP: 애플리케이션이 직접 구현 (필요한 만큼만)

  Q4. Head-of-Line Blocking이 문제인가?
      다중 스트림, 손실 환경 → QUIC (UDP 기반)
      단일 스트림, 안정적 환경 → TCP로 충분

실무 선택:
  HTTP REST API       → TCP (HTTP/1.1, HTTP/2)
  고성능 웹 서비스     → QUIC (HTTP/3)
  DNS 쿼리            → UDP (응답 512B 이하)
  대용량 DNS 응답      → TCP (EDNS0, DNSSEC)
  실시간 게임          → UDP + 자체 신뢰성
  스트리밍 (지연 민감) → UDP (RTP over UDP)
  스트리밍 (품질 우선) → TCP (HLS, DASH)
```

---

## 🔬 내부 동작 원리

### 1. TCP 헤더 vs UDP 헤더

```
TCP 헤더 (최소 20 bytes):
┌─────────────────────────────────────────────────────────────────┐
│  Source Port (16) │  Destination Port (16)                      │
├─────────────────────────────────────────────────────────────────┤
│                Sequence Number (32)                             │
├─────────────────────────────────────────────────────────────────┤
│              Acknowledgment Number (32)                         │
├─────────────────────────────────────────────────────────────────┤
│  Data Offset(4) │ Reserved │ Flags(6) │    Window Size (16)     │
├─────────────────────────────────────────────────────────────────┤
│    Checksum (16)           │    Urgent Pointer (16)             │
├─────────────────────────────────────────────────────────────────┤
│                    Options (가변, 최대 40 bytes)                  │
└─────────────────────────────────────────────────────────────────┘

Flags (6 bits):
  URG: Urgent Pointer 유효
  ACK: Acknowledgment 유효
  PSH: 즉시 전달 요청
  RST: 연결 리셋
  SYN: 연결 요청
  FIN: 연결 종료

모든 필드의 목적:
  Sequence/Ack: 신뢰성 (손실/중복/순서)
  Flags:        연결 상태 관리
  Window Size:  흐름 제어
  Options:      MSS, SACK, Timestamps, Window Scale

───────────────────────────────────────────────────────────────────

UDP 헤더 (고정 8 bytes):
┌─────────────────────────────────────────────────────────────────┐
│  Source Port (16)          │  Destination Port (16)             │
├─────────────────────────────────────────────────────────────────┤
│  Length (16)               │  Checksum (16)                     │
└─────────────────────────────────────────────────────────────────┘

4개 필드가 전부:
  Source Port:  출발지 포트
  Dest Port:    목적지 포트
  Length:       UDP 헤더 + 데이터 전체 크기
  Checksum:     오류 감지 (선택적, IPv4에서는 0으로 비활성화 가능)

없는 것들:
  Sequence Number → 순서 보장 없음
  Acknowledgment  → 수신 확인 없음
  Window Size     → 흐름 제어 없음
  Flags           → 연결 상태 없음
  
→ "연결"이라는 개념 자체가 없음 (Connectionless)
→ 전송하고 잊어버림 (Fire and Forget)
```

### 2. UDP의 "신뢰성 없음"의 정확한 의미

```
UDP가 보장하지 않는 것:
  ① 전달 보장 없음: 패킷이 사라져도 모름
  ② 순서 보장 없음: 1,2,3으로 보냈는데 3,1,2로 도착 가능
  ③ 중복 없음 보장 없음: 같은 패킷이 두 번 도착 가능

UDP가 제공하는 것:
  ① 포트 기반 다중화/역다중화: 어느 프로세스에게 전달할지
  ② 체크섬: 오류 감지 (수정은 불가)
  ③ 데이터그램 경계 보존: TCP와 달리 메시지 단위 유지

UDP Checksum:
  IPv4: 선택적 (0으로 설정하면 체크섬 비활성화)
  IPv6: 필수
  
  체크섬 비활성화 이유:
    로컬 네트워크(LAN)에서는 Ethernet FCS가 이미 오류 감지
    불필요한 CPU 오버헤드 제거 (1990년대 기준)
  
  현대: 대부분 체크섬 사용 (오류 감지 비용 < 데이터 손상 비용)

"Best-Effort Delivery":
  IP도 Best-Effort, TCP가 그 위에 신뢰성 추가
  UDP도 Best-Effort, 신뢰성 추가 없음
  → 신뢰성이 필요하면 애플리케이션이 직접 구현
```

### 3. DNS — UDP 위의 신뢰성

```
DNS가 UDP를 쓰는 이유:
  전형적인 DNS 쿼리: 28 bytes (UDP 헤더 8 + DNS 쿼리 20)
  전형적인 DNS 응답: 60~200 bytes
  
  만약 TCP를 쓴다면:
    3-Way Handshake (1.5 RTT)
    DNS 쿼리 전송 (0.5 RTT)
    DNS 응답 수신
    4-Way Handshake (2 RTT)
    총: 4 RTT
  
  UDP를 쓰면:
    DNS 쿼리 전송 (0 RTT, 단방향)
    DNS 응답 수신
    총: 0.5 RTT (단일 왕복)

DNS가 신뢰성을 보장하는 방법:
  ① 타임아웃 + 재시도:
     /etc/resolv.conf:
     options timeout:2 attempts:3
     → 2초 대기 후 응답 없으면 재전송, 최대 3회
  
  ② 트랜잭션 ID (16비트):
     각 쿼리에 무작위 ID 포함
     응답의 ID가 쿼리 ID와 일치해야 처리
     → 늦은 응답이나 중간자 공격 방어

  ③ 응답이 512바이트 초과 시 TCP 사용:
     "TC" (Truncated) 비트 = 1 → 클라이언트가 TCP로 재시도
     DNSSEC, EDNS0 → 큰 응답 → TCP로 전환

DNS over UDP 패킷 분석:
  sudo tcpdump -nn -i any 'port 53'
  
  # 쿼리:
  # 15:00:01 IP 192.168.1.100.54321 > 8.8.8.8.53: UDP
  #           53+ A? example.com. (29)
  # 53+: 트랜잭션 ID=53, +는 재귀 쿼리 요청
  
  # 응답:
  # 15:00:01 IP 8.8.8.8.53 > 192.168.1.100.54321: UDP
  #           53 1/0/0 A 93.184.216.34 (45)
  # 53: 같은 트랜잭션 ID, 1 Answer, 0 Authority, 0 Additional
```

### 4. QUIC — UDP 위에서 TCP보다 더 잘

```
QUIC (RFC 9000):
  구글이 개발, 표준화
  UDP 위에서 동작
  HTTP/3의 전송 계층

QUIC이 TCP보다 나은 점:
┌───────────────────────────────────────────────────────────────────┐
│  기능                  │  TCP              │  QUIC                 │
├───────────────────────────────────────────────────────────────────┤
│  연결 수립              │  1.5 RTT          │  1 RTT (또는 0-RTT)    │
│  암호화 통합             │  TCP + TLS 별도    │  TLS 1.3 내장          │
│  HOL Blocking         │  있음(스트림 전체)    │  없음(스트림별)          │
│  연결 마이그레이션        │  IP 변경 시 끊김     │  Connection ID로       │
│                       │                   │  유지 (WiFi→LTE)       │
│  헤더 압축              │  없음(HTTP/2 HPACK │  QPACK                │
│                       │  은 별도)           │  (QUIC 네이티브)        │
└───────────────────────────────────────────────────────────────────┘

QUIC의 신뢰성 구현:
  Packet Number (단방향 증가):
    TCP Sequence Number와 달리 재전송 시에도 새 번호 사용
    → Karn's Algorithm 불필요 (재전송 모호성 없음)
    → RTT 측정 정확도 향상

  Stream 기반 다중화:
    각 Stream이 독립적인 Sequence Number
    Stream 1 패킷 손실 → Stream 2,3,4는 계속 진행
    → TCP의 연결 수준 HOL Blocking 없음

  ACK 확인:
    QUIC ACK 프레임: SACK와 유사하게 불연속 범위 알림
    큰 ACK 블록 지원 (TCP SACK는 최대 4 블록)

0-RTT 연결 재개:
  이전 세션의 서버 설정(Resumption Token) 저장
  다음 연결에서 TLS 협상 없이 즉시 데이터 전송
  
  주의: 재전송 공격(Replay Attack) 위험
  → 멱등한 요청(GET)에만 0-RTT 적용 권장

Connection Migration:
  모바일 환경에서 Wi-Fi → LTE 전환 시
  TCP: IP 주소 변경 → 연결 끊김 → 재연결
  QUIC: Connection ID로 연결 유지
        IP가 바뀌어도 같은 연결 계속 사용
        → 모바일 환경에서 큰 장점
```

### 5. 게임/스트리밍에서 UDP 선택 이유

```
실시간 게임:
  요구사항: 낮은 지연, 높은 빈도 업데이트
  
  TCP가 문제인 이유:
    플레이어 위치 업데이트: 초당 60번 전송
    패킷 1개 손실 → TCP 재전송 대기 (최소 1 RTT)
    → 재전송 중에는 이후 패킷도 받지 못함
    → 체감: 0.1초 멈춤 (매우 눈에 띔)
  
  UDP 해결책:
    패킷 손실 → 그냥 버림 (이미 오래된 정보)
    다음 패킷에 현재 위치 담아서 전송
    → 1프레임 손실해도 다음 프레임에 자동 보정
  
  게임 전용 재전송 로직:
    중요 이벤트(아이템 획득, 피격)만 ACK 요구
    위치 업데이트는 손실 허용

VoIP/화상통화:
  RTP (Real-time Transport Protocol) over UDP
  오디오/비디오: 손실이 잡음으로 처리됨 (재전송보다 나음)
  재전송 데이터는 이미 늦어서 쓸모없음
  → 재전송 없이 그냥 전송

IPTV/스트리밍:
  HLS, DASH (TCP 기반): 버퍼링으로 품질 유지
  IPTV (UDP 기반): 낮은 지연 (라이브 방송)
  선택 기준: 지연 vs 품질 트레이드오프
```

---

## 💻 실전 실험

### 실험 1: TCP와 UDP 헤더 크기 직접 확인

```bash
# TCP 패킷 캡처 (HTTP)
sudo tcpdump -nn -i any 'tcp and port 80 and host example.com' -vv 2>/dev/null &
curl -s http://example.com > /dev/null
# 출력에서 "length" 확인: TCP 헤더 20bytes + 옵션 20bytes = 40bytes 오버헤드

# UDP 패킷 캡처 (DNS)
sudo tcpdump -nn -i any 'udp and port 53' -vv 2>/dev/null &
dig example.com
# 출력에서 UDP 헤더: 8bytes만
# 쿼리 총 크기: ~37 bytes (UDP 8 + DNS 쿼리 29)
```

### 실험 2: DNS UDP vs TCP 직접 비교

```bash
# DNS UDP (기본)
time dig @8.8.8.8 example.com
# 총 시간: 약 30~60ms

# DNS TCP 강제
time dig @8.8.8.8 +tcp example.com
# 총 시간: 더 길어짐 (Handshake 포함)

# 트랜잭션 ID 확인
sudo tcpdump -nn -i any 'port 53 and host 8.8.8.8' &
dig @8.8.8.8 example.com
# 쿼리와 응답의 ID가 동일한지 확인
```

### 실험 3: UDP 손실률 실험

```bash
# iperf3로 UDP 처리량과 손실률 측정
# 서버
iperf3 -s

# 클라이언트 (UDP 모드, 100Mbps 목표)
iperf3 -c server-ip -u -b 100M -t 30
# 출력:
# [ ID] ... 0.00-30.00 sec  357 MBytes  99.9 Mbits/sec  1.23%  (datagrams)
# → 1.23% 손실률 표시 (TCP는 재전송으로 0%를 유지)

# 손실률 5% 주입 후 비교
sudo tc qdisc add dev eth0 root netem loss 5%

# UDP: 손실률 그대로 5% 반영
iperf3 -c server-ip -u -b 100M -t 10

# TCP: 재전송으로 손실 숨김 (대신 처리량 감소)
iperf3 -c server-ip -t 10

sudo tc qdisc del dev eth0 root
```

### 실험 4: QUIC (HTTP/3) 직접 확인

```bash
# curl로 HTTP/3 요청
curl -v --http3 https://cloudflare.com 2>&1 | head -30
# "using HTTP/3" 표시 확인

# QUIC 패킷 캡처
sudo tcpdump -nn -i any 'udp and port 443 and host 1.1.1.1' &
curl --http3 https://1.1.1.1 -o /dev/null

# QUIC은 UDP 443을 사용
# 암호화됐지만 UDP 데이터그램임을 확인

# HTTP/2(TCP)와 시간 비교
curl -w "%{time_connect} %{time_appconnect} %{time_total}\n" \
     --http2 -o /dev/null -s https://cloudflare.com

curl -w "%{time_connect} %{time_appconnect} %{time_total}\n" \
     --http3 -o /dev/null -s https://cloudflare.com
# HTTP/3이 특히 time_appconnect 이후 빠를 수 있음
```

---

## 📊 성능/비용 비교

```
TCP vs UDP 오버헤드 비교:

헤더 크기:
  TCP: 20 bytes (최소) + 옵션 최대 40 bytes = 최대 60 bytes
  UDP: 8 bytes (고정)

작은 메시지(100 bytes)에서 헤더 비율:
  TCP: 60 / (60+100) = 37.5% 오버헤드
  UDP: 8 / (8+100)  = 7.4% 오버헤드

연결 수립 비용:
  TCP: 3-Way Handshake = 1.5 RTT
  UDP: 없음
  
  RTT=50ms 환경, 초당 1000 쿼리:
    TCP(새 연결마다): 75ms × 1000 = 75초 (불가능)
    UDP:              없음

재전송 지연:
  TCP: RTO 후 재전송 (최소 200ms)
  UDP: 애플리케이션 결정 (예: 500ms 후 재시도, 또는 포기)

처리량 (안정적 네트워크):
  TCP ≈ UDP (헤더 오버헤드 차이만 존재)

처리량 (고손실 네트워크, 5% 손실):
  TCP: AIMD로 cwnd 줄임 → 처리량 약 70% 감소
  UDP: 손실 그대로 → 처리량은 유지, 5% 데이터 손실
  UDP+재전송: 선택적 재전송 → TCP보다 효율적 가능
```

---

## ⚖️ 트레이드오프

```
TCP 선택 시:
  장점:
    신뢰성 보장 (OS가 처리)
    순서 보장
    흐름/혼잡 제어 내장
    대부분의 방화벽 통과
  
  단점:
    Handshake 오버헤드 (1.5 RTT)
    HOL Blocking
    연결 마이그레이션 불가 (IP 변경 시 끊김)
    모든 재전송을 기다려야 함

UDP 선택 시:
  장점:
    저지연 (Handshake 없음)
    손실 허용 가능한 데이터에 최적
    신뢰성을 필요한 만큼만 구현 가능
    멀티캐스트/브로드캐스트 지원
  
  단점:
    신뢰성을 직접 구현해야 함
    혼잡 제어 부재 → 네트워크에 공격적
    일부 방화벽이 UDP를 차단 (포트 443 UDP도)

QUIC 선택 시:
  장점:
    0-RTT/1-RTT 연결 (TCP+TLS의 3 RTT 대비)
    스트림별 HOL Blocking 없음
    연결 마이그레이션 (모바일 환경)
    신뢰성 + 암호화 통합
  
  단점:
    방화벽이 UDP 443을 차단하는 경우 fallback 필요
    CPU 오버헤드: TLS 등을 사용자 공간에서 처리
    아직 일부 네트워크 장비에서 최적화 부족
```

---

## 📌 핵심 정리

```
TCP vs UDP 핵심 요약:

헤더 차이:
  TCP: 20~60 bytes (Seq, ACK, Flags, Window, Options)
  UDP: 8 bytes (Port, Length, Checksum만)

TCP가 제공하는 것 (UDP에 없는 것):
  연결 수립/종료 (3-Way, 4-Way Handshake)
  신뢰성 (Seq/ACK, 재전송)
  순서 보장 (Seq Number로 재조합)
  흐름 제어 (rwnd)
  혼잡 제어 (cwnd, AIMD)

UDP 선택 조건:
  손실 허용 (게임 위치, 스트리밍)
  낮은 지연 필수 (VoIP, 게임)
  단발성 쿼리 (DNS, DHCP, NTP)
  Handshake 비용이 데이터보다 클 때
  직접 신뢰성 구현이 필요할 때 (QUIC)

QUIC:
  UDP 기반이지만 TCP보다 더 풍부한 신뢰성
  스트림별 독립 재전송 → HOL Blocking 없음
  TLS 1.3 내장 → 1 RTT 연결 수립
  Connection ID → IP 변경 시 연결 유지

DNS의 선택:
  UDP: 작은 쿼리/응답, Handshake 없음
  TCP: 응답 512B 초과 시 (TC 비트)
  트랜잭션 ID로 응답 검증
```

---

## 🤔 생각해볼 문제

**Q1.** DNS 쿼리가 UDP를 사용하는데 DNS 응답이 위조될 가능성은 없는가? 어떻게 방어하는가?

<details>
<summary>해설 보기</summary>

**DNS Spoofing이 가능한 이유:**
UDP는 연결이 없으므로 공격자가 올바른 트랜잭션 ID와 소스 포트를 맞춘 위조 응답을 먼저 보내면 클라이언트가 그것을 받아들일 수 있습니다.

**Kaminsky Attack (2008):**
트랜잭션 ID (16비트, 65536가지)와 소스 포트를 무차별 대입으로 맞춰 DNS 캐시를 오염시키는 공격. 이후 대부분의 DNS 서버가 소스 포트를 무작위로 선택하도록 패치됐습니다.

**방어 방법:**

1. **무작위 소스 포트**: UDP 포트도 무작위 → 트랜잭션 ID × 포트 번호 조합 = 2^32가지
2. **DNSSEC**: DNS 응답에 디지털 서명 추가. 공개키로 서명 검증 → 위조 불가
3. **DNS over TLS (DoT)**: TCP + TLS → 중간자 공격 불가
4. **DNS over HTTPS (DoH)**: HTTPS로 DNS 쿼리 → 암호화 + 방화벽 우회

현실적 선택:
- 내부 네트워크: 신뢰할 수 있는 DNS 서버 + 소스 포트 랜덤화
- 외부/공용: DoH (Cloudflare 1.1.1.1, Google 8.8.8.8 지원)
- 중요 인프라: DNSSEC + DoT/DoH 병행

</details>

---

**Q2.** "TCP를 사용하면 데이터가 반드시 전달된다"는 것은 완전히 참인가?

<details>
<summary>해설 보기</summary>

**완전히 참이 아닙니다.** TCP는 "전달되거나 오류를 보고하거나"를 보장하지, "반드시 전달"을 보장하지 않습니다.

**TCP가 데이터를 전달하지 못하는 상황:**

1. **연결 타임아웃**: 네트워크가 너무 불안정해서 재전송이 모두 실패하면 연결이 끊어집니다 (`tcp_retries2` 기본 15회 후 포기).

2. **RST에 의한 강제 종료**: 중간 방화벽이나 NAT가 세션을 강제로 끊으면 미전송 데이터가 사라집니다.

3. **Send Buffer에 있는 데이터**: `close()`를 호출해도 OS Send Buffer에 있는 데이터가 ACK를 받기 전에 연결이 끊어지면 손실됩니다.

4. **SO_LINGER(0)**: RST로 즉시 종료하면 Send Buffer의 데이터가 폐기됩니다.

5. **시스템 크래시**: OS가 죽으면 메모리의 모든 버퍼 데이터가 사라집니다.

**TCP가 보장하는 것:**
- 전달되면 순서대로, 중복 없이 전달됨
- 전달 실패 시 (일정 재시도 후) 오류 보고 (연결 끊김 또는 오류 코드)

**애플리케이션 수준의 신뢰성 보장:** 중요한 데이터는 애플리케이션 레벨 ACK (예: HTTP 200 응답, 메시지 큐의 명시적 ack)를 받을 때까지 재전송할 준비를 해야 합니다.

</details>

---

**Q3.** QUIC이 UDP 기반인데 일부 방화벽에서 차단될 수 있다. HTTP/3 클라이언트는 이 상황을 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**HTTP/3의 Fallback 메커니즘:**

1. **ALPN (Application-Layer Protocol Negotiation)**: TLS 핸드쉐이크에서 지원하는 프로토콜을 협상합니다.

2. **Alt-Svc 헤더**: 서버가 HTTP/2 응답에 `Alt-Svc: h3=":443"` 헤더를 포함해 HTTP/3 지원을 알립니다.

3. **Connection Coalescing**: 브라우저가 처음에는 HTTP/2(TCP)로 연결 후 서버의 Alt-Svc를 받으면 다음 요청부터 HTTP/3 시도.

4. **Happy Eyeballs 유사 방식**: TCP와 UDP 연결을 병렬로 시도해 먼저 성공하는 것 사용.

**실제 Chrome의 동작:**
```
1. TCP:443으로 HTTP/2 연결 시도
2. 서버: Alt-Svc: h3=":443" 헤더 반환
3. Chrome: UDP:443으로 QUIC 연결 시도
4. QUIC 성공 → HTTP/3 사용
   QUIC 실패 (방화벽 차단) → HTTP/2 유지
```

**방화벽 차단 빈도:**
- UDP 443을 차단하는 네트워크: 약 3~7% (기업 방화벽, 일부 ISP)
- 이 경우 자동으로 HTTP/2로 fallback
- 사용자는 차이를 모름

**Cloudflare 통계:** HTTP/3 지원 환경에서 HTTP/2 대비 약 15-17% 빠른 로딩 시간.

</details>

---

<div align="center">

**[⬅️ 이전: 혼잡 제어](./05-congestion-control.md)** | **[홈으로 🏠](../README.md)** | **[다음: TCP 소켓 상태 머신 ➡️](./07-tcp-socket-state-machine.md)**

</div>
