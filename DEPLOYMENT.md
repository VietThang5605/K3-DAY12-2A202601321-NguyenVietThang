# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Việt Thắng |
| Mã học viên | 2A202601321 |
| Repo | https://github.com/VietThang5605/K3-DAY12-2A202601321-NguyenVietThang |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://redis-production-b117.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | Railway tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `LOG_LEVEL` | ✅ | INFO |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `REDIS_PASSWORD` | ✅ | Railway Redis Database |
| `REDIS_URL` | ✅ | Railway Redis Add-on / fakeredis fallback |
| `REDISHOST` | ✅ | Railway Redis Database (`redis.railway.internal`) |
| `REDISPASSWORD` | ✅ | Railway Redis Database |
| `REDISPORT` | ✅ | Railway Redis Database (`6379`) |
| `REDISUSER` | ✅ | Railway Redis Database (`default`) |


## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://redis-production-b117.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://redis-production-b117.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://redis-production-b117.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://redis-production-b117.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://redis-production-b117.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
# 1. Liveness
HTTP/2 200 
content-type: application/json
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness
HTTP/2 200 
content-type: application/json
{"status":"ready","redis":true}

# 3. Không có API Key
HTTP/2 401 
content-type: application/json
{"detail":"invalid or missing API key"}

# 4. Có API Key
HTTP/2 200 
content-type: application/json
{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud. (Mình đang nhớ 20 lượt trao đổi trước đó.)","user_id":"sv-test","history_length":20,"cost_usd":9.525e-05,"tokens":{"in":455,"out":45}}


# 5. Rate Limit (15 lượt)
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
