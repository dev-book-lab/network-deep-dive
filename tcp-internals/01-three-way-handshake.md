# TCP 연결 수립 — 3-Way Handshake가 왜 3번인가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- SYN/SYN-ACK/ACK 각 패킷이 실제로 어떤 정보를 교환하는가?
- 2-Way Handshake로는 왜 충분하지 않은가?
- ISN(Initial Sequence Number)은 무엇이고 왜 무작위로 선택하는가?
- MSS(Maximum Segment Size)와 Window Scale은 어느 시점에 협상되는가?
- `tcpdump`로 Handshake 패킷을 캡처했을 때 각 필드를 어떻게 해석하는가?
- Spring RestTemplate/WebClient가 요청마다 Handshake를 하면 어떤 비용이 발생하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"커넥션 풀을 왜 써야 하나요?" 에 제대로 답하려면:

  HTTP 요청 1번 = TCP 연결 1번 (커넥션 풀 없을 때)
  TCP 연결 1번 = 3-Way Handshake = 1.5 RTT 비용

  서울 → 미국 RTT ≈ 150ms
  Handshake 비용 = 150ms * 1.5 = 225ms
  HTTPS면 TLS까지 추가 = 225ms + 150ms = 375ms

  API 요청 자체가 10ms라도 연결 수립에 375ms
  → 요청의 97%가 연결을 맺는 데 소비됨

  Spring RestTemplate 기본 설정:
    매 요청마다 새 TCP 연결 → Handshake 반복
  
  올바른 설정 (Connection Pool):
    연결을 재사용 → Handshake는 최초 1회만
    → 응답시간 수십~수백 ms 단축

  Handshake를 모르면:
    "왜 이렇게 느리죠?" → 이유 설명 불가
    "커넥션 풀이 왜 필요한가요?" → 설명 불가
    "maxConnectionsPerRoute를 얼마로 설정해야 하나요?" → 기준 없음
```

---

## 😱 흔한 실수

```
Before — Handshake를 모를 때의 접근:

실수 1: RestTemplate 매 요청마다 new로 생성
  @Service
  public class ApiService {
      public String call() {
          RestTemplate rt = new RestTemplate(); // ← 매번 생성
          return rt.getForObject(url, String.class);
      }
  }
  → 매 요청마다 TCP 연결 + Handshake + TLS 핸드쉐이크
  → 연결 비용이 실제 처리 비용을 압도

실수 2: 외부 API 타임아웃 설정 시 Handshake 시간 미고려
  connectTimeout = 100ms 설정
  서버까지 RTT = 80ms → Handshake = 120ms
  → 항상 timeout → 원인 파악 못함
  → Handshake = 1.5 RTT임을 알면 connectTimeout 설계 가능

실수 3: 로컬 테스트와 운영 성능 차이 이해 못함
  localhost: RTT ≈ 0.1ms → Handshake = 0.15ms (무시 가능)
  운영 서버: RTT ≈ 30ms  → Handshake = 45ms (눈에 띄는 비용)
  → "로컬에서는 빠른데 운영에서 왜 느리죠?" 혼란
```

---

## ✨ 올바른 접근

```
After — Handshake를 알고 나면:

Spring RestTemplate Connection Pool 올바른 설정:
  @Bean
  public RestTemplate restTemplate() {
      HttpComponentsClientHttpRequestFactory factory =
          new HttpComponentsClientHttpRequestFactory();
      
      // Handshake 비용 = 1.5 RTT → connectTimeout은 RTT * 2 이상
      factory.setConnectTimeout(3000);   // TCP Handshake 대기
      factory.setReadTimeout(10000);     // 데이터 수신 대기
      
      // 연결 재사용 (Handshake 비용 제거)
      PoolingHttpClientConnectionManager cm =
          new PoolingHttpClientConnectionManager();
      cm.setMaxTotal(200);
      cm.setDefaultMaxPerRoute(50);
      
      return new RestTemplate(factory);
  }

WebClient (Reactor Netty) 설정:
  ConnectionProvider provider = ConnectionProvider.builder("custom")
      .maxConnections(500)
      .pendingAcquireMaxCount(1000)
      .maxIdleTime(Duration.ofSeconds(20))  // Keep-Alive 유지 시간
      .build();

커넥션 풀 효과:
  풀 없음: 요청 100개 → Handshake 100번 (RTT * 1.5 * 100)
  풀 있음: 요청 100개 → Handshake ~5번 (풀 초기화 시만)
```

---

## 🔬 내부 동작 원리

### 1. 3-Way Handshake 전체 흐름

```
Client (Spring App)              Server (API Server)
        │                                │
        │  ① SYN                         │
        │  seq=x, SYN=1                  │
        │ ─────────────────────────────► │
        │                                │  상태: SYN_RCVD
        │  ② SYN-ACK                     │
        │  seq=y, ack=x+1, SYN=1, ACK=1  │
        │ ◄───────────────────────────── │
상태:    │                                │
ESTABLISHED  ③ ACK                      │
        │  seq=x+1, ack=y+1, ACK=1       │
        │ ─────────────────────────────► │  상태: ESTABLISHED
        │                                │
        │  이제 데이터 전송 가능              │
        │ ─────────────────────────────► │

소요 시간: 1.5 RTT
  ① SYN 전송 ~ ② SYN-ACK 수신: 1 RTT
  ② SYN-ACK 수신 ~ ③ ACK 전송: 0 (즉시)
  하지만 서버는 ③ ACK를 받기까지 0.5 RTT 추가 대기
  → 클라이언트: 1 RTT 후 데이터 전송 가능
  → 서버: 1.5 RTT 후 ESTABLISHED 확정
```

### 2. 왜 2-Way로는 부족한가

```
2-Way Handshake 시도:

Client                    Server
  │  SYN (seq=x)            │
  │ ──────────────────────► │
  │SYN-ACK (seq=y, ack=x+1) │
  │ ◄────────────────────── │
  │  [연결 수립 선언]          │
  │ ──────────────────────► │ ← 이 ACK 없으면?

문제 1: 서버가 클라이언트의 수신 능력을 확인 못함
  서버는 SYN-ACK를 보냈지만
  클라이언트가 받았는지 모름
  → 클라이언트가 보내는 ACK(③)로 확인

문제 2: 오래된 SYN 패킷으로 인한 가짜 연결
  시나리오:
    ① 클라이언트가 SYN 전송 → 네트워크에서 지연
    ② 클라이언트가 타임아웃 → 다시 SYN 전송 → 정상 연결
    ③ 정상 연결 종료
    ④ 지연됐던 ① SYN이 뒤늦게 서버에 도착
    ⑤ 서버: SYN-ACK 전송 → 새 연결 대기
    
    2-Way라면: 서버가 이 오래된 SYN으로 연결을 수립함
    3-Way라면: 클라이언트가 예상하지 않은 SYN-ACK를 받음
              → ISN 불일치 → RST 전송 → 가짜 연결 거부

문제 3: 양방향 ISN 동기화 불가
  클라이언트 ISN(x)을 서버가 알아야 함 → ① SYN
  서버 ISN(y)을 클라이언트가 알아야 함 → ② SYN-ACK
  서버가 클라이언트의 ISN 수신 확인   → ③ ACK
  → 2-Way로는 서버 ISN을 클라이언트가 수신했다는 확인 불가
```

### 3. SYN 패킷 내부 구조

```
① SYN 패킷 (Client → Server):

TCP Header:
┌──────────────────────────────────────────────────────────────┐
│  Source Port: 54321       │  Destination Port: 443           │
├──────────────────────────────────────────────────────────────┤
│  Sequence Number: 1234567890  (ISN — 무작위 선택)               │
├──────────────────────────────────────────────────────────────┤
│  Acknowledgment Number: 0  (아직 받은 것 없음)                   │
├──────────────────────────────────────────────────────────────┤
│  Data Offset │ Flags: SYN=1, 나머지=0                          │
├──────────────────────────────────────────────────────────────┤
│  Window Size: 65535  (내 수신 버퍼 크기)                         │
├──────────────────────────────────────────────────────────────┤
│  Checksum │ Urgent Pointer                                   │
├──────────────────────────────────────────────────────────────┤
│  Options:                                                    │
│    MSS: 1460              ← 내가 받을 수 있는 최대 세그먼트 크기     │
│    SACK Permitted         ← Selective ACK 지원 여부            │
│    Timestamps             ← RTT 측정용                         │
│    Window Scale: 7        ← Window Size를 2^7=128배로 확장      │
└──────────────────────────────────────────────────────────────┘

ISN(Initial Sequence Number) 무작위 선택 이유:
  예측 가능한 ISN = 보안 취약점
  공격자가 ISN을 예측하면:
    연결에 끼어들어 위조 패킷 삽입 가능 (TCP Session Hijacking)
  → RFC 793: ISN은 전역 시계(clock) 기반
  → 현대: 암호학적 난수 + 호스트/포트 기반 해시
```

### 4. SYN-ACK 패킷 내부 구조

```
② SYN-ACK 패킷 (Server → Client):

┌──────────────────────────────────────────────────────────────┐
│  Source Port: 443         │  Destination Port: 54321         │
├──────────────────────────────────────────────────────────────┤
│  Sequence Number: 9876543210  (서버의 ISN — 무작위)             │
├──────────────────────────────────────────────────────────────┤
│  Acknowledgment Number: 1234567891  (클라이언트 ISN + 1)        │
│    → "네 ISN 받았어. 다음엔 seq=1234567891부터 보내"               │
├──────────────────────────────────────────────────────────────┤
│  Flags: SYN=1, ACK=1                                         │
│    SYN=1: 나도 내 ISN을 알려주는 중                               │
│    ACK=1: 동시에 네 SYN을 확인하는 중                              │
├──────────────────────────────────────────────────────────────┤
│  Window Size: 65535  (서버의 수신 버퍼 크기)                      │
├──────────────────────────────────────────────────────────────┤
│  Options:                                                    │
│    MSS: 1460              ← 서버가 받을 수 있는 최대 크기           │
│    SACK Permitted                                            │
│    Window Scale: 7                                           │
└──────────────────────────────────────────────────────────────┘

실제 MSS 협상:
  클라이언트 MSS: 1460 (Ethernet MTU 1500 - IP 20 - TCP 20)
  서버 MSS: 1460
  사용 MSS = min(1460, 1460) = 1460
  → 각자의 MSS 중 작은 값 사용 (상대방이 받을 수 있는 크기로 전송)
```

### 5. ACK 패킷 내부 구조

```
③ ACK 패킷 (Client → Server):

┌──────────────────────────────────────────────────────────────┐
│  Sequence Number: 1234567891  (ISN+1, 데이터는 없지만 +1)        │
├──────────────────────────────────────────────────────────────┤
│  Acknowledgment Number: 9876543211  (서버 ISN + 1)            │
│    → "네 SYN-ACK 받았어. 다음엔 seq=9876543211부터 보내"           │
├──────────────────────────────────────────────────────────────┤
│  Flags: ACK=1 (SYN=0)                                        │
│    → SYN 플래그가 없음: 데이터 전송 가능 상태                        │
└──────────────────────────────────────────────────────────────┘

이 ③ ACK 이후:
  클라이언트: 즉시 HTTP 요청 데이터 전송 가능
  서버: ③ ACK 수신 후 ESTABLISHED → 클라이언트 요청 처리

  TCP Fast Open (TFO):
    ③ ACK와 함께 첫 번째 HTTP 요청을 동시에 전송 가능
    → Handshake 비용을 0.5 RTT 줄임
    → Linux: net.ipv4.tcp_fastopen = 3
    → 지원 여부 확인 필요 (방화벽이 차단하는 경우 있음)
```

### 6. SYN Backlog — 연결 수립 대기 큐

```
서버의 SYN 처리 내부:

SYN 수신 시 서버의 동작:
  ① SYN 도착 → SYN-ACK 전송
  ② SYN_RCVD 상태로 SYN Backlog(Half-Open Queue)에 추가
  ③ ACK 기다림 (기본 타임아웃: 75초)
  ④ ACK 수신 → ESTABLISHED → Accept Queue로 이동
  ⑤ 애플리케이션이 accept() 호출 → 연결 처리

두 개의 큐:
  ┌─────────────────────────────────────────────────────────┐
  │  SYN Queue (Half-Open Queue)                            │
  │  SYN_RCVD 상태 연결 대기                                   │
  │  크기: net.ipv4.tcp_max_syn_backlog (기본: 512~1024)      │
  ├─────────────────────────────────────────────────────────┤
  │  Accept Queue (Completed Connection Queue)              │
  │  ESTABLISHED 완료, 애플리케이션 accept() 대기                │
  │  크기: listen() backlog, min(backlog, somaxconn)         │
  │  Tomcat: server.tomcat.accept-count=100                 │
  └─────────────────────────────────────────────────────────┘

SYN Flood 공격:
  공격자: 위조 SrcIP로 SYN을 대량 전송
  → SYN Queue를 가득 채움
  → 정상 연결이 SYN-ACK를 받지 못함

SYN Cookie 방어:
  SYN Queue 저장 없이 ISN에 정보를 인코딩
  ACK가 오면 Cookie를 검증해서 ESTABLISHED
  → SYN Flood에도 정상 연결 가능
  → net.ipv4.tcp_syncookies = 1 (대부분 기본 활성화)

ss로 확인:
  ss -tlnp | grep 8080
  LISTEN 0 100 *:8080 *:*
         ↑ Recv-Q: 현재 Accept Queue의 대기 연결 수
           100 = Accept Queue 최대값 (listen backlog)
  Recv-Q가 지속적으로 차 있으면 애플리케이션이 accept() 못함
  → 처리 스레드 부족 또는 처리 속도 저하
```

---

## 💻 실전 실험

### 실험 1: tcpdump로 3-Way Handshake 캡처 및 분석

```bash
# 터미널 1: SYN/SYN-ACK/ACK 패킷만 필터링
sudo tcpdump -nn -i any \
  'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0 and host example.com' &

# 터미널 2: HTTP 요청 (새 연결 강제)
curl -v --no-keepalive http://example.com 2>&1 | grep -E "Connected|Trying"

# 예상 출력:
# 15:00:01.001 IP 192.168.1.100.54321 > 93.184.216.34.80: Flags [S]
#   seq 1234567890, win 65535, options [mss 1460,sackOK,TS,nop,wscale 7]
#   ← SYN: ISN=1234567890, MSS=1460, Window Scale=7 협상
#
# 15:00:01.051 IP 93.184.216.34.80 > 192.168.1.100.54321: Flags [S.]
#   seq 9876543210, ack 1234567891, win 65535, options [mss 1460,...]
#   ← SYN-ACK: 서버 ISN=9876543210, 클라이언트 ISN+1 확인
#
# 15:00:01.052 IP 192.168.1.100.54321 > 93.184.216.34.80: Flags [.]
#   ack 9876543211
#   ← ACK: 서버 ISN+1 확인, Handshake 완료
#
# RTT = 51ms (SYN 전송 ~ SYN-ACK 수신)
```

### 실험 2: Handshake 비용 직접 측정

```bash
# curl의 단계별 타이밍 측정
for i in 1 2 3 4 5; do
  curl -w "TCP Connect: %{time_connect}s | Total: %{time_total}s\n" \
       -o /dev/null -s --no-keepalive http://example.com
done

# 출력 예시:
# TCP Connect: 0.051s | Total: 0.187s  ← Handshake 51ms
# TCP Connect: 0.049s | Total: 0.183s
# TCP Connect: 0.052s | Total: 0.189s

# Keep-Alive 사용 시 (연결 재사용):
for i in 1 2 3 4 5; do
  curl -w "TCP Connect: %{time_connect}s | Total: %{time_total}s\n" \
       -o /dev/null -s http://example.com
done
# TCP Connect: 0.051s | Total: 0.187s  ← 첫 번째는 연결 수립
# TCP Connect: 0.000s | Total: 0.134s  ← 이후는 재사용 (Handshake 없음)
# TCP Connect: 0.000s | Total: 0.132s
```

### 실험 3: SYN Backlog와 Accept Queue 관찰

```bash
# Spring Boot 실행 중 (포트 8080, accept-count=100)

# 현재 backlog 상태
ss -tlnp | grep 8080
# LISTEN 0 100 *:8080
#        ^ Recv-Q: 현재 Accept Queue 대기 수

# Accept Queue 최대값 확인
cat /proc/sys/net/core/somaxconn
# 기본: 128 (실제 backlog = min(listen_backlog, somaxconn))

# SYN Queue 크기
cat /proc/sys/net/ipv4/tcp_max_syn_backlog

# 부하 테스트 중 Accept Queue 모니터링
ab -n 10000 -c 500 http://localhost:8080/ &
watch -n 0.5 "ss -tlnp | grep 8080"
# Recv-Q가 0에서 벗어나면 처리 속도 병목
```

### 실험 4: ISN 관찰 — 무작위성 확인

```bash
# 연속 연결의 ISN이 무작위인지 확인
sudo tcpdump -nn -i any 'tcp[tcpflags] & tcp-syn != 0 and port 80' &

# 10번 연속 연결
for i in $(seq 1 10); do
  curl -s --no-keepalive http://example.com -o /dev/null
done

# 각 SYN의 seq 값 추출
# 출력 예시:
# seq 3421857234  ← 무작위
# seq 1872345671  ← 무작위
# seq 4283956123  ← 무작위
# → 예측 불가능 → TCP Session Hijacking 방지
```

---

## 📊 성능/비용 비교

```
Handshake 비용 시나리오별 비교:

┌─────────────────────────────────────────────────────────────────┐
│  환경                   │  RTT    │  Handshake 비용  │  비고       │
├─────────────────────────────────────────────────────────────────┤
│  localhost             │  <1ms   │  <1.5ms         │  무시 가능   │
│  같은 데이터센터 내        │  0.3ms  │  0.45ms         │  낮음       │
│  서울 → 부산             │  3ms    │  4.5ms          │  낮음       │
│  서울 → 미국 서부         │  150ms  │  225ms          │  높음       │
│  서울 → 유럽             │  250ms  │  375ms          │  매우 높음   │
└─────────────────────────────────────────────────────────────────┘

커넥션 풀 효과 (외부 API 호출, RTT=50ms, 초당 100 요청):

풀 없음:
  매 요청 Handshake = 75ms
  100 req/s × 75ms = 7,500ms 누적 Handshake 비용/s
  실제 처리: 10ms → 전체 응답시간: 85ms

풀 있음 (풀 크기=10):
  초기 10개 연결 수립 후 재사용
  Handshake 비용 = 0 (재사용)
  실제 처리: 10ms → 전체 응답시간: 10ms

TCP Fast Open (TFO):
  ③ ACK + 첫 번째 데이터 동시 전송
  기존: 1.5 RTT 후 데이터 전송
  TFO:  1 RTT 후 데이터 전송 (0.5 RTT 절약)
  단, 재연결 시에만 효과 (TFO Cookie 필요)
```

---

## ⚖️ 트레이드오프

```
3-Way Handshake의 트레이드오프:

장점:
  ① 양방향 신뢰성 보장:
     양쪽이 서로의 ISN을 확인한 후 통신 시작
  ② 오래된 SYN 제거:
     지연된 SYN 패킷이 새 연결을 방해하지 않음
  ③ 파라미터 협상:
     MSS, Window Scale, SACK 등을 Handshake에서 한 번에 협상

단점:
  ① 1.5 RTT 지연:
     연결 수립에 네트워크 왕복 1.5회 필요
     고지연 환경에서 큰 비용
  ② SYN Flood 공격에 취약:
     Half-Open 연결로 SYN Queue 고갈 가능
     → SYN Cookie로 완화

개선 시도:
  TCP Fast Open (RFC 7413):
    첫 번째 SYN에 데이터 포함 가능 → 0-RTT 데이터 (두 번째 연결부터)
    단, 멱등성 없는 요청(POST)에는 재전송 위험

  QUIC (HTTP/3):
    0-RTT 또는 1-RTT로 연결 수립 + 암호화 동시 처리
    → TCP 3-Way Handshake + TLS Handshake를 1번으로 통합
    → 고지연 환경에서 TCP 대비 큰 개선
```

---

## 📌 핵심 정리

```
3-Way Handshake 핵심 요약:

3번이어야 하는 이유:
  ① SYN:     클라이언트 ISN 전달 + 연결 요청
  ② SYN-ACK: 서버 ISN 전달 + 클라이언트 ISN 수신 확인
  ③ ACK:     서버 ISN 수신 확인 → 양방향 ISN 동기화 완료
  2-Way로는 서버가 클라이언트의 수신 능력을 확인할 수 없음

비용:
  1.5 RTT → 클라이언트는 1 RTT 후 데이터 전송 가능
  서울-미국(150ms RTT): Handshake = 225ms

Handshake에서 협상되는 것:
  ISN (Initial Sequence Number): 양방향 각각 무작위 선택
  MSS (Maximum Segment Size):   양쪽 중 작은 값 사용
  Window Scale:                  수신 버퍼 크기 확장 계수
  SACK Permitted:                Selective ACK 지원 여부
  Timestamps:                    RTT 측정용

실무 적용:
  Connection Pool 필수: Handshake는 최초 1회로 제한
  connectTimeout = RTT * 2 이상으로 설정
  SYN Backlog: /proc/sys/net/ipv4/tcp_max_syn_backlog
  Accept Queue: server.tomcat.accept-count

진단 명령어:
  ss -tlnp: LISTEN 소켓의 Accept Queue 확인
  tcpdump 'tcp[tcpflags] & tcp-syn != 0': SYN 패킷만 캡처
  curl -w "%{time_connect}": TCP 연결 시간 측정
```

---

## 🤔 생각해볼 문제

**Q1.** Spring Boot 서버에서 `server.tomcat.accept-count=100`으로 설정했다. `ss -tlnp`에서 Recv-Q가 지속적으로 80~100을 유지하고 있다면 어떤 상황이고, 해결 방법은?

<details>
<summary>해설 보기</summary>

Recv-Q는 LISTEN 소켓에서 **ESTABLISHED는 됐지만 아직 애플리케이션이 `accept()`하지 않은 연결의 수**입니다. 100에 가깝다면 Accept Queue가 거의 가득 찬 상태입니다.

원인: 새 연결이 수립되는 속도 > 애플리케이션이 처리하는 속도

구체적으로:
- Tomcat 스레드 풀이 모두 사용 중 (maxThreads 고갈)
- DB 쿼리, 외부 API 응답 등 처리 시간이 길어 스레드가 묶임
- GC Stop-the-World

Accept Queue가 가득 차면 새로 오는 SYN-ACK에 대한 ACK를 받아도 연결이 Drop됩니다. 클라이언트는 Connection Timeout을 경험합니다.

해결 순서:
```bash
# 1. 스레드 상태 확인 (Blocked/Waiting 스레드 수)
jstack <pid> | grep -E "BLOCKED|WAITING" | wc -l

# 2. 어디서 막히는지 확인
jstack <pid> | grep -A 10 "BLOCKED"

# 3. Actuator로 Tomcat 스레드 확인
curl http://localhost:8080/actuator/metrics/tomcat.threads.busy

# 해결:
# - DB 쿼리 최적화 (처리 시간 단축)
# - maxThreads 증가 (단, 근본 해결은 아님)
# - 비동기 처리 도입
```

</details>

---

**Q2.** 같은 클라이언트에서 같은 서버로 새 TCP 연결을 맺을 때, 이전 연결과 동일한 SrcPort를 사용할 수 있는가? 어떤 조건에서 가능/불가능한가?

<details>
<summary>해설 보기</summary>

**이전 연결이 완전히 종료(CLOSED)된 후라면 가능**합니다. 단, TIME_WAIT 상태가 남아 있는 동안은 같은 4-Tuple(SrcIP:SrcPort:DstIP:DstPort)의 재사용이 제한됩니다.

TCP 연결은 4-Tuple로 식별됩니다:
```
(SrcIP, SrcPort, DstIP, DstPort)
```

TIME_WAIT 동안 재사용이 제한되는 이유: 이전 연결의 지연 패킷이 새 연결에 섞여들어올 수 있기 때문입니다.

예외적으로 재사용이 허용되는 경우:
1. **`SO_REUSEADDR`**: 서버 포트 재사용 (TIME_WAIT 상태의 서버 포트 즉시 재바인드)
2. **새 연결의 ISN이 이전 연결의 마지막 seq보다 커야 함**: OS가 자동으로 처리
3. **`net.ipv4.tcp_tw_reuse=1`**: TIME_WAIT 상태 연결을 아웃바운드 연결에 재사용 (클라이언트 측)

실무적으로 Ephemeral Port(클라이언트 측 포트)는 OS가 자동 선택하므로 TIME_WAIT인 포트는 자동으로 피합니다. 단, Ephemeral Port 범위(`/proc/sys/net/ipv4/ip_local_port_range`)가 고갈되면 재사용 문제가 발생합니다.

</details>

---

**Q3.** `tcpdump`로 캡처했을 때 SYN 패킷은 보이는데 SYN-ACK가 돌아오지 않는다. 가능한 원인 3가지와 각각의 진단 방법은?

<details>
<summary>해설 보기</summary>

SYN이 도달했지만 SYN-ACK가 없는 경우:

**원인 1: 서버가 해당 포트를 LISTEN하지 않음**
- SYN을 받으면 서버 OS가 RST로 응답해야 정상
- RST도 없으면 방화벽이 차단 중

진단:
```bash
# 서버에서
ss -tlnp | grep <port>   # LISTEN 없으면 프로세스 문제
```

**원인 2: 방화벽/보안 그룹이 인바운드 SYN을 DROP**
- iptables `-j DROP`이면 응답 없음 (RST도 없음)
- `-j REJECT`이면 즉시 RST 또는 ICMP Port Unreachable

진단:
```bash
# 서버에서 SYN이 도달하는지 확인
sudo tcpdump -nn -i any 'port <port> and tcp[tcpflags] & tcp-syn != 0'
# SYN이 보이지 않으면 중간 방화벽, 보이면 서버 방화벽
sudo iptables -L -n | grep DROP
```

**원인 3: SYN Queue가 가득 참 (SYN Flood 또는 처리 지연)**
- SYN Queue가 가득 차면 새 SYN을 버림

진단:
```bash
# SYN Queue 고갈 확인
netstat -s | grep "SYNs to LISTEN"
# "N SYNs to LISTEN sockets dropped" 증가 중이면 고갈

# SYN Cookie가 활성화됐는지 확인
sysctl net.ipv4.tcp_syncookies   # 1이면 활성화
```

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: TCP 연결 종료와 TIME_WAIT ➡️](./02-four-way-handshake-time-wait.md)**

</div>
