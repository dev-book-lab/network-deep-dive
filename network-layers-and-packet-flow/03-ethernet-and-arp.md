# Ethernet과 ARP — 같은 네트워크 안에서 통신하는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- IP 주소가 있는데 왜 MAC 주소가 별도로 필요한가?
- ARP(Address Resolution Protocol)는 어떤 과정으로 IP → MAC 매핑을 해결하는가?
- ARP 캐시가 만료되면 어떤 일이 일어나는가?
- Gratuitous ARP는 무엇이고 어디에 쓰이는가?
- ARP Spoofing이 가능한 이유와 방어 방법은?
- Docker 컨테이너가 같은 bridge 네트워크 안에서 통신할 때 ARP가 어떻게 동작하는가?

---

## 🔍 왜 이 개념이 중요한가

### MAC 주소가 없으면 같은 네트워크에서 통신이 불가능하다

```
"IP 주소만 있으면 되는 거 아닌가?"라는 질문:

  시나리오: 같은 스위치에 연결된 두 서버
  서버A (192.168.1.10) → 서버B (192.168.1.20)으로 패킷 전송

  문제:
    스위치는 L2 장비 → IP 주소를 모름
    스위치는 MAC 주소 테이블만 보고 포트를 결정
    
    "192.168.1.20으로 보내세요" → 스위치: "그게 몇 번 포트인지 모름"
    "MAC 주소 aa:bb:cc:dd:ee:ff로 보내세요" → 스위치: "2번 포트!"

  결론:
    IP 주소: "어느 호스트인가?" (L3, 논리 주소, 라우팅에 사용)
    MAC 주소: "지금 이 링크의 어느 장치인가?" (L2, 물리 주소, 전달에 사용)

  IP는 종단간(End-to-End) 불변
  MAC은 링크별로 변경 (라우터를 지날 때마다 교체)

ARP를 모르면 겪는 상황:
  "같은 서버실에 있는 서버끼리 왜 통신이 안 되죠?"
  → 네트워크가 연결된 것 같은데 ping 안 됨
  → ARP 캐시 문제일 수 있음 → arp -n으로 즉시 확인 가능
  
  Kubernetes에서:
  "Pod IP를 바꿨는데 이전 IP로 요청이 여전히 가요"
  → ARP 캐시에 이전 매핑이 남아 있음
  → Gratuitous ARP로 캐시를 무효화해야 함
```

---

## 🔬 내부 동작 원리

### 1. Ethernet 프레임 구조

```
Ethernet II 프레임 구조 (가장 일반적인 형식):

┌──────────────┬──────────────┬────────────┬────────────────┬──────┐
│  DstMAC      │  SrcMAC      │  EtherType │    Payload     │  FCS │
│  (6 bytes)   │  (6 bytes)   │  (2 bytes) │  (46~1500 bytes│(4B)  │
└──────────────┴──────────────┴────────────┴────────────────┴──────┘
총 크기: 64~1518 bytes (Payload 최소 46 bytes)

DstMAC (6 bytes):
  목적지 MAC 주소
  FF:FF:FF:FF:FF:FF = 브로드캐스트 (같은 네트워크의 모든 장치에 전달)
  첫 번째 바이트의 LSB = 0: 유니캐스트
  첫 번째 바이트의 LSB = 1: 멀티캐스트

SrcMAC (6 bytes):
  출발지 MAC 주소
  NIC 제조사가 부여한 고유 값 (OUI 3bytes + 고유번호 3bytes)

EtherType (2 bytes):
  0x0800 = IPv4
  0x0806 = ARP
  0x86DD = IPv6
  0x8100 = VLAN (802.1Q)

FCS (Frame Check Sequence, 4 bytes):
  CRC-32 체크섬 → 프레임 손상 여부 확인
  수신자가 계산해서 FCS와 비교 → 불일치 시 프레임 폐기
  → 오류 감지만, 수정은 불가 (재전송은 TCP가 담당)

Payload 최소 46 bytes 이유:
  Ethernet 최소 프레임 크기: 64 bytes
  헤더(14) + FCS(4) = 18 bytes
  Payload 최소 = 64 - 18 = 46 bytes
  부족하면 Padding(0)으로 채움
```

### 2. MAC 주소의 특성

```
MAC 주소 형식:
  6 bytes = 48 bits
  표기: aa:bb:cc:dd:ee:ff (리눅스) 또는 AA-BB-CC-DD-EE-FF (윈도우)

  OUI (Organizationally Unique Identifier): 앞 3 bytes
    → NIC 제조사 식별
    → 00:50:56 = VMware
    → 08:00:27 = VirtualBox
    → 02:42:xx = Docker (소프트웨어 생성)

  NIC 고유 번호: 뒤 3 bytes
    → 제조사가 각 NIC마다 고유하게 부여

  특수 MAC 주소:
    FF:FF:FF:FF:FF:FF = 브로드캐스트
    01:xx:xx:xx:xx:xx = 멀티캐스트
    00:00:00:00:00:00 = 무효 (사용 안 함)

가상 환경에서의 MAC:
  VMware/VirtualBox: OUI로 가상 NIC임을 알 수 있음
  Docker 컨테이너: 02:42:로 시작하는 소프트웨어 생성 MAC
  Kubernetes: veth 페어의 MAC도 소프트웨어 할당

MAC 주소는 변경 가능:
  ip link set eth0 address 02:42:ac:11:00:02
  → MAC 스푸핑 가능 → 보안 취약점의 원인
  → ARP Spoofing도 이를 이용
```

### 3. ARP 동작 원리 — 4단계

```
시나리오: 서버A(192.168.1.10)가 서버B(192.168.1.20)에게 패킷 전송

사전 조건: 서버A의 ARP 캐시에 192.168.1.20 항목 없음

Step 1: ARP Request (브로드캐스트)
  서버A → 스위치 → 모든 포트로 전달

  Ethernet 프레임:
  ┌─────────────────────┬─────────────────────┬──────────┐
  │ DstMAC: FF:FF:FF:FF:│ SrcMAC: aa:bb:cc:11 │EtherType │
  │         FF:FF       │                     │  0x0806  │
  └─────────────────────┴─────────────────────┴──────────┘
  ARP 페이로드:
  ┌──────────────────────────────────────────────────────────┐
  │ Operation: Request (1)                                   │
  │ Sender MAC: aa:bb:cc:dd:ee:11  ← 서버A의 MAC               │
  │ Sender IP:  192.168.1.10       ← 서버A의 IP                │
  │ Target MAC: 00:00:00:00:00:00  ← 모름 (0으로 채움)          │
  │ Target IP:  192.168.1.20       ← 찾고 싶은 IP              │
  └──────────────────────────────────────────────────────────┘
  
  의미: "192.168.1.20을 가진 분, MAC 주소 알려주세요"

Step 2: 다른 호스트들의 반응
  같은 스위치의 모든 호스트가 ARP Request를 받음
  192.168.1.20이 아닌 호스트들: 무시
  192.168.1.20인 서버B: ARP Reply 전송

Step 3: ARP Reply (유니캐스트)
  서버B → 서버A (직접)

  Ethernet 프레임:
  ┌─────────────────────┬─────────────────────┬──────────┐
  │ DstMAC: aa:bb:cc:11 │ SrcMAC: aa:bb:cc:22 │EtherType │
  │                     │                     │  0x0806  │
  └─────────────────────┴─────────────────────┴──────────┘
  ARP 페이로드:
  ┌──────────────────────────────────────────────────────────┐
  │ Operation: Reply (2)                                     │
  │ Sender MAC: aa:bb:cc:dd:ee:22  ← 서버B의 MAC (답!)         │
  │ Sender IP:  192.168.1.20       ← 서버B의 IP                │
  │ Target MAC: aa:bb:cc:dd:ee:11  ← 서버A의 MAC               │
  │ Target IP:  192.168.1.10       ← 서버A의 IP                │
  └──────────────────────────────────────────────────────────┘
  
  의미: "저요! 192.168.1.20은 제 MAC aa:bb:cc:dd:ee:22입니다"

Step 4: ARP 캐시 업데이트 + 원래 통신 시작
  서버A의 ARP 캐시:
    192.168.1.20 → aa:bb:cc:dd:ee:22 (TTL: 300초 기본)
  
  이제 Ethernet 프레임에 올바른 DstMAC을 설정해서 전송 가능
```

### 4. ARP 캐시와 만료

```
ARP 캐시 (ARP Cache / ARP Table):
  각 호스트가 관리하는 IP→MAC 매핑 테이블
  반복되는 ARP 브로드캐스트를 방지하기 위한 캐시

Linux ARP 캐시 항목:
  arp -n 출력:
  Address         HWtype  HWaddress           Flags Mask  Iface
  192.168.1.254   ether   aa:bb:cc:dd:ee:ff   C           eth0
  192.168.1.20    ether   aa:bb:cc:11:22:33   C           eth0
  
  C = Complete (통신 완료, 유효한 항목)
  I = Incomplete (ARP Request 보냈으나 Reply 미수신)

캐시 만료 타이머 (Linux):
  gc_stale_time: 60초 (유효하지만 재검증 필요)
  gc_interval:   30초 (가비지 컬렉션 주기)
  
  만료 과정:
    항목 생성 → 사용 중 갱신 → 사용 안 하면 STALE → 재검증 → 삭제
    
  STALE 상태:
    만료된 항목이지만 아직 삭제되진 않음
    다음 통신 시 ARP Request를 다시 보내서 재검증
    재검증 실패 시 FAILED → 삭제

직접 확인:
  ip neigh show  (neigh = neighbor, ARP 캐시의 현대적 표현)
  출력 예시:
  192.168.1.254 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
  192.168.1.20  dev eth0 lladdr aa:bb:cc:11:22:33 STALE
  
  상태: REACHABLE (최근 사용) → STALE (오래됨) → DELAY → PROBE → FAILED
```

### 5. Gratuitous ARP — 캐시 선제 업데이트

```
Gratuitous ARP (GARP):
  자기 자신의 IP에 대한 ARP Request/Reply를 스스로 브로드캐스트
  
  특징:
    Sender IP = Target IP = 자신의 IP
    Sender MAC = 자신의 MAC
    네트워크의 모든 호스트가 자신의 ARP 캐시를 즉시 업데이트

  사용 사례 1: IP 중복 감지
    호스트가 부팅 시 Gratuitous ARP 전송
    Reply가 돌아오면 같은 IP를 가진 다른 호스트 존재 → 충돌 경고

  사용 사례 2: VM/컨테이너 이동 후 캐시 무효화
    VM이 다른 호스트로 마이그레이션 → MAC 주소가 바뀜
    Gratuitous ARP → 네트워크의 모든 장비가 새 MAC으로 캐시 업데이트
    
    Kubernetes에서:
    Pod가 재시작되면 IP는 같지만 MAC이 새 veth 인터페이스로 바뀜
    → Gratuitous ARP로 네트워크 장비들이 새 MAC을 학습

  사용 사례 3: High Availability Failover
    Active 서버 장애 → Standby 서버가 가상 IP를 인수
    Standby 서버가 Gratuitous ARP → 모든 클라이언트가 새 MAC으로 연결
    → 애플리케이션 레벨 재연결 없이 투명한 Failover

  Gratuitous ARP 전송:
    arping -U -I eth0 192.168.1.10  # -U = Gratuitous ARP (Unsolicited)
```

### 6. ARP Spoofing — 취약점과 방어

```
ARP Spoofing (ARP Poisoning):
  ARP는 인증이 없음 → 누구나 위조된 ARP Reply를 보낼 수 있음

공격 원리:
  네트워크: 피해자(192.168.1.10), 게이트웨이(192.168.1.1), 공격자(192.168.1.99)
  
  공격자가 피해자에게 위조 ARP Reply 전송:
    "192.168.1.1(게이트웨이)의 MAC은 aa:bb:cc:99:99:99(공격자)입니다"
  
  공격자가 게이트웨이에게 위조 ARP Reply 전송:
    "192.168.1.10(피해자)의 MAC은 aa:bb:cc:99:99:99(공격자)입니다"
  
  결과:
    피해자 → 게이트웨이로 보내는 패킷 → 공격자 경유
    게이트웨이 → 피해자로 보내는 패킷 → 공격자 경유
    → Man-in-the-Middle 완성 (트래픽 도청/변조 가능)

방어 방법:
  1. Dynamic ARP Inspection (DAI):
     스위치 레벨에서 ARP Reply의 IP-MAC 매핑을 DHCP Snooping 테이블과 비교
     불일치 시 차단
  
  2. Static ARP 항목:
     arp -s 192.168.1.1 aa:bb:cc:dd:ee:ff  (수동 고정)
     변경되지 않지만 유지 관리 부담
  
  3. ARP Spoofing 탐지:
     arpwatch: ARP 트래픽 모니터링, MAC 변경 감지 시 알림
  
  4. 암호화 (TLS):
     패킷이 도청되더라도 내용을 알 수 없음
     → HTTPS를 쓰면 ARP Spoofing으로 내용 탈취는 방지
     → 단, 연결 자체를 차단하는 것은 막기 어려움

현실적 위협:
  같은 스위치(L2 도메인) 안에서만 가능
  → 클라우드 환경에서는 각 가상 포트가 격리되어 있어 어려움
  → 사무실, 카페 등 공유 네트워크에서 위험
```

---

## 😱 흔한 실수

```
Before — ARP를 모를 때:

실수 1: "같은 IP로 서버를 바꿨는데 이전 서버로 계속 요청이 가요"
  → ARP 캐시에 이전 서버의 MAC이 남아 있음
  → 해결: 새 서버에서 Gratuitous ARP 전송 (또는 캐시 만료 대기)
  → arping -U -I eth0 192.168.1.10

실수 2: Docker 컨테이너 통신 안 됨
  → 같은 bridge 네트워크인데 ping 안 됨
  → ARP 캐시 확인: docker exec container1 arp -n
  → INCOMPLETE 항목 → ARP Request에 Reply 안 옴
  → 원인: iptables FORWARD 규칙이 ARP를 차단

실수 3: VM 마이그레이션 후 간헐적 연결 실패
  → VM이 다른 호스트로 이동 → MAC 변경
  → 네트워크 장비의 ARP 캐시가 만료되기 전까지 패킷이 구 MAC으로 전달
  → 해결: 마이그레이션 완료 후 Gratuitous ARP 자동 전송 설정
```

---

## ✨ 올바른 접근

```
After — ARP를 알고 나면:

ARP 캐시 확인:
  ip neigh show
  → 통신 안 되는 상대방 IP의 상태가 FAILED이면
    → ARP Request가 응답받지 못함
    → 상대방 서버가 다운됐거나 네트워크 격리 확인

ARP 캐시 강제 삭제:
  ip neigh del 192.168.1.20 dev eth0
  → 다음 통신 시 ARP를 다시 수행

ARP 캐시 갱신 강제:
  arping -I eth0 192.168.1.20
  → ARP Request를 직접 전송해서 응답 확인

Gratuitous ARP 전송 (IP 변경 후):
  arping -U -c 3 -I eth0 192.168.1.10
  → -c 3: 3번 전송 (일부 장비가 첫 번째를 놓칠 수 있어 반복)
```

---

## 💻 실전 실험

### 실험 1: ARP 동작 직접 관찰

```bash
# ARP 캐시 현재 상태 확인
ip neigh show
arp -n  # 구식이지만 여전히 사용됨

# ARP 캐시 완전히 비우기 (특정 항목)
sudo ip neigh del 192.168.1.1 dev eth0

# ARP 트래픽 캡처 (다른 터미널에서 실행)
sudo tcpdump -i eth0 -nn arp &

# ping을 통해 ARP 발생시키기
ping -c 1 192.168.1.1

# tcpdump 출력 분석:
# ARP Request:
# 14:00:01 ARP, Request who-has 192.168.1.1 tell 192.168.1.100, length 28
# → DstMAC = FF:FF:FF:FF:FF:FF (브로드캐스트)

# ARP Reply:
# 14:00:01 ARP, Reply 192.168.1.1 is-at aa:bb:cc:dd:ee:ff, length 28
# → DstMAC = 요청한 호스트의 MAC (유니캐스트)

# 캐시에 추가됐는지 확인
ip neigh show | grep 192.168.1.1
# 출력: 192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

### 실험 2: Ethernet 프레임 헤더 분석

```bash
# Ethernet 헤더 포함해서 캡처 (-e 옵션)
sudo tcpdump -i eth0 -nn -e 'host 192.168.1.1' &
ping -c 1 192.168.1.1

# 출력 예시:
# 14:00:01 aa:bb:cc:11:22:33 > aa:bb:cc:dd:ee:ff, ethertype IPv4 (0x0800)
#           ↑ SrcMAC              ↑ DstMAC             ↑ EtherType
#
# ARP 패킷:
# 14:00:00 aa:bb:cc:11:22:33 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806)
#                               ↑ 브로드캐스트!

# 내 MAC 주소 확인
ip link show eth0 | grep ether
# 출력: link/ether aa:bb:cc:11:22:33 brd ff:ff:ff:ff:ff:ff
```

### 실험 3: Docker 네트워크에서 ARP 관찰

```bash
# Docker bridge 네트워크 생성 및 컨테이너 실행
docker network create --subnet=172.20.0.0/16 test-net

docker run -d --name c1 --network test-net --ip 172.20.0.10 \
  alpine sleep infinity

docker run -d --name c2 --network test-net --ip 172.20.0.11 \
  alpine sleep infinity

# c1의 ARP 캐시 확인 (처음에는 비어있음)
docker exec c1 arp -n

# c1에서 c2로 ping
docker exec c1 ping -c 1 172.20.0.11

# ARP 캐시 업데이트 확인
docker exec c1 arp -n
# 출력: 172.20.0.11  ether  02:42:ac:14:00:0b  C  eth0

# Docker bridge의 MAC 주소 확인
docker exec c1 ip link show eth0
# 출력: link/ether 02:42:ac:14:00:0a brd ff:ff:ff:ff:ff:ff
# 02:42는 Docker가 생성하는 MAC의 OUI

# 정리
docker rm -f c1 c2
docker network rm test-net
```

### 실험 4: ARP 캐시 타이머 관찰

```bash
# ARP 캐시 타이머 설정 확인
cat /proc/sys/net/ipv4/neigh/eth0/gc_stale_time
# 출력: 60 (초)

# 현재 캐시 상태와 상태 전환 관찰
watch -n 1 "ip neigh show"
# ping을 중단한 후:
# REACHABLE → (일정 시간 후) STALE → (재검증 후) REACHABLE 또는 FAILED

# 강제로 캐시 재검증
ip neigh change 192.168.1.1 dev eth0 nud stale
# → 다음 통신 시 ARP 재수행
```

---

## 📊 ARP 관련 성능 고려사항

```
ARP 브로드캐스트 부하:

소규모 네트워크 (/24, 254 호스트):
  영향 미미 → 각 호스트가 ARP 브로드캐스트를 받아 처리
  CPU 부하: 무시할 수준

대규모 L2 도메인 (수천 호스트):
  ARP 브로드캐스트 폭풍(ARP Storm) 가능
  모든 호스트가 모든 ARP Request를 받아야 함
  → L2 도메인을 작게 분할하는 이유
  → VLAN으로 브로드캐스트 도메인 분리

클라우드/데이터센터:
  AWS VPC, Azure VNet: 소프트웨어 정의 네트워크
  실제 ARP 브로드캐스트 대신 중앙 집중식 MAC 테이블 관리
  → ARP Proxy가 응답을 대신 처리

ARP 캐시 성능:
  ARP 캐시 HIT: 즉시 전송 가능
  ARP 캐시 MISS: ARP Request + Reply 대기 → 첫 패킷 지연
  
  처음 연결 시 지연:
    ARP RTT: 1~2 ms (같은 서브넷)
    → 연결 지속 시 캐시로 이 지연 제거
    → 새 서버와 처음 통신 시 1~2 ms 추가 지연
```

---

## ⚖️ 트레이드오프

```
ARP 설계의 한계와 대안:

한계:
  ① 인증 없음:
     누구나 ARP Reply를 위조 가능 → ARP Spoofing
  
  ② 브로드캐스트 의존:
     L2 도메인 전체에 브로드캐스트 → 확장성 제한
  
  ③ IPv4 전용:
     IPv6는 NDP(Neighbor Discovery Protocol)로 대체
     멀티캐스트 기반 → 브로드캐스트 폭풍 없음

IPv6의 개선:
  ARP → NDP(Neighbor Discovery Protocol)
  브로드캐스트 → 멀티캐스트 (해당 호스트만 수신)
  Link-Local 주소 자동 생성 (DHCPv6 없이도 통신 가능)

소프트웨어 정의 네트워크(SDN):
  ARP를 중앙 컨트롤러가 처리
  → 브로드캐스트 없이 즉시 응답
  → Kubernetes CNI, AWS VPC가 이 방식 사용
```

---

## 📌 핵심 정리

```
Ethernet과 ARP 핵심 요약:

왜 MAC 주소가 필요한가:
  IP: 최종 목적지 식별 (종단간, 라우터 통과해도 불변)
  MAC: 현재 링크에서 어느 NIC인지 식별 (링크마다 변경)
  스위치: IP를 모름 → MAC으로 포트 결정

ARP 4단계:
  1. ARP Request: 브로드캐스트 (FF:FF:FF:FF:FF:FF)
     "192.168.1.20 가진 분 MAC 알려주세요"
  2. 해당 호스트가 ARP Reply: 유니캐스트
     "저요, MAC은 aa:bb:cc:22:22:22"
  3. ARP 캐시에 저장 (60~300초)
  4. 이후 통신 시 ARP 없이 바로 전송

Gratuitous ARP:
  자기 IP에 대한 ARP를 스스로 브로드캐스트
  용도: IP 충돌 감지, MAC 변경 알림 (VM 이동, Failover)

ARP Spoofing:
  인증 없는 ARP → 위조 Reply → Man-in-the-Middle
  방어: DAI (Dynamic ARP Inspection), TLS 암호화

진단 명령어:
  ip neigh show       → ARP 캐시 확인 (REACHABLE/STALE/FAILED)
  arp -n              → ARP 캐시 (구식)
  arping <IP>         → ARP Request 직접 전송
  arping -U -I eth0 <IP> → Gratuitous ARP 전송
  tcpdump arp         → ARP 트래픽 캡처
```

---

## 🤔 생각해볼 문제

**Q1.** 서버가 재시작되지 않았는데 "IP는 같은데 MAC이 바뀐" 것처럼 동작하는 경우가 있는가? 어떤 상황인가?

<details>
<summary>해설 보기</summary>

예, 이런 경우가 실제로 발생합니다.

**Kubernetes Pod 재시작**: Pod가 재시작되면 같은 IP를 받더라도 새로운 veth(가상 ethernet) 인터페이스가 생성되어 새 MAC 주소가 할당됩니다. 클러스터 내 다른 노드의 ARP 캐시에 이전 MAC이 남아 있으면 일시적으로 통신 실패가 발생합니다. 이를 해결하기 위해 Kubernetes CNI 플러그인이 Pod 시작 시 Gratuitous ARP를 전송합니다.

**NIC 교체**: 물리 서버의 NIC를 교체하면 MAC이 바뀝니다. 같은 IP를 유지해도 네트워크 장비들의 ARP 캐시가 갱신될 때까지 통신이 불안정합니다.

**가상 MAC 변경**: 일부 환경에서 보안 목적으로 MAC을 주기적으로 변경합니다. `ip link set eth0 address <new_mac>`으로 MAC을 변경하면 ARP 캐시가 stale 상태가 됩니다.

진단: `ip neigh show`에서 FAILED 또는 STALE 상태의 항목 확인 → `arping`으로 재검증.

</details>

---

**Q2.** `ping 192.168.1.20` 이 실패했다. ARP 캐시에 `192.168.1.20 dev eth0 FAILED`가 보인다. 이것이 의미하는 것은 무엇이고, 다음 진단 단계는?

<details>
<summary>해설 보기</summary>

`FAILED` 상태는 ARP Request를 보냈으나 Reply를 받지 못했음을 의미합니다. 즉, **L3(IP) 이전 단계인 L2(Ethernet) 레벨에서 이미 통신이 안 되고 있습니다.**

원인 후보:
1. 대상 호스트가 다운됨 (물리적 전원 꺼짐, OS 다운)
2. 대상 호스트가 다른 서브넷으로 이동함 (IP는 같지만 물리 위치 변경)
3. 스위치 포트가 비활성화됨
4. Firewall에서 ARP 차단 (드문 경우)
5. 잘못된 VLAN 설정

다음 진단 단계:
```bash
# 1. 직접 ARP Request 전송해서 응답 확인
arping -I eth0 192.168.1.20
# → 응답 없음: 호스트 문제 또는 L2 연결 문제

# 2. 스위치 포트 상태 확인 (스위치 접근 가능한 경우)
# 3. 해당 IP의 서버 콘솔로 직접 접근해서 네트워크 상태 확인
# 4. 다른 호스트에서 같은 IP로 ARP 시도 → 동일한 FAILED?
#    → 네트워크 문제 (스위치, 케이블)
#    → 응답 있음? → 출발 서버의 VLAN/라우팅 문제
```

</details>

---

**Q3.** 같은 스위치에 연결된 두 서버가 서로 다른 서브넷이다 (A: 192.168.1.10/24, B: 192.168.2.10/24). A에서 B로 직접 통신이 가능한가?

<details>
<summary>해설 보기</summary>

**직접 통신 불가**입니다. 물리적으로 같은 스위치에 연결되어 있더라도, IP 레이어에서 서로 다른 서브넷으로 인식하기 때문입니다.

A(192.168.1.10)가 B(192.168.2.10)에게 통신하려 할 때:
1. A가 자신의 서브넷 마스크(/24)로 계산: 192.168.2.10은 다른 네트워크
2. 다른 네트워크이므로 라우터(Default Gateway)로 전달
3. 라우터가 192.168.2.0/24 경로를 알면 B로 전달, 모르면 폐기

따라서 같은 L2 스위치에 있더라도 **L3 라우터**가 필요합니다.

예외: Proxy ARP
일부 라우터나 스위치가 Proxy ARP를 지원하면, 라우터가 대신 ARP Reply를 보내고 중간에서 패킷을 전달해줍니다. 이 경우 같은 스위치에 있는 것처럼 통신할 수 있지만, 모든 트래픽이 라우터를 경유하므로 비효율적입니다.

실무 교훈: VLAN과 서브넷 설계 시, 빈번하게 통신하는 서버들은 같은 서브넷에 두어 라우터 경유를 피하는 것이 좋습니다.

</details>

---

<div align="center">

**[⬅️ 이전: IP와 라우팅](./02-ip-and-routing.md)** | **[홈으로 🏠](../README.md)** | **[다음: NAT와 포트 포워딩 ➡️](./04-nat-and-port-forwarding.md)**

</div>
