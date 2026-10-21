# NOVA SHOWCASE - Hệ thống WordPress DevOps Chuyên Nghiệp

Dự án triển khai website quảng bá sản phẩm "Nova Showcase" trên nền tảng WordPress, tự động hóa hoàn toàn bằng Docker Compose. Hệ thống tích hợp toàn diện các tiêu chuẩn DevOps bao gồm Reverse Proxy (HTTPS), Giám sát tài nguyên (Prometheus/Grafana), Thu thập log tập trung (Loki/Promtail) và Tôi cứng bảo mật (Hardening).

## 1. Điểm nổi bật của Hệ thống (Cập nhật mới nhất)

* **Kiến trúc mạng cô lập (Network Isolation):** Sử dụng external network `doan_network`. Database hoàn toàn ẩn khỏi host, chỉ giao tiếp nội bộ.
* **Database Nâng cao (Custom Mock Data):** Cơ sở dữ liệu `wordpress_db` được thiết kế mở rộng với các bảng nghiệp vụ thực tế (`nova_contacts`, `nova_analytics`, `nova_audit_logs`) giúp đa dạng hóa dữ liệu thay vì chỉ dùng các bảng mặc định của WordPress.
* **Bảo mật toàn diện (Hardening):**
* Chạy container với quyền Non-root (`user: "999:999"` cho MySQL, `user: "nobody"` cho Prometheus).
* Nginx cấu hình nghiêm ngặt 4 HTTP Security Headers.
* Ẩn phiên bản PHP (`expose_php = Off`) qua file `99-security.ini`.


* **Observability 100%:** Fix triệt để lỗi DNS và lỗi Timestamp quá hạn. Promtail và Loki hoạt động mượt mà, sẵn sàng truy vấn LogQL theo thời gian thực.

## 2. Công nghệ sử dụng

| Lớp hệ thống | Công nghệ | Vai trò cốt lõi |
| --- | --- | --- |
| **Ứng dụng Web** | WordPress & Theme Nova | CMS quản trị nội dung và giao diện sản phẩm |
| **Cơ sở dữ liệu** | MySQL 8.0 & phpMyAdmin | Lưu trữ dữ liệu và cung cấp GUI quản trị trực quan |
| **Reverse Proxy** | Nginx | Điều phối traffic, SSL Offloading, Security Headers |
| **Monitoring** | Prometheus, cAdvisor, Blackbox | Cụm giám sát tài nguyên (Container, Web, Database) |
| **Logging** | Loki & Promtail | Cụm thu thập, lưu trữ và truy xuất Log tập trung |
| **Data Viz** | Grafana | Trực quan hóa Metrics và LogQL trên giao diện duy nhất |

## 3. Yêu cầu hệ thống (Prerequisites)

* Hệ điều hành: Ubuntu 20.04/22.04 LTS (hoặc môi trường Linux tương thích).
* Đã cài đặt **Docker Engine** và **Docker Compose V2**.
* Các port trống trên host: `80`, `443`, `3000`, `3100`, `8080`, `8081`, `9090`.

## 4. Hướng dẫn Triển khai (Từng bước chi tiết)

### Bước 1: Chuẩn bị cấu hình môi trường

Clone repository về máy chủ. Mở terminal tại thư mục gốc của dự án và tạo file biến môi trường:

```bash
cp .env.example .env

```

Mở file `.env` và cấu hình các thông số bảo mật cho Database (TUYỆT ĐỐI KHÔNG commit file này lên Git):

```dotenv
MYSQL_ROOT_PASSWORD=rootpassword_strong
MYSQL_DATABASE=wordpress_db
MYSQL_USER=wp_user
MYSQL_PASSWORD=wp_password_strong

```

### Bước 2: Khởi tạo Mạng nội bộ (Network)

Tạo mạng bridge dùng chung cho toàn bộ hệ thống để đảm bảo tính cô lập:

```bash
docker network create doan_network

```

### Bước 3: Khởi chạy Khối Ứng dụng (App & Database)

Tại thư mục gốc, khởi động WordPress, MySQL và phpMyAdmin:

```bash
docker compose up -d

```

*Hệ thống sẽ tự động tạo named volumes `wp_data` và `db_data` để duy trì dữ liệu vĩnh viễn.*

### Bước 4: Khởi chạy Khối Giám sát (Monitoring)

Khởi động cụm Prometheus, cAdvisor và các Exporter:

```bash
cd monitoring
docker compose up -d
cd ..

```

### Bước 5: Khởi chạy Khối Logging tập trung

Khởi động Loki và Promtail để bắt đầu thu thập log hệ thống:

```bash
cd logging
docker compose up -d
cd ..

```

### Bước 6: Kiểm tra tổng thể

Xác nhận toàn bộ các container đang chạy ở trạng thái `Up` và không có lỗi:

```bash
docker ps

```

## 5. Bảng định tuyến Truy cập (Access URLs)

| Dịch vụ | Đường dẫn truy cập | Ghi chú |
| --- | --- | --- |
| **Website (HTTPS)** | `[https://demo.local](https://demo.local)` | Hoặc `http://localhost`. Đi qua Nginx Proxy. |
| **WordPress Admin** | `[https://demo.local/wp-admin](https://demo.local/wp-admin)` | Tài khoản quản trị cấp trong lúc setup web. |
| **phpMyAdmin** | `http://localhost:8081` | Đăng nhập bằng thông tin trong file `.env`. |
| **Grafana Dashboard** | `http://localhost:3000` | Giao diện giám sát tập trung. |
| **Prometheus** | `http://localhost:9090` | Trạng thái các Targets (cAdvisor, mysql_exporter). |

## 6. Hướng dẫn Giám sát & Truy vấn

### Truy vấn Metrics (PromQL) trên Grafana

* **Kiểm tra RAM của WordPress:**
```promql
container_memory_working_set_bytes{name="wp_app"} / 1024 / 1024

```


* **Kiểm tra tính sẵn sàng của Database:**
```promql
mysql_up

```



### Truy vấn Log (LogQL) trên Grafana Explore

Để xem log, chọn Data source là **Loki** và nhập các lệnh sau:

* **Xem toàn bộ Log truy cập Web (Nginx/WordPress):**
```logql
{container="wp_app"}

```


* **Lọc các luồng tấn công hoặc lỗi HTTP 4xx/5xx:**
```logql
{container="wp_app"} |~ "\" [45][0-9][0-9] "

```


* **Giám sát sự kiện đăng nhập Admin:**
```logql
{container="nginx_proxy"} |= "wp-login.php"

```



## 7. Quy định Quản lý Mã nguồn & Backup

* **Version Control:** Mọi thay đổi cấu hình (`.yml`, `.conf`, `.ini`) phải được commit rõ ràng với thông điệp mô tả thay đổi. Không commit thư mục volume hoặc file `.env`.
* **Backup:** Để sao lưu database an toàn mà không ảnh hưởng hệ thống đang chạy, sử dụng lệnh:
```bash
docker exec mysql_db /usr/bin/mysqldump -u root -p'PASSWORD' wordpress_db > backup_$(date +%F).sql

```
