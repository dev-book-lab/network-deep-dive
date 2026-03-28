# 리버스 프록시 — Nginx 내부 동작과 upstream 연결 관리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Nginx가 클라이언트 연결과 upstream 연결을 어떻게 분리해서 관리하는가?
- `keepalive` 지시어로 upstream TCP 연결을 어떻게 재사용하는가?
- 버퍼링(Buffering)이 없을 때 느린 클라이언트가 upstream 서버에 미치는 영향은?
- `upstream_response_time`과 `request_time`의 차이는 무엇인가?
- Nginx worker 프로세스와 이벤트 루프가 대용량 연결을 처리하는 원리는?
- Nginx 설정에서 성능을 크게 좌우하는 핵심 파라미터는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

```
"Nginx → Tomcat 구성에서 503이 자주 발생한다":
  원인 후보:
  1. Upstream Tomcat이 실제로 다운 → 502 Bad Gateway
  2. Upstream 연결 타임아웃 초과 → 504 Gateway Timeout
  3. Nginx upstream keepalive 설정 없음 → 매 요청마다 새 TCP 연결
     Tomcat의 동시 연결 수 제한 초과 → 503
  
  해결: upstream keepalive 32; 추가
  → Nginx가 연결 풀에서 재사용 → Tomcat 부하 감소

"access_log에서 request_time이 너무 길다":
  request_time: 클라이언트가 요청 전송 완료 ~ 응답 전달 완료
  upstream_response_time: Nginx가 upstream에 요청 ~ 응답 수신 완료
  
  request_time >> upstream_response_time:
  → upstream은 빨리 응답했는데 클라이언트에게 전달이 느림
  → 클라이언트가 느린 네트워크 (Nginx 버퍼링 동작)
  
  upstream_response_time이 큼:
  → 실제 애플리케이션 처리가 느림 → 코드/DB 최적화 필요

"Nginx가 1만 개 동시 연결을 처리하는 방법":
  Apache (구식): 연결당 스레드/프로세스 → 1만 스레드 = 불가능
  Nginx: 이벤트 루프 + 비동기 I/O → 수만 연결을 소수의 Worker로 처리
  → C10K 문제의 해결책
```

---

## 😱 흔한 실수

```
Before — Nginx 내부를 모를 때:

실수 1: upstream keepalive 없이 고트래픽 서비스
  기본 설정: 매 요청마다 새 TCP 연결
  초당 1000 요청 → 초당 1000번 TCP+TLS 핸드쉐이크
  → Tomcat 연결 오버헤드 → 지연 증가
  
  해결:
  upstream backend {
      server tomcat:8080;
      keepalive 32;  ← 최대 32개 유휴 연결 유지
  }
  
  설정 추가 필요:
  proxy_http_version 1.1;
  proxy_set_header Connection "";  # Connection: keep-alive 강제

실수 2: 버퍼링 비활성화로 모든 문제 해결 시도
  proxy_buffering off;  # 응답 즉시 전달
  → 느린 클라이언트 = upstream 연결 장시간 점유
  → Tomcat 스레드 풀 고갈 → 503
  → 버퍼링은 특정 경우만 비활성화 (SSE, 실시간 스트리밍)

실수 3: worker_processes를 CPU 수보다 많이 설정
  worker_processes 100;  # 32코어 서버에서
  → 컨텍스트 스위칭 오버헤드 증가
  → 올바른 설정: worker_processes auto; (CPU 수만큼 자동)

실수 4: proxy_pass에 슬래시 차이 무시
  proxy_pass http://backend;       # /api/users → /api/users
  proxy_pass http://backend/;      # /api/users → /
  → 슬래시 하나가 URL 매핑을 완전히 바꿈
```

---

## ✨ 올바른 접근

```
After — Nginx 내부를 알면:

성능 최적화된 Nginx 설정:
  worker_processes auto;         # CPU 수만큼
  worker_connections 10000;      # 워커당 최대 연결 수
  use epoll;                     # Linux 이벤트 모델
  multi_accept on;               # 여러 연결 한번에 수락

  upstream backend {
      server tomcat-1:8080;
      server tomcat-2:8080;
      keepalive 32;              # 유휴 연결 풀
      keepalive_requests 1000;   # 연결당 최대 요청 수
      keepalive_timeout 60s;     # 유휴 연결 유지 시간
  }

  server {
      # 버퍼 설정 (느린 클라이언트 보호)
      proxy_buffering on;
      proxy_buffer_size 4k;
      proxy_buffers 8 8k;
      proxy_busy_buffers_size 16k;
      
      # 타임아웃
      proxy_connect_timeout 5s;   # upstream 연결 시도 제한
      proxy_send_timeout 60s;     # upstream으로 요청 전송 제한
      proxy_read_timeout 60s;     # upstream 응답 대기 제한
      
      # Keep-Alive (keepalive 지시어 사용 시 필수)
      proxy_http_version 1.1;
      proxy_set_header Connection "";
      
      location /api/ {
          proxy_pass http://backend;
      }
  }

유용한 access_log 포맷:
  log_format detailed '$remote_addr - $request '
      'status=$status bytes=$body_bytes_sent '
      'req_time=$request_time '
      'upstream_time=$upstream_response_time '
      'upstream_addr=$upstream_addr';
  
  # upstream_response_time이 크면 → 앱 최적화
  # request_time - upstream_response_time이 크면 → 클라이언트 네트워크
```

---

## 🔬 내부 동작 원리

### 1. Nginx 이벤트 기반 아키텍처

```
Apache 전통 모델 (연결당 스레드):
  클라이언트 1 → 스레드 1 (블로킹 대기)
  클라이언트 2 → 스레드 2 (블로킹 대기)
  클라이언트 1000 → 스레드 1000 (메모리 부족!)
  
  각 스레드: 수 MB 메모리
  1000 스레드 = 수 GB → 한계 명확

Nginx 이벤트 루프 모델 (epoll):
  Master Process
  ├── Worker Process 1 (이벤트 루프)
  ├── Worker Process 2 (이벤트 루프)
  └── Worker Process N (CPU 수만큼)
  
  각 Worker Process:
    epoll에 수천 개의 소켓 등록
    
    while(true) {
        events = epoll_wait(all_sockets)  // 이벤트 대기 (블로킹)
        for each event:
            if event.type == READ:
                data = read(socket)       // 비동기 읽기
                process(data)
            if event.type == WRITE:
                write(socket, response)   // 비동기 쓰기
    }
    
    → 단일 스레드가 수만 개의 연결 처리
    → 스레드가 블로킹되지 않음 (이벤트 기반)
    → 메모리 효율: 연결당 ~4KB vs 스레드당 수MB

C10K 문제 해결:
  초당 10,000 동시 연결 처리
  epoll: O(1) 이벤트 감지 (select는 O(n) → 비효율)
  sendfile(): 커널에서 직접 파일 → 소켓 복사 (사용자 공간 거치지 않음)
  → Nginx: 적은 스레드로 수만 연결 처리
```

### 2. 클라이언트 연결 vs Upstream 연결 분리

```
Nginx가 두 연결을 분리하는 방식:

클라이언트 연결 (외부):
  ┌─────────────────────────────────────────────────────┐
  │  Client 1 ──HTTP/1.1──► Nginx worker                │
  │  Client 2 ──HTTPS────► Nginx worker                 │
  │  Client 3 ──HTTP/2───► Nginx worker (멀티플렉싱)       │
  │  ...                                                │
  │  수천 개의 클라이언트 연결 (이벤트 루프로 관리)                │
  └─────────────────────────────────────────────────────┘

Upstream 연결 (내부):
  ┌─────────────────────────────────────────────────────┐
  │  Nginx worker ──HTTP/1.1──► Tomcat A:8080           │
  │  Nginx worker ──HTTP/1.1──► Tomcat B:8080           │
  │  ...                                                │
  │  keepalive 32개 유휴 연결 풀 유지                        │
  └─────────────────────────────────────────────────────┘

요청 흐름:
  1. 클라이언트 연결 수신 (이벤트 감지)
  2. HTTP 요청 파싱
  3. upstream 연결 풀에서 유휴 연결 꺼냄 (없으면 새 연결 생성)
  4. upstream에 요청 전송
  5. upstream 응답 수신 (이벤트 감지)
  6. 응답 버퍼링 (클라이언트로 전달)
  7. upstream 연결 풀에 반환 (keepalive)

분리의 이점:
  클라이언트가 100Mbps, upstream이 1Gbps:
  → Nginx가 upstream 응답을 버퍼에 받음
  → upstream 연결 해제 (빠르게 처리 완료)
  → 버퍼에서 클라이언트로 천천히 전달
  → upstream 스레드 점유 시간 최소화
```

### 3. upstream keepalive — 연결 풀

```
keepalive 없는 경우:
  요청 1: TCP SYN → SYN-ACK → ACK (3 RTT)
           HTTP 요청 → HTTP 응답
           TCP FIN → FIN-ACK (4 RTT 종료)
  요청 2: TCP SYN → SYN-ACK → ACK (또 3 RTT!)
  ...
  → 초당 1000 요청 = 초당 1000번 TCP 핸드쉐이크

keepalive 32 설정:
  요청 1: TCP 연결 수립 → HTTP 요청 → 응답 → 연결 유지(풀 반환)
  요청 2: 풀에서 연결 꺼냄 (이미 연결됨!) → HTTP 요청 → 응답
  요청 3: 풀에서 연결 꺼냄 → ...
  → 연결 오버헤드 제거

keepalive 설정:
  upstream backend {
      server 10.0.1.10:8080;
      server 10.0.1.11:8080;
      
      keepalive 32;           # 풀의 최대 유휴 연결 수
      keepalive_requests 100; # 연결당 최대 처리 요청 수 (이후 재생성)
      keepalive_timeout 60s;  # 유휴 연결 유지 시간 (이후 종료)
  }
  
  server {
      location / {
          proxy_pass http://backend;
          proxy_http_version 1.1;  # HTTP/1.1 필수 (Keep-Alive 지원)
          proxy_set_header Connection "";  # "close" 헤더 제거 (연결 유지)
      }
  }

keepalive 파라미터 선택:
  keepalive N: 활성 Worker 수 × 단위 시간 동시 업스트림 요청 고려
  서버 수 2개, 요청 집중: keepalive 16~32 (각 서버당 8~16 연결)
  서버 수 10개: keepalive 64~128
  
  너무 크면: upstream 서버의 FD(파일 디스크립터) 고갈
  너무 작으면: 유휴 연결 부족 → 새 연결 생성 빈번
```

### 4. 버퍼링 — 느린 클라이언트로부터 upstream 보호

```
버퍼링 없는 경우 (proxy_buffering off):
  
  Nginx: upstream 응답을 클라이언트로 즉시 전달
  
  빠른 클라이언트: 정상 동작
  느린 클라이언트 (2G 모바일):
    upstream 응답 시작 → 클라이언트로 조금씩 전달
    upstream: 클라이언트가 받을 때까지 연결 유지
    → Tomcat 스레드: 응답 전달 완료까지 점유!
    → 느린 클라이언트 100명 = Tomcat 스레드 100개 점유
    → 실제 처리보다 클라이언트 네트워크가 병목

버퍼링 있는 경우 (기본값):

  Nginx: upstream 응답을 버퍼에 모두 받음
  → upstream 연결 즉시 해제
  → 버퍼에서 클라이언트로 전달 (클라이언트 속도에 맞춰)
  → Tomcat 스레드: 빠르게 응답 후 다음 요청 처리
  
  버퍼 크기 설정:
    proxy_buffer_size 4k;     # 응답 헤더 버퍼
    proxy_buffers 8 8k;       # 응답 바디 버퍼 (8개 × 8KB = 64KB)
    proxy_busy_buffers_size 16k; # 클라이언트 전달 중 최대 버퍼
  
  응답이 버퍼 초과 시:
    임시 파일에 저장 → 클라이언트에게 전달
    proxy_max_temp_file_size 1024m;

버퍼링 비활성화가 필요한 경우:
  SSE (Server-Sent Events):
    → 실시간 이벤트 스트리밍 → 버퍼에 다 모을 수 없음
    proxy_buffering off;
    proxy_cache off;
    proxy_set_header X-Accel-Buffering no;
  
  대용량 파일 다운로드:
    → 버퍼 초과 → 임시 파일 사용 → 디스크 I/O
    → 버퍼링 off가 오히려 효율적일 수 있음
  
  WebSocket:
    → HTTP Upgrade 후 양방향 스트리밍
    proxy_buffering off;  # 실시간성 보장
```

### 5. access_log 분석 — request_time vs upstream_response_time

```
Nginx access_log 핵심 변수:
  $request_time:
    클라이언트에서 첫 바이트 수신 ~ 응답 마지막 바이트 전송까지
    = 전체 요청 처리 시간 (클라이언트 관점)
  
  $upstream_response_time:
    Nginx가 upstream에 연결 ~ 응답 수신 완료까지
    = 실제 앱 처리 시간 + upstream 네트워크 시간
  
  $upstream_connect_time:
    Nginx → upstream TCP 연결 수립 시간
    keepalive 사용 시 0 (이미 연결됨)
  
  $upstream_header_time:
    upstream이 응답 헤더 전송까지의 시간
    TTFB(Time To First Byte) 확인에 유용

진단 패턴:
  Case 1: request_time ≈ upstream_response_time
    → Nginx 오버헤드 없음, upstream 처리 시간이 대부분
    → 애플리케이션 최적화 필요
  
  Case 2: request_time >> upstream_response_time
    → upstream은 빠른데 전달이 느림
    → 클라이언트 네트워크 느림 (버퍼링으로 처리 중)
    → 정상 동작 (Nginx 버퍼링 효과)
  
  Case 3: upstream_connect_time이 큼
    → upstream TCP 연결 수립이 느림
    → keepalive 설정 확인, upstream 서버 부하 확인
  
  Case 4: upstream_response_time이 큰데 upstream_header_time이 작음
    → 응답 헤더는 빨리 왔는데 바디 전송이 느림
    → upstream 서버의 응답 스트리밍 문제

로그 포맷 설정:
  log_format main
    '$remote_addr [$time_local] "$request" '
    '$status $body_bytes_sent '
    'rt=$request_time '
    'uct=$upstream_connect_time '
    'uht=$upstream_header_time '
    'urt=$upstream_response_time '
    'ua=$upstream_addr';

로그 분석 (평균 upstream 응답 시간):
  awk '{print $NF}' /var/log/nginx/access.log | \
    grep "urt=" | cut -d= -f2 | \
    awk '{sum+=$1; n++} END {print "Avg:", sum/n}'
```

---

## 💻 실전 실험

### 실험 1: upstream keepalive 효과 측정

```bash
# keepalive 없는 Nginx 설정
cat > /tmp/nginx-no-keepalive.conf << 'EOF'
upstream backend { server localhost:8080; }
server {
    listen 8888;
    location / { proxy_pass http://backend; }
}
EOF

# keepalive 있는 Nginx 설정
cat > /tmp/nginx-keepalive.conf << 'EOF'
upstream backend {
    server localhost:8080;
    keepalive 16;
}
server {
    listen 8889;
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
EOF

# 부하 테스트로 비교
ab -n 1000 -c 100 http://localhost:8888/api/  # keepalive 없음
ab -n 1000 -c 100 http://localhost:8889/api/  # keepalive 있음
# → keepalive 있을 때 rps 높고 지연 낮음 확인

# ss로 연결 상태 확인
ss -s | grep ESTABLISHED
watch -n 1 'ss -s'
```

### 실험 2: 버퍼링 효과 관찰

```bash
# 느린 응답 시뮬레이션 (Python 서버)
python3 -c "
from http.server import HTTPServer, BaseHTTPRequestHandler
import time

class SlowHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-Length', '10000')
        self.end_headers()
        for i in range(100):
            self.wfile.write(b'x' * 100)
            self.wfile.flush()
            time.sleep(0.1)  # 청크당 0.1초 지연
    def log_message(self, *args): pass

HTTPServer(('127.0.0.1', 8080), SlowHandler).serve_forever()
" &

# upstream_response_time과 request_time 비교
curl -v -w "request_time=%{time_total}\n" \
     http://localhost/slow-endpoint 2>&1 | tail -3

# Nginx 로그에서 시간 차이 확인
tail -f /var/log/nginx/access.log
```

### 실험 3: Nginx 상태 모니터링

```bash
# ngx_http_stub_status_module 활성화
cat >> /etc/nginx/conf.d/status.conf << 'EOF'
server {
    listen 8080;
    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
EOF

nginx -s reload

curl http://127.0.0.1:8080/nginx_status
# Active connections: 291
# server accepts handled requests
#  16630948 16630948 31070465
# Reading: 6 Writing: 179 Waiting: 106

# 해석:
# Active: 현재 활성 연결 수
# Reading: 요청 헤더 읽는 중
# Writing: 응답 전송 중
# Waiting: Keep-Alive 유휴 연결
```

### 실험 4: proxy_pass 슬래시 차이 확인

```bash
# 슬래시 없는 경우
# proxy_pass http://backend;
# /api/users → 백엔드로 /api/users 전달

# 슬래시 있는 경우
# location /api/ { proxy_pass http://backend/; }
# /api/users → 백엔드로 /users 전달 (/api/ 제거)

# 테스트
nginx -t -c /tmp/test-nginx.conf
curl -v http://localhost/api/users 2>&1 | grep "< HTTP"
# 백엔드 로그에서 받은 경로 확인
```

---

## 📊 성능/비용 비교

```
keepalive 설정별 성능:

초당 1000 요청, upstream 서버 2대:

keepalive 없음:
  TCP 핸드쉐이크: 1000회/초 (각 0.5ms = 500ms/s 낭비)
  upstream 연결 FD: 순간 최대 1000개
  Tomcat 연결 처리 부하: 높음

keepalive 32:
  새 TCP 연결: 드물게 (풀 고갈 시만)
  upstream FD: 최대 32개 유지
  → CPU, 메모리, 지연 모두 개선

버퍼 크기별 영향:
  버퍼 너무 작음 (1KB):
    응답이 자주 버퍼 초과 → 임시 파일 → 디스크 I/O → 느림
  
  버퍼 적당함 (8~64KB):
    대부분의 API 응답 버퍼 내에서 처리
    → 디스크 I/O 없음 → 빠름
  
  버퍼 너무 큼 (1MB):
    메모리 낭비
    연결당 1MB × 10,000 연결 = 10GB 메모리

worker 설정 영향:
  worker_processes 1 (4코어 서버):
    단일 CPU 사용 → 75% CPU 낭비
  
  worker_processes 4 (= auto):
    4개 CPU 모두 사용 → 4배 처리량
  
  worker_connections 1024 (기본):
    최대 동시 연결 4096개 (4 × 1024)
  
  worker_connections 10000:
    최대 동시 연결 40000개 (파일 디스크립터 한계 확인 필요)
```

---

## ⚖️ 트레이드오프

```
keepalive 크기:

keepalive 낮음 (8):
  연결 풀 작음 → 자주 새 연결 생성
  upstream 서버의 FD 절약
  burst 트래픽 시 연결 부족 가능

keepalive 높음 (128):
  연결 풀 충분 → 새 연결 드뭄
  upstream FD 많이 사용
  유휴 연결 유지 비용

적절한 값: 동시 요청 수 / upstream 서버 수 × 1.5배

버퍼링 on vs off:
  on:  느린 클라이언트 보호, upstream 빠른 해제, 디스크 I/O 가능
  off: 실시간 스트리밍에 필요, upstream 연결 장시간 점유

proxy_read_timeout 설정:
  짧음 (10초): 느린 응답 빠르게 504로 처리 → 사용자에게 빠른 피드백
  길음 (300초): 긴 작업(파일 업로드, 보고서 생성) 허용
  → API 유형별로 다르게 설정
  
  location /api/quick/ {
      proxy_read_timeout 10s;
  }
  location /api/report/ {
      proxy_read_timeout 300s;
  }
```

---

## 📌 핵심 정리

```
Nginx 리버스 프록시 핵심 요약:

이벤트 기반 아키텍처:
  Worker 프로세스 × (CPU 수) + epoll
  → 단일 Worker가 수만 연결을 비동기로 처리
  → Apache 스레드 모델 대비 메모리 효율 극대화

두 연결 분리:
  클라이언트 연결 ≠ Upstream 연결
  → 느린 클라이언트가 upstream 스레드 점유 방지
  → 버퍼링으로 upstream 빠르게 해제

upstream keepalive:
  연결 풀 유지 → TCP 핸드쉐이크 비용 제거
  필수 설정: proxy_http_version 1.1; proxy_set_header Connection "";
  크기: 동시요청수 / 서버수 × 1.5배

버퍼링:
  기본 활성화 (느린 클라이언트 보호)
  비활성화: SSE, WebSocket, 대용량 스트리밍에서만
  크기: API 응답 크기에 맞게 (4~64KB)

로그 분석:
  upstream_response_time: 앱 처리 시간 지표
  request_time - upstream_response_time: 네트워크/버퍼 시간
  upstream_connect_time = 0: keepalive 정상 동작

핵심 설정:
  worker_processes auto;
  worker_connections 10000;
  keepalive 32 (upstream에)
  버퍼 크기 응답 크기에 맞게
  타임아웃 API 특성에 맞게
```

---

## 🤔 생각해볼 문제

**Q1.** Nginx 로그에서 `upstream_response_time`이 0.001초인데 `request_time`이 30초라면 무슨 일이 일어나고 있는가?

<details>
<summary>해설 보기</summary>

**상황:** upstream은 1ms에 응답했는데 전체 요청 처리에 30초가 걸림.

**가능한 원인:**

1. **느린 클라이언트 (가장 유력):**
   - 클라이언트가 2G 또는 매우 느린 네트워크
   - Nginx가 upstream 응답(예: 1MB 파일)을 버퍼에 받음 → upstream 연결 즉시 해제
   - 버퍼에서 클라이언트로 1MB를 30초에 걸쳐 천천히 전달
   - `request_time = 30s`, `upstream_response_time = 0.001s`
   - 이것은 **정상 동작** (Nginx 버퍼링이 upstream 보호하는 것)

2. **클라이언트 요청 전송 지연:**
   - POST 요청에서 대용량 파일 업로드 중
   - `request_time`은 요청 수신 시작부터 측정
   - 클라이언트가 30초에 걸쳐 업로드 → 그제서야 upstream으로 전달 → 빠른 응답
   - 해결: 대용량 업로드는 Nginx에서 직접 S3 등으로 처리

3. **Nginx 자체 처리 지연 (드묾):**
   - SSL 처리, Lua 스크립트 등이 blocking인 경우
   - 점검: `nginx -V`로 모듈 확인

**진단:**
```bash
# 동시에 다른 클라이언트는 빠른지 확인
# 빠른 클라이언트: request_time ≈ upstream_response_time → 정상
# 모든 클라이언트가 느림 → upstream 문제

# 클라이언트 IP별 request_time 분포 확인
awk '{print $1, $NF}' /var/log/nginx/access.log | sort -k2 -n | tail -20
```

</details>

---

**Q2.** Nginx의 `keepalive_requests` 파라미터는 왜 필요한가? 무한대로 설정하면 안 되는 이유는?

<details>
<summary>해설 보기</summary>

**`keepalive_requests N`의 의미:**
하나의 keepalive 연결에서 최대 N개의 요청을 처리한 후 연결을 재생성합니다.

**왜 필요한가:**

1. **메모리 누수 방지:**
   - 오래 사용된 TCP 연결은 커널 버퍼, 앱 레벨 상태가 쌓일 수 있음
   - 주기적으로 연결을 갱신하면 메모리 정리됨

2. **로드밸런싱 재분산:**
   - keepalive 연결이 특정 upstream 서버에 영구적으로 묶이면
   - 새 서버 추가 시 기존 연결들은 여전히 이전 서버로 감
   - `keepalive_requests 1000`: 1000 요청 후 새 연결 → 새 서버로 이동 가능

3. **upstream 서버 교체:**
   - 배포 시 서버 IP가 바뀌면 오래된 연결은 이전 서버를 계속 사용
   - 주기적 재생성으로 새 서버 연결 보장

4. **성능 vs 안정성 균형:**
   - 너무 낮음 (10): 자주 재생성 → 핸드쉐이크 비용 증가
   - 너무 높음 (무한): 위의 문제들 발생
   - 권장값: 100~1000 (트래픽 패턴에 따라)

**실무 예시:**
```nginx
upstream backend {
    server 10.0.1.10:8080;
    keepalive 32;
    keepalive_requests 1000;  # 1000 요청 후 재생성
    keepalive_timeout 60s;    # 60초 유휴 시 종료
}
```

초당 100 요청 시: 평균 10초마다 각 연결 재생성 → 적절한 균형

</details>

---

**Q3.** Nginx `worker_connections 10000` 설정에서 실제 최대 동시 클라이언트 수는 얼마인가?

<details>
<summary>해설 보기</summary>

**단순 계산:** `worker_processes × worker_connections`
= 4 × 10000 = 40,000

**하지만 실제로는 절반이 상한:**

Nginx가 각 클라이언트 요청을 처리하려면:
- 클라이언트 연결: 1개 FD
- upstream 연결: 1개 FD

→ 클라이언트 1개당 FD 2개 필요

**따라서:**
```
최대 실제 클라이언트 수 = (worker_processes × worker_connections) / 2
= (4 × 10000) / 2 = 20,000
```

**OS FD 한계도 확인 필요:**
```bash
# 현재 FD 한계
ulimit -n  # 기본 1024 → 너무 낮음

# 증가
ulimit -n 65536
# 또는 /etc/security/limits.conf
nginx soft nofile 65536
nginx hard nofile 65536

# Nginx 설정에도 반영
worker_rlimit_nofile 65536;
events {
    worker_connections 10000;
}
```

**실제 계산 예:**
- `worker_processes 4`
- `worker_connections 16384`
- OS `ulimit -n 65536`
- 최대 클라이언트: 4 × 16384 / 2 = 32,768

이것이 `worker_connections`를 늘릴 때 OS FD 한계도 함께 높여야 하는 이유입니다.

</details>

---

<div align="center">

**[⬅️ 이전: L4 vs L7 LB](./01-l4-vs-l7-load-balancer.md)** | **[홈으로 🏠](../README.md)** | **[다음: Sticky Session vs Stateless ➡️](./03-sticky-session-vs-stateless.md)**

</div>
