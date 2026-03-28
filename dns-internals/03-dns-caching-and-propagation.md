# DNS 캐싱과 전파 — TTL이 배포에 미치는 영향

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- TTL이 만료되기 전에 DNS 레코드를 변경하면 어떤 일이 생기는가?
- 서비스 마이그레이션 시 TTL을 언제, 얼마나 낮춰야 하는가?
- Negative Caching(NXDOMAIN)은 어떻게 동작하고 왜 위험한가?
- `dig +norecurse`로 Resolver 캐시 상태를 어떻게 확인하는가?
- DNS 전파 시간이 "최대 48시간"이라는 말의 실제 의미는?
- 블루-그린 배포와 카나리 배포에서 DNS를 어떻게 활용하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"DNS 변경했는데 왜 아직도 이전 서버로 요청이 가나요?":
  클라이언트: TTL=3600이면 최대 1시간 이전 IP 사용
  이게 "DNS 전파"라고 불리는 현상
  
  실제로 일어나는 일:
  1. Authoritative NS: 즉시 변경 (수 초)
  2. 전 세계 Recursive Resolver 캐시: TTL 만료까지 이전 값
  3. 브라우저/OS 캐시: 짧은 시간 (수십 초)
  4. JVM 캐시: 설정에 따라 영구까지
  
  "48시간 전파"라는 말:
  TTL=86400(24시간) 레코드를 두 번 변경하면 최대 48시간
  실제로는 대부분 TTL=3600이면 1~2시간 내 전파 완료

"스타트업 마이그레이션 중 롤백이 안 됐다":
  원인: TTL=86400 상태에서 마이그레이션
  새 서버 문제 발생 → 이전 IP로 롤백
  → 전 세계 캐시가 24시간 동안 새 IP 사용
  → 롤백 불가능!
  → 마이그레이션 24시간 전에 TTL을 낮춰야 함

"배포 중 NXDOMAIN 오류":
  배포 자동화 스크립트가 없는 도메인 조회 → NXDOMAIN
  NXDOMAIN 캐시 = SOA의 Minimum TTL (보통 300~3600초)
  → 도메인 생성 후에도 캐시 만료까지 오류 지속
```

---

## 😱 흔한 실수

```
Before — TTL과 전파를 모를 때:

실수 1: TTL 낮추지 않고 DNS 마이그레이션
  TTL=86400 상태에서 IP 변경
  → 이전 IP로 최대 24시간 트래픽 유입
  → 롤백 불가능한 상황 발생

실수 2: NXDOMAIN 캐시 무시
  도메인 생성 직전에 테스트 → NXDOMAIN
  도메인 생성 후에도 NXDOMAIN 캐시 때문에 접근 안 됨
  → SOA Minimum TTL (보통 300~3600초) 후에야 접근 가능
  → 배포 시 도메인 생성 순서와 서비스 시작 순서 고려 필요

실수 3: TTL 낮춘 후 되돌리는 것을 잊음
  마이그레이션 전: TTL=60으로 낮춤
  마이그레이션 후: TTL=3600으로 복구 안 함
  → Authoritative NS에 불필요한 쿼리 지속
  → DNS 서비스 비용 증가 (Route53 등)

실수 4: "DNS 전파 완료"를 어떻게 확인하는지 모름
  dig 결과가 새 IP → "전파 완료"로 오해
  → 특정 Resolver에서만 전파된 것일 수 있음
  → whatsmydns.net으로 전 세계 Resolver 확인 필요
```

---

## ✨ 올바른 접근

```
After — TTL과 전파를 알면:

서비스 마이그레이션 체크리스트:
  T-72h: 현재 TTL 확인 (dig example.com | grep TTL)
  T-48h: TTL=300(5분)으로 변경 → 반드시 적용 대기
          (기존 TTL만큼 기다려야 전 세계 적용)
  T-0:   DNS 레코드 변경 (IP 교체)
          5분 후 대부분의 Resolver에서 새 IP 조회
  T+1h:  안정화 확인 → 문제 없으면 TTL=3600 복구

전파 확인 방법:
  # 여러 Resolver에서 동시 확인
  for resolver in 8.8.8.8 1.1.1.1 9.9.9.9 208.67.222.222; do
    printf "%-20s" "$resolver:"
    dig @$resolver api.example.com +short
  done
  
  # 글로벌 전파 확인
  # whatsmydns.net (웹) 또는 dnschecker.org

블루-그린 DNS 전략:
  blue.example.com  A  203.0.113.10  (Blue 환경)
  green.example.com A  203.0.113.20  (Green 환경)
  api.example.com   CNAME  blue.example.com  (현재 라이브)
  
  전환:
  api.example.com   CNAME  green.example.com  (Green으로 변경)
  → CNAME은 TTL 짧게 유지 (300초)
  → blue/green A 레코드는 TTL 높게 유지 (3600초)
  → 롤백: CNAME을 다시 blue로 변경 (5분 내 완료)
```

---

## 🔬 내부 동작 원리

### 1. TTL과 캐시 만료

```
TTL(Time To Live) 동작:

Authoritative NS가 A 레코드 반환:
  api.example.com.  300  IN  A  203.0.113.10  ← TTL=300

Recursive Resolver:
  수신 시각: 12:00:00
  캐시 저장: {api.example.com: 203.0.113.10, expires: 12:05:00}
  
  12:00:00 이후 요청: 캐시에서 반환, TTL 감소
  클라이언트 1 (12:01): 응답 TTL = 240 (5분 - 1분 경과)
  클라이언트 2 (12:04): 응답 TTL = 60  (1분 남음)
  
  12:05:00 이후: 캐시 만료 → Authoritative NS 재조회

DNS 레코드 변경 시 전파 과정:

12:00 - Authoritative NS에서 IP 변경:
  api.example.com.  300  IN  A  203.0.113.20  (새 IP)

전 세계 Resolver 상황:
  방금 조회한 Resolver: 12:05까지 이전 IP 캐시 (300초)
  5분 전에 조회한 Resolver: 이미 만료 → 새 IP 조회
  1시간 전에 조회한 Resolver: 이미 만료 → 새 IP 조회

결론:
  모든 Resolver가 새 IP를 반환하는 데 걸리는 시간 = 최대 TTL 초
  TTL=300 → 최대 5분
  TTL=3600 → 최대 1시간
  TTL=86400 → 최대 24시간

"48시간 전파"의 진실:
  오래된 조언: "DNS 전파는 48시간 걸린다"
  현실: TTL=86400이면 최대 24시간, TTL=3600이면 최대 1시간
  "48시간"은 TTL=86400 레코드를 두 번 변경하던 시절의 말
  현재 권장: 평상시 TTL=3600, 변경 전 TTL=300으로 낮춤
```

### 2. Negative Caching — NXDOMAIN 캐시

```
존재하지 않는 도메인 조회 시:
  Authoritative NS → NXDOMAIN (Non-Existent Domain)
  Recursive Resolver: NXDOMAIN도 캐시!

Negative TTL 설정:
  SOA 레코드의 마지막 값 (Minimum TTL)
  
  example.com.  IN  SOA  ns1.example.com.  admin.example.com. (
      2026032801  ; Serial
      3600        ; Refresh
      900         ; Retry
      604800      ; Expire
      300         ; ← 이게 Negative TTL (NXDOMAIN 캐시 시간)
  )
  
  notexist.example.com 조회 → NXDOMAIN
  Resolver: 300초 동안 NXDOMAIN 캐시
  → 300초 내 재조회해도 NXDOMAIN 반환 (실제 존재해도!)

실무 문제:
  배포 시나리오:
  1. 애플리케이션 시작 전 DNS 레코드 접근 시도 (readiness 체크 등)
  2. 도메인이 아직 없음 → NXDOMAIN 캐시됨
  3. 도메인 생성 (수 초 후)
  4. 애플리케이션이 도메인 재조회 → 여전히 NXDOMAIN (캐시!)
  5. Negative TTL 만료까지 기다려야 접근 가능
  
  → 해결: 도메인 먼저 생성, 서비스 나중에 시작
  → 또는 nscd 캐시 플러시, 재시작으로 캐시 강제 초기화

Negative Caching 예:
  K8s에서 Service가 삭제되고 재생성되는 경우:
  삭제 직후 → CoreDNS: NXDOMAIN 캐시 (ndotsN 설정 기반)
  재생성 후 → CoreDNS Negative TTL 만료 전: 아직 NXDOMAIN
  → 재생성 후 수초~수분 내 불안정 구간 존재
```

### 3. 캐시 계층과 각 계층의 TTL 적용

```
DNS 캐시 계층 (각각 독립적):

Layer 1: 브라우저 DNS 캐시
  Chrome: chrome://net-internals/#dns
  최대 1분 (OS 이하로 제한하는 경우도)
  시크릿 모드: 캐시 없음

Layer 2: OS DNS 캐시
  Linux: systemd-resolved (캐시 O) 또는 nscd
  Windows: DNS Client 서비스
  macOS: mDNSResponder
  TTL 그대로 준수 또는 자체 상한 적용

Layer 3: JVM DNS 캐시
  networkaddress.cache.ttl 설정값
  기본 영구 가능 (보안 매니저 활성화 시)

Layer 4: 로컬 Recursive Resolver (캐싱 Resolver)
  기업 내부 DNS 서버, systemd-resolved, dnsmasq
  TTL 그대로 준수

Layer 5: ISP/Public Recursive Resolver
  8.8.8.8, 1.1.1.1 등
  TTL 그대로 준수 (일부 최솟값 적용)
  수백만 사용자 공유 캐시

Layer 6: Authoritative NS
  실제 레코드 보유 (캐시 없음, 즉시 갱신)

DNS 레코드 변경이 모두 반영되려면:
  Layer 1~5 모두에서 TTL 만료 필요
  가장 긴 캐시 = 가장 오래된 Resolver의 남은 TTL
  
  JVM이 영구 캐시라면:
  → 재시작 전까지 DNS 변경 무효
  → 이것이 실무에서 가장 자주 놓치는 문제
```

### 4. 배포 전략에서 DNS 활용

```
블루-그린 배포:

현재:
  api.example.com  A  10.0.0.10  (Blue)

준비:
  Blue: 10.0.0.10 (현재 라이브)
  Green: 10.0.0.20 (새 버전 배포 완료)

전환:
  api.example.com  A  10.0.0.20  (Green으로 변경)
  
  롤백 준비:
  TTL 낮춰놨으면 → 5분 내 Blue로 복구 가능

CNAME 기반 블루-그린 (더 유연):
  api.example.com  CNAME  blue.api.example.com  TTL=60
  blue.api.example.com   A  10.0.0.10           TTL=3600
  green.api.example.com  A  10.0.0.20           TTL=3600
  
  전환: api → blue 대신 green을 가리키도록 CNAME만 변경
  → CNAME은 TTL=60이므로 60초 내 전환
  → A 레코드 TTL=3600이므로 IP는 Resolver에 오래 캐시 (OK, 변경 없음)
  
  롤백: CNAME을 다시 blue로 변경 → 60초 내 롤백 완료

가중치 기반 카나리:
  Route53 Weighted Routing:
  api.example.com  A  10.0.0.10  Weight=90  (기존)
  api.example.com  A  10.0.0.20  Weight=10  (카나리)
  
  Route53이 90:10 비율로 랜덤하게 다른 IP 응답
  단, 클라이언트 캐시로 인해 실제 비율은 불균등할 수 있음
  → TTL을 짧게 (60초) 유지해야 더 균등

헬스 체크 기반 장애 조치:
  Route53 Failover:
  api.example.com  A  10.0.0.10  PRIMARY   (헬스 체크 연결)
  api.example.com  A  10.0.0.20  SECONDARY (Primary 실패 시)
  
  헬스 체크 실패 → Route53이 자동으로 Secondary IP 반환
  Recovery: Primary 복구 → 자동으로 다시 Primary IP 반환
  TTL=60으로 빠른 장애 조치 (60초 내)
```

### 5. DNS 캐시 플러시와 확인 방법

```
각 계층별 캐시 강제 초기화:

Linux (systemd-resolved):
  sudo systemd-resolve --flush-caches
  sudo resolvectl flush-caches
  확인: sudo resolvectl statistics

Linux (nscd):
  sudo service nscd restart
  또는 sudo nscd -i hosts

macOS:
  sudo dscacheutil -flushcache
  sudo killall -HUP mDNSResponder

Windows:
  ipconfig /flushdns

JVM:
  재시작 (프로그래밍적으로 초기화 어려움)
  또는 arthas: ognl "java.net.InetAddress.invalidateCache()"

캐시 상태 확인 (캐시 히트 여부):
  # +norecurse: Resolver에 재귀 요청 안 함 → 캐시 확인
  dig +norecurse @8.8.8.8 api.example.com
  
  응답에 ANSWER 있음: 캐시 히트 (남은 TTL 확인)
  응답에 ANSWER 없음: 캐시 없음 (AUTHORITY에 위임 정보)
  
  # 남은 TTL 확인 (두 번 조회해서 감소 확인)
  dig api.example.com | awk '/IN A/{print $2}'
  sleep 10
  dig api.example.com | awk '/IN A/{print $2}'
  # 10 감소 확인

전파 상태 모니터링:
  # 여러 Resolver 동시 확인
  for resolver in 8.8.8.8 1.1.1.1 9.9.9.9 208.67.222.222 64.6.64.6; do
    echo -n "$resolver: "
    dig @$resolver api.example.com +short 2>/dev/null || echo "ERROR"
  done
  
  # whatsmydns.net API (지역별 확인)
  curl "https://api.whatsmydns.net/v1/check?name=api.example.com&type=A"
```

---

## 💻 실전 실험

### 실험 1: TTL 감소 관찰

```bash
# TTL이 실시간으로 감소하는 것 확인
while true; do
  TTL=$(dig api.example.com +short +ttlunits 2>/dev/null | head -1 | awk '{print $1}')
  echo "$(date +%H:%M:%S) - TTL: ${TTL}초"
  sleep 30
done

# 캐시 히트 vs 미스 구분
# 캐시 히트: 요청마다 TTL 감소
# 캐시 미스: TTL이 다시 원래값으로 (재조회)
```

### 실험 2: Negative Caching 실험

```bash
# 1. 없는 도메인 조회 → NXDOMAIN 발생
dig notexist.example.com @8.8.8.8
# Status: NXDOMAIN
# AUTHORITY SECTION: example.com. SOA ... 300 (Negative TTL)

# 2. SOA Minimum TTL 확인
dig SOA example.com @8.8.8.8 | grep "SOA"
# 마지막 숫자 = Negative Caching TTL

# 3. 도메인 생성 후 NXDOMAIN 캐시 만료 대기 확인
# (실제 도메인이 있는 환경에서)
# 새 서브도메인 조회 → NXDOMAIN
# DNS 레코드 추가 → 즉시 조회 (NXDOMAIN 캐시로 실패)
# SOA Minimum TTL 후 재조회 → 성공
```

### 실험 3: 마이그레이션 시뮬레이션

```bash
# 현재 TTL 확인
dig example.com | grep -E "^example|IN A"
# example.com. 3600 IN A 203.0.113.10
# TTL=3600: 1시간 내 전파 (마이그레이션 전 낮춰야 함)

# TTL 낮춘 후 전파 대기 시뮬레이션
echo "1. TTL 300으로 변경 (Route53 등에서)"
echo "2. 3600초(기존 TTL) 대기"
echo "3. 전 세계 Resolver에서 TTL=300 확인:"
for resolver in 8.8.8.8 1.1.1.1; do
  TTL=$(dig @$resolver example.com | awk '/IN A/{print $2}' | head -1)
  echo "   $resolver: TTL=$TTL"
done
echo "4. TTL이 300이면 IP 변경 가능 (5분 내 전파)"
```

### 실험 4: DNS 캐시 플러시 및 재조회

```bash
# Linux systemd-resolved 캐시 통계
sudo resolvectl statistics
# Current Cache Size: N
# Cache Hits: N
# Cache Misses: N

# 캐시 플러시 전후 비교
time dig api.example.com  # 첫 조회 (느림)
time dig api.example.com  # 캐시 조회 (빠름)

sudo resolvectl flush-caches

time dig api.example.com  # 플러시 후 재조회 (다시 느림)

# K8s CoreDNS 캐시 확인
kubectl logs -n kube-system deployment/coredns | grep "cache"
kubectl exec -n kube-system deployment/coredns -- \
  curl -s localhost:9153/metrics | grep "coredns_cache"
```

---

## 📊 성능/비용 비교

```
TTL 설정별 트레이드오프:

┌──────────────────────────────────────────────────────────────────┐
│  TTL     │  변경 전파 시간   │  DNS 쿼리 수  │  롤백 속도               │
├──────────────────────────────────────────────────────────────────┤
│  60초    │  최대 1분        │  매우 많음     │  1분 이내               │
│  300초   │  최대 5분        │  많음         │  5분 이내               │
│  3600초  │  최대 1시간      │  보통         │  1시간 이내              │
│  86400초 │  최대 24시간     │  적음         │  24시간 이내             │
└──────────────────────────────────────────────────────────────────┘

Route53 비용 영향:
  TTL=60 (초당 쿼리 10만 개):
    월 DNS 쿼리 수 = 10만 × 60초 × 86400 / 60 ≈ 86억
    Route53 비용 ≈ $3,440/월 (Standard zone)
  
  TTL=3600 (동일 트래픽):
    Resolver 캐시로 쿼리 수 60배 감소
    Route53 비용 ≈ $57/월
  → TTL=60으로 유지하면 60배 비용 증가

Authoritative NS 부하:
  전 세계 Resolver 수 × 조회 빈도 = NS 부하
  TTL 낮을수록 Resolver가 자주 재조회 → NS 부하 증가
  고트래픽 도메인: TTL 너무 낮으면 NS가 DDoS 같은 상황
```

---

## ⚖️ 트레이드오프

```
TTL 설계 원칙:

빠른 전파 vs 낮은 DNS 비용:
  낮은 TTL (60~300s): 빠른 변경, 높은 비용
  높은 TTL (3600s+):  느린 변경, 낮은 비용

권장 전략:
  평상시: TTL=3600 (낮은 비용, 캐시 효율)
  마이그레이션 준비: TTL=300 (최소 현재 TTL만큼 기다린 후)
  마이그레이션 완료 후: TTL=3600으로 복구

Negative Caching TTL:
  SOA Minimum TTL이 너무 높으면:
  → 존재하지 않는 도메인 조회 후 생성해도 오래 기다려야 함
  → 300~600초 권장

클라우드 서비스 특성:
  AWS ALB, ECS: IP가 자주 바뀜
  → TTL=60 또는 Route53 Alias 사용
  → ALB 앞에 Alias 설정 시 AWS가 IP 추적 자동화

K8s CoreDNS TTL:
  기본값: 30초 (cluster.local), 30초 (external)
  Service 삭제 → 재생성 시 30초 내 DNS 반영
  StatefulSet: 각 Pod에 안정적 DNS 이름 → TTL 중요

블루-그린에서 TTL:
  CNAME으로 환경 스위칭: TTL=60 (빠른 전환)
  실제 서버 IP (A 레코드): TTL=3600 (안정)
  이 조합이 최적: 빠른 스위칭 + 낮은 비용
```

---

## 📌 핵심 정리

```
DNS 캐싱과 전파 핵심 요약:

TTL 동작:
  Authoritative NS가 설정한 시간 동안 Resolver 캐시
  캐시 중 레코드 변경해도 TTL 만료까지 이전 값 사용
  "전파 시간" = 최대 TTL 초 (이미 캐시된 Resolver 기준)

마이그레이션 전략:
  T-24h~T-48h: 현재 TTL 확인 후 300s로 낮춤
               반드시 기존 TTL만큼 대기!
  T-0:         IP 변경 (이제 5분 내 전파)
  T+1h:        안정화 후 TTL=3600으로 복구

Negative Caching:
  NXDOMAIN(없는 도메인)도 SOA Minimum TTL만큼 캐시
  배포 시 도메인 생성 순서 주의
  300~600초 권장

캐시 계층:
  브라우저 → OS → JVM → 로컬 Resolver → ISP Resolver
  JVM 영구 캐시 주의 (networkaddress.cache.ttl 설정 필수)

전파 확인:
  dig +norecurse: Resolver 캐시 상태 확인
  dig @8.8.8.8 / @1.1.1.1: 다른 Resolver 비교
  whatsmydns.net: 전 세계 동시 확인

블루-그린 DNS:
  CNAME(TTL=60)으로 환경 스위칭
  A 레코드(TTL=3600)는 실제 IP
  → 빠른 롤백 + 낮은 DNS 비용 균형
```

---

## 🤔 생각해볼 문제

**Q1.** TTL을 0으로 설정하면 어떻게 되는가? DNS 캐싱이 전혀 일어나지 않아 이상적인 것 아닌가?

<details>
<summary>해설 보기</summary>

**이론적으로는:** TTL=0은 "즉시 만료", 캐시하지 말라는 의미입니다. 모든 조회마다 Authoritative NS에 직접 쿼리.

**실제 문제:**

1. **RFC 1035 준수 여부:** TTL=0은 유효하지만, 일부 Resolver가 최소 TTL을 강제(예: 최소 5~30초). 실제로 매 요청마다 Authoritative NS까지 가지 않을 수 있음.

2. **Authoritative NS 폭발적 부하:**
   - 모든 클라이언트가 매 요청마다 직접 조회
   - 초당 100만 요청 서비스 → Authoritative NS에 100만 쿼리/초
   - Route53 비용: $400/월 × 100배 이상 = 수십만 달러

3. **지연 증가:**
   - 캐시 없음 → Recursive Resolver가 항상 Authoritative NS까지 쿼리
   - 지리적 거리에 따라 50~200ms 추가 지연 (모든 요청마다)

4. **DNS 서버 장애 영향:**
   - Authoritative NS 장애 → 즉시 모든 조회 실패
   - TTL이 있으면 캐시가 완충 역할

**현실적 권장:** TTL=60초가 "빠른 전파"와 "캐시 효율"의 현실적 하한선. 특별한 경우(헬스 체크 기반 장애 조치)에 TTL=30초까지 낮출 수 있지만 비용 상승 감수.

</details>

---

**Q2.** 지역별로 다른 IP를 반환하는 서비스에서 DNS TTL이 낮을수록 항상 좋은가?

<details>
<summary>해설 보기</summary>

**GeoDNS의 원리:**
GeoDNS(Route53 Latency, Geolocation 라우팅)는 Recursive Resolver의 위치를 기반으로 다른 IP를 반환합니다.

**TTL이 낮을 때의 문제:**

클라이언트가 서울에서 부산으로 이동한 경우:
- Resolver: 서울(KT) → 부산(SK)로 변경
- 부산 Resolver가 DNS를 처음 조회 → 부산 가까운 IP 반환 (정상)
- 하지만 클라이언트 OS/브라우저 캐시: TTL 동안 서울 IP 유지

**TTL이 높을 때:**
- 클라이언트: 이동 후에도 서울 IP 계속 사용 (TTL 만료까지)
- 서울 서버에 부산에서 요청 → 지연 증가

**균형점:**
- GeoDNS 서비스: TTL=60~300초 권장
- 너무 낮으면: Authoritative NS 부하, 비용 증가
- 너무 높으면: 이동 시 최적 서버로 라우팅 안 됨

**CDN의 접근:**
대형 CDN(Cloudflare, AWS CloudFront)은 Anycast를 사용하여 DNS 기반이 아닌 네트워크 라우팅 레벨에서 최적 서버 선택. TTL 영향을 최소화하면서 지역 최적화 달성.

</details>

---

**Q3.** 사이트 마이그레이션 중 일부 사용자는 새 서버로, 일부는 이전 서버로 접속한다. 이 분할 상태를 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

**"DNS Split"이라고 불리는 상황:**
TTL 내에 캐시를 가진 사용자(이전 IP)와 캐시가 만료된 사용자(새 IP)가 동시에 존재합니다.

**대응 전략:**

1. **양쪽 서버에서 동일 응답:**
   마이그레이션 기간 동안 이전/새 서버 모두 동일한 데이터 제공.
   - 이전 서버: 읽기 전용 또는 새 서버로 프록시
   - 데이터베이스: 동기화 유지 (Replication 등)

2. **쿠키 기반 라우팅:**
   이전 서버에 접속한 사용자 쿠키에 "old server" 표시 → 이전 서버로 계속 라우팅
   DNS 전파 완료 후 모든 사용자 새 서버로 이동

3. **로드밸런서 앞에서 처리:**
   DNS는 LB를 가리키게 유지
   LB에서 트래픽을 이전/새 서버로 분기
   → DNS 전파 문제 없이 즉각 제어 가능

4. **세션 공유:**
   이전/새 서버가 같은 Redis 세션 저장소 공유
   어느 서버로 가든 동일한 사용자 경험

**모니터링:**
```bash
# 이전 IP와 새 IP의 트래픽 비율 모니터링
# 로그에서 서버별 요청 수 추적
# 시간이 지나면서 이전 서버 트래픽 감소 확인
```

**결론:** 완벽한 동시 전환은 불가능합니다. TTL을 낮추고 두 서버가 동일하게 동작하게 만드는 것이 최선입니다.

</details>

---

<div align="center">

**[⬅️ 이전: DNS 레코드](./02-dns-records.md)** | **[홈으로 🏠](../README.md)** | **[다음: DNS 보안과 고급 ➡️](./04-dns-security-advanced.md)**

</div>
