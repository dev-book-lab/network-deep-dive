# TCP 신뢰성 보장 — Sequence Number, ACK, 재전송

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Sequence Number는 어떻게 패킷 손실, 중복, 순서 뒤바뀜을 감지하는가?
- ACK Number가 "다음에 받기를 기대하는 번호"인 이유는 무엇인가?
- RTO(Retransmission Timeout)는 어떻게 계산되며, 왜 고정값이 아닌가?
- Karn's Algorithm은 왜 필요하고 어떻게 동작하는가?
- Cumulative ACK와 Selective ACK(SACK)의 차이는 무엇인가?
- `tcpdump`에서 재전송 패킷을 어떻게 식별하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
네트워크는 신뢰할 수 없다:
  패킷 손실: 라우터 버퍼 고갈, 링크 오류
  패킷 중복: 라우팅 루프, 네트워크 장비 버그
  순서 뒤바뀜: 멀티패스 라우팅, ECMP

TCP가 이 위에서 신뢰성을 제공하는 방법:
  모든 바이트에 Sequence Number 부여
  수신 확인(ACK) 없으면 재전송
  중복 → 버림, 순서 뒤바뀜 → 버퍼에서 재조합

실무에서 재전송이 중요한 이유:
  "간헐적으로 요청이 느려요" 원인의 상당수:
    tcpdump에서 TCP Retransmission 패킷 발견
    → 패킷 손실 → 재전송 → RTO만큼 지연
    
  ss -ti로 bytes_retrans 확인:
    bytes_retrans:0    → 정상
    bytes_retrans:5000 → 재전송 발생 → 네트워크 불안정

  재전송은 성능 문제 뿐 아니라 장애의 신호:
    NIC 오류, 케이블 문제, 혼잡한 업스트림 링크 등
```

---

## 😱 흔한 실수

```
Before — 재전송 원리를 모를 때:

실수 1: "타임아웃을 짧게 설정하면 빠른 재시도가 가능하다"
  connectTimeout = 100ms 설정
  RTO가 200ms인 상황에서:
  → TCP가 재전송 시도 중인데 애플리케이션이 포기
  → 실제로는 패킷이 잠시 후 도달할 수 있었음
  → 너무 짧은 타임아웃 = 불필요한 재연결 비용

실수 2: 패킷 손실과 서버 다운을 혼동
  "요청 타임아웃이 발생했어요"
  → 서버 문제로 판단 후 재시작
  → 실제 원인: 네트워크 혼잡으로 인한 일시적 패킷 손실 + 재전송
  → tcpdump로 확인하면 Retransmission 이후 정상 응답 확인 가능

실수 3: SACK가 기본 활성화인 줄 모름
  오래된 방화벽이 SACK 옵션을 포함한 패킷을 차단
  → "특정 환경에서만 연결이 느림"
  → SACK 차단 시 Cumulative ACK만으로 처리 → 성능 저하
```

---

## ✨ 올바른 접근

```
After — 재전송 원리를 알고 나면:

재전송 모니터링:
  # 연결별 재전송 통계
  ss -ti 'dst 192.168.1.1'
  → retrans:5/1000 → 0.5% 손실률 → 네트워크 점검 필요

  # 시스템 전체 재전송 통계
  netstat -s | grep "segments retransmitted"
  # 증가 추세이면 네트워크 불안정

tcpdump로 재전송 식별:
  sudo tcpdump -nn 'tcp' | grep -i retransmit
  # 또는 Wireshark에서:
  # 필터: tcp.analysis.retransmission

타임아웃 설계:
  connectTimeout > RTT * 2 (Handshake 포함)
  readTimeout > 서버 처리 시간 + RTT + 재전송 마진
  
  예: 서버 처리 3초, RTT 50ms, 재전송 한 번 허용
  readTimeout = 3000 + 50 + 200(RTO) = 3250ms → 4000ms 설정

SACK 활성화 확인:
  sysctl net.ipv4.tcp_sack
  # 1 = 활성화 (Linux 기본값)
```

---

## 🔬 내부 동작 원리

### 1. Sequence Number — 모든 바이트에 번호를

```
Sequence Number 부여 원리:

  연결 수립 시 ISN(Initial Sequence Number) = 1000 (예시)
  
  데이터 전송:
  ┌─────────────────────────────────────────────────────────┐
  │  세그먼트 1: seq=1001, data="Hello " (6바이트)              │
  │  세그먼트 2: seq=1007, data="World"  (5바이트)              │
  │  세그먼트 3: seq=1012, data="!"      (1바이트)              │
  └─────────────────────────────────────────────────────────┘

  seq=1001: 이 세그먼트의 첫 번째 바이트는 전체 스트림의 1001번
  seq=1007: 이 세그먼트의 첫 번째 바이트는 전체 스트림의 1007번

ACK Number = 다음에 기대하는 번호:
  세그먼트 1(seq=1001, 6바이트) 수신 후:
    ACK = 1007 ("1007번 바이트부터 보내줘")
  세그먼트 2(seq=1007, 5바이트) 수신 후:
    ACK = 1012 ("1012번 바이트부터 보내줘")

패킷 손실 감지:
  세그먼트 2가 손실된 경우:
    수신자: seq=1001(6B) 수신 → ACK=1007
    수신자: seq=1012(1B) 수신 → ACK=1007 (여전히! 1007~1011이 없음)
    수신자: seq=1007이 없으니 1012는 버퍼에 보관
    → ACK=1007을 계속 보냄 → Duplicate ACK
    → 송신자: 같은 ACK 3번 → Fast Retransmit (재전송)

중복 패킷 감지:
  세그먼트 1이 중복 도착:
    수신자: 이미 ACK=1007 보냄 → seq=1001 재도착
    → seq 1001~1006은 이미 처리 완료
    → 버림 + ACK=1007 다시 전송 (중복 ACK)

순서 뒤바뀜 처리:
  세그먼트 3(seq=1012)이 세그먼트 2(seq=1007)보다 먼저 도착:
    seq=1012를 수신 버퍼에 보관 (Out-of-Order)
    ACK=1007 전송 (아직 1007을 못 받음)
    이후 seq=1007 도착 → 버퍼의 seq=1012와 합쳐서 처리
    ACK=1013 전송
```

### 2. RTO — 동적 재전송 타이머

```
RTO(Retransmission Timeout):
  ACK를 기다리는 시간 → 초과 시 재전송

왜 고정값이 아닌가:
  네트워크 환경에 따라 RTT가 다름
  RTT=1ms → RTO=1ms면 너무 민감 (정상 ACK도 타임아웃 처리)
  RTT=200ms → RTO=1ms면 너무 촉박 (재전송 폭증)

Jacobson's Algorithm (RFC 6298):
  SRTT = Smoothed RTT (지수 이동 평균)
  RTTVAR = RTT Variance (변동성)

  초기값: RTO = 1초

  RTT 샘플 측정할 때마다:
    RTTVAR = (1 - β) × RTTVAR + β × |SRTT - RTT_sample|
    SRTT   = (1 - α) × SRTT   + α × RTT_sample
    RTO    = SRTT + max(G, 4 × RTTVAR)

    α = 0.125 (새 샘플 반영 비율)
    β = 0.25  (변동성 반영 비율)
    G = 클락 그래뉼러리티 (보통 1ms)

예시:
  RTT 샘플 = 10ms
  초기: SRTT=10, RTTVAR=5
  RTO = 10 + max(1, 4×5) = 30ms

  다음 RTT 샘플 = 12ms:
  RTTVAR = 0.75×5 + 0.25×|10-12| = 4.25
  SRTT   = 0.875×10 + 0.125×12 = 10.25
  RTO    = 10.25 + max(1, 4×4.25) = 27.25ms

재전송 시 RTO 변화 (Exponential Backoff):
  1차 재전송: RTO × 2
  2차 재전송: RTO × 4
  3차 재전송: RTO × 8
  ...
  최대: 64×RTO (Linux 기본 약 120초)
  → 지수적 증가로 네트워크 추가 혼잡 방지
```

### 3. Karn's Algorithm — 재전송 시 RTT 측정 문제

```
문제: 재전송 후 ACK가 오면 어느 전송의 RTT인가?

  ① 세그먼트 A 전송 (t=0)
  ② 타임아웃 → 세그먼트 A 재전송 (t=100ms)
  ③ ACK 수신 (t=150ms)

  RTT = 150 - 0 = 150ms? (원본 기준) → 원본이 손실됐으므로 부정확
  RTT = 150 - 100 = 50ms? (재전송 기준) → 재전송 시점 기준

  어느 RTT 샘플을 써야 하는가? → 알 수 없음

Karn's Algorithm 해결책:
  규칙 1: 재전송된 세그먼트에 대한 ACK로는 RTT 측정하지 않음
  규칙 2: 재전송 발생 시 RTO를 두 배로 증가 (backoff)
           정상 ACK가 올 때까지 이 doubled RTO 유지

결과:
  재전송 직후에는 보수적인(큰) RTO 사용
  → 불안정한 네트워크에서 재전송 폭풍 방지
  → 정상 전송의 ACK를 받으면 다시 정상 RTT 측정 재개

Timestamps 옵션 (RFC 1323):
  Karn's Algorithm 한계 극복
  각 세그먼트에 타임스탬프 포함
  ACK에 원본 타임스탬프 echo
  → 재전송이든 원본이든 정확한 RTT 측정 가능
  → SYN에서 Timestamps 옵션 협상 (TS val, TS ecr 필드)
```

### 4. Cumulative ACK vs SACK

```
Cumulative ACK (기본):
  ACK = "이 번호까지 모두 받았음"

  수신 상황: 1,2,3,5,6 수신 (4 손실)
  ACK = 4 ("3까지 받았고, 4를 기다리는 중")
  → 4가 재전송되면 1~6 전체를 한번에 ACK 가능
  
  문제: 5,6을 받았는데 4만 재전송하면 됨
        하지만 송신자는 5,6도 재전송할 수 있음 (불필요한 재전송)
        → 손실이 많을 때 매우 비효율

SACK (Selective Acknowledgment, RFC 2018):
  "어떤 범위를 받았는지" 구체적으로 알림

  수신 상황: 1,2,3,5,6 수신 (4 손실)
  ACK = 4, SACK = {5-6}
  → "4는 없고, 5~6은 받았어"

  수신 상황: 1,2,3,5,6,8,9 수신 (4,7 손실)
  ACK = 4, SACK = {5-6, 8-9}
  → "4,7이 없고, 5-6, 8-9는 있어"

  → 송신자: 4와 7만 재전송 (5,6,8,9는 불필요)

SACK 동작 흐름:
  SYN에서 "SACK Permitted" 옵션으로 협상
  
  ┌──────────────────────────────────────────────────────────┐
  │  수신됨:   [1][2][3]  [5][6]  [8][9]                       │
  │  손실:                [4]      [7]                        │
  │                                                          │
  │  Cumulative ACK: 4 (3까지 연속 수신)                        │
  │  SACK Block 1: {5-6} (연속 수신 블록)                       │
  │  SACK Block 2: {8-9} (연속 수신 블록)                       │
  │                                                          │
  │  송신자: 4와 7만 재전송                                      │
  └──────────────────────────────────────────────────────────┘

D-SACK (Duplicate SACK):
  중복 수신된 범위를 알림
  "1-5 받았는데 1-3이 또 왔어"
  → 불필요한 재전송이 발생했음을 감지
  → 재전송 정책 개선에 활용
```

### 5. Fast Retransmit — 타임아웃 전에 재전송

```
세그먼트 손실 시 타임아웃을 기다리지 않는 방법:

Duplicate ACK 기반 Fast Retransmit:
  
  송신: [1][2][3][4][5][6][7]
        [3] 손실 가정
  
  수신자:
    [1] 수신 → ACK=2
    [2] 수신 → ACK=3
    [3] 없음
    [4] 수신 → ACK=3 (Duplicate ACK #1: "3을 아직 못 받음")
    [5] 수신 → ACK=3 (Duplicate ACK #2)
    [6] 수신 → ACK=3 (Duplicate ACK #3)
  
  송신자:
    같은 ACK(3)를 3번 받음 → 3이 손실됐다고 판단
    → RTO 타임아웃 전에 즉시 재전송 (Fast Retransmit)
    → [3] 재전송
  
  수신자:
    [3] 수신 → 버퍼에 있던 [4][5][6]와 합쳐서 → ACK=7

왜 3번인가:
  1~2번 Duplicate ACK: 순서 뒤바뀜으로도 발생 가능 (오탐 방지)
  3번: 높은 확률로 손실 → 재전송 결정
  
  이 임계값을 "dupthresh" 또는 "tcpRexmitThresh"라 부름
  기본값: 3

Fast Retransmit vs RTO Retransmit:
  Fast Retransmit: Duplicate ACK 3회 → 즉각 재전송
                   → 재전송 지연: 수 ms ~ 수십 ms
  RTO Retransmit:  타임아웃 후 재전송
                   → 재전송 지연: 수백 ms ~ 수초
  
  Fast Retransmit가 훨씬 빠름
  → 네트워크 처리량과 지연에 큰 차이
```

---

## 💻 실전 실험

### 실험 1: tcpdump로 재전송 패킷 식별

```bash
# 재전송 패킷 캡처 (seq 번호 중복 확인)
sudo tcpdump -nn -i any 'tcp and host target-ip' -w /tmp/retrans.pcap &

# 부하 테스트로 패킷 손실 유발 (실험적)
# tc (traffic control)로 인위적 패킷 손실
sudo tc qdisc add dev eth0 root netem loss 5%  # 5% 손실 주입
curl http://target-ip/large-file > /dev/null
sudo tc qdisc del dev eth0 root  # 정리

# Wireshark에서 열기
wireshark /tmp/retrans.pcap
# 필터: tcp.analysis.retransmission
# → 빨간색으로 표시된 재전송 패킷 확인

# tshark (커맨드라인 Wireshark)로 재전송만 추출
tshark -r /tmp/retrans.pcap -Y "tcp.analysis.retransmission"
```

### 실험 2: ss로 재전송 통계 확인

```bash
# 연결별 TCP 상세 정보
ss -ti 'dst example.com'
# 출력:
# ESTAB ...
#   cubic wscale:7,7 rto:204 rtt:50/25 ato:40 mss:1460
#   rcvmss:1448 advmss:1448 cwnd:10
#   retrans:0/5 lost:0 sacked:0 dsack_dups:0

# retrans:0/5 의미:
#   현재 재전송 대기 중: 0
#   누적 재전송 횟수: 5

# 시스템 전체 재전송 통계
netstat -s | grep -E "segments (sent|retransmitted|received)"
# segments sent out:        1000000
# segments retransmitted:   1500     ← 0.15% 재전송률

# 재전송률 계산
SENT=$(netstat -s | grep "segments sent" | awk '{print $1}')
RETRANS=$(netstat -s | grep "segments retransmitted" | awk '{print $1}')
echo "재전송률: $(echo "scale=4; $RETRANS/$SENT*100" | bc)%"
```

### 실험 3: SACK 동작 관찰

```bash
# SACK 활성화 확인
sysctl net.ipv4.tcp_sack
# 1 = 활성화

# tcpdump에서 SACK 옵션 확인 (-vv 옵션)
sudo tcpdump -nn -i any -vv 'tcp and host target-ip' | grep -A 5 "sack"
# 출력 예시:
# options [nop,nop,sack 1 {1461:2921}]
# → SACK Block: seq 1461~2921 수신 완료 알림
# → 1~1460은 아직 없음 (Cumulative ACK)

# SACK 비활성화 테스트 (비교용)
sudo sysctl -w net.ipv4.tcp_sack=0
# 패킷 손실 시 재전송 횟수 증가 확인
# 테스트 후 복원:
sudo sysctl -w net.ipv4.tcp_sack=1
```

### 실험 4: RTO 동작 관찰

```bash
# 인위적으로 패킷 손실 후 RTO 변화 관찰
sudo tc qdisc add dev lo root netem loss 30%  # 30% 손실

# ss -ti로 rto 값 모니터링
watch -n 0.5 "ss -ti 'dst 127.0.0.1' | grep rto"
# rto 값이 재전송마다 2배씩 증가하는 것 확인:
# rto:200 → rto:400 → rto:800 → ...

# 복원
sudo tc qdisc del dev lo root
```

---

## 📊 성능/비용 비교

```
재전송 방식별 성능 비교:

시나리오: 패킷 손실 1개, RTT=50ms

RTO Retransmit만 사용:
  손실 감지: RTO = 200ms (초기값)
  재전송 후 ACK: +50ms
  총 지연: 250ms

Fast Retransmit 사용 (SACK 없음):
  Duplicate ACK 3개 수집: 3 × RTT = 150ms
  재전송 후 ACK: +50ms
  총 지연: 200ms
  불필요 재전송 가능성: 높음 (손실 이후 패킷도 재전송)

Fast Retransmit + SACK:
  Duplicate ACK 3개 수집: 150ms
  필요한 것만 재전송: 50ms
  총 지연: 200ms
  불필요 재전송: 없음

패킷 손실률과 처리량의 관계 (Mathis Formula):
  Throughput ≈ MSS / (RTT × √loss_rate)

  RTT=50ms, MSS=1460B:
    손실률 0%:   이론적 최대
    손실률 0.1%: 약 924 Kbps
    손실률 1%:   약 292 Kbps
    손실률 10%:  약 92 Kbps

  → 1% 손실이 처리량을 1/10 이하로 감소
  → 네트워크 안정성이 성능에 직결
```

---

## ⚖️ 트레이드오프

```
신뢰성 vs 성능:

Cumulative ACK만 사용:
  장점: 구현 단순
  단점: 다중 손실 시 비효율적 재전송
        한 번에 하나씩 재전송 → 고손실 환경에서 느림

SACK:
  장점: 필요한 것만 재전송 → 고손실 환경에서 효율적
  단점: 구현 복잡, ACK에 옵션 공간 추가 필요
        옛 방화벽이 SACK 옵션 포함 패킷 차단 가능

RTO 조정:
  짧게: 빠른 재전송 감지, 하지만 오탐 가능 (정상 지연도 손실로 판단)
  길게: 안정적, 하지만 실제 손실 대응이 느림

TCP vs UDP + 애플리케이션 재전송:
  TCP: OS가 재전송 처리, 애플리케이션 투명
       단점: Head-of-Line Blocking (연결 수준)
  UDP + App: 유연한 재전송 정책 (필요한 것만, 시간 제한)
             단점: 구현 복잡
             예: QUIC, RTP, RUDP

Aggressive vs Conservative 설정:
  tcp_retries2 = 15 (기본): 최대 15번 재전송 후 포기
  줄이면: 불안정 연결 빠르게 포기 (클라우드 환경에서 유리)
  늘리면: 더 오래 시도 (이동 환경, 불안정 링크에서 유리)
```

---

## 📌 핵심 정리

```
TCP 신뢰성 보장 핵심 요약:

Sequence Number 역할:
  모든 바이트에 번호 부여
  손실 감지: ACK가 같은 번호에서 멈춤 (Duplicate ACK)
  중복 감지: 이미 받은 번호 재도착 → 버림
  순서 재조합: 번호로 정렬 후 애플리케이션에 전달

ACK Number:
  "다음에 받기를 기대하는 바이트 번호"
  누적(Cumulative): 해당 번호까지 모두 수신 완료

RTO(Retransmission Timeout):
  SRTT + 4×RTTVAR (동적 계산)
  재전송마다 2배 증가 (Exponential Backoff)
  재전송 시 RTT 측정 안 함 (Karn's Algorithm)

Fast Retransmit:
  Duplicate ACK 3회 → 타임아웃 전에 즉시 재전송
  RTO 재전송보다 훨씬 빠름 (ms vs 수백ms)

SACK:
  수신한 불연속 범위를 구체적으로 알림
  → 필요한 것만 재전송 (고손실 환경에서 효율적)
  sysctl net.ipv4.tcp_sack (기본 1=활성화)

진단 명령어:
  ss -ti 'dst X': rto, rtt, retrans 확인
  netstat -s | grep retransmit: 시스템 전체 재전송 수
  tcpdump: tcp.analysis.retransmission 필터
```

---

## 🤔 생각해볼 문제

**Q1.** 패킷이 손실됐는데 Duplicate ACK가 3개 오지 않아서 Fast Retransmit이 발동하지 않는 경우는 어떤 상황인가?

<details>
<summary>해설 보기</summary>

Duplicate ACK가 3개 오려면 손실된 세그먼트 이후에 **3개의 세그먼트가 도착**해야 합니다.

**Fast Retransmit이 발동하지 않는 경우:**

1. **전송 중인 데이터가 부족한 경우**: 전체 스트림에서 세그먼트가 3개 미만 남아있을 때 (예: 마지막 세그먼트가 손실되면 그 이후 세그먼트가 없음)

2. **스트림 끝의 세그먼트 손실**: 마지막 패킷이 손실되면 Duplicate ACK가 오지 않음 → RTO 타임아웃만 동작

3. **TCP Window가 너무 작을 때**: 한 번에 3개 이상의 세그먼트를 보낼 수 없으면 발동 불가

이 경우 RTO 타임아웃을 기다려야 합니다. 이것이 TCP의 "tail loss" 문제이며, **TLP(Tail Loss Probe)** 라는 메커니즘으로 개선됩니다. Linux는 RTO 전에 probe 패킷을 보내 빠른 손실 감지를 시도합니다.

`sysctl net.ipv4.tcp_early_retrans`: TLP 설정 확인

</details>

---

**Q2.** `ss -ti` 출력에서 `retrans:3/150`을 발견했다. 이것이 의미하는 것과 어떻게 추가 진단하는가?

<details>
<summary>해설 보기</summary>

`retrans:현재대기/누적횟수`를 의미합니다.
- `retrans:3/150`: 현재 3개의 세그먼트가 재전송 대기 중, 이 연결에서 누적 재전송이 150회

**150회 누적 재전송 해석:**
- 이 연결 동안 150번의 패킷 손실(또는 타임아웃)이 있었음
- 연결 수명과 전송 데이터량 대비 비율로 심각도 판단

**추가 진단:**
```bash
# 현재 연결의 상세 정보
ss -ti 'dst <target-ip>'
# rtt: 값이 불안정하게 변동하는지 확인
# cwnd: 혼잡 윈도우가 작게 줄어 있는지 확인

# 패킷 레벨 확인
sudo tcpdump -nn -i any 'host <target-ip>' -w /tmp/debug.pcap
# Wireshark에서 tcp.analysis.retransmission 필터

# 네트워크 경로 품질 확인
mtr --report <target-ip>
# Loss% 컬럼에서 어느 홉에서 손실이 발생하는지 확인

# NIC 오류 확인
ip -s link show eth0
# RX errors, TX errors 증가 여부
ethtool -S eth0 | grep -i error
```

</details>

---

**Q3.** 같은 파일을 HTTP로 다운로드할 때, 네트워크 손실률이 1%인 환경에서 TCP와 UDP+자체재전송 중 어느 것이 더 빠른가?

<details>
<summary>해설 보기</summary>

**상황에 따라 다릅니다.** 핵심 차이는 TCP의 Head-of-Line Blocking입니다.

**TCP:**
- 패킷 손실 시 해당 패킷이 재전송될 때까지 이후 모든 데이터 차단
- SACK가 있어도 연결 단위에서 블로킹 발생
- 1% 손실에서 Mathis 공식: `Throughput ≈ MSS / (RTT × √0.01)` → 처리량 1/10 수준

**UDP + 자체 재전송 (QUIC 방식):**
- 스트림별 독립 재전송: 패킷 1이 손실됐어도 패킷 2,3,4는 전달 가능
- 손실된 것만 재전송 → 전체 처리량 영향 최소화
- 단, 구현 복잡도가 높음

**단일 파일 다운로드의 경우:**
- 단일 TCP 연결에서는 두 방식 차이가 큰 차이 없을 수 있음
- 다만 여러 스트림을 동시에 처리하는 HTTP/3(QUIC)이 HTTP/2(TCP)보다 손실 환경에서 훨씬 유리
- 손실률 1%에서 HTTP/3은 HTTP/2 대비 3~5배 처리량 차이도 가능

**결론:** 단순 파일 다운로드 1개는 TCP가 충분하지만, 여러 리소스(이미지, JS, CSS)를 동시에 받는 웹 환경에서는 손실 환경에서 QUIC(HTTP/3)이 명확히 유리합니다.

</details>

---

<div align="center">

**[⬅️ 이전: TIME_WAIT](./02-four-way-handshake-time-wait.md)** | **[홈으로 🏠](../README.md)** | **[다음: TCP 흐름 제어 ➡️](./04-flow-control-sliding-window.md)**

</div>
