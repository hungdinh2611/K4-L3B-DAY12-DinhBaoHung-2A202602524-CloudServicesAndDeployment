# Triển khai CP5 — Railway

## Thông tin học viên

| Mục | Nội dung |
|---|---|
| Họ và tên | Đinh Bảo Hưng |
| Mã học viên | 2A202602524 |
| Repo | https://github.com/hungdinh2611/K4-L3B-DAY12-DinhBaoHung-2A202602524-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|---|---|
| Platform | Railway (project `day12-agent-dinhbaohung`) |
| Ngày deploy | 2026-09-29 |
| Public URL | https://day12-agent-production-8ce2.up.railway.app |
| Dashboard | https://railway.com/project/95191ca4-1152-4a5f-8c32-96fd9aee4879 |
| Redis | Railway Redis service trong cùng project |

## Biến môi trường trên Railway

| Biến | Nguồn |
|---|---|
| `AGENT_API_KEY` | Secret đặt trong Railway Variables qua stdin của CLI; không lưu trong repo |
| `REDIS_URL` | Reference tới `Redis.REDIS_URL` trong cùng project |
| `RATE_LIMIT_PER_MINUTE` | Cấu hình service: 10 |
| `MONTHLY_BUDGET_USD` | Cấu hình service: 10.0 |
| `LOG_LEVEL` | Cấu hình service: INFO |
| `PORT` | Đọc từ môi trường Railway nếu có; Dockerfile dùng 8000 khi không được cấp |

## Kiểm tra URL thật

```bash
URL=https://day12-agent-production-8ce2.up.railway.app
curl -i "$URL/health"
curl -i "$URL/ready"
curl -i -X POST "$URL/ask" -H "Content-Type: application/json" -d '{"question":"Hello"}'
# Để thử request hợp lệ, nạp AGENT_API_KEY từ môi trường cục bộ; không in giá trị.
curl -i -X POST "$URL/ask" -H "Content-Type: application/json" -H "X-API-Key: $AGENT_API_KEY" -H "X-User-Id: cp5-smoke" -d '{"question":"Deploy la gi?"}'
```

Kết quả kiểm tra thực tế ngày 2026-09-29:

```text
GET /health                 200  {"status":"ok","service":"day12-agent","version":"1.0.0"}
GET /ready                  200  {"status":"ready","redis":true}
POST /ask (không có key)    401  {"detail":"invalid or missing API key"}
POST /ask (key hợp lệ)      200  user_id=cp5-smoke, history_length=0, có answer
15 POST /ask liên tiếp      200 x 10, sau đó 429 x 5 (user thử riêng)
```

## Ảnh minh chứng

- `screenshots/dashboard.png`: Railway project hiển thị agent và Redis.
- `screenshots/health.png`: public URL `/health` và kết quả 200.

Hai ảnh này cần được chụp trực tiếp từ trình duyệt đã đăng nhập Railway trước khi nộp bài.
