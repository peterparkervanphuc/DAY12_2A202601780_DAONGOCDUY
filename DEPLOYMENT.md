# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đào Ngọc Duy |
| Mã học viên | 2A202601780 |
| Repo | DAY12_2A202601780_DAONGOCDUY |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | http://localhost:8000 |
| Platform | Railway (local fallback do không có tài khoản cloud) |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong .env, không nằm trong repo |
| `REDIS_URL` | ✅ | redis://redis:6379/0 (Docker Compose service) |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i http://localhost:8000/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i http://localhost:8000/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'
```

## Kết Quả Chạy Thật

```
HTTP/1.1 200 OK
{"status":"ok","service":"day12-chat-service","version":"1.0.0"}

HTTP/1.1 200 OK
{"status":"ready","redis":true}

HTTP/1.1 401 Unauthorized
www-authenticate: Bearer
{"detail":"invalid or missing bearer token"}

HTTP/1.1 200 OK
{"reply":"...","client_id":"sv-test","turns_before":0,"usd_cost":0.0001,...}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Dùng phương án dự phòng LOCAL_FALLBACK=true vì không có tài khoản cloud.

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`

```
Lý do: Sử dụng phương án dự phòng với Docker Compose chạy local.
```
