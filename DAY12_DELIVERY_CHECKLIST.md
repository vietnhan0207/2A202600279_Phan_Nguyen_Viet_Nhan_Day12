#  Delivery Checklist — Day 12 Lab Submission

> **Student Name:** _________________________  
> **Student ID:** _________________________  
> **Date:** _________________________

---

##  Submission Requirements

Submit a **GitHub repository** containing:

### 1. Mission Answers (40 points)

Create a file `MISSION_ANSWERS.md` with your answers to all exercises:

```markdown
# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found
1. API key hardcode trong code
2. Không có config management
3. Print thay vì proper logging
4. Không có health check endpoint
5. Port cố định — không đọc từ environment

### Exercise 1.3: Comparison table
| Feature | Develop | Production | Why Important? |
|---------|---------|------------|----------------|
| Config | Hardcode trực tiếp trong code (`OPENAI_API_KEY = "sk-..."`, port=8000) | Đọc từ environment variables qua `settings` (config.py) | Nếu hardcode, push lên GitHub là lộ secret ngay; env vars cho phép thay đổi config mà không cần sửa code |
| Health check | Không có endpoint `/health` hay `/ready` | Có `/health` (liveness) và `/ready` (readiness) trả về JSON status | Cloud platform (Railway, Render) cần health check để biết khi nào restart container; load balancer cần readiness để không route traffic vào instance chưa sẵn sàng |
| Logging | `print()` thô, kể cả in ra secret (`print(f"Using key: {OPENAI_API_KEY}")`) | Structured JSON logging qua `logging` module, không bao giờ log secret | JSON logs dễ parse bởi log aggregator (Datadog, Loki); print() mất khi container restart và có thể lộ thông tin nhạy cảm |
| Shutdown | Không xử lý — tắt đột ngột, request đang chạy bị mất | Graceful shutdown với SIGTERM handler + lifespan context manager, chờ request hiện tại hoàn thành | Nếu tắt đột ngột, user đang chờ response sẽ nhận lỗi; graceful shutdown đảm bảo hoàn thành in-flight requests trước khi exit |
| Host binding | `host="localhost"` — chỉ nhận kết nối từ chính máy đó | `host="0.0.0.0"` — nhận kết nối từ bên ngoài container | Trong Docker/cloud, nếu bind localhost thì không thể truy cập từ bên ngoài container — app deploy xong nhưng không ai gọi được |

## Part 2: Docker

### Exercise 2.1: Dockerfile questions
1. Base image: `python:3.11` — full Python distribution (~1 GB), chứa toàn bộ compiler và tools
2. Working directory: `/app` — tất cả lệnh tiếp theo (COPY, RUN, CMD) đều chạy trong thư mục này bên trong container
3. Tại sao COPY requirements.txt trước: Docker build theo từng layer và cache lại. Nếu `requirements.txt` không đổi, Docker dùng cached layer, bỏ qua bước `pip install` tốn thời gian. Chỉ khi file này thay đổi thì mới cài lại dependencies. Nếu COPY code trước, mỗi lần sửa code đều trigger pip install lại — rất chậm.
4. CMD vs ENTRYPOINT: `CMD` là lệnh mặc định nhưng **có thể bị override** khi chạy `docker run image <lệnh_khác>`. `ENTRYPOINT` là lệnh cố định, **không bị override** dễ dàng — arguments từ `docker run` sẽ được append vào ENTRYPOINT thay vì thay thế. Dùng ENTRYPOINT khi muốn container luôn chạy đúng một chương trình nhất định.

### Exercise 2.3: Image size comparison
- Develop: [1.66] GB
- Production: [236] MB
- Difference: [85.79]%

## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment
- URL: https://your-app.railway.app
- Screenshot: 
  ![alt text](2A202600279_Phan_Nguyen_Viet_Nhan_Day12/Railway-Server_Image.png) check the image in the folder 2A202600279_Phan_Nguyen_Viet_Nhan_Day12 called Railway-Server_Image.png

## Part 4: API Security

### Exercise 4.1-4.3: Test results
Cho 4.1: 
![alt text](2A202600279_Phan_Nguyen_Viet_Nhan_Day12/Railway-API_Key-Response-To-Test.png)
![alt text](2A202600279_Phan_Nguyen_Viet_Nhan_Day12/Railway-Server-Response-To-API-Key-Test.png)

Cho 4.2: 

```bash
phana@VietNhanLaptop MINGW64 /c/GET_A_JOB/VIN_AI/Lab/Day12/2A202600279_Phan_Nguyen_Viet_Nhan_Day12/04-api-gateway/production (main)
$ curl -X POST "http://localhost:8000/auth/token" -H "Content-Type: application/json" -d "{\"username\":\"student\",\"password\":\"demo123\"}"
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3NzY0MzU1MjgsImV4cCI6MTc3NjQzOTEyOH0.xT4vQbrG1ky5uefVFUbVKs0ZFbOiDccBIHGBvme9nU4","token_type":"bearer","expires_in_minutes":60,"hint":"Include in header: Authorization: Bearer eyJhbGciOiJIUzI1NiIs..."}

phana@VietNhanLaptop MINGW64 /c/GET_A_JOB/VIN_AI/Lab/Day12/2A202600279_Phan_Nguyen_Viet_Nhan_Day12/04-api-gateway/production (main)
$  curl -X POST "http://localhost:8000/ask" -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3NzY0MzU1MjgsImV4cCI6MTc3NjQzOTEyOH0.xT4vQbrG1ky5uefVFUbVKs0ZFbOiDccBIHGBvme9nU4" -H "Content-Type: application/json" -d "{\"question\":\"Explain JWT\"}"
{"question":"Explain JWT","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":9,"budget_remaining_usd":1.9e-05}}

```

cho 4.3: 

```bash

phana@VietNhanLaptop MINGW64 /c/GET_A_JOB/VIN_AI/Lab/Day12/2A202600279_Phan_Nguyen_Viet_Nhan_Day12/04-api-gateway/production (main)
$ for i in $(seq 1 12); do curl -s -X POST "http://localhost:8000/ask" -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3NzY0MzU1MjgsImV4cCI6MTc3NjQzOTEyOH0.xT4vQbrG1ky5uefVFUbVKs0ZFbOiDccBIHGBvme9nU4" -H "Content-Type: application/json" -d "{\"question\":\"test\"}"; e
cho; done
{"question":"test","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","usage":{"requests_remaining":9,"budget_remaining_usd":3.9e-05}}
{"question":"test","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":8,"budget_remaining_usd":5.8e-05}}
{"question":"test","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":7,"budget_remaining_usd":7.6e-05}}
{"question":"test","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","usage":{"requests_remaining":6,"budget_remaining_usd":9.7e-05}}
{"question":"test","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","usage":{"requests_remaining":5,"budget_remaining_usd":0.000112}}
{"question":"test","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","usage":{"requests_remaining":4,"budget_remaining_usd":0.000133}}
{"question":"test","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","usage":{"requests_remaining":3,"budget_remaining_usd":0.000154}}
{"question":"test","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":2,"budget_remaining_usd":0.000172}}
{"question":"test","answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé.","usage":{"requests_remaining":1,"budget_remaining_usd":0.000188}}
{"question":"test","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic.","usage":{"requests_remaining":0,"budget_remaining_usd":0.000209}}
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":57}}
{"detail":{"error":"Rate limit exceeded","limit":10,"window_seconds":60,"retry_after_seconds":56}}

```


### Exercise 4.4: Cost guard implementation

**Approach** (từ `04-api-gateway/production/cost_guard.py`):

**Luồng hoạt động — 2 bước mỗi request:**
1. **Trước khi gọi LLM** → `cost_guard.check_budget(user_id)` kiểm tra budget còn không
2. **Sau khi gọi LLM xong** → `cost_guard.record_usage(user_id, input_tokens, output_tokens)` ghi nhận chi phí thực tế

**Cách tính chi phí:**
- Input: `(tokens / 1000) × $0.00015` (GPT-4o-mini: $0.15/1M tokens)
- Output: `(tokens / 1000) × $0.0006` (GPT-4o-mini: $0.60/1M tokens)
- Token được ước tính: `len(question.split()) × 2`

**2 lớp kiểm soát:**
| Lớp | Limit | HTTP Code | Ý nghĩa |
|-----|-------|-----------|---------|
| Per-user | $1/ngày | 402 Payment Required | Mỗi user bị giới hạn riêng |
| Global | $10/ngày | 503 Service Unavailable | Bảo vệ toàn bộ hệ thống |

**Cảnh báo sớm:** Log warning khi user dùng ≥80% budget — cho phép can thiệp trước khi bị block

**Reset tự động:** So sánh `record.day` với ngày hiện tại — nếu khác ngày thì tạo `UsageRecord` mới (reset về 0)

**Hạn chế hiện tại:** Lưu in-memory → mất khi restart. Production thực tế phải dùng Redis với key `budget:{user_id}:{YYYY-MM-DD}` để persist qua restart và share giữa nhiều instances.

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks — giải thích

`develop/app.py` implement 2 endpoints:
- **`/health` (Liveness probe):** Trả về `{"status":"ok"}` kèm uptime, version, memory usage. Cloud platform gọi định kỳ — nếu fail → restart container tự động.

```bash
phana@VietNhanLaptop MINGW64 /c/GET_A_JOB/VIN_AI/Lab/Day12/2A202600279_Phan_Nguyen_Viet_Nhan_Day12/05-scaling-reliability/develop (main)
$ curl http://localhost:8000/health
{"status":"ok","uptime_seconds":7.3,"version":"1.0.0","environment":"development","timestamp":"2026-04-17T15:01:46.580637+00:00","checks":{"memory":{"status":"ok","used_percent":88.6}}}
```

- **`/ready` (Readiness probe):** Trả về 200 chỉ khi `_is_ready=True` (sau startup xong). Trả về 503 khi đang khởi động hoặc shutdown. Load balancer dùng cái này để không route traffic vào instance chưa sẵn sàng.

```bash
phana@VietNhanLaptop MINGW64 /c/GET_A_JOB/VIN_AI/Lab/Day12/2A202600279_Phan_Nguyen_Viet_Nhan_Day12/05-scaling-reliability/develop (main)
$ curl http://localhost:8000/ready
{"ready":true,"in_flight_requests":1}
```

### Exercise 5.2: Graceful shutdown — giải thích

Cơ chế trong `develop/app.py`:
1. `lifespan` context manager: khi shutdown, set `_is_ready=False` và chờ `_in_flight_requests == 0` (tối đa 30 giây)
2. Middleware `track_requests` đếm số request đang xử lý bằng counter `_in_flight_requests`
3. `signal.signal(SIGTERM, handle_sigterm)` bắt tín hiệu từ platform
4. uvicorn chạy với `timeout_graceful_shutdown=30` — đảm bảo hoàn thành request trước khi tắt


```
$ python app.py &
PID=$!

# Gửi request
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Long task"}' &

# Ngay lập tức kill
kill -TERM $PID
[1] 784
[2] 801
2026-04-17 20:09:48,430 INFO Starting agent on port 8000
INFO:     Started server process [2148]
INFO:     Waiting for application startup.
2026-04-17 20:09:48,469 INFO Agent starting up...
2026-04-17 20:09:48,469 INFO Loading model and checking dependencies...
[1]-  Terminated              python app.py

$ curl: (7) Failed to connect to localhost port 8000 after 2240 ms: Couldn't connect to server

### Exercise 5.3: Stateless design — giải thích

**Anti-pattern (có state trong memory):**
```python
conversation_history = {}  # Mất khi restart, không share giữa instances
```

**Production pattern (Redis-backed):**
`production/app.py` lưu session vào Redis với key `session:{session_id}` và TTL 3600s.
- `save_session()` / `load_session()` đọc/ghi Redis
- Bất kỳ instance nào cũng đọc được cùng session → scale ngang không bị mất data
- Tự fallback về in-memory nếu Redis không có (cho dev local)
- `served_by: INSTANCE_ID` trong response chứng minh request có thể được serve bởi instance khác nhau mà vẫn có đúng history

### Exercise 5.4: Load balancing — giải thích

`docker compose up --scale agent=3` khởi động 3 agent instances + Nginx làm load balancer.
- Nginx dùng round-robin phân tán traffic đều cho 3 instances
- Mỗi request có thể vào instance khác nhau — quan sát `served_by` thay đổi
- Nếu 1 instance die, Nginx tự chuyển traffic sang 2 instance còn lại (health check của Nginx)

```bash
Test log:
agent-1  | {"ts":"...","lvl":"INFO","msg":"Processing request on instance-d0a9a9"}
agent-2  | {"ts":"...","lvl":"INFO","msg":"Processing request on instance-027688"}
agent-3  | {"ts":"...","lvl":"INFO","msg":"Processing request on instance-710668"}

```


### Exercise 5.5: Test stateless

`test_stateless.py` tự động:
1. Tạo conversation trên instance 1
2. Gửi request tiếp theo — có thể vào instance 2 hoặc 3
3. Xác nhận conversation history vẫn còn nguyên → stateless hoạt động đúng
```

PS C:\GET_A_JOB\VIN_AI\Lab\Day12\2A202600279_Phan_Nguyen_Viet_Nhan_Day12\05-scaling-reliability\production> python test_stateless.py
============================================================
Stateless Scaling Demo
============================================================

Session ID: c6fa96e8-4654-4915-97a0-3013920eeed4

Request 1: [instance-d0a9a9]
  Q: What is Docker?
  A: Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!...

Request 2: [instance-027688]
  Q: Why do we need containers?
  A: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận....      

Request 3: [instance-710668]
  Q: What is Kubernetes?
  A: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận....      

Request 4: [instance-d0a9a9]
  Q: How does load balancing work?
  A: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận....      

Request 5: [instance-027688]
  Q: What is Redis used for?
  A: Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé....        

------------------------------------------------------------
Total requests: 5
Instances used: {'instance-710668', 'instance-d0a9a9', 'instance-027688'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 2
  [user]: What is Kubernetes?...
  [assistant]: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã đư...    

✅ Session history preserved across all instances via Redis!
---

---

### 2. Full Source Code - Lab 06 Complete (60 points)

Your final production-ready agent with all files:

```
your-repo/
├── app/
│   ├── main.py              # Main application
│   ├── config.py            # Configuration
│   ├── auth.py              # Authentication
│   ├── rate_limiter.py      # Rate limiting
│   └── cost_guard.py        # Cost protection
├── utils/
│   └── mock_llm.py          # Mock LLM (provided)
├── Dockerfile               # Multi-stage build
├── docker-compose.yml       # Full stack
├── requirements.txt         # Dependencies
├── .env.example             # Environment template
├── .dockerignore            # Docker ignore
├── railway.toml             # Railway config (or render.yaml)
└── README.md                # Setup instructions
```

**Requirements:**
-  All code runs without errors
-  Multi-stage Dockerfile (image < 500 MB)
-  API key authentication
-  Rate limiting (10 req/min)
-  Cost guard ($10/month)
-  Health + readiness checks
-  Graceful shutdown
-  Stateless design (Redis)
-  No hardcoded secrets

---

### 3. Service Domain Link

Create a file `DEPLOYMENT.md` with your deployed service information:

```markdown
# Deployment Information

## Public URL
https://your-agent.railway.app

## Platform
Railway / Render / Cloud Run

## Test Commands

### Health Check
```bash
curl https://your-agent.railway.app/health
# Expected: {"status": "ok"}
```

### API Test (with authentication)
```bash
curl -X POST https://your-agent.railway.app/ask \
  -H "X-API-Key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```

## Environment Variables Set
- PORT
- REDIS_URL
- AGENT_API_KEY
- LOG_LEVEL

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
```

##  Pre-Submission Checklist

- [ ] Repository is public (or instructor has access)
- [ ] `MISSION_ANSWERS.md` completed with all exercises
- [ ] `DEPLOYMENT.md` has working public URL
- [ ] All source code in `app/` directory
- [ ] `README.md` has clear setup instructions
- [ ] No `.env` file committed (only `.env.example`)
- [ ] No hardcoded secrets in code
- [ ] Public URL is accessible and working
- [ ] Screenshots included in `screenshots/` folder
- [ ] Repository has clear commit history

---

##  Self-Test

Before submitting, verify your deployment:

```bash
# 1. Health check
curl https://your-app.railway.app/health

# 2. Authentication required
curl https://your-app.railway.app/ask
# Should return 401

# 3. With API key works
curl -H "X-API-Key: YOUR_KEY" https://your-app.railway.app/ask \
  -X POST -d '{"user_id":"test","question":"Hello"}'
# Should return 200

# 4. Rate limiting
for i in {1..15}; do 
  curl -H "X-API-Key: YOUR_KEY" https://your-app.railway.app/ask \
    -X POST -d '{"user_id":"test","question":"test"}'; 
done
# Should eventually return 429
```

---

##  Submission

**Submit your GitHub repository URL:**

```
https://github.com/your-username/day12-agent-deployment
```

**Deadline:** 17/4/2026

---

##  Quick Tips

1.  Test your public URL from a different device
2.  Make sure repository is public or instructor has access
3.  Include screenshots of working deployment
4.  Write clear commit messages
5.  Test all commands in DEPLOYMENT.md work
6.  No secrets in code or commit history

---

##  Need Help?

- Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- Review [CODE_LAB.md](CODE_LAB.md)
- Ask in office hours
- Post in discussion forum

---

**Good luck! **
