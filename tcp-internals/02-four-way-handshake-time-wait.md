# TCP 연결 종료 — 4-Way Handshake와 TIME_WAIT

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 연결 종료가 왜 3-Way가 아닌 4-Way인가?
- Half-Close란 무엇이고 왜 TCP가 이를 지원하는가?
- TIME_WAIT 상태가 2MSL 동안 유지되는 이유는 무엇인가?
- TIME_WAIT이 없다면 어떤 문제가 생기는가?
- CLOSE_WAIT이 서버에 대량 누적되는 원인과 해결 방법은?
- `SO_REUSEADDR`와 `SO_REUSEPORT`는 언제 사용해야 하고, 남용하면 어떤 위험이 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"서버 재시작 후 이전 포트로 바인드가 안 돼요":
  → TIME_WAIT 때문
  → 이유를 모르면 임의로 SO_REUSEADDR을 추가 → 잠재적 버그 도입

"CLOSE_WAIT이 수천 개 쌓였어요":
  → 애플리케이션 버그 (소켓 close() 미호출)
  → 모르면 서버 재시작으로 임시 해결 반복

"TIME_WAIT가 너무 많아서 줄여야 하지 않나요?":
  → TIME_WAIT는 네트워크 안정성 보호 장치
  → 줄이면 안 되는 이유를 설명 못함

실무에서 빈번히 마주치는 현상:
  고트래픽 API 서버:
    초당 수천 요청 → 수천 연결 종료 → TIME_WAIT 수천 개 누적
    → Ephemeral Port 고갈 → 새 아웃바운드 연결 불가
    → 원인: Connection Pool 미사용 또는 Pool 크기 부족

  Spring Boot + RestTemplate:
    풀 없이 사용 → 초당 1000 요청 → TIME_WAIT 1000/s 생성
    → /proc/sys/net/ipv4/ip_local_port_range 범위(약 28000개) 소진
    → "Cannot assign requested address" 에러
```

---

## 😱 흔한 실수

```
Before — TIME_WAIT와 CLOSE_WAIT를 모를 때:

실수 1: TIME_WAIT를 줄이는 것이 좋다고 생각
  sysctl -w net.ipv4.tcp_tw_recycle=1  ← 위험! (Linux 4.12에서 제거됨)
  sysctl -w net.ipv4.tcp_fin_timeout=15  ← TIME_WAIT 단축
  → 지연 패킷이 새 연결에 섞여들 위험
  → 드물지만 재현하기 어려운 데이터 오염 버그 발생

실수 2: CLOSE_WAIT를 서버 재시작으로만 해결
  netstat -an | grep CLOSE_WAIT | wc -l  → 3000
  → 서버 재시작 → 잠시 후 다시 3000
  → 근본 원인: 특정 코드 경로에서 소켓/연결을 close() 안 함
  → try-with-resources 또는 finally에서 close() 필요

실수 3: SO_REUSEADDR 남용
  서버가 8080 포트로 재시작 안 됨
  → SO_REUSEADDR 추가 → 해결된 것처럼 보임
  → 하지만 이전 연결의 지연 패킷이 새 연결에 도달할 위험
  → 서버 포트 재사용은 괜찮지만, 클라이언트 포트에 무분별하게 적용하면 위험
```

---

## ✨ 올바른 접근

```
After — TIME_WAIT를 이해하면:

TIME_WAIT 누적 근본 해결:
  원인: 연결을 너무 자주 맺고 끊음
  해결: Connection Pool로 연결 재사용
    → TIME_WAIT 자체를 만들지 않는 전략

Ephemeral Port 고갈 예방:
  cat /proc/sys/net/ipv4/ip_local_port_range
  # 기본: 32768 60999 (약 28000개)
  
  sysctl -w net.ipv4.ip_local_port_range="1024 65535"
  # 약 64000개로 확장 (임시 방편)
  
  근본 해결: Connection Pool 사용

CLOSE_WAIT 방지 패턴 (Spring):
  // 잘못된 방법
  HttpURLConnection conn = url.openConnection();
  // close() 없음 → CLOSE_WAIT 누적

  // 올바른 방법 (try-with-resources)
  try (CloseableHttpClient client = HttpClients.createDefault()) {
      // ...
  }  // 자동으로 close() 호출

서버 포트 재바인드 (SO_REUSEADDR):
  // Spring Boot는 기본으로 SO_REUSEADDR 설정
  // 서버 재시작 시 TIME_WAIT 상태에서도 같은 포트 바인드 가능
  // 이것은 안전 (SrcPort가 서버 포트이므로 4-Tuple이 달라짐)
```

---

## 🔬 내부 동작 원리

### 1. 4-Way Handshake 전체 흐름

```
왜 4번인가? — TCP는 양방향 독립 종료를 지원 (Half-Close)

Client (먼저 종료 요청)         Server
        │                          │
        │  ① FIN                   │
        │  seq=x, FIN=1, ACK=1     │  Client: 더 이상 보낼 데이터 없음
        │ ───────────────────────► │  상태: FIN_WAIT_1
        │                          │
        │  ② ACK                   │
        │  ack=x+1                 │  Server: FIN 받음, 아직 보낼 데이터 있을 수 있음
        │ ◄─────────────────────── │  상태: CLOSE_WAIT
        │  상태: FIN_WAIT_2         │
        │                          │  (서버가 남은 데이터 전송 중...)
        │                          │
        │  ③ FIN                  │
        │  seq=y, FIN=1, ACK=1     │  Server: 이제 나도 종료
        │ ◄─────────────────────── │  상태: LAST_ACK
        │  상태: TIME_WAIT          │
        │                          │
        │  ④ ACK                  │
        │  ack=y+1                 │  Client: 서버 FIN 확인
        │ ───────────────────────► │  상태: CLOSED
        │                          │
        │  [2MSL 대기]              │
        │  상태: CLOSED             │

핵심: ①②가 하나의 FIN-ACK이 아닌 이유
  서버는 ① FIN을 받고 ② ACK를 즉시 보냄
  하지만 서버 애플리케이션이 아직 데이터를 보내는 중일 수 있음
  → 서버 데이터 전송 완료 후에야 ③ FIN 전송
  → ②와 ③ 사이에 시간 간격이 있을 수 있음
  → 따라서 FIN-ACK를 하나로 합칠 수 없음 (Half-Close)
```

### 2. TIME_WAIT — 왜 2MSL인가

```
MSL (Maximum Segment Lifetime):
  네트워크에서 패킷이 살아있을 수 있는 최대 시간
  IP 헤더의 TTL이 0이 될 때까지의 시간 상한
  RFC 793: MSL = 2분 (현실: 1분 또는 30초로 구현)
  Linux 기본: /proc/sys/net/ipv4/tcp_fin_timeout = 60초 (1 MSL)
  TIME_WAIT = 2 MSL = 120초

TIME_WAIT가 필요한 이유 1: 마지막 ACK 손실 방어

  Client                    Server
    │  ③ FIN                   │
    │ ◄──────────────────────  │
    │  ④ ACK (손실!)            │
    │ ───────────────────────X │ ← ACK가 네트워크에서 사라짐
    │                          │
    │  [서버 관점]               │
    │  ④ ACK 없음 → 재전송       │
    │  ③ FIN 재전송             │
    │ ◄──────────────────────  │
    │                          │
  TIME_WAIT 상태에서 ③ FIN 재전송을 받으면:
    → ④ ACK를 다시 전송
    → 서버가 정상적으로 CLOSED 상태로 전환 가능

  TIME_WAIT가 없다면:
    → Client는 CLOSED 상태
    → 재전송된 ③ FIN 도착 → "모르는 연결" → RST 전송
    → 서버: RST를 받으면 정상 종료가 아님 (에러 상태로 종료)

TIME_WAIT가 필요한 이유 2: 지연 패킷 혼입 방지

  시나리오:
    연결 A: (src=1.2.3.4:54321, dst=5.6.7.8:80)
    연결 A가 종료 후 즉시 같은 4-Tuple로 연결 B 수립
    연결 A의 지연된 데이터 패킷이 뒤늦게 도착
    → 연결 B가 이 패킷을 자신의 데이터로 오인

  2 MSL 대기:
    모든 지연 패킷은 최대 2 MSL 안에 소멸(TTL 0)
    → TIME_WAIT 이후에는 지연 패킷 없음
    → 새 연결이 안전하게 같은 4-Tuple 사용 가능
```

### 3. CLOSE_WAIT — 서버 버그의 신호

```
CLOSE_WAIT 상태:
  상대방(클라이언트)이 FIN을 보냈음
  내가(서버) ACK는 보냈음
  하지만 내 애플리케이션이 아직 소켓을 close()하지 않음
  → FIN을 보내지 못하고 CLOSE_WAIT 상태에 머무름

정상 흐름:
  클라이언트 FIN 수신 → OS: ACK 전송, 소켓에 EOF 전달
  → 애플리케이션: read()에서 0 반환 (EOF 감지)
  → 애플리케이션: close() 호출
  → OS: FIN 전송 → LAST_ACK → CLOSED

버그가 있는 흐름:
  클라이언트 FIN 수신 → OS: ACK 전송
  → 애플리케이션: close() 호출을 누락 (예외 처리 미흡)
  → 소켓이 CLOSE_WAIT에서 영원히 대기
  → 파일 디스크립터 누수 + 포트 점유 + 메모리 누수

CLOSE_WAIT 대량 누적의 일반적인 원인:

  원인 1: 예외 발생 시 연결 close() 미호출
    public void callApi() throws Exception {
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        // 예외 발생 → conn.disconnect() 미호출 → CLOSE_WAIT
    }

  원인 2: HTTP 클라이언트의 잘못된 사용
    try {
        CloseableHttpResponse response = client.execute(request);
        // response.close() 누락 → 커넥션이 풀로 반환 안 됨
        // 클라이언트가 연결 종료하면 CLOSE_WAIT
    }

  원인 3: Tomcat의 연결 타임아웃과 클라이언트 타임아웃 불일치
    Nginx keepalive_timeout: 65s
    Tomcat connectionTimeout: 20s
    → Nginx가 먼저 FIN 전송 → Tomcat이 CLOSE_WAIT
    → 설정 정합성 필요

진단:
  ss -tanp | grep CLOSE_WAIT
  # 어떤 프로세스인지 확인
  lsof -p <pid> | grep CLOSE_WAIT | wc -l
  # 해당 프로세스의 CLOSE_WAIT 소켓 수 확인
```

### 4. 소켓 옵션 — SO_REUSEADDR, SO_LINGER

```
SO_REUSEADDR:
  서버 포트를 TIME_WAIT 상태에서도 재바인드 허용

  사용 안전 조건:
    서버가 고정 포트(80, 443, 8080)를 재시작할 때
    → 4-Tuple에서 DstIP:DstPort(클라이언트 주소)가 새 연결과 다름
    → 지연 패킷이 새 연결에 영향 없음

  위험한 사용:
    클라이언트 소켓에 SO_REUSEADDR로 같은 SrcPort 재사용
    → 완전히 같은 4-Tuple → 지연 패킷 혼입 가능

  Spring Boot:
    내장 Tomcat이 기본으로 SO_REUSEADDR 설정
    → 재시작 시 "Address already in use" 없음

SO_LINGER (Linger on Close):
  소켓 close() 시 동작 제어

  기본 (linger OFF):
    close() 호출 → 즉시 반환 → OS가 백그라운드에서 FIN 전송
    → 우아한 종료 (Graceful Shutdown)

  linger ON, timeout=0:
    close() 호출 → 즉시 RST 전송 → 강제 종료
    → TIME_WAIT 없음 (하지만 데이터 손실 가능)
    → 성능 테스트, 빠른 포트 재사용 필요 시 사용

  linger ON, timeout=N:
    close() 호출 → N초까지 데이터 전송 대기 후 FIN
    → close()가 N초 동안 블록될 수 있음

TCP_NODELAY (Nagle 알고리즘 비활성화):
  Nagle 알고리즘: 작은 패킷을 모아서 보냄 (대역폭 효율)
  단점: 지연이 발생 (ACK 대기)
  
  실시간 애플리케이션 (게임, 채팅):
    TCP_NODELAY = true → 작은 패킷도 즉시 전송

  Spring Boot:
    server.tomcat.connection-timeout (소켓 읽기 타임아웃)
```

---

## 💻 실전 실험

### 실험 1: 4-Way Handshake 캡처

```bash
# FIN/RST 패킷만 캡처
sudo tcpdump -nn -i any \
  'tcp[tcpflags] & (tcp-fin|tcp-rst) != 0 and host example.com' &

# 연결 종료 강제 (Keep-Alive 없이 단일 요청)
curl --no-keepalive http://example.com

# 예상 출력:
# ... Flags [F.] seq=N, ack=M  ← 클라이언트 FIN (①)
# ... Flags [.]  ack=N+1        ← 서버 ACK (②)
# ... Flags [F.] seq=M, ack=N+1 ← 서버 FIN (③)
# ... Flags [.]  ack=M+1        ← 클라이언트 ACK (④)
```

### 실험 2: TIME_WAIT 상태 관찰

```bash
# TIME_WAIT 소켓 실시간 모니터링
watch -n 0.5 "ss -tan state time-wait | wc -l"

# 다른 터미널에서 연결 반복 생성 (Keep-Alive 없음)
for i in $(seq 1 100); do
  curl -s --no-keepalive http://example.com -o /dev/null
done

# TIME_WAIT 수가 증가하다가 60~120초 후 소멸됨을 확인

# 구체적인 TIME_WAIT 소켓 목록
ss -tan state time-wait | head -20
# Local Address:Port  Peer Address:Port
# 192.168.1.100:54321  93.184.216.34:80  ← 각 연결의 TIME_WAIT
```

### 실험 3: CLOSE_WAIT 재현

```bash
# CLOSE_WAIT 재현을 위한 서버 (Python)
python3 -c "
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('127.0.0.1', 9090))
s.listen(1)
print('waiting...')
conn, addr = s.accept()
print('connected')
# FIN을 받아서 ACK를 보내지만 close()를 안 함
data = conn.recv(1024)
print('received:', data[:50])
time.sleep(60)  # close() 없이 대기 → CLOSE_WAIT
" &

# 클라이언트: 데이터 보내고 연결 종료
(echo -e "GET / HTTP/1.0\r\n\r\n"; sleep 2) | nc 127.0.0.1 9090 &

# CLOSE_WAIT 확인
sleep 3
ss -tanp | grep 9090
# CLOSE_WAIT 상태 확인됨

# 프로세스에서 열린 소켓 확인
lsof -i :9090
```

### 실험 4: Ephemeral Port 고갈 시뮬레이션

```bash
# 현재 Ephemeral Port 범위 확인
cat /proc/sys/net/ipv4/ip_local_port_range
# 32768 60999

# TIME_WAIT 통계 확인
ss -s
# 출력 예시:
# Total: 1200
# TCP:   800 (estab 50, closed 0, orphaned 0, timewait 750)
# → TIME_WAIT 750개: Ephemeral Port 750개 점유 중

# 포트 범위 내에서 TIME_WAIT가 많으면:
# "Cannot assign requested address" 에러 발생
# → 새 아웃바운드 연결 불가

# 진단
netstat -s | grep "TCP:"
ss -s | grep timewait
```

---

## 📊 성능/비용 비교

```
연결 종료 방식별 비교:

Graceful Shutdown (FIN):
  소요 시간: 2 RTT (FIN + ACK + FIN + ACK)
  안전성:   높음 (데이터 손실 없음)
  TIME_WAIT: 발생 (Active Close 측)
  사용:     일반적인 연결 종료

RST (SO_LINGER timeout=0):
  소요 시간: 즉시
  안전성:   낮음 (미전송 데이터 손실 가능)
  TIME_WAIT: 없음
  사용:     에러 상황, 성능 테스트 포트 빠른 재사용

TIME_WAIT 존재 비용:
  메모리: 소켓당 약 280 bytes
  포트:   Ephemeral Port 1개 점유
  10,000 TIME_WAIT → 약 2.8 MB 메모리

TIME_WAIT 소멸 시간:
  Linux 기본 (tcp_fin_timeout=60): 60~120초
  → 고트래픽에서 수천~수만 개 누적 가능
  → Connection Pool로 근본 해결이 최선

Nginx upstream keepalive 설정:
  upstream backend {
      server 127.0.0.1:8080;
      keepalive 32;  ← 업스트림 연결 재사용 (TIME_WAIT 감소)
  }
```

---

## ⚖️ 트레이드오프

```
TIME_WAIT 존재의 트레이드오프:

TIME_WAIT의 비용:
  소켓 메모리 점유 (280 bytes / 소켓)
  Ephemeral Port 점유 (아웃바운드 연결 수 제한)
  고트래픽 API 서버에서 수만 개 누적 가능

TIME_WAIT를 없애는 방법들과 위험:
  ① tcp_tw_recycle (Linux 4.12에서 완전 제거):
     NAT 환경에서 동일 공인 IP의 연결을 잘못 차단
     → 이미 사용 불가

  ② tcp_tw_reuse=1 (아웃바운드 연결에만):
     조건: 새 연결 타임스탬프 > 기존 연결 타임스탬프
     → 비교적 안전하지만 완벽한 보장은 아님

  ③ SO_LINGER timeout=0 (RST):
     TIME_WAIT 없음 → 미전송 데이터 손실 가능
     → 테스트 환경 외 프로덕션 사용 주의

권장 접근:
  TIME_WAIT는 TCP의 정상 동작
  줄이려 하지 말고 Connection Pool로 연결 자체를 재사용
  → Handshake도 줄고, TIME_WAIT도 줄고
```

---

## 📌 핵심 정리

```
4-Way Handshake와 TIME_WAIT 핵심 요약:

4번인 이유:
  TCP는 Half-Close 지원
  각 방향의 종료가 독립적 → FIN/ACK가 각 방향마다 필요
  3-Way가 아닌 이유: ②ACK와 ③FIN 사이에 시간 간격 가능

TIME_WAIT 존재 이유 2가지:
  ① 마지막 ACK(④) 손실 시 재전송된 FIN(③)에 ACK 재전송
  ② 이전 연결의 지연 패킷이 새 연결에 혼입되는 것 방지

TIME_WAIT 기간: 2 MSL (Linux 기본 ≈ 60~120초)
TIME_WAIT 발생 위치: Active Close 측 (먼저 FIN을 보낸 쪽)

CLOSE_WAIT:
  상대방 FIN 받음 + ACK 전송했지만 내가 close() 안 함
  원인: 애플리케이션 버그 (예외 발생 시 소켓 close() 누락)
  해결: try-with-resources, finally에서 close()

진단 명령어:
  ss -tan state time-wait | wc -l    → TIME_WAIT 개수
  ss -tan state close-wait | wc -l   → CLOSE_WAIT 개수
  ss -s                              → 전체 소켓 상태 요약
  lsof -p <pid> | grep -c CLOSE_WAIT → 특정 프로세스 CLOSE_WAIT

핵심 원칙:
  TIME_WAIT는 줄이지 말고 Connection Pool로 방지
  CLOSE_WAIT는 코드 버그 → 반드시 수정
  SO_REUSEADDR은 서버 포트 재바인드에만 사용
```

---

## 🤔 생각해볼 문제

**Q1.** 클라이언트가 먼저 FIN을 보내는 경우(Active Close)와 서버가 먼저 FIN을 보내는 경우, TIME_WAIT는 각각 어느 쪽에 발생하는가? 실무에서 TIME_WAIT를 줄이려면 어느 쪽에서 연결을 먼저 끊는 것이 유리한가?

<details>
<summary>해설 보기</summary>

**TIME_WAIT는 Active Close 측, 즉 FIN을 먼저 보낸 쪽에 발생합니다.**

일반적인 HTTP 통신에서:
- **HTTP/1.0**: 서버가 응답 후 먼저 FIN → 서버에 TIME_WAIT
- **HTTP/1.1 Keep-Alive**: 클라이언트가 연결 종료 시 먼저 FIN → 클라이언트에 TIME_WAIT

실무적으로:
- **클라이언트(Spring App)** 가 먼저 끊으면: 클라이언트 서버의 Ephemeral Port 점유 → 아웃바운드 연결 제한
- **서버** 가 먼저 끊으면: 서버의 고정 포트(80 등)는 TIME_WAIT가 발생해도 4-Tuple이 달라 실질적 영향 적음

따라서 고트래픽 API 서버에서는:
- 서버 측에서 `Connection: close`를 먼저 보내서 서버가 Active Close를 하게 하면 클라이언트의 Ephemeral Port 고갈을 줄일 수 있습니다.
- 하지만 근본 해결책은 Connection Pool 사용입니다.

</details>

---

**Q2.** Spring Boot 애플리케이션에서 특정 외부 API 호출 후 CLOSE_WAIT가 누적된다. 코드 어디를 확인해야 하는가?

<details>
<summary>해설 보기</summary>

CLOSE_WAIT는 상대방(외부 API 서버)이 FIN을 보냈는데 우리 쪽에서 `close()`를 호출하지 않은 상태입니다.

확인 포인트:

**1. HTTP 클라이언트 응답 처리**
```java
// 문제: response를 close하지 않음
CloseableHttpResponse response = httpClient.execute(request);
String body = EntityUtils.toString(response.getEntity());
// response.close() 누락!

// 해결:
try (CloseableHttpResponse response = httpClient.execute(request)) {
    return EntityUtils.toString(response.getEntity());
}
```

**2. RestTemplate 사용 시 예외 경로**
```java
// RestTemplate은 응답을 자동으로 처리하지만
// 4xx, 5xx 응답에서 예외 처리 도중 연결이 반환 안 되는 경우
// → ErrorHandler 설정 확인
```

**3. WebClient 구독 취소**
```java
// Flux/Mono 구독을 중간에 취소하면
// 연결이 정상 종료되지 않을 수 있음
```

**진단 코드:**
```bash
# 어떤 파일 디스크립터가 CLOSE_WAIT인지 확인
PID=$(pgrep -f spring-boot-app)
ss -tanp | grep CLOSE_WAIT | grep "pid=$PID"
lsof -p $PID | grep -c "CLOSE_WAIT"

# 스택 트레이스로 어느 스레드가 해당 소켓을 잡고 있는지
jstack $PID | grep -A 20 "HttpConn"
```

</details>

---

**Q3.** `tcp_fin_timeout`을 60초에서 10초로 줄이면 어떤 이점이 있고 어떤 위험이 생기는가?

<details>
<summary>해설 보기</summary>

**`tcp_fin_timeout`은 Linux에서 FIN_WAIT_2 타임아웃을 제어하며, TIME_WAIT 기간(2MSL)과는 별개입니다.**

실제로 Linux의 TIME_WAIT 기간은 커널 내부에 하드코딩(60초)되어 있고, `tcp_fin_timeout`은 FIN_WAIT_2 상태에서 상대방의 FIN을 기다리는 시간을 제어합니다.

10초로 줄이면:
- **이점**: FIN_WAIT_2 상태에서 소켓이 빠르게 해제됨 (상대방이 FIN을 보내지 않는 비정상 상황에서)
- **위험**: 상대방이 느리게 FIN을 보내는 정상적인 경우(고지연 네트워크, 처리 중인 데이터)에도 연결이 강제 종료됨

TIME_WAIT 자체를 줄이는 방법:
- `net.ipv4.tcp_tw_reuse=1`: 아웃바운드 연결에서 TIME_WAIT 소켓 재사용 (비교적 안전)
- RST 기반 종료 (SO_LINGER=0): TIME_WAIT 없음, 하지만 데이터 손실 위험

**결론**: `tcp_fin_timeout` 단축은 FIN_WAIT_2 해소에는 도움되지만, 근본적인 TIME_WAIT 문제 해결이 아닙니다. Connection Pool 사용이 올바른 접근입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 3-Way Handshake](./01-three-way-handshake.md)** | **[홈으로 🏠](../README.md)** | **[다음: TCP 신뢰성 보장 ➡️](./03-tcp-reliability.md)**

</div>
