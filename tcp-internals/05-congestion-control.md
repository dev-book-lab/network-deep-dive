# TCP 혼잡 제어 — Slow Start, AIMD, Fast Recovery

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 흐름 제어(rwnd)와 혼잡 제어(cwnd)는 각각 무엇을 보호하는가?
- Slow Start에서 cwnd가 "지수적으로" 증가하는 원리는?
- ssthresh에 도달하면 증가 방식이 왜 선형으로 바뀌는가?
- 패킷 손실 감지 시 Fast Recovery와 Timeout Retransmit에서 cwnd가 각각 어떻게 달라지는가?
- BBR과 CUBIC은 어떻게 다르고 어느 환경에서 각각 유리한가?
- `ss -ti`에서 cwnd, ssthresh, rtt 값을 어떻게 해석하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"왜 연결 초기에 느리다가 점점 빨라지는가?":
  Slow Start 때문
  cwnd=10(MSS)으로 시작 → 매 RTT마다 두 배 증가
  → 초기 연결: 14.6KB → 29.2KB → 58.4KB → ... 점점 빨라짐

"짧은 HTTP 요청은 왜 긴 파일 전송보다 대역폭 활용률이 낮은가?":
  요청 응답 전에 Slow Start가 완료 안 됨
  ssthresh에 도달하기 전에 연결이 끊김
  → HTTP/2의 연결 재사용이 중요한 이유

  HTTP/1.1 매 요청 새 연결:
    Slow Start 반복 → 처음 몇 RTT는 항상 느림
  
  HTTP/2 커넥션 재사용:
    이미 Congestion Avoidance 단계 → cwnd 큼 → 즉시 빠름

"갑자기 응답이 느려졌다가 회복됐어요":
  Timeout Retransmit → cwnd = 1 MSS (가장 심각한 리셋)
  → Slow Start 다시 시작 → 회복하는 데 수 초 소요
  → 진단: ss -ti에서 cwnd 값이 1 또는 매우 작으면 신호
```

---

## 😱 흔한 실수

```
Before — 혼잡 제어를 모를 때:

실수 1: 초기 cwnd가 작은 것을 서버 성능 문제로 오인
  "연결 직후 처음 몇 초가 느린데 서버 문제 아닌가요?"
  → Slow Start: 연결 초기에는 cwnd가 작음
  → 시간이 지나면 cwnd가 커져서 정상 속도
  → 해결: initcwnd를 높이거나 연결 재사용

실수 2: 혼잡 창이 리셋된 후 원인을 모름
  "잘 되다가 갑자기 느려졌어요"
  → Timeout Retransmit → cwnd = 1
  → 패킷 손실 → 재전송 → 혼잡 창 리셋
  → ss -ti: cwnd:1 + retrans 증가 → 네트워크 혼잡 신호

실수 3: BBR vs CUBIC 선택 기준 모름
  클라우드 환경에서 기본 알고리즘(CUBIC)으로 방치
  → 고지연 환경에서 BBR이 훨씬 유리
  → sysctl net.ipv4.tcp_congestion_control 확인 필요
```

---

## ✨ 올바른 접근

```
After — 혼잡 제어를 알고 나면:

초기 cwnd 최적화:
  # 서버에서 initcwnd 확인
  ip route show default
  # default via 192.168.1.1 dev eth0 proto dhcp initcwnd 10
  
  # initcwnd 증가 (10 → 30)
  ip route change default via 192.168.1.1 initcwnd 30
  # → 연결 초기 전송량 증가 (30 × 1460 = 43.8KB)
  # → 소용량 HTTP 응답에서 Slow Start 영향 감소

혼잡 알고리즘 전환:
  # 현재 알고리즘 확인
  sysctl net.ipv4.tcp_congestion_control
  
  # BBR로 전환 (고지연/고손실 환경에서 유리)
  sysctl -w net.ipv4.tcp_congestion_control=bbr
  
  # 적용 가능한 알고리즘 목록
  sysctl net.ipv4.tcp_available_congestion_control

cwnd 상태 모니터링:
  ss -ti 'dst target-ip'
  # cwnd:10 → 정상 Congestion Avoidance
  # cwnd:1  → Timeout 발생 후 리셋됨 (패킷 손실)
  # cwnd:46 → 정상 대용량 전송 중
```

---

## 🔬 내부 동작 원리

### 1. 혼잡 제어의 목적 — 네트워크를 보호하라

```
흐름 제어 vs 혼잡 제어:

  흐름 제어(Flow Control):
    목적: 수신자 버퍼 보호
    기준: rwnd (수신자가 광고)
    제어자: 수신자

  혼잡 제어(Congestion Control):
    목적: 네트워크(라우터, 링크) 보호
    기준: cwnd (송신자가 자체 추정)
    제어자: 송신자

  실제 전송 속도 = min(rwnd, cwnd)
  
  rwnd=1MB, cwnd=64KB → 64KB까지만 전송
  rwnd=64KB, cwnd=1MB → 64KB까지만 전송

혼잡이 없으면 어떻게 되는가:
  모든 호스트가 전선 속도로 전송
  → 라우터 버퍼 가득 참
  → 패킷 드롭
  → 모두 재전송 → 더 많은 패킷
  → "혼잡 붕괴(Congestion Collapse)"
  → 네트워크 처리량이 0에 수렴

TCP 혼잡 제어의 철학:
  "패킷 손실 = 네트워크 혼잡의 신호"
  → 손실 감지 시 전송 속도 줄임
  → 손실 없으면 점진적으로 늘림
  → 공정성(Fairness): 각 TCP 연결이 공평하게 대역폭 나눔
```

### 2. 4단계 상태 머신

```
TCP 혼잡 제어 4단계:

상태 1: Slow Start (SS)
  초기 cwnd = IW (Initial Window, 보통 10 MSS)
  매 RTT마다: cwnd += 1 per ACK received
            = 매 RTT마다 cwnd 두 배 증가 (지수 증가)
  
  구체적으로:
    cwnd=10: 10개 세그먼트 전송 → 10개 ACK → cwnd=20
    cwnd=20: 20개 전송 → 20개 ACK → cwnd=40
    cwnd=40: → cwnd=80 ...
  
  종료 조건:
    cwnd >= ssthresh → Congestion Avoidance로 전환
    패킷 손실 감지 → 손실 처리

상태 2: Congestion Avoidance (CA)
  cwnd < ssthresh에서 증가 종료 후
  매 RTT마다: cwnd += MSS × MSS / cwnd (≈ +1 MSS per RTT)
  → 선형 증가 (1 RTT마다 1 MSS씩)
  
  왜 선형인가:
    지수 증가는 네트워크에 너무 공격적
    ssthresh = 이전에 혼잡이 발생한 지점의 절반
    → 그 근처에서는 조심스럽게 탐색

상태 3: Fast Recovery (FR)
  Duplicate ACK 3번 감지 시 진입
  cwnd = ssthresh + 3 × MSS
  ssthresh = cwnd / 2
  → 중간 수준에서 빠르게 재개

상태 4: Recovery after Timeout
  RTO 타임아웃 발생 시
  cwnd = 1 MSS (완전 리셋!)
  ssthresh = cwnd / 2
  → Slow Start 처음부터 재시작
  → 가장 심각한 상태
```

### 3. Slow Start 상세

```
초기 연결 시 cwnd 증가 과정 (IW=10, MSS=1460B):

RTT 1:
  cwnd = 10 MSS (14.6KB 전송 가능)
  10개 ACK 수신 → cwnd = 20 MSS

RTT 2:
  cwnd = 20 MSS (29.2KB)
  20개 ACK → cwnd = 40 MSS

RTT 3:
  cwnd = 40 MSS (58.4KB)
  → cwnd 80, 160, 320...

ssthresh 도달 (가령 ssthresh=200 MSS):
  RTT N: cwnd 200 → Congestion Avoidance 진입
  이후: 매 RTT마다 +1 MSS

시간축으로:
  t=0ms:    cwnd=10 (14.6KB)
  t=50ms:   cwnd=20 (29.2KB)   [RTT=50ms 가정]
  t=100ms:  cwnd=40 (58.4KB)
  t=150ms:  cwnd=80 (116.8KB)
  t=200ms:  cwnd=160 (233.6KB)
  t=250ms:  cwnd=200 → CA (선형 증가 시작)
  t=300ms:  cwnd=201
  t=350ms:  cwnd=202
  ...

작은 HTTP 응답 (64KB):
  64KB / 1460B ≈ 44 MSS
  cwnd=44에서 전송 완료 → Slow Start 완료 전에 연결 종료
  → 다음 연결에서 다시 Slow Start부터
  → HTTP/2 keep-alive의 가치
```

### 4. Fast Recovery vs Timeout Recovery

```
패킷 손실 감지 방법 2가지:

방법 1: Duplicate ACK × 3 (Mild signal)
  → Fast Retransmit + Fast Recovery
  cwnd 변화:
    감지 전: cwnd = 100 MSS
    ssthresh = 100 / 2 = 50 MSS
    cwnd = ssthresh + 3 = 53 MSS  ← 50으로 줄지만 1까지 내려가지 않음
    재전송 후 각 Dup ACK마다: cwnd += 1 MSS (인플라이트 유지)
    정상 ACK 복귀 → Congestion Avoidance (cwnd=ssthresh=50)

방법 2: RTO 타임아웃 (Severe signal)
  → Slow Start 재시작
  cwnd 변화:
    감지 전: cwnd = 100 MSS
    ssthresh = 100 / 2 = 50 MSS
    cwnd = 1 MSS  ← 완전 리셋!
    Slow Start 재시작: 1→2→4→8→...→50 (ssthresh)
    ssthresh 도달 후 Congestion Avoidance
  
  처리량 회복 시간 비교:
    Fast Recovery: 1~2 RTT 만에 회복
    Timeout Recovery: Slow Start로 인해 수십 RTT 필요

TCP CUBIC (Linux 기본):
  패킷 손실 시: cwnd = cwnd × 0.7 (30% 감소, AIMD의 50% 감소보다 덜함)
  회복: 3차 함수(Cubic) 기반으로 급격히 증가 후 안정화
  → 손실 직후 빠르게 복구 → 고대역폭-고지연 환경에서 유리

TCP BBR (Bottleneck Bandwidth and RTT):
  손실 기반이 아닌 대역폭+RTT 측정 기반
  패킷 손실 = 혼잡이 아닐 수 있음 (무선 환경의 오류 손실)
  → 대역폭과 RTT를 직접 측정해 최적 전송 속도 계산
  → 불필요한 버퍼링(Bufferbloat) 감소
```

### 5. AIMD — 공정성의 수학

```
AIMD (Additive Increase, Multiplicative Decrease):

  Additive Increase: 혼잡 없을 때 +1 MSS (선형 증가)
  Multiplicative Decrease: 혼잡 감지 시 ×0.5 (절반으로 감소)

왜 이 조합인가:
  AI (Additive): 공정성 보장
    두 연결이 각자 +1씩 증가 → 결국 같은 속도로 수렴
  MD (Multiplicative): 빠른 대역폭 양보
    혼잡 시 큰 연결이 더 많이 양보 → 빠른 안정화

공정성 증명:
  연결 A: cwnd=100, 연결 B: cwnd=50
  둘 다 AI: +1 → A=101, B=51 (차이 유지)
  혼잡 → MD: A=50.5, B=25.5 (차이 절반으로 줄어듦)
  반복 → 점점 같아짐

  그래프 (cwnd_A vs cwnd_B):
    fairness line: cwnd_A = cwnd_B
    AI: 45도 방향으로 이동 (둘 다 증가)
    MD: 원점 방향으로 이동 (절반 감소)
    → 지그재그로 fairness line 수렴

BBR과 공정성:
  BBR은 AIMD를 사용하지 않음
  → CUBIC과 BBR이 같은 링크를 쓰면 불공정
  → BBR이 더 공격적 → CUBIC 연결이 손해
  → 동일 알고리즘 사용 권장
```

---

## 💻 실전 실험

### 실험 1: cwnd 변화 실시간 관찰

```bash
# 대용량 파일 다운로드 중 cwnd 추적
wget http://speedtest.server.com/100MB -O /dev/null &

# 0.5초마다 cwnd 출력
while true; do
  ss -ti 'dst speedtest.server.com' 2>/dev/null | \
    grep -oP 'cwnd:\d+|ssthresh:\d+|rtt:\S+' | tr '\n' ' '
  echo ""
  sleep 0.5
done

# 예상 출력:
# cwnd:10 ssthresh:2147483647 rtt:50/25     ← Slow Start 초기
# cwnd:20 ssthresh:2147483647 rtt:50/25     ← 두 배 증가
# cwnd:40 ssthresh:2147483647 rtt:50/25
# cwnd:80 ssthresh:2147483647 rtt:50/25
# cwnd:100 ssthresh:100 rtt:50/25           ← ssthresh 도달, CA 진입
# cwnd:101 ssthresh:100 rtt:50/25           ← 선형 증가
# cwnd:102 ssthresh:100 rtt:50/25
```

### 실험 2: 패킷 손실 시 cwnd 변화

```bash
# 인위적 패킷 손실 주입
sudo tc qdisc add dev eth0 root netem loss 2%

# 연결 중 cwnd 모니터링
wget http://target-server/largefile -O /dev/null &

while true; do
  ss -ti 'dst target-server' 2>/dev/null | \
    grep -oP 'cwnd:\d+|retrans:\d+/\d+' | tr '\n' ' '
  echo ""
  sleep 0.2
done

# 예상 출력:
# cwnd:80 retrans:0/0
# cwnd:40 retrans:1/1    ← Fast Recovery: 절반으로 감소
# cwnd:80 retrans:0/1    ← 회복 중
# cwnd:1  retrans:2/3    ← RTO: 1로 리셋!
# cwnd:2  retrans:0/3    ← Slow Start 재시작

# 정리
sudo tc qdisc del dev eth0 root
```

### 실험 3: BBR vs CUBIC 비교

```bash
# 현재 알고리즘 확인
sysctl net.ipv4.tcp_congestion_control

# CUBIC로 테스트
sysctl -w net.ipv4.tcp_congestion_control=cubic
iperf3 -c target-server -t 30 -i 1
# 결과 기록

# BBR로 전환
sysctl -w net.ipv4.tcp_congestion_control=bbr
iperf3 -c target-server -t 30 -i 1
# 결과 비교

# 고지연 환경 시뮬레이션 후 비교
sudo tc qdisc add dev eth0 root netem delay 100ms loss 1%
# BBR이 CUBIC보다 높은 처리량을 보일 것
sudo tc qdisc del dev eth0 root
```

### 실험 4: initcwnd 조정 효과

```bash
# 현재 initcwnd 확인
ip route show default
# default via 192.168.1.1 dev eth0 proto dhcp initcwnd 10

# 작은 응답(64KB)의 전송 시간 측정 (현재 initcwnd)
time curl -s http://target/64kb-file > /dev/null

# initcwnd 증가 (10 → 50)
sudo ip route change default via 192.168.1.1 initcwnd 50

# 같은 파일 재측정
time curl -s http://target/64kb-file > /dev/null
# → 초기 전송이 빨라짐 (Slow Start 구간 축소)

# 원복
sudo ip route change default via 192.168.1.1 initcwnd 10
```

---

## 📊 성능/비용 비교

```
혼잡 제어 알고리즘 비교:

┌─────────────────────────────────────────────────────────────────────┐
│  알고리즘   │  손실 기반    │  대역폭 탐색     │  고지연 성능    │ 공정성       │
├─────────────────────────────────────────────────────────────────────┤
│  CUBIC    │  예         │  3차 함수       │  보통         │  좋음       │
│  BBR      │  아니오      │  BDP 측정       │  우수         │  보통       │
│  RENO     │  예         │  선형           │  낮음         │  좋음       │
│  Vegas    │  아니오      │  RTT 기반       │  우수         │  낮음       │
└─────────────────────────────────────────────────────────────────────┘

환경별 추천:
  LAN/데이터센터 (저지연):  CUBIC (안정적, 공정)
  고지연 WAN (>50ms RTT):   BBR (처리량 우수)
  무선 환경 (손실≠혼잡):   BBR (손실 무관)
  혼재 환경:               CUBIC (공정성 우선)

Slow Start 영향:
  응답 크기 64KB, RTT=50ms, IW=10:
    RTT 1: 14.6KB 전송
    RTT 2: 29.2KB 전송 → 총 43.8KB
    RTT 3: 58.4KB → 64KB 완료 (3 RTT = 150ms)
  
  IW=50으로 증가 시:
    RTT 1: 73KB → 64KB 완료 (1 RTT = 50ms)
    → 100ms 단축
```

---

## ⚖️ 트레이드오프

```
혼잡 제어 설계의 트레이드오프:

공격적 vs 보수적:
  공격적(BBR, 큰 IW):
    장점: 대역폭 빠르게 활용, 짧은 응답에 유리
    단점: 혼잡 기여 가능, 다른 연결에 불공정

  보수적(Reno, 작은 IW):
    장점: 공정성, 네트워크 안정성
    단점: 대역폭 활용 느림

손실 기반 vs 모델 기반:
  손실 기반(CUBIC, Reno):
    장점: 구현 단순, 공정성 검증됨
    단점: 무선 환경에서 오탐 (손실=혼잡 아닐 수 있음)

  모델 기반(BBR, Vegas):
    장점: 무선 환경, 고지연 환경에 강함
    단점: 복잡한 구현, 혼재 환경에서 불공정

레거시 연결과의 공정성:
  인터넷은 혼합 환경
  BBR 연결이 CUBIC 연결을 압도할 수 있음
  → CDN, 클라우드가 BBR을 채택하면 개인 사용자 영향
  → 표준화 필요 (IETF에서 논의 중)
```

---

## 📌 핵심 정리

```
TCP 혼잡 제어 핵심 요약:

cwnd(Congestion Window):
  송신자가 자체적으로 관리하는 전송 한도
  실제 전송 = min(rwnd, cwnd)

4단계:
  Slow Start:          cwnd 지수 증가 (1→2→4→8...)
  Congestion Avoidance: cwnd 선형 증가 (+1/RTT)
  Fast Recovery:       Dup ACK 3개 시, cwnd/2로 줄이고 빠른 재개
  Timeout Recovery:    RTO 시, cwnd=1로 리셋, Slow Start 재시작

ssthresh:
  Slow Start와 CA의 경계
  패킷 손실 시: ssthresh = cwnd / 2

Fast Recovery vs Timeout:
  Fast Recovery: cwnd = cwnd/2 → 빠른 회복
  Timeout:       cwnd = 1 → Slow Start 재시작 → 느린 회복

알고리즘:
  CUBIC (Linux 기본): 3차 함수 기반, 공정성 좋음
  BBR: 대역폭+RTT 측정 기반, 고지연/무선에 강함

진단 명령어:
  ss -ti 'dst X': cwnd, ssthresh, rtt, retrans 확인
  cwnd=1 + retrans 증가: RTO 발생 (네트워크 불안정)
  sysctl net.ipv4.tcp_congestion_control: 현재 알고리즘
```

---

## 🤔 생각해볼 문제

**Q1.** HTTP/1.1과 HTTP/2에서 같은 웹 페이지를 로딩할 때, 혼잡 제어 관점에서 어떤 차이가 있는가?

<details>
<summary>해설 보기</summary>

**HTTP/1.1:**
- 리소스별로 새 TCP 연결 (또는 병렬 연결 6개)
- 각 연결마다 Slow Start 시작 → cwnd=10부터
- 6개 연결 합산 cwnd = 60 MSS
- 각 연결이 독립적으로 혼잡 탐지 → 일부만 손실돼도 해당 연결 cwnd 급감

**HTTP/2:**
- 하나의 TCP 연결에서 다중 스트림 (멀티플렉싱)
- TCP 연결이 오래 유지 → cwnd가 충분히 커진 상태 유지
- 초기 이후 요청은 Slow Start 없이 바로 큰 cwnd 활용
- 단, TCP 수준의 Head-of-Line Blocking: 패킷 1개 손실 → 전체 스트림 블로킹

**HTTP/3 (QUIC):**
- UDP 기반, 스트림별 독립 재전송
- 초기 연결에서 cwnd 시작은 동일 (0-RTT 제외)
- 패킷 손실이 특정 스트림만 영향 → HOL Blocking 없음
- 단, UDP이므로 일부 네트워크 장비에서 차단 가능

**결론:** 짧은 수명 연결이 많은 HTTP/1.1보다 HTTP/2가 Slow Start 영향을 덜 받습니다. 고손실 환경에서는 HTTP/3이 가장 유리합니다.

</details>

---

**Q2.** `ss -ti`에서 특정 연결의 `cwnd:1, ssthresh:10, retrans:3/15`를 발견했다. 이 연결에 무슨 일이 일어나고 있는가?

<details>
<summary>해설 보기</summary>

이 수치들의 의미:
- `cwnd:1`: RTO 타임아웃이 발생해 혼잡 창이 1 MSS로 리셋됨
- `ssthresh:10`: 이전에 cwnd가 약 20 MSS였고 손실로 절반(10)으로 줄어든 상태
- `retrans:3/15`: 현재 3개 재전송 대기 중, 누적 15번의 재전송 발생

**판단:** 이 연결은 반복적인 패킷 손실을 겪고 있습니다. cwnd=1에서 Slow Start를 재시작 중이며, 매우 낮은 처리량 상태입니다.

**원인 가능성:**
1. 네트워크 혼잡 (중간 라우터 버퍼 고갈)
2. 케이블/NIC 오류 (물리적 손실)
3. 방화벽/미들박스가 패킷을 산발적으로 드롭
4. 수신자 측 처리 지연 (rwnd=0 이후 타임아웃)

**추가 진단:**
```bash
# 네트워크 경로 품질
mtr --report target-ip
# Loss% 컬럼 확인

# NIC 오류
ethtool -S eth0 | grep -i error

# 재전송 패킷 캡처
sudo tcpdump -nn -w retrans.pcap 'host target-ip'
# Wireshark: tcp.analysis.retransmission
```

</details>

---

**Q3.** BBR과 CUBIC이 같은 병목 링크를 공유할 때 무슨 일이 생기는가? 왜 CDN 회사들이 BBR 전환을 신중하게 결정해야 하는가?

<details>
<summary>해설 보기</summary>

**BBR vs CUBIC 혼재 시 문제:**

CUBIC은 손실 신호를 받을 때까지 계속 증가하다가 손실 시 cwnd를 절반으로 줄입니다. BBR은 손실과 무관하게 측정된 대역폭으로 전송합니다.

같은 병목 링크에서:
- BBR: 손실이 나도 전송 속도를 크게 줄이지 않음 → 링크 계속 점유
- CUBIC: 손실 감지 → cwnd 절반 → BBR에게 양보 상태가 됨

결과: **BBR 연결이 CUBIC 연결보다 불균형하게 많은 대역폭을 가져감.**

Google 내부 측정(2016): BBR이 같은 링크의 CUBIC 연결 처리량을 40% 감소시키는 경우 발생.

**CDN 관점에서 BBR 전환의 고려사항:**
1. CDN → 사용자: CDN이 BBR 사용, 사용자도 같은 링크에 다른 CUBIC 연결이 있을 수 있음 → 사용자 다른 트래픽에 영향
2. 이미 같은 링크에 BBR이 많다면 CUBIC이 손해
3. 사용자의 ISP 네트워크 특성에 따라 효과가 다름

**실용적 접근:** 측정 후 결정. 대부분의 환경에서 BBR이 유리하지만, 혼재 환경에서는 불공정성 문제를 인지하고 모니터링이 필요합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 흐름 제어](./04-flow-control-sliding-window.md)** | **[홈으로 🏠](../README.md)** | **[다음: TCP vs UDP ➡️](./06-tcp-vs-udp.md)**

</div>
