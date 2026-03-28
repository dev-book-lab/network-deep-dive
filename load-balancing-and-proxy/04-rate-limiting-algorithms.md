# Rate Limiting — Token Bucket, Leaky Bucket, Sliding Window

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Fixed Window의 경계 시점 burst 문제가 정확히 어떻게 발생하는가?
- Token Bucket이 순간 burst를 허용하면서 평균 처리율을 어떻게 제한하는가?
- Leaky Bucket과 Token Bucket의 차이는 무엇인가?
- Sliding Window Log vs Sliding Window Counter의 트레이드오프는?
- Nginx `limit_req_zone`과 Spring의 Bucket4j는 어떻게 구현되는가?
- 분산 환경(다중 서버)에서 Rate Limiting을 일관되게 적용하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"특정 IP가 초당 10,000 요청으로 API 서버를 마비시켰다":
  DDoS 공격 또는 버그 있는 클라이언트
  → Rate Limiting 없으면 서버 과부하 → 전체 서비스 다운
  → Rate Limiting: 429 Too Many Requests 반환 → 공격자 차단
  
"OpenAI API처럼 사용량을 제한하고 싶다":
  Free tier: 초당 3 요청, 월 1000 요청
  Pro tier: 초당 60 요청, 월 무제한
  
  → Token Bucket: 평균 rate 제한 + 순간 burst 허용
  → 사용자별 Redis 키로 관리
  → 429 응답 + Retry-After 헤더

"Rate Limiting인데 왜 burst가 가능한가":
  Token Bucket: 토큰이 쌓여있으면 순간적으로 많이 처리
  → 사용자가 30초 동안 안 쓰면 토큰 30개 적립
  → 갑자기 30개 요청 → 허용 (정상 사용 패턴)
  
  Leaky Bucket: 항상 일정한 속도로만 처리
  → burst 없음 (엄격한 속도 제한)
```

---

## 😱 흔한 실수

```
Before — Rate Limiting 알고리즘을 모를 때:

실수 1: Fixed Window의 경계 시점 burst 인지 못함
  limit: 100 req/min
  12:00:00~12:00:59: 100 요청 허용
  12:01:00~12:01:59: 다시 100 요청 허용
  
  공격: 12:00:59에 100개 + 12:01:00에 100개 = 200개/2초
  → 분당 100개 제한인데 2초에 200개 가능!
  → 실제 서버 부하: 200 req/2s = 100 req/s (분당 제한의 60배)

실수 2: 분산 환경에서 로컬 Rate Limiting 사용
  서버 A: localhost 카운터 (100 req/s 허용)
  서버 B: localhost 카운터 (100 req/s 허용)
  LB: 요청을 A, B에 분산
  
  결과: 사용자가 200 req/s 가능 (각 서버에 100개씩)
  → 중앙화 Rate Limiting 필요 (Redis 기반)

실수 3: Rate Limit 응답에 Retry-After 없음
  429 Too Many Requests 반환
  클라이언트: "언제 재시도하지?" → 즉시 재시도 → 더 많은 429
  → Retry-After: 60 헤더 추가 필수
  → X-RateLimit-Remaining, X-RateLimit-Reset도 유용

실수 4: 전체 서비스에 동일한 Rate Limit 적용
  로그인 API: 초당 1000 요청 제한 (공격 방어)
  파일 업로드 API: 초당 10 요청 (무거운 처리)
  내부 헬스 체크: 제한 없음
  
  → API 엔드포인트별로 다른 Rate Limit 설정 필요
```

---

## ✨ 올바른 접근

```
After — Rate Limiting 알고리즘을 알면:

Nginx Rate Limiting 설정:
  # 전역 설정 (http 블록)
  # $binary_remote_addr: IP당 카운터 (Token Bucket 방식)
  limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
  limit_req_zone $binary_remote_addr zone=login_limit:10m rate=1r/s;
  
  server {
      # API 일반 Rate Limit (burst 허용)
      location /api/ {
          limit_req zone=api_limit burst=20 nodelay;
          # rate=10r/s: 평균 10 req/s
          # burst=20: 순간 최대 20개 초과 허용 (토큰 버킷)
          # nodelay: burst 내에서는 지연 없이 즉시 처리
      }
      
      # 로그인 엄격 제한 (burst 없음)
      location /api/auth/login {
          limit_req zone=login_limit;
          # burst 없음: 초당 1개만 허용 (브루트포스 방어)
      }
      
      # 429 응답 커스터마이징
      limit_req_status 429;
  }

Spring Boot + Bucket4j (Token Bucket):
  @Bean
  public Bucket createBucket() {
      Bandwidth limit = Bandwidth.classic(
          100,                          // 최대 토큰 수 (burst 용량)
          Refill.greedy(10, Duration.ofSeconds(1))  // 초당 10개 충전
      );
      return Bucket.builder().addLimit(limit).build();
  }
  
  @GetMapping("/api/data")
  public ResponseEntity<?> getData() {
      if (!bucket.tryConsume(1)) {
          return ResponseEntity.status(429)
              .header("Retry-After", "1")
              .header("X-RateLimit-Remaining", "0")
              .build();
      }
      return ResponseEntity.ok(service.getData());
  }

Redis 기반 분산 Rate Limiting (Bucket4j + Redis):
  @Bean
  public BucketProxyManager<String> bucketManager(RedissonClient redisson) {
      ProxyManager<String> proxyManager =
          new RedissonBasedProxyManager(redisson, ...);
      return proxyManager;
  }
  
  // 사용자별 Rate Limiting
  Bucket bucket = bucketManager.builder()
      .build(userId, () -> createBucketConfig());
  if (!bucket.tryConsume(1)) {
      // 429 반환
  }
```

---

## 🔬 내부 동작 원리

### 1. Fixed Window Counter — 가장 단순하지만 취약

```
구현:
  window_key = "{IP}:{floor(current_time / window_size)}"
  count = INCR window_key
  if count == 1: EXPIRE window_key window_size
  if count > limit: reject

예시 (limit=100, window=60초):
  12:00:00: window_key = "1.2.3.4:1200"
             count = 1
  12:00:30: count = 50
  12:00:59: count = 100 (한도 도달)
  12:01:00: 새 window → "1.2.3.4:1201" → count = 1
  12:01:01: count = 2

경계 시점 burst 문제:
  12:00:59: 100개 요청 → 허용 (100이 한도)
  12:01:00: 100개 요청 → 허용 (새 윈도우, count=100)
  
  2초 동안 200개 처리 = 초당 100개 처리 (분당 6000개 페이스!)
  분당 100개를 의도했는데 최악의 경우 2배 burst 허용

장점:
  구현 매우 단순 (Redis INCR 한 줄)
  메모리 효율 (윈도우당 1개 카운터)

단점:
  경계 burst 문제
  → 정확한 Rate Limiting이 필요한 경우 부적합
```

### 2. Sliding Window Log — 정확하지만 메모리 비용

```
구현:
  log = Redis Sorted Set (score = timestamp, member = request_id)
  
  요청 시:
  현재 시각 = now
  window_start = now - window_size
  ZADD log_key now request_id
  ZREMRANGEBYSCORE log_key 0 window_start  # 윈도우 밖 제거
  count = ZCARD log_key
  if count > limit: reject
  EXPIRE log_key window_size

예시 (limit=10, window=60초):
  12:00:01: 요청 1 → log=[1]        count=1 → 허용
  12:00:10: 요청 2 → log=[1,2]      count=2 → 허용
  ...
  12:00:59: 요청 10 → log=[1..10]   count=10 → 허용
  12:01:00: 요청 11 → log=[2..10,11] (1은 60초 밖) count=10 → 허용
  12:01:01: 요청 12 → 윈도우 내 10개 이미 있음 → 거부!

Fixed Window 경계 문제 해결:
  12:00:59에 100개 → 12:01:00에 100개 시도
  → 12:01:00에 sliding window: 12:00:00~12:01:00 사이 요청 수 확인
  → 이미 100개 있음 → 거부 (burst 방지!)

단점:
  메모리: 요청마다 로그 저장 → 고트래픽 시 메모리 폭발
  limit=1000, 사용자 10만 명 → 최대 1억 개 로그 항목!
  → 사용자가 많은 서비스에서 현실적 적용 어려움
```

### 3. Sliding Window Counter — 정확성과 효율의 균형

```
아이디어:
  Sliding Window를 두 Fixed Window 카운터의 가중 합으로 근사

계산:
  현재 윈도우 시작 = floor(now / window) × window
  이전 윈도우 시작 = 현재 윈도우 시작 - window
  
  이전 윈도우의 가중치 = (현재 윈도우 시작 - now_within_window) / window
  
  예시 (window=60초, limit=100):
  현재 시각: 12:01:45 (현재 윈도우 시작부터 45초 경과)
  
  이전 윈도우(12:00~12:01): count=80개
  현재 윈도우(12:01~12:02): count=30개
  
  이전 가중치 = (60-45)/60 = 0.25
  슬라이딩 count = 80 × 0.25 + 30 = 50개
  → 50 < 100이므로 허용

메모리 효율:
  윈도우당 1개 카운터 (2개 × 사용자 수)
  → Sliding Window Log 대비 극적 메모리 절감

근사 오차:
  최악의 경우 이전 윈도우가 균등 분포라고 가정
  실제 오차 범위: ±0~10% 정도
  → 완벽하지 않지만 실용적 (Cloudflare가 이 방식 사용)

Redis 구현:
  MULTI
  INCR current_window_key
  EXPIRE current_window_key (window * 2)  # 이전 윈도우도 필요
  EXEC
  
  # 두 윈도우 값 조회 후 가중치 계산 (애플리케이션 레이어)
```

### 4. Token Bucket — burst 허용 + 평균 제한

```
원리:
  버킷 최대 용량: capacity (순간 최대 burst)
  충전 속도: rate (초당 토큰 추가)
  요청 비용: 1 요청 = 1 토큰 소비
  
  토큰 있으면: 소비 후 처리
  토큰 없으면: 거부 (또는 대기)

상태 추적 (Redis):
  tokens:  현재 토큰 수
  last_refill: 마지막 충전 시각

요청 시 처리 (원자적으로):
  elapsed = now - last_refill
  new_tokens = elapsed × rate
  tokens = min(capacity, tokens + new_tokens)
  last_refill = now
  
  if tokens >= 1:
      tokens -= 1
      허용
  else:
      거부

예시 (capacity=10, rate=1/s):
  초기: tokens=10
  1초 후: tokens=10 (이미 최대)
  10개 연속 요청: tokens=0 (버스트!)
  1초 대기: tokens=1
  1개 요청: tokens=0
  
  → 순간 10개 burst 가능 (capacity 만큼)
  → 평균적으로 초당 1개 (rate)

Nginx Token Bucket (limit_req_zone):
  limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
  limit_req zone=api burst=20 nodelay;
  
  rate=10r/s: 초당 10개 토큰 충전
  burst=20:   버킷 최대 용량 20 (순간 20개 burst)
  nodelay:    burst 내에서 즉시 처리 (지연 없음)
  
  nodelay 없으면:
  burst=20이지만 20개를 초당 1개씩 처리
  (2초에 걸쳐 처리 → 사실상 큐)
```

### 5. Leaky Bucket — 엄격한 속도 제한

```
원리:
  버킷에 요청이 들어옴
  버킷에서 일정한 속도(rate)로 처리
  버킷이 가득 차면 새 요청 거부

구현 (큐 방식):
  queue: 처리 대기 요청 (최대 capacity)
  
  요청 시:
  if queue.size < capacity:
      queue.add(request)
      허용 (나중에 처리)
  else:
      거부

처리:
  매 1/rate 초마다 queue에서 1개 꺼내서 처리

특성:
  출력 속도 완전 일정 (rate 초과 불가)
  burst가 허용되지만 처리는 rate로 고정
  → 초당 100개 요청이 와도 처리는 항상 10/s
  → 큐가 가득 차면 거부

Token Bucket vs Leaky Bucket:
  Token Bucket:
    순간 burst → rate를 순간 초과해서 처리 가능
    "사용 안 한 토큰 적립 → 나중에 몰아서 사용"
    → 실제 처리 속도가 rate를 초과하는 순간이 있음
    
  Leaky Bucket:
    처리 속도 = 항상 rate (초과 불가)
    "요청이 쌓여도 처리는 일정 속도"
    → 업스트림 서버 보호에 더 적합 (일정한 부하)
  
  API Rate Limiting: Token Bucket (사용자 경험 좋음)
  백엔드 보호: Leaky Bucket (일정한 부하 유지)
```

### 6. 분산 Rate Limiting — Redis + Lua

```
왜 Redis가 필요한가:
  서버 A: 사용자 X의 로컬 카운터 = 50
  서버 B: 사용자 X의 로컬 카운터 = 50
  실제 요청 수 = 100 (한도 초과!)
  → 서버별 로컬 카운터는 분산 환경에서 부정확

Redis Lua 스크립트 (원자적 Rate Limiting):
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  
  local current = redis.call('INCR', key)
  if current == 1 then
      redis.call('EXPIRE', key, window)
  end
  
  if current > limit then
      return 0  -- 거부
  end
  return 1  -- 허용

Lua 스크립트의 원자성:
  INCR + 비교 + EXPIRE를 하나의 원자적 트랜잭션으로 처리
  → 여러 서버에서 동시 실행해도 정확한 카운터

Bucket4j + Hazelcast/Redis:
  @Bean
  public ProxyManager<Long> proxyManager(RedissonClient redisson) {
      return Bucket4jRedisson.casBasedBuilder(redisson)
          .expirationAfterWrite(ExpirationAfterWriteStrategy
              .basedOnTimeForRefillingBucketUpToMax(ofSeconds(60)))
          .build();
  }
  
  // 요청 처리
  long userId = getCurrentUserId();
  BucketConfiguration config = BucketConfiguration.builder()
      .addLimit(Bandwidth.classic(100, Refill.greedy(10, ofSeconds(1))))
      .build();
  
  Bucket bucket = proxyManager.builder().build(userId, () -> config);
  if (!bucket.tryConsume(1)) {
      return ResponseEntity.status(429)
          .header("X-RateLimit-Remaining", "0")
          .header("Retry-After", "1")
          .build();
  }
```

---

## 💻 실전 실험

### 실험 1: Fixed Window Burst 문제 시연

```bash
# 간단한 Fixed Window Rate Limiter (Python)
python3 << 'EOF'
import time, collections

LIMIT = 5
WINDOW = 10  # 10초

requests = collections.deque()

def is_allowed(timestamp):
    window_start = timestamp - WINDOW
    while requests and requests[0] < window_start:
        requests.popleft()
    
    if len(requests) < LIMIT:
        requests.append(timestamp)
        return True
    return False

# 정상 사용
for i in range(5):
    print(f"요청 {i+1}: {is_allowed(time.time())}")
    time.sleep(2)

# Fixed Window 경계 burst
now = int(time.time())
window_end = (now // 10 + 1) * 10  # 다음 윈도우 시작
time.sleep(window_end - time.time() - 0.1)  # 경계 직전

print("=== 경계 시점 burst ===")
for i in range(5):
    print(f"경계 전 요청 {i+1}: {is_allowed(time.time())}")

time.sleep(0.2)  # 윈도우 넘어가기

for i in range(5):
    print(f"경계 후 요청 {i+1}: {is_allowed(time.time())}")
EOF
```

### 실험 2: Nginx Rate Limiting 설정 테스트

```bash
# Nginx 설정
cat > /tmp/rate-limit-test.conf << 'EOF'
limit_req_zone $binary_remote_addr zone=test:1m rate=5r/s;

server {
    listen 8080;
    location / {
        limit_req zone=test burst=10 nodelay;
        limit_req_status 429;
        return 200 "OK\n";
    }
}
EOF

nginx -c /tmp/rate-limit-test.conf

# burst 테스트: 10개 동시 요청 (burst 내)
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code} " http://localhost:8080/
done
# 모두 200 OK (burst=10 내)

echo ""

# burst 초과: 11개째
for i in $(seq 1 11); do
  curl -s -o /dev/null -w "%{http_code} " http://localhost:8080/
done
# 11번째: 429 Too Many Requests
```

### 실험 3: Redis 기반 Rate Limiting

```bash
# Redis CLI로 Token Bucket 시뮬레이션
redis-cli

# 토큰 초기화
SET tokens:user:1 10
SET last_refill:user:1 $(date +%s)

# 요청 처리 (Lua 스크립트)
redis-cli EVAL "
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local last_refill = tonumber(redis.call('GET', 'last_refill:' .. key) or 0)
local tokens = tonumber(redis.call('GET', 'tokens:' .. key) or capacity)

local elapsed = now - last_refill
local new_tokens = elapsed * rate
tokens = math.min(capacity, tokens + new_tokens)

redis.call('SET', 'last_refill:' .. key, now)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call('SET', 'tokens:' .. key, tokens)
    return 1
else
    redis.call('SET', 'tokens:' .. key, tokens)
    return 0
end
" 1 user:1 10 1 $(date +%s)
```

### 실험 4: Rate Limit 응답 헤더 확인

```bash
# Rate Limit 적용된 API 호출
curl -v http://localhost:8080/api/data 2>&1 | grep -E "X-RateLimit|Retry-After|429"
# X-RateLimit-Limit: 100
# X-RateLimit-Remaining: 99
# X-RateLimit-Reset: 1711603660

# 한도 초과 시
# HTTP/1.1 429 Too Many Requests
# Retry-After: 60
# X-RateLimit-Remaining: 0

# 429 후 Retry-After 준수하는 클라이언트
retry_after=$(curl -s -I http://api.example.com/data | grep "Retry-After" | awk '{print $2}')
echo "Waiting ${retry_after}s before retry..."
sleep $retry_after
curl http://api.example.com/data
```

---

## 📊 성능/비용 비교

```
알고리즘별 비교:

┌─────────────────────────────────────────────────────────────────────┐
│  알고리즘                │  메모리   │  정확성  │  burst 허용  │  복잡도    │
├─────────────────────────────────────────────────────────────────────┤
│  Fixed Window          │ 매우 낮음 │  낮음    │  경계 burst  │  낮음     │
│  Sliding Window Log    │  높음    │  정확    │  없음        │  중간     │
│  Sliding Window Counter│  낮음    │  ~90%   │  없음        │  중간     │
│  Token Bucket          │  낮음    │  높음    │  있음 (cap)  │  중간     │
│  Leaky Bucket          │  낮음    │  높음    │  없음        │  중간     │
└─────────────────────────────────────────────────────────────────────┘

Redis 기반 Rate Limiting 성능:
  Redis 단일 명령: ~0.1ms
  Lua 스크립트 (원자적 Rate Limiting): ~0.3ms
  → 초당 3000~5000 Rate Limit 결정 가능 (단일 Redis)
  → Redis Cluster로 수평 확장 가능

Nginx limit_req 성능:
  메모리 기반 (limit_req_zone 10m = 10MB)
  10MB에 약 160,000개 IP 상태 저장 가능
  → 처리 오버헤드: ~0.01ms (메모리 접근)
  → 수백만 rps에서도 동작 가능
```

---

## ⚖️ 트레이드오프

```
알고리즘 선택:

API Rate Limiting (사용자 경험 중심):
  Token Bucket 권장
  → 순간 burst 허용 (정상 사용 패턴 지원)
  → 평균 rate 제한 (과부하 방지)

백엔드 보호 (일정 부하):
  Leaky Bucket 또는 Sliding Window Counter
  → 항상 일정한 처리 속도
  → burst로 인한 순간 과부하 없음

정확도 vs 메모리:
  정확도 필요: Sliding Window Log (메모리 높음)
  메모리 절약: Sliding Window Counter (근사, 오차 ~10%)
  실용적 선택: Token Bucket (정확 + 메모리 효율)

분산 vs 로컬:
  단일 서버: 로컬 카운터 (빠름, 정확)
  다중 서버: Redis 기반 (약간 느림, 정확)
  
  Redis 타임아웃 대비:
  Redis 다운 → fallback: 로컬 카운터로 near-accurate 제한
  또는 "allow all when Redis down" (가용성 우선)
  또는 "deny all when Redis down" (보안 우선)

Rate Limit 적용 레벨:
  IP 기반: DDoS 방어 (Nginx 레이어)
  사용자 기반: API quota 관리 (애플리케이션 레이어)
  API 키 기반: 서비스별 limit (애플리케이션 레이어)
  전역: 서비스 전체 보호 (Nginx 또는 API Gateway)
```

---

## 📌 핵심 정리

```
Rate Limiting 알고리즘 핵심 요약:

Fixed Window:
  구현 단순, 경계 시점 burst 취약 (최대 2배 초과)
  → 단순 abuse 방어에는 충분, 정밀 제어에는 부족

Sliding Window Log:
  요청마다 타임스탬프 저장, 정확한 제한
  메모리 비용 높음 → 소규모 서비스에 적합

Sliding Window Counter:
  두 Fixed Window 가중치 합산 근사
  ~10% 오차, 메모리 효율적 → Cloudflare 채택

Token Bucket:
  토큰 충전(rate) + 소비, burst 허용(capacity)
  → API Rate Limiting 표준 (Nginx, AWS, 대부분의 API)
  파라미터: rate (평균 속도), capacity (최대 burst)

Leaky Bucket:
  일정한 처리 속도 보장 (burst 없음)
  → 백엔드 보호, 일정 부하 유지에 적합

구현:
  Nginx: limit_req_zone (Token Bucket 방식)
  Spring: Bucket4j + Redis (분산 Token Bucket)
  응답: 429 + Retry-After + X-RateLimit-* 헤더

분산 환경:
  Redis + Lua 스크립트 (원자적 처리)
  Redis 장애 대비 fallback 전략 필요
```

---

## 🤔 생각해볼 문제

**Q1.** Rate Limiting을 Nginx 레이어에서 하는 것과 Spring 애플리케이션 레이어에서 하는 것의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

**Nginx 레이어 Rate Limiting:**

장점:
- 앱 서버 도달 전에 차단 → CPU/메모리 절약
- 앱 코드 수정 없이 설정 변경 가능
- IP 기반으로 빠르고 정확
- 초고속 처리 (메모리 기반, 초당 수백만 결정)

단점:
- 사용자 인증 정보 없음 (IP만 볼 수 있음)
- 비즈니스 로직 기반 제한 어려움 (사용자 티어, API 키 등)
- 분산 환경에서 각 Nginx 노드가 독립적으로 카운터 관리 (여러 Nginx가 있으면 부정확)

**Spring 애플리케이션 레이어:**

장점:
- 사용자 ID, API 키, 사용자 티어 기반 세밀한 제한
- Redis 기반 분산 정확한 카운터
- 비즈니스 로직과 통합 (DB에서 quota 조회 등)
- 세밀한 응답 제어 (남은 토큰 수, 초기화 시간 헤더)

단점:
- 앱 서버까지 요청이 도달 → 리소스 소비
- 코드 복잡도 증가

**실무 조합:**
```
인터넷 → Nginx(IP 기반 DDoS 방어) → Spring(사용자 기반 quota 관리)
```
- Nginx: 초당 1000 req 이상의 단일 IP → 즉시 차단 (DDoS 방어)
- Spring: 사용자별 API quota (무료 100/일, 프로 10000/일)

</details>

---

**Q2.** Token Bucket에서 `capacity`와 `rate`를 어떻게 설정해야 하는가? 실제 서비스에서 값을 결정하는 방법은?

<details>
<summary>해설 보기</summary>

**파라미터 의미:**
- `rate`: 평균 처리율 (초당 N개 토큰 충전)
- `capacity`: 버킷 최대 크기 (순간 burst 허용량)

**설정 방법:**

1. **서버 최대 처리량 측정:**
```bash
# wrk 또는 ab로 서버 한계 측정
wrk -t4 -c100 -d30s http://server/api/endpoint
# 서버 최대 처리량: 예) 1000 rps
```

2. **rate = 최대 처리량 / 사용자 수:**
```
최대 1000 rps, 동시 100 사용자:
→ 사용자당 rate = 10 rps
```

3. **capacity = 정상 burst 패턴 분석:**
```
사용자가 페이지 로딩 시 동시 요청: 5~20개
→ capacity = 20 (페이지 로딩 burst 허용)
```

**비즈니스 요구사항 반영:**
```java
// 사용자 티어별 다른 설정
BucketConfiguration config = switch (user.getTier()) {
    case FREE -> BucketConfiguration.builder()
        .addLimit(Bandwidth.classic(100, Refill.greedy(1, ofSeconds(1))))  // 1 rps, burst 100
        .build();
    case PRO -> BucketConfiguration.builder()
        .addLimit(Bandwidth.classic(1000, Refill.greedy(60, ofSeconds(1))))  // 60 rps, burst 1000
        .build();
    case ENTERPRISE -> BucketConfiguration.builder()
        .addLimit(Bandwidth.classic(10000, Refill.greedy(1000, ofSeconds(1))))
        .build();
};
```

**점진적 조정:**
1. 초기: 느슨하게 설정 (정상 트래픽 파악)
2. 로그 분석: 실제 사용 패턴 확인 (P99 요청 수 등)
3. 조정: burst가 자주 차단되면 capacity 증가
4. 모니터링: 429 비율 ≤ 1% 목표

</details>

---

**Q3.** 다중 Rate Limit 규칙(초당 10개, 분당 100개, 시간당 1000개)을 동시에 적용하려면 어떻게 구현하는가?

<details>
<summary>해설 보기</summary>

**여러 Bandwidth를 겹쳐서 적용 (Bucket4j):**

```java
BucketConfiguration config = BucketConfiguration.builder()
    // 초당 10개
    .addLimit(Bandwidth.classic(10, Refill.greedy(10, ofSeconds(1))))
    // 분당 100개
    .addLimit(Bandwidth.classic(100, Refill.greedy(100, ofMinutes(1))))
    // 시간당 1000개
    .addLimit(Bandwidth.classic(1000, Refill.greedy(1000, ofHours(1))))
    .build();

// 요청 시 모든 limit을 동시에 확인
// 하나라도 부족하면 거부
if (!bucket.tryConsume(1)) {
    // 429 반환
}
```

**작동 원리:**
`tryConsume(1)`은 모든 bandwidth의 토큰이 있을 때만 성공. 하나라도 부족하면 실패.

**응답 헤더에 어느 limit이 걸렸는지 표시:**
```java
ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
if (!probe.isConsumed()) {
    long waitSeconds = probe.getNanosToWaitForRefill() / 1_000_000_000L;
    return ResponseEntity.status(429)
        .header("Retry-After", String.valueOf(waitSeconds))
        .header("X-RateLimit-Remaining-Second", 
            String.valueOf(bucket.getAvailableTokens()))  // 근사값
        .build();
}
```

**Nginx에서 여러 zone 겹치기:**
```nginx
location /api/ {
    limit_req zone=per_second burst=5 nodelay;  # 초당 10개
    limit_req zone=per_minute burst=20 nodelay;  # 분당 100개
    # 둘 다 만족해야 통과
}
```

**실무 주의사항:**
- 여러 limit이 있을 때 가장 제한적인 것이 병목
- 복잡한 다단계 limit은 디버깅 어려움 → 로그에 어느 limit이 걸렸는지 기록 필요

</details>

---

<div align="center">

**[⬅️ 이전: Sticky Session](./03-sticky-session-vs-stateless.md)** | **[홈으로 🏠](../README.md)** | **[다음: 서킷 브레이커 ➡️](./05-circuit-breaker.md)**

</div>
