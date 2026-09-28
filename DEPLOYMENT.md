# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Thu Trang |
| Mã học viên | 2A202602435 |
| Repo | https://github.com/trangnguyen2k5-sketch/K4-L3A-DAY12-NguyenThuTrang-2A202602435-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://k4-l3a-day12-nguyenthutrang-2a202602435-cloudser-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Railway |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://k4-l3a-day12-nguyenthutrang-2a202602435-cloudser-production.up.railway.app/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://k4-l3a-day12-nguyenthutrang-2a202602435-cloudser-production.up.railway.app/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://k4-l3a-day12-nguyenthutrang-2a202602435-cloudser-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://k4-l3a-day12-nguyenthutrang-2a202602435-cloudser-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://k4-l3a-day12-nguyenthutrang-2a202602435-cloudser-production.up.railway.app/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
1. GET /health
HTTP/2 200 
content-type: application/json
date: Mon, 28 Sep 2026 14:39:11 GMT
server: railway-hikari
x-railway-request-id: -jR5b8vsTNy3HSLsxtoGcA
content-length: 57

{"status":"ok","service":"day12-agent","version":"1.0.0"}

2. GET /ready
HTTP/2 200 
content-type: application/json
date: Mon, 28 Sep 2026 14:39:11 GMT
server: railway-hikari
x-railway-request-id: 3GZx2e0NQu-hmoIUwUFZXw
content-length: 31

{"status":"ready","redis":true}

3. POST /ask (Không có API Key)
HTTP/2 401 
content-type: application/json
date: Mon, 28 Sep 2026 14:39:12 GMT
server: railway-hikari
x-railway-request-id: Fa8al50HQ5u9BQwYn6XIxQ
content-length: 39

{"detail":"invalid or missing API key"}

4. POST /ask (Có API Key)
HTTP/2 200 
content-type: application/json
date: Mon, 28 Sep 2026 14:39:13 GMT
server: railway-hikari
x-railway-request-id: Kbj5iopRSlWLoB1dnpoFkQ
content-length: 279

{"answer":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","user_id":"sv-test","history_length":0,"cost_usd":2.145e-05,"tokens":{"in":3,"out":35}}

5. Rate Limiting (gửi 15 lượt):
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429
```

## Ảnh Chụp Màn Hình

Đã đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên Railway
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl
