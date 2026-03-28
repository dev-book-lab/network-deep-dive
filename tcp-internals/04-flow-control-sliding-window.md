# TCP 흐름 제어 — Sliding Window와 수신 버퍼

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- TCP 흐름 제어와 혼잡 제어는 무엇이 다른가?
- 수신 버퍼(rwnd)는 어떻게 송신 속도를 제한하는가?
- Sliding Window 프로토콜은 Stop-and-Wait와 어떻게 다른가?
- Zero Window 상태는 언제 발생하고 Window Probe는 무엇인가?
- Window Scale 옵션은 왜 필요하고 SYN에서 어떻게 협상되는가?
- Wireshark에서 `tcp.window_size`로 흐름 제어 동작을 어떻게 확인하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"빠른 서버가 느린 클라이언트의 버퍼를 넘치게 하면 어떻게 되는가?"

  시나리오: Spring Boot 서버가 대용량 파일을 전송
  클라이언트: 저사양 기기, 처리 속도 느림
  
  흐름 제어 없다면:
    서버: 10Gbps 속도로 전송
    클라이언트: 처리 못한 패킷 → 버퍼 가득 → 패킷 드롭
    → 재전송 폭풍 → 오히려 더 느려짐
  
  TCP 흐름 제어:
    클라이언트: 수신 버퍼 여유를 ACK에 실어 전달 (rwnd)
    서버: rwnd 이내에서만 전송
    → 클라이언트 처리 능력에 맞게 자동 조절

실무 시나리오:
  1. 대용량 응답 (Spring Boot → 클라이언트):
     클라이언트가 느리면 Window가 작아짐
     → 서버의 전송 속도 자동 제한
     → 서버 측 Send Buffer에 데이터 누적
     → ss -ti에서 Send-Q 증가로 감지

  2. DB 연결에서 느린 쿼리 결과 수신:
     Spring → MySQL 대용량 결과셋
     Spring 처리 속도 < MySQL 전송 속도
     → Spring의 수신 버퍼(rwnd) 감소 → MySQL이 전송 속도 줄임

  3. Zero Window 상태 관찰:
     tcpdump에서 "win 0" 패킷 발견
     → 수신자 버퍼가 완전히 찬 상태
     → 애플리케이션 처리 지연 신호
```

---

## 😱 흔한 실수

```
Before — 흐름 제어를 모를 때:

실수 1: SO_RCVBUF/SO_SNDBUF를 무작정 크게 설정
  sysctl -w net.core.rmem_max=16777216  # 무조건 16MB
  → 수신 버퍼가 커지면 rwnd가 커짐
  → 고지연 환경에서는 효과적 (BDP = bandwidth × RTT)
  → 하지만 메모리 과다 사용, L4/L7 처리 지연 증가
  → 실제 링크 대역폭과 RTT를 기반으로 계산해야 함

실수 2: 애플리케이션이 소켓에서 느리게 읽음
  // 잘못된 패턴:
  InputStream in = socket.getInputStream();
  byte[] buf = new byte[1];
  while ((b = in.read(buf)) != -1) {  // 1바이트씩 읽기
      process(buf[0]);  // 느린 처리
  }
  → 수신 버퍼가 차면 Zero Window → 상대방 전송 중단
  → BufferedInputStream 또는 큰 버퍼로 읽어야 함

실수 3: Zero Window를 네트워크 문제로 오인
  tcpdump에서 "win 0" 발견
  → "네트워크 장비 문제" 판단 후 인프라 팀 문의
  → 실제 원인: 애플리케이션이 수신 버퍼를 빠르게 비우지 못함
```

---

## ✨ 올바른 접근

```
After — 흐름 제어를 알고 나면:

Zero Window 원인 진단:
  tcpdump에서 "win 0" 발견 시:
  → ss -ti로 수신 버퍼 확인
  → Recv-Q 값 확인 (애플리케이션이 읽지 않은 데이터)
  → Recv-Q가 계속 차 있으면 애플리케이션 처리 병목

고지연 환경 버퍼 최적화:
  # BDP(Bandwidth-Delay Product) 계산
  # RTT=100ms, 대역폭=1Gbps
  # BDP = 1Gbps × 0.1s = 100Mb = 12.5MB
  # → 수신 버퍼가 12.5MB 이상이어야 링크 최대 활용

  # Linux 자동 버퍼 조정 (권장)
  sysctl net.ipv4.tcp_rmem  # min default max
  # 기본: 4096 131072 6291456 (6MB max)
  
  # 고성능 환경:
  sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"
  sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
  sysctl -w net.core.rmem_max=16777216

Spring 애플리케이션 소켓 버퍼 설정:
  # application.properties
  server.tomcat.max-http-response-header-size=8192
  # NIO connector가 내부적으로 소켓 버퍼 관리
```

---

## 🔬 내부 동작 원리

### 1. Stop-and-Wait vs Sliding Window

```
Stop-and-Wait (비효율적):

  송신자: [세그먼트 1 전송] → 대기
  수신자:                    [ACK 1]
  송신자: [세그먼트 2 전송] → 대기
  수신자:                    [ACK 2]
  
  효율: 1 RTT마다 1개 세그먼트
  RTT=100ms, MSS=1460B → 처리량 = 1460 × 8 / 0.1 = 116 Kbps
  → 1Gbps 링크를 0.01%만 사용!

Sliding Window (실제 TCP):

  Window Size = 5 (5개 세그먼트를 ACK 없이 전송 가능)
  
  송신자:
  [1][2][3][4][5] → 한꺼번에 전송 (Window 꽉 채움)
  
  ACK=1 수신 → Window 슬라이딩 → [6] 전송 가능
  [2][3][4][5][6] → 계속 전송
  
  ACK=3 수신 → Window 슬라이딩 → [7][8] 전송 가능
  [4][5][6][7][8] → 계속 전송

  "파이프라인" 효과:
    네트워크에 항상 Window Size만큼의 세그먼트가 "날아다님"
    ACK를 기다리는 동안 다음 데이터 전송 → 링크 최대 활용

링크 활용률:
  Window Size = W (세그먼트 수)
  RTT = R
  MSS = M
  
  처리량 ≈ W × M / R
  
  W=100, RTT=100ms, MSS=1460B:
  처리량 = 100 × 1460 × 8 / 0.1 = 116 Mbps
  
  W=10000, RTT=100ms, MSS=1460B:
  처리량 = 10000 × 1460 × 8 / 0.1 = 11.6 Gbps (1Gbps 링크 포화!)
```

### 2. rwnd (Receive Window) — 수신자 버퍼 기반 흐름 제어

```
rwnd 동작:

  수신자의 수신 버퍼 = 8KB (예시)
  애플리케이션이 데이터를 읽는 속도 = 초당 4KB

  ┌──────────────────────────────────────────────────────────────┐
  │  수신 버퍼 (8KB)                                               │
  │  ┌──────────────┬────────────────────────────────────────┐   │
  │  │ 읽기 대기      │           여유 공간                      │   │
  │  │ (4KB)        │           (4KB)                        │   │
  │  └──────────────┴────────────────────────────────────────┘   │
  │   ← 애플리케이션이 가져감                                          │
  └──────────────────────────────────────────────────────────────┘
  
  ACK에 담기는 rwnd = 여유 공간 = 4KB
  → 송신자: 4KB 이상 보내지 않음

  애플리케이션이 데이터를 읽어가면:
  여유 공간 증가 → rwnd 증가 → ACK에 큰 rwnd 전달
  → 송신자 전송 속도 증가

  애플리케이션이 데이터를 읽지 않으면:
  버퍼 가득 참 → rwnd = 0 → Zero Window
  → 송신자 전송 중단

TCP 헤더의 Window 필드:
  16비트 → 최대 65535 bytes (64KB)
  고속 네트워크(기가비트)에서는 부족
  → Window Scale 옵션으로 확장
```

### 3. Window Scale — 65KB 한계 극복

```
문제: RTT=100ms, 대역폭=1Gbps
  최대 처리량 = Window Size / RTT
  Window Size max = 65535 bytes
  최대 처리량 = 65535 / 0.1 = 655 Kbps
  → 1Gbps 링크를 0.06%만 사용!

Window Scale 옵션 (RFC 1323):
  SYN 패킷에서 협상
  Scale Factor = S
  실제 Window = Window_field × 2^S

  SYN에서: options [wscale 7]  → Scale Factor = 7
  실제 Window = 65535 × 2^7 = 8,388,480 bytes ≈ 8MB

  최대 확장:
  Scale Factor 최대 = 14
  최대 Window = 65535 × 2^14 = 1,073,725,440 bytes ≈ 1GB

협상:
  SYN:     options [wscale 7]   ← 클라이언트 제안
  SYN-ACK: options [wscale 7]   ← 서버 제안
  → 이후 모든 패킷의 Window 필드에 Scale 적용

  어느 한쪽이 wscale을 보내지 않으면:
  → Scale Factor = 0 (확장 없음, 최대 65535)

tcpdump에서 확인:
  SYN 패킷:
  options [mss 1460,sackOK,TS val 12345 ecr 0,nop,wscale 7]
  → wscale 7 = Window × 128배 확장
```

### 4. Zero Window와 Window Probe

```
Zero Window 발생:

  수신자: 수신 버퍼 가득 참 → ACK의 Window 필드 = 0
  송신자: Window = 0 수신 → 전송 중단
  
  위험: 데드락
    수신자: Window가 열릴 때까지 대기 (ACK 안 보냄)
    송신자: Window Probe 없으면 영원히 대기

Zero Window 상태:
  ┌──────────────────────────────────────────────────────────────┐
  │  수신 버퍼 (8KB)                                               │
  │  ┌────────────────────────────────────────────────────────┐  │
  │  │              읽기 대기 데이터 (8KB 가득)                    │  │
  │  └────────────────────────────────────────────────────────┘  │
  │                                                              │
  │  여유 공간 = 0 → rwnd = 0                                      │
  └──────────────────────────────────────────────────────────────┘
  
  송신자: 전송 중단 → "Zero Window Timer" 시작

Window Probe (Zero Window Probe):
  Zero Window 수신 후 일정 시간(약 200ms~수 초) 경과 시:
  1바이트짜리 "탐색 패킷" 전송
  
  수신자 반응:
  버퍼 여유 생김 → Window > 0인 ACK → 송신 재개
  버퍼 아직 가득 → Window = 0인 ACK → 다시 대기 후 재시도
  
  Window Probe 간격: 지수적 증가 (처음: 200ms, 이후: 400ms, 800ms...)
  최대 약 75초 대기 후 연결 종료

Wireshark에서 확인:
  "[TCP ZeroWindow]"  → 수신자가 Window=0 알림
  "[TCP Window Full]" → 송신자가 Window를 꽉 채웠음 (곧 Zero Window)
  "[TCP ZeroWindowProbe]" → 송신자가 1바이트 탐색 패킷 전송

tcpdump에서 확인:
  패킷의 "win 0" → Window = 0
  "length 1" (Window Probe 크기)
```

### 5. Silly Window Syndrome (SWS)

```
문제: 작은 Window 반복 → 비효율

상황 1 — 수신자 SWS:
  수신 버퍼에 2바이트 여유 생김
  → ACK에 rwnd=2 전달
  → 송신자가 2바이트 전송 (헤더 40바이트 + 데이터 2바이트)
  → 40:2 비율 → 매우 비효율

상황 2 — 송신자 SWS:
  애플리케이션이 1바이트씩 write()
  → TCP가 매번 1바이트 패킷 전송
  → 헤더 오버헤드 절대적

해결책:

Clark's Algorithm (수신자 SWS 방지):
  수신 버퍼 여유가 max(MSS, 수신 버퍼의 절반) 미만이면
  → rwnd = 0 광고 (실제로는 여유 있어도)
  → 의미 있는 크기가 될 때까지 Window를 열지 않음

Nagle's Algorithm (송신자 SWS 방지):
  조건: 미확인 데이터 있음 AND 현재 데이터 < MSS
  → 전송 보류 (이전 ACK를 기다리면서 데이터 누적)
  → MSS 이상 모이거나, 미확인 데이터 없으면 전송

  단점: 지연 발생 (실시간 애플리케이션에 부적합)
  
  TCP_NODELAY = true: Nagle 알고리즘 비활성화
  → 즉시 전송, 처리량보다 지연 최소화 우선
  → 게임, 원격 터미널, 실시간 API에 사용
```

---

## 💻 실전 실험

### 실험 1: Window Size 변화 관찰

```bash
# 대용량 파일 다운로드 중 Window Size 모니터링
wget -q http://speedtest.example.com/100MB.bin -O /dev/null &

# 다른 터미널에서 Window Size 실시간 확인
sudo tcpdump -nn -i any 'host speedtest.example.com' -e 2>/dev/null | \
  awk '{for(i=1;i<=NF;i++) if($i~/^win/) print $i}' | head -50

# 또는 Wireshark에서:
# tcp.window_size 필터로 Window 크기 변화 그래프 확인
```

### 실험 2: Zero Window 재현

```bash
# 서버: 큰 데이터를 보내는 서버
python3 -c "
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('127.0.0.1', 9090))
s.listen(1)
conn, addr = s.accept()
# 1MB 데이터 전송
data = b'X' * 1024 * 1024
conn.sendall(data)
conn.close()
" &

# 클라이언트: 수신 버퍼를 작게 설정하고 천천히 읽기
python3 -c "
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 4096)  # 4KB 버퍼
s.connect(('127.0.0.1', 9090))
while True:
    data = s.recv(100)  # 100바이트씩 느리게 읽기
    if not data: break
    time.sleep(0.1)  # 처리 지연 시뮬레이션
" &

# tcpdump로 Zero Window 관찰
sudo tcpdump -nn -i lo 'port 9090' | grep -E "win [0-9]"
# win 4096 → win 2048 → win 1024 → win 0 감소하다가
# [ZeroWindowProbe] 패킷 발생
```

### 실험 3: ss로 버퍼 상태 실시간 모니터링

```bash
# 파일 다운로드 중
wget http://example.com/large-file -O /dev/null &

# 소켓 버퍼 실시간 확인
watch -n 0.2 "ss -ti 'dst example.com' | grep -E 'rcv_space|snd_wnd|rcv_wnd'"

# 출력에서:
# rcv_space: 수신 버퍼 총 크기
# rcv_wnd:   현재 수신 윈도우 (실제 rwnd)
# snd_wnd:   상대방이 광고한 수신 윈도우
# → snd_wnd가 0에 가까워지면 Zero Window 임박

# Send-Q 모니터링 (애플리케이션→OS 전달 대기)
ss -tan | grep ESTAB
# Send-Q 값이 지속적으로 크면:
# → 네트워크가 느리거나 수신자 Window가 작음
```

### 실험 4: BDP 계산 및 버퍼 최적화

```bash
# 현재 수신 버퍼 설정 확인
sysctl net.ipv4.tcp_rmem
# 출력: 4096 131072 6291456
# min: 4KB, default: 128KB, max: 6MB

# RTT 측정
ping -c 10 target-ip | tail -1
# rtt min/avg/max/mdev = 10.0/12.5/15.0/1.5 ms

# BDP 계산 (대역폭 1Gbps, RTT 12.5ms)
# BDP = 1Gbps × 0.0125s = 12.5Mb = 1.5625MB
# → 수신 버퍼 max가 1.5MB 이상이면 OK

# 고성능 설정
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 16777216"  # max 16MB
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 16777216"
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl -w net.core.wmem_max=16777216
```

---

## 📊 성능/비용 비교

```
Window Size와 처리량 관계:

RTT=100ms, MSS=1460 bytes

┌──────────────────────────────────────────────────────────────────┐
│  Window Size  │  최대 처리량           │  실제 사용 예                 │
├──────────────────────────────────────────────────────────────────┤
│  65KB (기본)   │  5.2 Mbps           │  Window Scale 없음          │
│  512KB        │  41 Mbps            │  wscale=3 (×8)             │
│  1MB          │  83 Mbps            │  wscale=4 (×16)            │
│  8MB          │  655 Mbps           │  wscale=7 (×128)           │
│  16MB         │  1.31 Gbps          │  wscale=8 (×256)           │
└──────────────────────────────────────────────────────────────────┘

같은 RTT, 같은 대역폭에서 Window 크기가 성능을 결정!

BDP(Bandwidth-Delay Product):
  최적 Window = 대역폭 × RTT

  1Gbps, RTT=1ms:  BDP = 125KB  (LAN)
  1Gbps, RTT=10ms: BDP = 1.25MB (국내)
  1Gbps, RTT=100ms: BDP = 12.5MB (해외)
  1Gbps, RTT=300ms: BDP = 37.5MB (위성)

  BDP < 수신 버퍼: 링크 100% 활용 가능
  BDP > 수신 버퍼: 버퍼가 병목 (Window Scale 필요)
```

---

## ⚖️ 트레이드오프

```
수신 버퍼 크기 조정:

버퍼 크게:
  장점: 고지연-고대역폭 환경에서 처리량 향상
        rwnd 크게 → 송신자가 더 많이 보낼 수 있음
  단점: 메모리 사용 증가
        소켓당 메모리 비용
        연결이 많은 서버에서는 총 메모리 크게 증가

버퍼 작게:
  장점: 메모리 절약
        빠른 버퍼 처리 (L1/L2 캐시 효율)
  단점: 고지연 환경에서 처리량 제한
        Zero Window 발생 빈도 증가

Linux Auto-Tuning (권장):
  tcp_moderate_rcvbuf = 1 (기본 활성화)
  → OS가 연결별로 버퍼를 동적 조정
  → RTT와 대역폭 측정 후 자동 최적화
  → 대부분의 환경에서 수동 설정보다 좋음

TCP_NODELAY vs Nagle:
  Nagle ON (기본):
    장점: 작은 패킷을 묶어서 전송 → 대역폭 효율
    단점: 최대 40ms 지연 (Delayed ACK와 결합 시)
  
  TCP_NODELAY:
    장점: 즉각 전송 → 지연 최소화
    단점: 작은 패킷이 많으면 헤더 오버헤드 증가
  
  HTTP/1.1: TCP_NODELAY 권장 (패킷 지연 방지)
  대용량 파일 전송: Nagle 유지 (처리량 최적화)
```

---

## 📌 핵심 정리

```
TCP 흐름 제어 핵심 요약:

흐름 제어 vs 혼잡 제어:
  흐름 제어: 수신자 버퍼 보호 (rwnd 기반)
  혼잡 제어: 네트워크 혼잡 방지 (cwnd 기반)
  실제 전송량 = min(rwnd, cwnd)

Sliding Window:
  Stop-and-Wait 대비: 파이프라인 효과 → 링크 활용률 극대화
  Window 내에서는 ACK 없이 연속 전송
  ACK 수신 시 Window 슬라이딩

rwnd (수신자 윈도우):
  수신 버퍼의 여유 공간
  ACK 패킷의 Window 필드로 전달
  애플리케이션이 데이터를 읽으면 증가

Window Scale:
  SYN에서 협상 (wscale 옵션)
  실제 Window = Window_field × 2^scale
  최대 2^14 = 16384배 확장 가능

Zero Window:
  수신 버퍼 가득 → rwnd = 0 → 송신 중단
  Window Probe: 1바이트 탐색 패킷으로 버퍼 여유 확인
  원인: 애플리케이션 처리 속도 < 수신 속도

진단 명령어:
  ss -ti: rcv_wnd, snd_wnd 확인
  ss -tan: Recv-Q (애플리케이션 미처리 데이터)
  tcpdump: "win 0" = Zero Window
  Wireshark: [TCP ZeroWindow], [TCP Window Full]
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Boot 서버에서 대용량 파일을 스트리밍으로 전송할 때, 클라이언트가 연결은 끊지 않고 데이터를 읽지 않고 있다. 서버에서는 어떤 현상이 관찰되는가?

<details>
<summary>해설 보기</summary>

다음 순서로 현상이 나타납니다:

1. **Send-Q 증가**: 서버 애플리케이션이 계속 write()를 호출하지만, 클라이언트의 rwnd가 0이 되면 OS의 송신 버퍼(Send Buffer)에 데이터가 쌓입니다.
   ```bash
   ss -tan | grep ESTAB  # Send-Q 값이 큰 경우
   ```

2. **Zero Window 상태**: 클라이언트 수신 버퍼가 가득 차면 rwnd=0을 보내옵니다. tcpdump에서 `win 0` 확인.

3. **서버 write() 블로킹**: 송신 버퍼도 가득 차면 `write()`/`send()`가 블로킹됩니다. Spring의 스트리밍 응답 스레드가 멈춥니다.

4. **Window Probe 발생**: 서버 OS가 주기적으로 1바이트 탐색 패킷을 보내 클라이언트 버퍼 여유를 확인합니다.

5. **Keep-Alive Timeout 초과 시**: 아무 활동이 없으면 keep-alive probe를 보내고 응답 없으면 연결을 종료합니다.

결론: 클라이언트가 읽지 않으면 서버 스레드가 묶이고 메모리가 낭비됩니다. 따라서 스트리밍 API에서는 클라이언트 응답 타임아웃을 적절히 설정해야 합니다.

</details>

---

**Q2.** 같은 1Gbps 네트워크에서 서울-서울(RTT=1ms)과 서울-뉴욕(RTT=150ms) 파일 전송의 실제 처리량 차이는 얼마나 나는가? Window Scale을 쓰지 않는 경우와 쓰는 경우를 비교하라.

<details>
<summary>해설 보기</summary>

**Window Scale 없음 (max Window = 65535 bytes):**

- 서울-서울 (RTT=1ms):
  처리량 = 65535 / 0.001 = 65.5 MB/s = 524 Mbps (링크의 52%)

- 서울-뉴욕 (RTT=150ms):
  처리량 = 65535 / 0.150 = 437 KB/s = 3.5 Mbps (링크의 0.35%)

  → 같은 1Gbps 링크에서 150배 차이!

**Window Scale 있음 (wscale=7, max Window = 8.4MB):**

- 서울-서울 (RTT=1ms):
  처리량 = 8.4MB / 0.001 = 8.4 GB/s → 1Gbps 링크가 병목, 실제 ≈ 1Gbps

- 서울-뉴욕 (RTT=150ms):
  처리량 = 8.4MB / 0.150 = 56 MB/s = 448 Mbps (링크의 44.8%)

  BDP = 1Gbps × 0.150s = 150Mb = 18.75MB
  → 8.4MB 버퍼는 BDP보다 작아 여전히 병목
  → wscale=8(256배) 적용 시: 16.7MB → 더 개선

**결론:** Window Scale이 없으면 고지연 환경에서 고속 링크를 전혀 활용하지 못합니다. 현대 OS는 기본적으로 Window Scale을 협상합니다.

</details>

---

**Q3.** TCP_NODELAY를 활성화하면 무조건 성능이 좋아지는가? 언제 오히려 나빠지는가?

<details>
<summary>해설 보기</summary>

**무조건 좋아지지 않습니다.** 상황에 따라 다릅니다.

**TCP_NODELAY가 유리한 경우:**
- 짧고 빈번한 요청/응답 (원격 터미널, 게임, 실시간 채팅)
- 요청마다 즉각적인 ACK가 필요한 경우
- Delayed ACK(40ms)와 Nagle이 결합할 때 발생하는 지연 제거

**TCP_NODELAY가 불리한 경우:**

1. **대용량 데이터 스트리밍**: Nagle이 작은 패킷을 묶어 전송하므로 헤더 오버헤드 감소. TCP_NODELAY를 켜면 MSS 미만의 작은 패킷을 즉시 전송 → 헤더:데이터 비율 악화.

2. **Write-Write-Read 패턴 문제**: 
   ```
   write(헤더 4바이트)  → 즉시 전송 (TCP_NODELAY)
   write(바디 1460바이트) → 즉시 전송
   → 2개의 패킷 = 불필요한 왕복
   
   Nagle이면: 묶어서 1개 패킷으로 전송
   ```

3. **소규모 버스트 트래픽**: 애플리케이션이 순간적으로 많은 작은 write()를 호출하면 패킷 수가 폭발적으로 증가.

**Spring Boot 기본값:** HTTP 통신이므로 TCP_NODELAY 활성화가 일반적으로 유리합니다. `server.tomcat.no-delay=true` (기본 true).

</details>

---

<div align="center">

**[⬅️ 이전: TCP 신뢰성](./03-tcp-reliability.md)** | **[홈으로 🏠](../README.md)** | **[다음: TCP 혼잡 제어 ➡️](./05-congestion-control.md)**

</div>
