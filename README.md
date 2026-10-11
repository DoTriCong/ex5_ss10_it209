# Bài 5: Docker Compose & file `.env`

Đóng gói bài EX4 (Backend + PostgreSQL, custom network, named volume) thành tệp `docker-compose.yml` theo chuẩn Infrastructure as Code, dùng **Docker Compose CLI v2** (`docker compose`).

## Cấu trúc thư mục

```
ex5/
├── docker-compose.yml   # Khai báo hạ tầng
├── .env.example         # Mẫu biến môi trường (được commit)
├── .gitignore           # Chặn commit file .env thật
└── README.md
```

## Thành phần trong `docker-compose.yml`

| Thành phần | Mô tả |
|-----------|-------|
| `services.database` | `postgres:15-alpine`, mount volume `pgdata` vào `/var/lib/postgresql/data`, có `healthcheck` bằng `pg_isready` |
| `services.backend` | Backend Web API, `DB_HOST=database` (tên service = hostname qua DNS nội bộ), publish `${PORT}:8080` |
| `depends_on` | `backend` chờ `database` ở trạng thái `service_healthy` rồi mới khởi động |
| `networks` (root) | `backend-net` (bridge) cho cả 2 service giao tiếp nội bộ |
| `volumes` (root) | `pgdata` lưu dữ liệu DB bền vững |

Mọi thông tin nhạy cảm (tên DB, user, password, port) đều lấy từ `.env` qua cú pháp `${VAR}`, không hard-code trong YAML.

> Backend dùng `python:3.12-alpine` chạy `http.server` làm dịch vụ giả lập vì đề không cung cấp mã nguồn. Nếu có Dockerfile, thay `image: ...` và `command: ...` bằng `build: .`.

## Cách chạy

```bash
# 1. Tạo file .env từ mẫu rồi sửa giá trị (đặc biệt là DB_PASSWORD)
cp .env.example .env

# 2. Kiểm tra cấu hình sau khi nội suy biến
docker compose config

# 3. Khởi động toàn bộ hạ tầng
docker compose up -d

# 4. Xem trạng thái, log
docker compose ps
docker compose logs -f

# 5. Dừng và xóa container + network (giữ lại volume)
docker compose down

# (Tùy chọn) Xóa luôn volume, mất dữ liệu
docker compose down -v
```

## Kết quả mong đợi

- `docker compose up -d` tự tạo network `<thư-mục>_backend-net` và volume `<thư-mục>_pgdata`, rồi đưa cả 2 service lên trạng thái Running (`database` hiển thị `healthy`).
- `docker compose ps` hiển thị `db-server` và `app-container` đều `Up`.
- Kiểm tra giao tiếp nội bộ:
  ```bash
  docker compose exec backend ping -c 2 database
  ```
- Kiểm tra dữ liệu bền vững: tạo dữ liệu, chạy `docker compose down` rồi `docker compose up -d`, dữ liệu vẫn còn vì volume `pgdata` không bị xóa.

## Lưu ý bảo mật

- File `.env` bị chặn bởi `.gitignore`, chỉ commit `.env.example` với giá trị mẫu.
- Cổng DB chỉ bind vào `127.0.0.1`, không phơi ra mạng ngoài.
- `docker compose` (v2, có dấu cách) thay cho `docker-compose` (v1). Thuộc tính `version:` ở đầu file đã lỗi thời nên không dùng.
