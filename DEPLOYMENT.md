# Thông Tin Deploy — Checkpoint 5

> `pytest tests/test_cp5.py` đọc file này để tìm địa chỉ service và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Vũ Đức Thiện |
| Mã học viên | 2A202602437 |
| Repo | https://github.com/vuthien3002-sys/K4-L3B-Day12-VuDucThien-2A202602437-Cloud-Service-And-Deployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-qv1c.onrender.com |
| Platform | Render (Blueprint từ `render.yaml`, runtime Docker, gói Free) |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Render tự gán (10000), app đọc qua `${PORT:-8000}` |
| `AGENT_API_KEY` | ✅ | `sync: false` trong `render.yaml` — nhập trên dashboard lúc deploy, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Key Value `day12-redis`, gắn tự động qua `fromService` → `connectionString` |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Chạy ngày 2026-09-29 vào `https://day12-agent-qv1c.onrender.com`:

```
# 1. GET /health
HTTP/1.1 200 OK
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. GET /ready
HTTP/1.1 200 OK
{"status":"ready","redis":true}

# 3. POST /ask (không có API key)
HTTP/1.1 401 Unauthorized
{"detail":"invalid or missing API key"}

# 4. POST /ask (có API key, X-User-Id: sv-test)
HTTP/1.1 200 OK
{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên.","user_id":"sv-test","history_length":0,"cost_usd":2.265e-05,"tokens":{"in":3,"out":37}}

# 5. Rate limit — 15 request liên tiếp (X-User-Id: sv-ratelimit)
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429
```

10 request đầu trong cửa sổ 60 giây được phục vụ, từ request thứ 11 bị chặn với `429` đúng theo `RATE_LIMIT_PER_MINUTE=10`.

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png` — trang service `day12-agent` trên Render (trạng thái Live, URL, commit đã deploy)
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt
