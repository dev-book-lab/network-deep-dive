# 네트워크 진단 도구 — tcpdump, Wireshark, ss 실전 활용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `tcpdump`의 BPF(Berkeley Packet Filter) 필터 문법을 어떻게 조합하는가?
- `ss`와 `netstat`의 차이는 무엇이고, 소켓 상태에서 어떤 정보를 읽는가?
- Wireshark에서 TCP Stream Follow로 전체 HTTP 요청/응답을 어떻게 복원하는가?
- `ping`이 성공해도 서비스 접속이 안 되는 경우 어떤 도구로 다음 단계를 진단하는가?
- `traceroute`의 `* * *`이 반드시 패킷 손실을 의미하지 않는 이유는?
- 네트워크 장애 상황에서 단계별로 어떤 도구를 순서대로 사용하는가?

---

## 🔍 왜 이 개념이 중요한가

### 도구를 모르면 장애 앞에서 손발이 묶인다

```
네트워크 장애 현장:

  개발팀: "API 응답이 없어요"
  
  도구를 모르는 경우:
    → 서버 재시작 (원인 파악 없이)
    → 로그 전체 grep (패턴 없음)
    → 네트워크 팀에 에스컬레이션 대기
    → 30분 이상 다운타임
  
  도구를 아는 경우:
    ping target      # 2초: L3 연결 확인
    telnet target 443  # 3초: L4 포트 확인
    curl -v https://target  # 5초: TLS + HTTP 확인
    ss -tanp | grep 8080    # 1초: 서버 소켓 상태
    tcpdump 'host target'   # 10초: 패킷 레벨 확인
    → 5분 내 원인 격리 가능

도구를 알아야 하는 이유:
  네트워크 문제는 어느 계층에서 발생하는지 모름
  각 계층에 맞는 도구가 있음
  도구를 모르면 계층별 진단 자체가 불가
```

---

## 🔬 내부 동작 원리

### 1. tcpdump — 패킷 캡처의 핵심

```
tcpdump 동작 원리:
  NIC에서 들어오는/나가는 모든 패킷을 OS 커널의 BPF 레이어에서 필터링
  필터를 통과한 패킷을 사용자 공간으로 복사해서 출력 또는 파일로 저장

  장점: 커널 공간에서 필터링 → 매우 낮은 오버헤드
  주의: sudo 권한 필요 (raw 소켓 접근)

기본 사용법:
  sudo tcpdump [옵션] [필터 표현식]

필수 옵션:
  -i <iface>  : 캡처할 인터페이스 (eth0, any 등)
  -n          : IP를 호스트명으로 역변환하지 않음 (빠름)
  -nn         : IP와 포트 번호 모두 숫자로 출력
  -v          : 상세 정보 (IP 헤더 필드)
  -vv         : 더 상세한 정보
  -e          : Ethernet 헤더 포함 출력
  -w <file>   : .pcap 파일로 저장 (Wireshark에서 열기)
  -r <file>   : 저장된 .pcap 파일 읽기
  -c <count>  : 지정한 개수의 패킷만 캡처
  -s <snaplen>: 패킷에서 캡처할 바이트 수 (0=전체)
  -A          : 패킷 내용을 ASCII로 출력 (HTTP 평문 확인 용도)
  -X          : 패킷 내용을 HEX+ASCII로 출력
  -l          : 라인 버퍼링 (실시간 파이프 처리 시)

BPF 필터 표현식:
  # 호스트 필터
  host 192.168.1.10          # 출발지 또는 목적지
  src host 192.168.1.10      # 출발지만
  dst host 192.168.1.10      # 목적지만
  not host 192.168.1.10      # 제외

  # 포트 필터
  port 80                    # 출발지 또는 목적지 포트
  src port 80
  dst port 80
  portrange 8000-9000        # 포트 범위

  # 프로토콜 필터
  tcp                        # TCP만
  udp                        # UDP만
  icmp                       # ICMP만
  arp                        # ARP만

  # 논리 연산자
  host 10.0.0.1 and port 443
  tcp and not port 22        # SSH 제외하고 TCP만
  src host 192.168.1.10 or dst host 192.168.1.10

  # 고급 필터
  tcp[tcpflags] & tcp-syn != 0     # SYN 패킷만
  tcp[tcpflags] & tcp-rst != 0     # RST 패킷만
  tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn  # SYN only (not SYN-ACK)
  'ip[6:2] & 0x1fff != 0'          # 단편화된 패킷만
```

### 2. tcpdump 출력 해석

```
tcpdump 출력 포맷:
  시간 출발지IP.포트 > 목적지IP.포트: 플래그 seq ack 윈도우 옵션 데이터길이

TCP Handshake 캡처 및 해석:
sudo tcpdump -nn -i any 'port 80 and host example.com'

실제 출력 예시:
──────────────────────────────────────────────────────────────────────────
15:00:01.000001 IP 192.168.1.100.54321 > 93.184.216.34.80: Flags [S],
  seq 1234567890, win 65535, options [mss 1460,sackOK,TS val 123 ecr 0,
  nop,wscale 7], length 0
──────────────────────────────────────────────────────────────────────────

분석:
  15:00:01.000001      : 타임스탬프 (마이크로초 단위)
  192.168.1.100.54321  : 출발지 IP.포트
  93.184.216.34.80     : 목적지 IP.포트
  Flags [S]            : SYN 플래그 (연결 요청)
  seq 1234567890       : 초기 Sequence Number (ISN)
  win 65535            : 수신 윈도우 크기
  mss 1460             : Maximum Segment Size 협상
  length 0             : TCP 페이로드 크기 (SYN은 데이터 없음)

TCP 플래그:
  [S]   = SYN     : 연결 요청
  [S.]  = SYN-ACK : 연결 요청 수락
  [.]   = ACK     : 확인 응답
  [P.]  = PSH+ACK : 데이터 전송 (즉시 전달 요청)
  [F.]  = FIN+ACK : 연결 종료 요청
  [R]   = RST     : 강제 연결 리셋
  [R.]  = RST+ACK

완전한 TCP Handshake:
  15:00:01.001 192.168.1.100.54321 > 93.184.216.34.80: Flags [S]   ← SYN
  15:00:01.051 93.184.216.34.80 > 192.168.1.100.54321: Flags [S.]  ← SYN-ACK
  15:00:01.052 192.168.1.100.54321 > 93.184.216.34.80: Flags [.]   ← ACK
  RTT = 51ms (SYN 전송 ~ SYN-ACK 수신)

RST 패킷 발견:
  15:00:05.001 93.184.216.34.80 > 192.168.1.100.54321: Flags [R.]
  → 서버가 강제로 연결을 끊음
  → 원인: 서버 프로세스 죽음, 방화벽, Keep-Alive 타임아웃 불일치
```

### 3. ss — 소켓 상태의 현대적 도구

```
ss vs netstat:
  netstat: 구식 (iproute 패키지 전), /proc/net 파싱
  ss:      현대적, netlink 소켓 사용 → 훨씬 빠름, 상세 정보

ss 주요 옵션:
  -t  : TCP 소켓
  -u  : UDP 소켓
  -l  : LISTEN 상태 소켓만
  -a  : 모든 상태 (LISTEN + ESTABLISHED + ...)
  -n  : 숫자로 출력 (호스트명 변환 안 함)
  -p  : 소켓을 사용하는 프로세스 정보
  -e  : 소켓 상세 정보 (uid, inode)
  -i  : TCP 내부 정보 (cwnd, rtt, rto)
  -s  : 통계 요약

자주 쓰는 조합:
  ss -tanp           # TCP 전체 + 숫자 + 프로세스
  ss -tlnp           # TCP LISTEN + 숫자 + 프로세스
  ss -tan state established  # ESTABLISHED만
  ss -tan state time-wait    # TIME_WAIT만

출력 분석:
  $ ss -tanp | head -5
  State    Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
  LISTEN   0       128     0.0.0.0:8080        0.0.0.0:*          pid=12345
  ESTAB    0       0       192.168.1.100:8080  192.168.1.50:54321 pid=12345
  TIME-WAIT 0      0       192.168.1.100:8080  192.168.1.50:54320

  State:   소켓 상태
  Recv-Q:  수신 버퍼에서 애플리케이션이 아직 읽지 않은 바이트 수
           LISTEN 상태에서는: SYN_RCVD 상태의 연결 수 (backlog 사용량)
  Send-Q:  송신 버퍼에서 아직 ACK를 받지 못한 바이트 수
           LISTEN 상태에서는: accept backlog 최대값

중요한 이상 신호:
  Recv-Q 지속적으로 증가 → 애플리케이션이 데이터를 처리 못함 (느린 consumer)
  Send-Q 지속적으로 증가 → 네트워크가 느리거나 수신자가 버퍼를 못 비움
  TIME_WAIT 대량 누적   → 단시간 대량 연결 생성/종료 (Connection Pool 미사용)
  CLOSE_WAIT 누적       → 애플리케이션이 FIN 받고 소켓을 close() 안 함

TCP 내부 정보 (cwnd, RTT):
  ss -ti  # TCP internal info
  출력 예시:
  ESTAB ... 
    cubic wscale:7,7 rto:204 rtt:1.5/0.75 ato:40 mss:1448 pmtu:1500
    rcvmss:1448 advmss:1448 cwnd:10 bytes_sent:1234 bytes_retrans:0
    bytes_acked:1234 segs_out:12 segs_in:11 ...
  
  rto:204    = Retransmission Timeout 204ms
  rtt:1.5    = 측정된 RTT 1.5ms
  cwnd:10    = 혼잡 윈도우 10 MSS (= 10 * 1448 = 14480 bytes)
  bytes_retrans:0  = 재전송 바이트 없음 (좋음)
```

### 4. Wireshark — GUI 패킷 분석

```
Wireshark 핵심 기능:

① 디스플레이 필터 (BPF가 아닌 Wireshark 전용 문법):
  tcp.port == 80           # 포트 80
  http                     # HTTP 프로토콜
  tcp.flags.syn == 1       # SYN 패킷
  tcp.flags.reset == 1     # RST 패킷
  ip.addr == 192.168.1.10  # IP 주소
  http.request.uri contains "/api"  # URL 포함
  tls.handshake.type == 1  # ClientHello
  tcp.analysis.retransmission  # 재전송 패킷

② Follow TCP Stream (가장 유용한 기능):
  패킷 우클릭 → Follow → TCP Stream
  → 해당 TCP 연결의 전체 데이터를 ASCII로 복원
  → HTTP 평문 트래픽이면 요청/응답 전체가 보임
  → 컬러: 빨강 = 클라이언트→서버, 파랑 = 서버→클라이언트

③ Statistics → TCP Stream Graph:
  Time-Sequence Graph: seq number 변화로 흐름 제어 시각화
  Throughput Graph:    처리량 변화
  RTT Graph:           RTT 변화 추이
  Window Scaling:      수신 윈도우 변화

④ tcpdump 파일을 Wireshark에서 열기:
  wireshark capture.pcap     # GUI
  tshark -r capture.pcap     # 커맨드라인 Wireshark

TLS 복호화 (개발 환경에서):
  SSLKEYLOGFILE 환경변수로 세션 키 저장
  export SSLKEYLOGFILE=/tmp/ssl-keys.log
  curl https://example.com
  
  Wireshark → Edit → Preferences → Protocols → TLS
  → (Pre)-Master-Secret log filename: /tmp/ssl-keys.log
  → TLS 트래픽이 복호화됨!
```

### 5. ping과 traceroute — L3 진단

```
ping 상세 활용:

  ping -c 5 target        # 5번만 전송
  ping -i 0.2 target      # 0.2초 간격으로 (빠른 테스트)
  ping -s 1400 target     # 패킷 크기 1400 bytes (MTU 테스트)
  ping -M do -s 1472 target  # DF=1, MTU Discovery
  ping -f target          # Flood ping (루트만 가능, 부하 테스트)
  ping -t 5 target        # TTL=5로 제한 (5홉 이내만)

ping 결과 해석:
  64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=15.2 ms
  
  ttl=118:  서버에서 출발한 TTL이 118로 도착
            128(Windows 기본) - 10홉 = 118
            → 경유 홉 수 ≈ 10개
  time=15.2 ms: RTT = 15.2ms

  정상: 일정한 RTT, 100% 응답
  이상: 높은 RTT → 혼잡 또는 먼 거리
        간헐적 손실 → 불안정한 경로
        100% 손실  → ICMP 차단 또는 호스트 다운

traceroute 심화:
  traceroute -n -w 1 target    # 응답 대기 1초, 빠른 결과
  traceroute -I target          # ICMP 기반 (일부 방화벽 우회)
  traceroute -U -p 53 target   # UDP + 포트 53
  traceroute -T -p 443 target  # TCP SYN 기반 (방화벽 우회)
  mtr target                   # traceroute + ping 실시간 결합 (최강)

mtr 출력:
  HOST               Loss%  Snt  Last  Avg  Best  Wrst  StDev
  1. 192.168.1.254   0.0%   10   0.8  0.9   0.7   1.1   0.1
  2. 10.0.0.1        0.0%   10   5.2  5.1   4.9   5.8   0.3
  3. ???             100%   10   0.0  0.0   0.0   0.0   0.0  ← ICMP 차단
  4. 72.14.233.85    0.0%   10  10.3 10.1   9.8  10.8   0.3

  Loss%: 패킷 손실률
  3번 홉의 100%는 ICMP 차단 (4번 홉이 정상이면 패킷 통과 중)
  StDev 높음: 지터(Jitter) 높음 → 불안정한 링크
```

### 6. 장애 진단 플로우 — 단계별 도구 사용

```
"API 응답이 없다" 상황에서의 체계적 진단:

Step 1: L3 연결 확인 (ping)
  ping -c 3 target-ip
  ✅ 응답: L3는 정상 → Step 2
  ❌ 무응답: ICMP가 차단됐을 가능성 → 다음 단계로 진행
  ❌ "Destination Host Unreachable": L3 경로 문제

Step 2: L4 포트 확인 (telnet 또는 nc)
  telnet target-ip 8080
  nc -zv target-ip 8080
  ✅ "Connected": 포트 열림 → Step 3
  ❌ "Connection refused": 서버가 해당 포트 안 열었음
  ❌ "Connection timed out": 방화벽이 SYN을 차단

Step 3: L7 확인 (curl)
  curl -v http://target-ip:8080/health
  curl -v --resolve host:8080:target-ip https://host:8080/health
  ✅ 200 OK: 정상
  ❌ SSL 오류: TLS 설정 문제
  ❌ 5xx: 서버 애플리케이션 오류
  ❌ 타임아웃: 서버가 응답을 안 줌

Step 4: 서버 소켓 상태 (ss)
  서버에서:
  ss -tanp | grep 8080
  ✅ LISTEN: 서버가 포트 열고 있음
  ❌ 없음: 서버 프로세스가 포트 안 열었음
  ⚠️  CLOSE_WAIT 많음: 소켓 누수 (FIN 받고 close() 안 함)
  ⚠️  Recv-Q 증가: 애플리케이션 처리 속도 느림

Step 5: 패킷 레벨 확인 (tcpdump)
  서버에서:
  sudo tcpdump -nn -i any 'port 8080 and host client-ip' -w /tmp/debug.pcap
  
  클라이언트에서 요청 재시도
  
  Ctrl+C 후 분석:
  sudo tcpdump -r /tmp/debug.pcap -nn
  
  SYN만 있고 SYN-ACK 없음 → iptables가 차단
  SYN-ACK 있는데 클라이언트에서 ACK 없음 → 클라이언트 측 문제
  RST 발생 → 연결이 강제 종료
  데이터 있는데 응답 없음 → 애플리케이션 문제

Step 6: 방화벽 확인
  sudo iptables -L -n -v | grep 8080
  sudo firewall-cmd --list-all  # CentOS/RHEL
  sudo ufw status               # Ubuntu
```

---

## 💻 실전 실험

### 실험 1: HTTP 요청 전체 패킷 분석

```bash
# HTTP 요청을 캡처하고 Wireshark용 파일로 저장
sudo tcpdump -i any -w /tmp/http-capture.pcap 'host example.com and port 80' &

# HTTP 요청 (평문)
curl -v http://example.com

# 캡처 종료
sudo kill %1

# 터미널에서 HTTP 내용 보기 (-A: ASCII 출력)
sudo tcpdump -r /tmp/http-capture.pcap -nn -A | head -100
# → GET / HTTP/1.1, Host, Accept 등 HTTP 헤더 전체가 평문으로 보임

# Wireshark가 있으면:
# wireshark /tmp/http-capture.pcap
# → TCP Stream Follow로 전체 HTTP 대화 복원
```

### 실험 2: TCP 상태 변화 실시간 관찰

```bash
# 터미널 1: 소켓 상태 실시간 모니터링
watch -n 0.5 "ss -tanp | grep -E 'State|ESTABLISHED|TIME|CLOSE|LISTEN'"

# 터미널 2: 연결 생성
curl http://httpbin.org/delay/3 &  # 3초 지연 응답

# 터미널 1에서 관찰:
# ESTABLISHED: curl 연결 중
# → 응답 받으면 FIN-WAIT → TIME-WAIT 전환 관찰

# CLOSE_WAIT 재현 (서버 측이 close()를 안 할 때)
# netcat으로 서버 역할:
nc -lk 9090 &
# 다른 터미널에서 연결 후 nc를 Ctrl+C
# → 클라이언트는 FIN을 보냈지만 nc(서버)가 응답 안 함 → CLOSE_WAIT

ss -tanp | grep 9090
# CLOSE_WAIT 상태 확인
```

### 실험 3: DNS 쿼리 패킷 분석

```bash
# DNS 쿼리 캡처 (UDP 53)
sudo tcpdump -nn -i any 'port 53' &

# DNS 조회
dig example.com

# 출력 분석:
# 15:00:01 IP 192.168.1.100.54321 > 8.8.8.8.53: 53+ A? example.com. (29)
#  → UDP, 출발지.랜덤포트 → 8.8.8.8.53
#  → 53+: 쿼리 ID 53, "+"는 재귀 쿼리 요청
#  → A?: A 레코드 조회
#  → (29): 패킷 크기 29 bytes

# 15:00:01 IP 8.8.8.8.53 > 192.168.1.100.54321: 53 1/0/0 A 93.184.216.34 (45)
#  → 응답: 쿼리 ID 53, 1개 Answer, 0개 Authority, 0개 Additional
#  → A 93.184.216.34: example.com의 IP

# HTTPS (DoH) 사용 시:
dig @1.1.1.1 example.com +https
sudo tcpdump -nn 'port 443 and host 1.1.1.1'
# → DNS 쿼리가 HTTPS로 암호화되어 전송됨 (port 53 패킷 없음)
```

### 실험 4: RST 패킷 재현 및 분석

```bash
# RST가 발생하는 상황 재현

# 터미널 1: 패킷 캡처
sudo tcpdump -nn -i lo 'port 9090' &

# 터미널 2: 서버 역할 (소켓을 열고 바로 닫음)
python3 -c "
import socket, time
s = socket.socket()
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.bind(('127.0.0.1', 9090))
s.listen(1)
conn, addr = s.accept()
# SO_LINGER로 즉시 RST 전송
import struct
s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', 1, 0))
conn.close()
"

# 터미널 3: 클라이언트로 연결
telnet 127.0.0.1 9090

# tcpdump에서 확인:
# ... Flags [S] → SYN
# ... Flags [S.] → SYN-ACK (연결 수립)
# ... Flags [R.] → RST (강제 종료!)
# → 클라이언트: "Connection reset by peer"
```

### 실험 5: ss로 Spring Boot 소켓 상태 분석

```bash
# Spring Boot 실행 가정 (포트 8080)

# LISTEN 소켓 확인
ss -tlnp | grep 8080
# LISTEN 0 100 *:8080 *:* users:(("java",pid=12345,fd=51))
# backlog: 100 (Tomcat의 acceptCount)

# 연결 부하 테스트 중 상태 확인
# ab -n 1000 -c 100 http://localhost:8080/api/test & 동시에:
watch -n 0.2 "ss -tan | grep 8080 | awk '{print \$1}' | sort | uniq -c"
# ESTABLISHED: 100 (동시 연결 수)
# TIME_WAIT:   점점 증가 (요청 처리 완료 후)

# cwnd, RTT 확인 (loopback은 항상 빠름)
ss -ti 'sport = :8080' | grep -A 5 ESTAB
# rtt:0.1/0.05 cwnd:10 → 매우 낮은 RTT (localhost)

# 실제 외부 연결에서는:
# rtt:5.2/2.6 cwnd:10 → 5ms RTT
# bytes_retrans:500 → 재전송 발생 중 (네트워크 불안정 신호)
```

---

## 📊 도구별 사용 상황 비교

```
장애 유형별 최적 도구:

┌──────────────────────────────────────────────────────────────────────┐
│  증상                    │  1차 도구          │  심화 도구                │
├──────────────────────────────────────────────────────────────────────┤
│  서버에 아예 못 붙음        │  ping, traceroute │  tcpdump (SYN 확인)      │
│  포트 연결 거부            │  telnet, nc       │  ss -tlnp (LISTEN 확인)  │
│  연결은 되는데 응답 없음     │  curl -v          │  tcpdump (데이터 확인)     │
│  간헐적 연결 실패          │  mtr              │  tcpdump -w (파일 저장)   │
│  응답이 매우 느림          │  curl -w "%{time}"│  ss -ti (cwnd, rtt)     │
│  CLOSE_WAIT 누적        │  ss -tan          │  lsof -p <pid>          │
│  TIME_WAIT 과다         │  ss -s            │  sysctl 확인             │
│  TLS 핸드쉐이크 실패       │  openssl s_client │  tcpdump TLS 필터        │
│  DNS 조회 실패           │  dig, nslookup    │  tcpdump port 53        │
│  패킷 손실               │  ping -f          │  tcpdump + 재전송 필터     │
└──────────────────────────────────────────────────────────────────────┘

curl -w 옵션으로 단계별 시간 측정:
  curl -w "
    DNS:        %{time_namelookup}s
    TCP 연결:   %{time_connect}s
    TLS 핸드쉐이크: %{time_appconnect}s
    첫 바이트:  %{time_starttransfer}s
    전체:       %{time_total}s
  " -o /dev/null -s https://example.com
  
  → 각 단계의 시간을 정확히 분리해서 병목 지점 확인
```

---

## ⚖️ 트레이드오프

```
도구 선택의 트레이드오프:

tcpdump vs Wireshark:
  tcpdump:
    ✅ 서버에서 바로 실행 가능 (CLI)
    ✅ 파이프로 grep, awk 연결 가능
    ❌ 복잡한 프로토콜 분석 어려움
    → 실시간 서버 진단에 적합
  
  Wireshark:
    ✅ GUI로 TCP Stream Follow, 그래프 시각화
    ✅ TLS 복호화, 프로토콜 자동 파싱
    ❌ 서버에서 직접 실행 어려움 (X11 필요)
    → tcpdump로 캡처(-w) 후 로컬에서 Wireshark로 분석

ss vs netstat:
  ss:
    ✅ 빠름 (netlink 기반)
    ✅ TCP 내부 정보 (-i 옵션)
    ✅ 최신 Linux에서 권장
  netstat:
    ⚠️ 느림 (/proc 파싱)
    ✅ 오래된 시스템에서도 사용 가능

tcpdump 오버헤드:
  -w 파일 저장: 낮음 (커널 BPF 필터링)
  -A ASCII 출력: 중간
  고트래픽 환경에서 tcpdump 자체가 CPU 사용
  → Ring buffer 설정: -B 4096 (버퍼 크기 증가)
  → 필터를 최대한 좁게 → 캡처 대상 최소화
```

---

## 📌 핵심 정리

```
네트워크 진단 도구 핵심 요약:

단계별 장애 진단 플로우:
  ping → L3 확인 (IP 도달)
  telnet/nc → L4 확인 (TCP 포트)
  curl -v → L7 확인 (HTTP/TLS)
  ss -tanp → 소켓 상태 확인
  tcpdump → 패킷 레벨 확인

tcpdump 핵심:
  -nn -i any: 숫자 출력, 모든 인터페이스
  -w file: Wireshark용 저장
  -A: ASCII 출력 (HTTP 평문 확인)
  필터: 'host X and port Y', 'tcp[tcpflags] & tcp-rst != 0'

ss 핵심:
  ss -tanp: TCP 전체 + 프로세스
  ss -tlnp: LISTEN만
  ss -ti: cwnd, rtt, 재전송 정보
  이상 신호: CLOSE_WAIT 누적, Recv-Q 증가

tcpdump 플래그:
  [S] SYN, [S.] SYN-ACK, [.] ACK
  [P.] PSH+ACK (데이터), [F.] FIN, [R] RST

curl 시간 측정:
  -w "%{time_connect}/%{time_appconnect}/%{time_total}"
  → DNS/TCP/TLS/HTTP 각 단계 시간 분리

mtr: traceroute + ping 실시간 (가장 강력한 경로 진단)
```

---

## 🤔 생각해볼 문제

**Q1.** `ss -tanp`에서 Spring Boot 서버의 `Recv-Q` 값이 지속적으로 증가하고 있다. 이것이 의미하는 것은 무엇이며, 원인과 해결 방법은?

<details>
<summary>해설 보기</summary>

`Recv-Q`가 증가한다는 것은 **OS의 수신 버퍼(TCP Receive Buffer)에 데이터가 쌓이고 있지만, 애플리케이션(Spring Boot)이 데이터를 충분히 빠르게 읽지 못하고 있다**는 의미입니다.

원인 분석:
1. **애플리케이션 처리 속도 부족**: 요청이 들어오는 속도보다 처리 속도가 느림 (스레드 부족, DB 병목 등)
2. **스레드 풀 고갈**: Tomcat의 maxThreads 초과 → 요청을 수신 버퍼에 쌓아둠
3. **GC Stop-the-World**: Java GC로 인한 처리 일시 중단

진단 단계:
```bash
# 1. 스레드 상태 확인
jstack <pid> | grep "WAITING\|BLOCKED" | wc -l

# 2. Tomcat 스레드 풀 모니터링 (Actuator)
curl http://localhost:8080/actuator/metrics/tomcat.threads.busy

# 3. GC 로그 확인
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps

# 4. CPU 사용률
top -p <pid>
```

해결 방법:
- `server.tomcat.max-threads` 증가 (기본 200)
- DB 쿼리 최적화 (병목 제거)
- 비동기 처리 도입 (`@Async`, WebFlux)
- 수평 스케일아웃

</details>

---

**Q2.** `tcpdump`로 캡처했더니 `Flags [R.]` 패킷이 자주 보인다. 어떤 상황에서 RST가 발생하는가? 각각을 어떻게 구별하는가?

<details>
<summary>해설 보기</summary>

RST(Reset)은 TCP 연결을 즉시 강제 종료하는 패킷입니다. 주요 발생 상황:

**1. Connection Refused (닫힌 포트)**
```
클라이언트 → SYN → 서버
서버 → RST+ACK → 클라이언트 (해당 포트에 LISTEN 소켓 없음)
```
확인: `telnet target port` → "Connection refused"

**2. 방화벽 REJECT**
```
iptables -j REJECT → RST 전송
iptables -j DROP   → 응답 없음 (타임아웃)
```
구별: RST는 즉시 오고, DROP은 타임아웃

**3. Keep-Alive 타임아웃 불일치**
```
방화벽: 5분 후 세션 삭제
TCP 연결: 아직 유효하다고 판단
다음 패킷 전송 → 방화벽이 RST 전송 (테이블에 없는 연결)
```
확인: RST 직전에 오랜 시간 데이터 없음

**4. 애플리케이션이 SO_LINGER로 강제 종료**
```java
socket.setSoLinger(true, 0);  // RST 전송
```

**5. 반만 열린 연결 (Half-Open)**
서버가 재시작됐는데 클라이언트가 모르는 상황 → 클라이언트가 데이터 전송 → 서버가 RST

tcpdump로 구별:
```bash
# RST 직전 패킷 확인
sudo tcpdump -nn -r capture.pcap | grep -B 5 "Flags \[R"
# → 직전에 SYN만 있음: Connection Refused
# → 직전에 오래된 타임스탬프: Keep-Alive 불일치
# → 직전에 데이터 있음: 애플리케이션 강제 종료
```

</details>

---

**Q3.** `curl -v`로 요청 시 "Connection timed out"이 나오고, `telnet`도 타임아웃이다. 하지만 `ping`은 성공한다. 어떤 원인이 가능하고 다음 진단 단계는?

<details>
<summary>해설 보기</summary>

이 증상은 **L3(IP)는 정상이지만 L4(TCP) 포트가 막혀 있음**을 의미합니다.

가능한 원인:
1. **방화벽(iptables/ufw)이 해당 포트 차단**: SYN을 DROP하면 응답 없이 타임아웃
2. **서버 프로세스가 해당 포트를 열지 않음**: 이 경우 RST가 와야 하는데 타임아웃이면 방화벽이 더 유력
3. **보안 그룹(클라우드)에서 인바운드 차단**: AWS Security Group 등

다음 진단 단계:
```bash
# 1. 서버에서 LISTEN 확인
ss -tlnp | grep <port>
# → LISTEN 없음: 프로세스가 포트 안 열었음
# → LISTEN 있음: 방화벽 문제

# 2. 방화벽 규칙 확인
sudo iptables -L -n | grep <port>
sudo ufw status

# 3. tcpdump로 SYN 패킷이 서버에 도착하는지 확인 (서버에서)
sudo tcpdump -nn 'port <port>'
# → SYN 패킷이 보이지 않음: 네트워크 경로 중간에 차단
# → SYN 패킷이 보이는데 응답 없음: 서버의 iptables가 DROP

# 4. REJECT vs DROP 구별
# REJECT: 즉시 RST 또는 ICMP Port Unreachable 응답
# DROP: 타임아웃 (응답 없음) ← 현재 증상
sudo iptables -L -n | grep DROP  # DROP 규칙 확인
```

</details>

---

<div align="center">

**[⬅️ 이전: NAT와 포트 포워딩](./04-nat-and-port-forwarding.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — TCP 3-Way Handshake ➡️](../tcp-internals/01-three-way-handshake.md)**

</div>
