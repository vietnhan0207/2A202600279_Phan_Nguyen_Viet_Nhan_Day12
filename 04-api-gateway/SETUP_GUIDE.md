# Part 4: API Security — Setup Guide

## 📋 Overview

Part 4 có 4 bài tập về bảo vệ API:
- **4.1:** API Key authentication
- **4.2:** JWT authentication
- **4.3:** Rate limiting
- **4.4:** Cost guard

**Thời gian:** ~40 phút  
**Độ khó:** Intermediate  
**Yêu cầu:** Python 3.11+, FastAPI

---

## 🚀 Quick Start

### Step 1: Cài dependencies

```bash
cd 04-api-gateway

# Cho exercise 4.1 (develop)
cd develop
pip install -r requirements.txt

# Hoặc cho exercises 4.2-4.4 (production)
cd ../production
pip install -r requirements.txt
```

### Step 2: Chọn bài tập

**Exercise 4.1 & 4.2:**
```bash
cd 04-api-gateway/develop
AGENT_API_KEY=my-secret-key python app.py
```

**Exercise 4.2-4.4 (Advanced):**
```bash
cd 04-api-gateway/production
python app.py
```

### Step 3: Đọc hướng dẫn

Mỗi bài tập có file hướng dẫn riêng:

```bash
# Hướng dẫn chi tiết (Vietnamese)
cat PART4_EXERCISES.md

# Hoặc start code + docstring
cat develop/app.py
cat production/app.py
cat production/auth.py
cat production/rate_limiter.py
cat production/cost_guard.py
```

---

## 📁 Folder Structure

```
04-api-gateway/
├── develop/                    # Exercise 4.1 (Basic API Key)
│   ├── app.py                 # Main app with API key auth
│   └── requirements.txt        # Dependencies
│
├── production/                 # Exercise 4.2-4.4 (Advanced JWT + Rate limit + Cost)
│   ├── app.py                 # Main app with full security stack
│   ├── auth.py                # JWT authentication
│   ├── rate_limiter.py        # Rate limiting
│   ├── cost_guard.py          # Cost protection
│   ├── utils/
│   │   └── mock_llm.py        # Mock LLM (shared)
│   └── requirements.txt        # Dependencies (includes pyjwt)
│
├── PART4_EXERCISES.md         # 📖 Detailed guide (this file)
├── ANSWERS_41.md              # Answer template for 4.1
├── ANSWERS_42.md              # Answer template for 4.2
├── ANSWERS_43.md              # Answer template for 4.3
├── ANSWERS_44.md              # Answer template for 4.4
└── test_part4.py              # 🧪 Automated test script
```

---

## 🎯 Exercise 4.1: API Key Authentication

### Khái niệm
- **API Key:** Token đơn giản để authentication
- **Cách gửi:** Header `X-API-Key: <key>`
- **Thích hợp:** Rapid prototyping, internal APIs, B2B

### Setup
```bash
cd develop
pip install -r requirements.txt
```

### Chạy
```bash
AGENT_API_KEY=my-secret-key python app.py
```

Thấy output:
```
API Key: my-secret-key
Test: curl -H 'X-API-Key: my-secret-key' http://localhost:8000/ask?question=hello
```

### Test endpoints

#### Public (không cần key)
```bash
curl http://localhost:8000/
curl http://localhost:8000/health
```

#### Protected (cần key)
```bash
# ❌ Không có key
curl -X POST http://localhost:8000/ask?question=Hello

# ❌ Key sai
curl -X POST -H "X-API-Key: wrong-key" http://localhost:8000/ask?question=Hello

# ✅ Key đúng
curl -X POST -H "X-API-Key: my-secret-key" http://localhost:8000/ask?question=Hello
```

### Trả lời câu hỏi
- Mở file `ANSWERS_41.md` và điền đáp án
- Tham khảo `PART4_EXERCISES.md` Exercise 4.1 section

---

## 🎯 Exercise 4.2: JWT Authentication

### Khái niệm
- **JWT:** Stateless token chứa user info + signature
- **Flow:** Login → token → use token
- **Ưu điểm:** Stateless (dễ scale), có expiry, có role

### Setup
```bash
cd production
pip install -r requirements.txt
```

### Chạy
```bash
python app.py
```

Thấy output:
```
=== Demo credentials ===
  student / demo123  (10 req/min, $1/day budget)
  teacher / teach456 (100 req/min, $1/day budget)

Docs: http://localhost:8000/docs
```

### Demo users

| Username | Password | Role | Req/min | Budget/day |
|----------|----------|------|---------|-----------|
| student | demo123 | user | 10 | $1 |
| teacher | teach456 | admin | 100 | $1 |

### Test endpoints

#### 1. Login → Get token
```bash
curl -X POST http://localhost:8000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}'

# Response:
# {"access_token": "eyJ0eXAi...", "token_type": "bearer", ...}
```

#### 2. Lưu token vào variable (Linux/Mac)
```bash
TOKEN=$(curl -s -X POST http://localhost:8000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}' | jq -r '.access_token')

echo $TOKEN
```

#### 3. Dùng token để gọi API
```bash
# ❌ Không có token
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'

# ✅ Có token
curl -X POST http://localhost:8000/ask \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```

#### 4. Xem usage
```bash
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/me/usage | jq
```

#### 5. Admin stats
```bash
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/admin/stats | jq
```

### Trả lời câu hỏi
- Mở file `ANSWERS_42.md` và điền đáp án
- Tham khảo `PART4_EXERCISES.md` Exercise 4.2 section

---

## 🎯 Exercise 4.3: Rate Limiting

### Khái niệm
- **Rate Limiting:** Giới hạn requests/user/phút
- **Algorithm:** Sliding Window Counter
- **Cách hoạt động:** Giữ deque timestamp, check khi request mới

### Limits
- **User:** 10 req/phút
- **Admin:** 100 req/phút

### Test

#### 1. Student: 10 requests (all succeed)
```bash
# Get student token
TOKEN=$(curl -s -X POST http://localhost:8000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}' | jq -r '.access_token')

# Gọi 10 lần
for i in {1..10}; do
  curl -s -X POST http://localhost:8000/ask \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"question": "test"}' | jq '.usage.requests_remaining'
done

# Expected: 9, 8, 7, ..., 0
```

#### 2. Student: 11th request (fail with 429)
```bash
curl -X POST http://localhost:8000/ask \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "test"}' | jq '.detail'

# Expected: {"error": "Rate limit exceeded", ...}
```

#### 3. Admin: 50 requests (all succeed)
```bash
# Get admin token
ADMIN_TOKEN=$(curl -s -X POST http://localhost:8000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username": "teacher", "password": "teach456"}' | jq -r '.access_token')

# Gọi 50 lần (admin limit: 100)
for i in {1..50}; do
  curl -s -X POST http://localhost:8000/ask \
    -H "Authorization: Bearer $ADMIN_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"question": "test"}' > /dev/null && echo "Request $i: OK"
done

# Expected: All OK
```

### Trả lời câu hỏi
- Mở file `ANSWERS_43.md` và điền đáp án
- Tham khảo `PART4_EXERCISES.md` Exercise 4.3 section

---

## 🎯 Exercise 4.4: Cost Guard

### Khái niệm
- **Cost Guard:** Bảo vệ budget LLM
- **Budget:** $1/user/day + $10 global/day
- **Tracking:** Đếm input/output tokens → tính chi phí
- **Lỗi:** 402 Payment Required khi vượt budget

### Pricing (mock)
- Input: $0.00015/1K tokens
- Output: $0.0006/1K tokens

### Token counting
```python
# Simplified (in reality use tiktoken)
input_tokens = len(question.split()) * 2
output_tokens = len(response.split()) * 2
```

### Test

#### 1. Check initial usage
```bash
TOKEN=$(curl -s -X POST http://localhost:8000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}' | jq -r '.access_token')

curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/me/usage | jq
```

#### 2. Make request and check cost
```bash
curl -X POST http://localhost:8000/ask \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain Docker"}'

# Check usage again
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/me/usage | jq '.cost_usd, .budget_remaining_usd'
```

#### 3. Make many requests to approach budget
```bash
# Gọi 20 lần (mỗi lần ~$0.05) → gần hết $1 budget
for i in {1..20}; do
  RESPONSE=$(curl -s -X POST http://localhost:8000/ask \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"question\": \"Request $i\"}")
  
  STATUS=$(echo $RESPONSE | jq '.usage.budget_remaining_usd // .detail.error')
  echo "Request $i: $STATUS"
  
  # Stop if hit budget
  if echo $RESPONSE | jq -e '.detail.error' > /dev/null 2>&1; then
    echo "❌ Budget exceeded at request $i"
    break
  fi
done
```

### Trả lời câu hỏi
- Mở file `ANSWERS_44.md` và điền đáp án
- Tham khảo `PART4_EXERCISES.md` Exercise 4.4 section

---

## 🧪 Automated Testing

Dùng script để test tất cả:

```bash
# Install test dependencies
pip install requests

# Run all tests
python test_part4.py

# Test specific exercise
python test_part4.py --exercise 4.1
python test_part4.py --exercise 4.2
python test_part4.py --exercise 4.3
python test_part4.py --exercise 4.4

# Test with custom URL
python test_part4.py --url http://your-server:8000
```

---

## 📊 Expected Results

| Exercise | Endpoint | Key/Token | Expected Status |
|----------|----------|-----------|-----------------|
| 4.1 | /ask | ❌ None | 401 |
| 4.1 | /ask | ❌ Wrong | 403 |
| 4.1 | /ask | ✅ Correct | 200 |
| 4.2 | /auth/token | ❌ Wrong password | 401 |
| 4.2 | /auth/token | ✅ Correct | 200 + token |
| 4.2 | /ask | ❌ No token | 401 |
| 4.2 | /ask | ✅ Valid token | 200 |
| 4.3 | /ask × 10 | student | 200 (all) |
| 4.3 | /ask × 11 | student | 429 |
| 4.3 | /ask × 50 | admin | 200 (all) |
| 4.4 | /me/usage | student | 200 + cost |
| 4.4 | /ask × many | student | 402 (budget exceeded) |

---

## ✅ Submission Checklist

Trước khi submit, kiểm tra:

### Code
- [ ] `develop/app.py` có `verify_api_key()` function
- [ ] `production/app.py` có `verify_token()` dependency
- [ ] `production/rate_limiter.py` có `check()` method
- [ ] `production/cost_guard.py` có `check_budget()` method
- [ ] Không có hardcoded secrets
- [ ] Không có print(secret) logs

### Answers
- [ ] `ANSWERS_41.md` đầy đủ
- [ ] `ANSWERS_42.md` đầy đủ
- [ ] `ANSWERS_43.md` đầy đủ
- [ ] `ANSWERS_44.md` đầy đủ

### Tests
- [ ] Tất cả endpoints test thành công
- [ ] Không có lỗi trong console
- [ ] Screenshot test results (optional)

---

## 🐛 Troubleshooting

### Issue: "Module not found: utils.mock_llm"

**Solution:**
```bash
# Make sure utils folder exists and has __init__.py
touch utils/__init__.py

# Or copy from develop
cp develop/utils/mock_llm.py production/utils/
```

### Issue: "Port 8000 already in use"

**Solution:**
```bash
# Tìm process trên port 8000
lsof -ti:8000 | xargs kill -9

# Hoặc chạy trên port khác
PORT=8001 python app.py
```

### Issue: "JWT secret is hardcoded"

**Solution:**
```python
# ❌ Sai
SECRET_KEY = "super-secret-key-123"

# ✅ Đúng
SECRET_KEY = os.getenv("JWT_SECRET", "default-for-dev-only")
```

### Issue: Token expired

**Solution:**
```bash
# Token hết hạn sau 60 phút
# Lấy token mới
TOKEN=$(curl -s -X POST http://localhost:8000/auth/token ...)
```

---

## 📚 References

- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)
- [JWT.io](https://jwt.io/)
- [RFC 7519 (JWT)](https://tools.ietf.org/html/rfc7519)
- [HTTP Status Codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)
  - 401: Unauthorized (auth required)
  - 402: Payment Required (budget exceeded)
  - 403: Forbidden (auth failed)
  - 429: Too Many Requests (rate limited)

---

## 💡 Tips & Tricks

### Beautify JSON responses
```bash
curl ... | jq '.'   # Pretty print
curl ... | jq '.key' # Extract key
```

### Save token to file
```bash
TOKEN=$(curl ...) 
echo $TOKEN > token.txt
cat token.txt | xargs -I {} curl -H "Authorization: Bearer {}" ...
```

### Test with different URLs
```bash
# Localhost
python test_part4.py --url http://localhost:8000

# Remote server
python test_part4.py --url https://your-app.railway.app
```

---

## 🎓 Learning Outcomes

Setelah menyelesaikan Part 4, Anda akan:
- ✅ Understand API authentication methods
- ✅ Implement API key auth (simple)
- ✅ Implement JWT auth (stateless)
- ✅ Implement rate limiting (sliding window)
- ✅ Implement cost protection (budget guard)
- ✅ Know difference between 401/403/429/402
- ✅ Be able to secure production APIs

---

**Happy coding! 🔐**
