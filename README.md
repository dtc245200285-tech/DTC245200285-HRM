# PEOPLEOS HRM — Docker, PostgreSQL, Monitoring & Hardening

Ứng dụng mẫu quản lý nhân sự theo đề tài 10: quản lý nhân viên, phòng ban và chấm công cơ bản; PostgreSQL + pgAdmin; Nginx reverse proxy và security headers; Prometheus + Grafana; Loki + Promtail; Docker Compose và hardening cơ bản.

## 1. Yêu cầu
- Docker Desktop (Windows/macOS) hoặc Docker Engine + Docker Compose plugin (Ubuntu).
- RAM khuyến nghị 4 GB trở lên để chạy cả bộ monitoring.

## 2. Cấu hình lần đầu
Linux/macOS:
```bash
cp .env.example .env
```
PowerShell:
```powershell
Copy-Item .env.example .env
```
Mở `.env`, đổi toàn bộ mật khẩu và `SESSION_SECRET` trước khi triển khai thật. Đây là dự án học tập, không có hệ thống đăng nhập/phân quyền người dùng; không đưa ra Internet công khai nếu chưa bổ sung xác thực và HTTPS.

## 3. Khởi chạy ứng dụng chính
```bash
docker compose up -d --build
```
- HRM: http://localhost:8080 (đổi `HTTP_PORT` trong `.env` nếu cổng bận)
- Kiểm tra sức khỏe: http://localhost:8080/health
- API: `/api/employees`, `/api/departments`, `/api/attendance`, `/api/dashboard`

## 4. pgAdmin
pgAdmin nằm trong profile `admin`, không chạy mặc định để giảm bề mặt tấn công:
```bash
docker compose --profile admin up -d
```
Mở http://localhost:5050 (hoặc cổng đặt trong `PGADMIN_PORT`). Cổng được bind vào `127.0.0.1` để không phơi bày pgAdmin ra mạng ngoài. Đăng nhập bằng `PGADMIN_EMAIL` và `PGADMIN_PASSWORD` trong `.env`.

Máy chủ `HRM PostgreSQL` được cấu hình sẵn và sẽ xuất hiện trong cây bên trái. Host là `db` vì pgAdmin nằm cùng Docker network. Khi kết nối lần đầu, nhập `POSTGRES_PASSWORD` trong `.env`; tên database là `POSTGRES_DB` và username là `POSTGRES_USER`. Nếu bạn đổi tên database hoặc username trong `.env`, cập nhật các trường `MaintenanceDB` và `Username` tương ứng trong `pgadmin/servers.json`.

## 5. Monitoring (Prometheus, Grafana, Loki)
```bash
docker compose --profile monitoring up -d
```
- Grafana: http://localhost:3001 (mặc định user `admin`, mật khẩu trong `.env`; đổi cổng bằng `GRAFANA_PORT`).
- Prometheus: http://localhost:9090 (đổi cổng bằng `PROMETHEUS_PORT`). Cả hai cổng chỉ bind vào localhost; Grafana truy cập Prometheus nội bộ qua `http://prometheus:9090`.
- Dashboard `HRM - Application Monitoring` và datasource Prometheus/Loki được provision tự động.
- Trong Grafana → Explore → Loki, thử LogQL: `{job=~"nginx|hrm-nginx|hrm-app"}`. Docker log discovery cần Docker socket; đây là quyền nhạy cảm, chỉ dùng trong lab.
- Prometheus scrape metrics tại `app:3000/metrics`; metrics có bộ đếm request và độ trễ. Nginx access logs ghi vào `access-file.log` trong volume `nginx_logs` và được Promtail đọc từ volume dùng chung này.

## 6. Kiểm thử nhanh
```bash
docker compose ps
docker compose logs --tail=100 app nginx db
curl http://localhost:8080/health
curl http://localhost:8080/api/employees
curl -X POST http://localhost:8080/api/departments -H 'Content-Type: application/json' -d '{"name":"Marketing","description":"Truyền thông và thương hiệu"}'
```
Trên Windows PowerShell có thể dùng `Invoke-RestMethod` thay cho `curl`.

## 7. Hardening đã áp dụng
- Tách mạng frontend/backend/monitoring; backend đặt `internal: true`.
- PostgreSQL và Loki không publish cổng ra host; Prometheus/Grafana chỉ publish cổng trên localhost.
- Container ứng dụng chạy user không đặc quyền, root filesystem read-only, tmpfs `/tmp`, drop Linux capabilities, `no-new-privileges`, giới hạn PID.
- Nginx ẩn phiên bản, giới hạn body/timeouts, thêm CSP, `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`.
- Express dùng Helmet, rate limiting, giới hạn JSON body, truy vấn SQL có tham số, validate dữ liệu và thông báo lỗi không lộ stack trace.
- Không commit `.env`; sử dụng `.env.example` làm mẫu.

## 8. Lệnh vận hành
```bash
docker compose down                 # dừng và xóa container/network, giữ volume
docker compose down -v              # XÓA CẢ dữ liệu PostgreSQL/Grafana/Loki
docker compose logs -f app
docker compose --profile monitoring ps
```

## 9. Lưu ý vận hành
- `postgres/init.sql` chỉ chạy tự động khi volume PostgreSQL được khởi tạo lần đầu. Nếu sửa schema cho môi trường lab mới, có thể chạy `docker compose down -v` rồi `docker compose up -d --build` (lệnh `down -v` sẽ xóa dữ liệu).
- Dự án được cung cấp dưới dạng bài thực hành, chưa có đăng nhập, RBAC, TLS/HTTPS, backup tự động, email hay workflow nhân sự nâng cao. Cần bổ sung các phần này trước khi dùng dữ liệu thật.
- Nếu cần truy cập pgAdmin an toàn, nên publish chỉ trên localhost: thêm `ports: ["127.0.0.1:5050:80"]` cho `pgadmin`.
