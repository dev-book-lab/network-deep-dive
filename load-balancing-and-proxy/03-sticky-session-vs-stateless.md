# 연결 유지 전략 — Sticky Session vs Stateless 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Sticky Session이 수평 확장을 어렵게 만드는 이유는?
- IP Hash 기반 Sticky Session이 갖는 한계는?
- Cookie 기반 Session Affinity는 어떻게 동작하는가?
- Spring Session + Redis로 Sticky Session 없이 세션을 어떻게 공유하는가?
- JWT 기반 Stateless 설계와 서버 세션의 트레이드오프는?
- 세션 스토어 장애 시 서비스가 완전히 다운되는 것을 어떻게 방지하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"배포 후 일부 사용자만 로그아웃된다":
  원인: Sticky Session → 배포 시 서버 재시작 → 세션 소실
  사용자 A (서버 1 Sticky): 서버 1 재시작 → 세션 없음 → 강제 로그아웃
  사용자 B (서버 2 Sticky): 영향 없음
  
  해결:
  1. 서버 메모리 → Redis 세션 스토어
  2. JWT Stateless 인증
  3. Rolling 배포 + 세션 Draining

"Scale-out 후 장바구니가 사라진다":
  1번 서버: 장바구니 세션 저장
  2번 서버 추가: LB가 새 서버로 라우팅 → 세션 없음
  → Sticky Session 설정 안 되어 있음
  → 또는 Stateless 설계로 전환 필요

"Redis 장애 시 모든 사용자가 로그아웃된다":
  Spring Session + Redis → Redis가 세션 저장소
  Redis 다운 → 세션 없음 → 전체 서비스 다운
  
  방어:
  Redis Sentinel 또는 Cluster (HA)
  세션이 없어도 일부 기능은 동작하도록 graceful degradation
  JWT + Redis 조합 (Redis 없어도 기본 인증 가능)
```

---

## 😱 흔한 실수

```
Before — 세션 관리를 모를 때:

실수 1: IP Hash Sticky Session으로 확장 문제 해결 시도
  IP Hash: 소스 IP → 항상 같은 서버
  문제 1: NAT 뒤 수천 명 → 같은 IP → 한 서버에 몰림
  문제 2: 서버 추가 시 IP Hash 재계산 → 일부 사용자 다른 서버로 이동
          → 세션 없는 서버 → 강제 로그아웃
  문제 3: 서버 장애 시 해당 서버 사용자 전원 로그아웃

실수 2: JWT 토큰에 민감한 정보 저장
  JWT payload는 Base64 인코딩 (암호화 아님!)
  누구나 디코딩 가능
  → 비밀번호, 카드번호, 개인정보 절대 포함 금지
  → 사용자 ID, 권한 정도만 포함

실수 3: JWT를 localStorage에 저장
  localStorage: JavaScript에서 접근 가능
  XSS 공격 → localStorage 탈취 → JWT 도용
  → 권장: HttpOnly 쿠키에 저장
    httpOnly: JS 접근 불가 → XSS 방어
    secure: HTTPS에서만 전송

실수 4: Redis 세션 스토어를 SPOF(단일 장애점)로 방치
  Redis 한 대 → 장애 = 전체 서비스 다운
  → Redis Sentinel (HA) 또는 Redis Cluster
  → 또는 Circuit Breaker로 Redis 장애 시 로컬 세션 폴백
```

---

## ✨ 올바른 접근

```
After — 세션 관리를 알면:

Spring Session + Redis 설정:
  # pom.xml
  <dependency>
      <groupId>org.springframework.session</groupId>
      <artifactId>spring-session-data-redis</artifactId>
  </dependency>
  
  # application.properties
  spring.session.store-type=redis
  spring.session.timeout=30m
  spring.data.redis.host=redis-cluster.internal
  spring.data.redis.port=6379
  
  # 설정 클래스
  @EnableRedisHttpSession(maxInactiveIntervalInSeconds = 1800)
  @Configuration
  public class SessionConfig { }
  
  → 이후 세션은 자동으로 Redis에 저장/조회
  → 어느 서버로 라우팅되어도 같은 세션

JWT 기반 Stateless (Spring Security):
  @Configuration
  public class SecurityConfig {
      @Bean
      public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
          http
              .sessionManagement(s -> s
                  .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
              .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
          return http.build();
      }
  }
  
  // JWT 필터
  @Component
  public class JwtFilter extends OncePerRequestFilter {
      @Override
      protected void doFilterInternal(HttpServletRequest request, ...) {
          String token = extractTokenFromCookie(request);
          if (token != null && jwtUtil.validateToken(token)) {
              Claims claims = jwtUtil.getClaims(token);
              // SecurityContext에 인증 정보 설정
              SecurityContextHolder.getContext()
                  .setAuthentication(createAuth(claims));
          }
          filterChain.doFilter(request, response);
      }
  }
```

---

## 🔬 내부 동작 원리

### 1. Sticky Session의 동작과 한계

```
IP Hash 방식:
  Hash(소스 IP) mod 서버 수 = 서버 인덱스
  
  서버 3대 (A, B, C):
  192.168.1.1 → hash → A
  192.168.1.2 → hash → B
  192.168.1.3 → hash → A
  ...
  
  문제 1 — NAT:
  기업 NAT: 직원 500명 → 공인 IP 1개
  → 500명 모두 서버 A → 불균등 분산
  
  문제 2 — 서버 추가/제거:
  서버 3대 → 4대 추가:
  Hash mod 3 → Hash mod 4 (결과 완전히 달라짐)
  → 대부분 사용자가 다른 서버로 이동 → 세션 소실
  
  Consistent Hashing으로 개선 가능하지만 여전히 한계 존재

Cookie 기반 Session Affinity:
  LB가 쿠키를 통해 특정 서버로 라우팅
  
  첫 요청:
  클라이언트 → LB → 서버 A (새 서버 선택)
  LB → 클라이언트: Set-Cookie: SERVERID=server-a; Path=/; HttpOnly
  
  이후 요청:
  클라이언트 → LB: Cookie: SERVERID=server-a
  LB: "server-a로 가야 함" → 서버 A로 전달
  
  AWS ALB: AWSALB 쿠키 사용
  Nginx: upstream_hash 또는 ip_hash
  
  Cookie 방식의 장점:
  NAT 문제 없음 (IP가 아닌 쿠키 기반)
  서버별로 균등 분산 (초기 랜덤 배정)
  
  Cookie 방식의 한계:
  서버 A 장애 → 서버 A 쿠키 사용자 전원 세션 소실
  배포/재시작 시 세션 소실
  수평 확장 시 새 서버로 이동 어려움

Sticky Session이 만드는 문제들:
  ① 부하 불균등:
     서버별 세션 수가 달라짐 → CPU/메모리 불균등
     장시간 세션 많은 사용자 → 특정 서버에 부하 집중
  
  ② 롤링 배포 어려움:
     서버 1 배포 중 → 세션 Drain (기존 세션 만료 대기)
     10분 만료 세션 → 10분 배포 지연
  
  ③ 단일 장애점:
     서버 장애 → 해당 서버 세션 모두 소실
     HA 목적이 퇴색됨
```

### 2. 분산 세션 — Spring Session + Redis

```
Spring Session 동작:
  HTTP 요청 → Spring Session Filter
  → Session ID (쿠키에서 추출)
  → Redis에서 세션 데이터 조회
  → 요청 처리 (세션 데이터 사용)
  → 세션 변경 사항 Redis에 저장
  → 응답

Redis 세션 구조:
  Key: "spring:session:sessions:{SESSION_ID}"
  Value: {
    creationTime: 1711600000000,
    lastAccessedTime: 1711603600000,
    maxInactiveInterval: 1800,
    sessionAttr:
      "userId": "12345",
      "username": "alice",
      "roles": ["USER", "ADMIN"]
  }
  TTL: maxInactiveInterval (자동 만료)

세션 직렬화 설정:
  기본: Java 직렬화 (JDK Serializable) → 호환성 문제
  권장: JSON 직렬화
  
  @Bean
  public RedisSerializer<Object> springSessionDefaultRedisSerializer() {
      return new GenericJackson2JsonRedisSerializer();
  }

Redis HA 설정:
  # Sentinel (자동 Failover)
  spring.data.redis.sentinel.master=mymaster
  spring.data.redis.sentinel.nodes=sentinel1:26379,sentinel2:26379,sentinel3:26379
  
  # Cluster
  spring.data.redis.cluster.nodes=node1:7001,node2:7002,node3:7003

세션 스토어 장애 대비:
  @Bean
  public SessionRepository sessionRepository(RedisTemplate redisTemplate) {
      RedisIndexedSessionRepository repository = 
          new RedisIndexedSessionRepository(redisTemplate);
      
      // Redis 장애 시 예외를 삼키고 새 세션 생성 (선택적)
      repository.setFlushMode(FlushMode.ON_SAVE);
      return repository;
  }
  
  // Circuit Breaker 패턴으로 Redis 장애 격리
  @Autowired
  private CircuitBreaker redisCircuitBreaker;
  
  public Session getSession(String id) {
      return redisCircuitBreaker.executeSupplier(
          () -> redisRepository.findById(id),
          e -> null  // 장애 시 세션 없음 처리
      );
  }
```

### 3. JWT 기반 Stateless 인증

```
JWT(JSON Web Token) 구조:
  Header.Payload.Signature
  
  Header (Base64):
    {"alg": "HS256", "typ": "JWT"}
  
  Payload (Base64, 평문!):
    {
      "sub": "12345",         // 사용자 ID
      "username": "alice",
      "roles": ["USER"],
      "iat": 1711600000,      // 발급 시각
      "exp": 1711603600       // 만료 시각 (1시간 후)
    }
  
  Signature:
    HMAC-SHA256(base64(Header) + "." + base64(Payload), SECRET)

JWT 검증 (서버):
  1. Header + Payload + 서버의 SECRET으로 Signature 재계산
  2. 전달받은 Signature와 비교
  3. exp 만료 확인
  → DB 조회 없이 토큰만으로 검증 (Stateless!)

토큰 저장 위치:
  localStorage:
    장점: 편리
    단점: XSS 공격으로 탈취 가능
  
  HttpOnly 쿠키 (권장):
    장점: JS 접근 불가 → XSS 방어
    단점: CSRF 공격 가능 → CSRF Token 함께 사용
  
  Memory (React state):
    장점: 탭 닫으면 자동 소멸
    단점: 새로고침 시 소멸 → 재로그인 필요

Access Token + Refresh Token 패턴:
  Access Token: 짧은 유효기간 (15분~1시간)
                매 요청마다 전달 (Authorization 헤더 또는 쿠키)
  
  Refresh Token: 긴 유효기간 (7일~30일)
                 HttpOnly 쿠키에만 저장 (서버로만 전달)
                 DB에 저장 (폐기 가능하도록)
  
  흐름:
  Access Token 만료 → Refresh Token으로 새 Access Token 요청
  Refresh Token 만료 또는 폐기 → 재로그인
  
  로그아웃:
  Refresh Token 삭제 (DB에서 폐기)
  클라이언트 측 Access Token 삭제 (메모리 또는 쿠키)
  → Access Token은 만료까지 기술적으로 유효하지만
    Refresh Token 폐기로 새 토큰 발급 불가

JWT 취소(Revocation) 문제:
  JWT는 Stateless → 서버가 개별 토큰 추적 불가
  유효기간 내 폐기 불가 (로그아웃, 계정 정지 등)
  
  해결 방법:
  1. 짧은 만료 시간 (15분): 영향 최소화
  2. JWT Blacklist (Redis):
     revoked_tokens:{jti} → "1"  (jti: JWT ID, 고유 식별자)
     → 요청마다 Redis 조회 (Stateless 이점 일부 포기)
  3. 버전 기반 검증:
     DB에 user_token_version 저장
     JWT에 version 포함
     버전 불일치 → 거부 (DB 조회 필요하지만 단순)
```

### 4. 세션 vs JWT 설계 결정

```
서버 세션 (Spring Session + Redis):
  
  흐름:
  로그인 → 서버에 세션 생성 → 세션 ID를 쿠키로 반환
  이후 요청 → 쿠키의 세션 ID → Redis 조회 → 인증
  
  장점:
  ① 즉각적인 세션 무효화 (Redis에서 삭제)
  ② 세션 데이터 서버 제어 (민감 정보 저장 가능)
  ③ 구현 단순 (Spring Security 기본)
  ④ 로그인 상태 추적 용이
  
  단점:
  ① Redis 의존성 → 중앙화 SPOF
  ② 모든 요청마다 Redis 조회 → 네트워크 비용
  ③ 마이크로서비스에서 세션 공유 복잡

JWT (Stateless):
  
  흐름:
  로그인 → JWT 발급 → 클라이언트 저장
  이후 요청 → JWT 전달 → 서버에서 서명 검증 → 인증
  
  장점:
  ① DB/Redis 조회 없음 → 빠른 인증
  ② 서버 상태 없음 → 완전한 수평 확장
  ③ 마이크로서비스 간 토큰 공유 용이
  ④ CORS 환경에 적합 (쿠키 제한 없음)
  
  단점:
  ① 즉각적 토큰 폐기 어려움 (짧은 만료 + Refresh 패턴 필요)
  ② payload 암호화 안 됨 (민감 정보 포함 금지)
  ③ 토큰 크기 → 요청마다 헤더 크기 증가
  ④ SECRET 노출 시 모든 토큰 위조 가능

선택 기준:
  즉각적 세션 무효화 필요 (보안 민감):
    → 서버 세션 (Redis)
  
  완전한 Stateless, 마이크로서비스, API:
    → JWT + Refresh Token
  
  둘 다 필요:
    → JWT (Access) + Redis (Refresh Token 저장 + Blacklist)
```

---

## 💻 실전 실험

### 실험 1: Sticky Session vs Spring Session 비교

```bash
# 서버 2대 (포트 8080, 8081)에서 세션 확인

# Sticky Session (IP Hash - Nginx)
for i in $(seq 1 10); do
  curl -c cookie.txt -b cookie.txt http://nginx/session/get
done
# 항상 같은 서버에서 응답 확인

# Spring Session + Redis (어느 서버에서나 같은 세션)
# 서버 8080에서 로그인
SESSION_ID=$(curl -c cookie.txt http://localhost:8080/login \
  -d "user=alice&pass=pass" -s -I | grep "Set-Cookie" | grep "SESSION=")

# 서버 8081에 같은 쿠키로 요청
curl -b cookie.txt http://localhost:8081/session/current
# alice로 인증됨 (Redis에서 세션 조회)
```

### 실험 2: JWT 동작 확인

```bash
# JWT 발급
TOKEN=$(curl -s -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"pass"}' | jq -r .token)

# JWT 디코딩 (Base64 - 평문!)
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool
# {"sub": "12345", "username": "alice", "exp": 1711603600, ...}

# JWT로 API 호출
curl http://localhost:8080/api/users/me \
  -H "Authorization: Bearer $TOKEN"

# 만료된 JWT로 호출
sleep $((exp_time - current_time + 1))  # 만료 대기
curl http://localhost:8080/api/users/me \
  -H "Authorization: Bearer $TOKEN"
# 401 Unauthorized
```

### 실험 3: Redis 세션 직접 확인

```bash
# Redis에서 세션 확인
redis-cli keys "spring:session:sessions:*"
redis-cli hgetall "spring:session:sessions:{SESSION_ID}"
# creationTime, lastAccessedTime, sessionAttr:userId 등 확인

# 세션 강제 만료 (로그아웃 시뮬레이션)
redis-cli del "spring:session:sessions:{SESSION_ID}"
# 이후 해당 세션 ID로 요청 → 401

# 세션 TTL 확인
redis-cli ttl "spring:session:sessions:{SESSION_ID}"
# 1800 (30분 남음)
```

### 실험 4: 세션 재해 복구 시뮬레이션

```bash
# Spring Session + Redis 서버에서 Redis 다운 시뮬레이션
docker stop redis

# 요청 보내기 (세션 사용)
curl -b "SESSION=abc123" http://localhost:8080/api/profile
# 예상: 500 Internal Server Error 또는 로그아웃

# Circuit Breaker 적용 시 fallback 동작 확인
# application.properties:
# spring.session.redis.flush-mode=on-save
# resilience4j.circuitbreaker.instances.redis.failure-rate-threshold=50

# Redis 복구
docker start redis
# 잠시 후 서비스 정상화 확인
```

---

## 📊 성능/비용 비교

```
세션 관리 방식별 성능:

요청당 인증 처리 시간:
  메모리 세션 (단일 서버): ~0.1ms (로컬 HashMap)
  Spring Session + Redis: ~1~5ms (Redis RTT)
  JWT 검증: ~0.3ms (HMAC 연산)
  JWT + Redis Blacklist: ~1~5ms (Redis 조회)

처리량:
  JWT: Redis 조회 없음 → 가장 높은 처리량
  Redis 세션: Redis가 병목 (Redis Cluster로 확장 가능)
  메모리 세션: 단일 서버 → 수평 확장 불가

Redis 메모리 사용:
  세션당 ~1KB × 100만 세션 = ~1GB
  TTL로 자동 만료 → 메모리 자동 회수

JWT 오버헤드:
  JWT 크기: ~200 bytes (최소) → ~1KB (payload 클 때)
  → 매 요청 헤더에 포함 → 네트워크 트래픽 증가
  → HTTP/2 HPACK으로 반복 헤더 압축 가능
```

---

## ⚖️ 트레이드오프

```
Sticky Session이 여전히 필요한 경우:
  WebSocket: 연결 지속 → 같은 서버 필요
  게임 서버: 실시간 상태가 메모리에
  레거시 레거시 레거시: 변경 불가한 경우
  
  이 경우에도 최선 방법:
  Redis Pub/Sub으로 서버 간 상태 동기화
  또는 서비스 메시(Istio)의 session affinity

JWT 단점 완화:
  짧은 만료 (15분) + Refresh Token (Redis 저장):
    로그아웃 → Refresh Token 폐기 → 즉각 효과
    Access Token은 15분 내 만료 → 영향 최소화
  
  이 패턴이 현재 표준:
    Access: JWT (Stateless, 빠름)
    Refresh: Redis 저장 (제어 가능, 폐기 가능)

Spring Session vs JWT 성능 차이:
  소규모 (초당 100 rps): 차이 없음
  중규모 (초당 1000 rps): JWT가 약간 유리
  대규모 (초당 10000 rps): JWT 또는 Redis Cluster 필요
  → 성능보다는 보안 요구사항이 선택 기준
```

---

## 📌 핵심 정리

```
세션 관리 핵심 요약:

Sticky Session:
  IP Hash: NAT로 인한 쏠림, 서버 추가 시 재해싱
  Cookie: 균등 분산, 서버 장애 시 세션 소실
  공통 한계: 수평 확장 어려움, 배포 시 세션 관리 복잡

Spring Session + Redis:
  세션을 Redis 중앙 저장 → 모든 서버에서 접근
  어느 서버로 라우팅되어도 같은 세션
  Redis HA 필수 (Sentinel/Cluster)
  모든 요청마다 Redis 조회 (~1~5ms)

JWT:
  서버 상태 없음 → 완전한 수평 확장
  서명 검증만으로 인증 (~0.3ms)
  payload는 평문 (민감 정보 금지!)
  즉각 폐기 어려움 → 짧은 만료 + Refresh Token 패턴

Access + Refresh Token 패턴 (현재 표준):
  Access Token: JWT, 짧은 만료 (15분), 요청마다 전달
  Refresh Token: 불투명 토큰, 긴 만료 (30일), Redis 저장
  로그아웃: Refresh Token 폐기 → 새 Access Token 발급 차단

저장 위치:
  HttpOnly 쿠키: XSS 방어, CSRF 주의
  localStorage: CSRF 안전, XSS 취약
  권장: Access Token → 메모리, Refresh Token → HttpOnly 쿠키
```

---

## 🤔 생각해볼 문제

**Q1.** 마이크로서비스 환경에서 서비스 A의 JWT를 서비스 B가 어떻게 검증하는가?

<details>
<summary>해설 보기</summary>

**방법 1 — 공유 Secret (HMAC):**
```java
// 모든 서비스가 같은 SECRET 보유
String SECRET = System.getenv("JWT_SECRET");
Claims claims = Jwts.parser().setSigningKey(SECRET).parseClaimsJws(token).getBody();
```
→ 단점: SECRET 배포 관리 복잡, 서비스가 많아지면 SECRET 유출 위험 증가

**방법 2 — 비대칭 키 (RSA/ECDSA):**
```java
// Auth 서비스: 개인키로 서명
// 다른 서비스: 공개키로 검증만
PublicKey publicKey = loadPublicKeyFromFile("/certs/auth-public.pem");
Claims claims = Jwts.parser().setSigningKey(publicKey).parseClaimsJws(token).getBody();
```
→ 공개키만 배포하면 됨 → 서명(개인키)은 Auth 서비스만 보유

**방법 3 — JWKS Endpoint:**
Auth 서비스가 공개키를 JWKS(JSON Web Key Set)로 제공:
```
GET https://auth.example.com/.well-known/jwks.json
{
  "keys": [{"kty": "RSA", "kid": "key1", "n": "...", "e": "AQAB"}]
}
```
각 서비스가 JWKS에서 공개키 자동 로드 (Spring Security OAuth2 자원 서버 지원):
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://auth.example.com/.well-known/jwks.json
```
→ 키 교체 시 JWKS만 업데이트 → 서비스 재배포 없음

**실무 권장:** JWKS Endpoint 방식 (표준화, 키 교체 용이, Spring Security 기본 지원)

</details>

---

**Q2.** JWT 로그아웃을 구현할 때 클라이언트에서 토큰을 삭제하는 것만으로는 왜 부족한가?

<details>
<summary>해설 보기</summary>

**클라이언트 삭제의 한계:**

클라이언트에서 JWT를 삭제해도 토큰 자체는 여전히 만료 전까지 유효합니다.

**공격 시나리오:**
1. 사용자 로그인 → JWT 발급 (1시간 만료)
2. 30분 후 로그아웃 → 클라이언트에서 JWT 삭제
3. 공격자가 이전에 훔친 JWT로 서버에 요청
4. 서버: JWT 서명 유효, 만료 안 됨 → 인증 성공!

**서버 측 대응:**

1. **짧은 만료 (15분):**
   - 로그아웃 후 최대 15분 내 토큰 무효화
   - 위험 창이 작아짐 (완전 해결 아님)

2. **Redis Blacklist:**
```java
// 로그아웃 시
String jti = claims.getId();  // JWT 고유 ID
redisTemplate.opsForValue().set(
    "blacklist:" + jti,
    "1",
    remainingTtl  // 남은 만료 시간만큼 유지
);

// 요청 처리 시
if (redisTemplate.hasKey("blacklist:" + jti)) {
    throw new TokenRevokedException();
}
```
→ 로그아웃 즉시 효과, Redis 조회 비용 추가

3. **Refresh Token 폐기:**
   - Access Token은 짧은 만료로 관리
   - Refresh Token만 DB에서 폐기 → 새 Access Token 발급 불가
   - Access Token 최대 15분간 유효 (허용 가능한 리스크)

**실무 결론:** 보안 민감도에 따라 선택:
- 일반 서비스: 짧은 만료 + Refresh Token 폐기
- 금융/의료: Redis Blacklist (로그아웃 즉시 무효화)

</details>

---

**Q3.** 세션 Draining이란 무엇이고 Rolling 배포에서 왜 필요한가?

<details>
<summary>해설 보기</summary>

**세션 Draining (Session Drain):**
배포를 위해 서버를 제외할 때, 기존 활성 세션/연결이 완료될 때까지 기다리는 것.

**Rolling 배포 시나리오:**

```
서버 A, B, C 실행 중 (Sticky Session 사용)
서버 A에 새 버전 배포:

1단계: LB에서 서버 A를 "Draining 모드"로 전환
  → 새 요청: B, C로만 라우팅
  → 기존 서버 A 연결: 완료될 때까지 유지

2단계: 서버 A의 기존 연결/세션이 자연스럽게 종료될 때까지 대기
  → Sticky Session 세션이 만료되거나 클라이언트가 닫을 때까지
  → 최대 Draining 시간 설정 (예: 300초)

3단계: 서버 A 연결 모두 종료 → 서버 A 재시작 (배포)

4단계: 서버 A가 새 버전으로 LB에 재등록

5단계: 서버 B를 Draining 모드로... (반복)
```

**AWS ALB Draining:**
```
# Target Group에서 Deregistration Delay 설정
aws elbv2 modify-target-group-attributes \
  --target-group-arn $ARN \
  --attributes Key=deregistration_delay.timeout_seconds,Value=300
```

**Spring Session + Redis 사용 시 Draining 불필요:**
- 어느 서버가 재시작해도 Redis에 세션이 유지됨
- 서버 A 재시작 → 클라이언트 다음 요청이 서버 B로 → 같은 세션 사용
- Draining 시간 대폭 단축 가능 (0으로 설정해도 됨)

**결론:** Stateless 세션 관리(Spring Session + Redis 또는 JWT)가 배포 프로세스도 단순화합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Nginx 리버스 프록시](./02-reverse-proxy-nginx.md)** | **[홈으로 🏠](../README.md)** | **[다음: Rate Limiting ➡️](./04-rate-limiting-algorithms.md)**

</div>
