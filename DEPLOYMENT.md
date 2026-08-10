# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Nguyễn Văn Quân |
| Mã học viên | 2A202601544 |
| Repo | https://github.com/nguyenvanquan2905/Day12-2A202601544-NguyenVanQuan.git |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-chat-production-4e57.up.railway.app |
| Platform | Railway  |
| Ngày deploy | 10/08/2026 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | 	Redis add-on của Railway (service day12-chat-redis), set bằng reference ${{day12-chat-redis.REDIS_URL}} |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
$ curl -i https://day12-chat-production-4e57.up.railway.app/healthz
HTTP/1.1 200 OK
Content-Type: application/json
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

$ curl -i https://day12-chat-production-4e57.up.railway.app/readyz
HTTP/1.1 200 OK
Content-Type: application/json
{"status":"ready","redis":true}

$ curl -i -X POST https://day12-chat-production-4e57.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"test"}'
HTTP/1.1 401 Unauthorized
Content-Type: application/json
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}

$ curl -i -X POST https://day12-chat-production-4e57.up.railway.app/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -d '{"message":"Xin chao"}'
HTTP/1.1 200 OK
Content-Type: application/json
{"reply":"Với Xin chao, cách làm phổ biến trong production là đặt một lớp gateway phía trước để lo authentication, rate limiting và bảo vệ chi phí.","client_id":"anonymous","turns_before":0,"usd_cost":2.07e-05,"usage":{"prompt":2,"completion":34}}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---


