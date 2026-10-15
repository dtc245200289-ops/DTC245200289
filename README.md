# Website quảng bá sản phẩm WordPress

Dự án triển khai website WordPress bằng Docker, kết hợp reverse proxy HTTPS, giám sát tài nguyên và thu thập log.

## 1. Mục tiêu

* Triển khai WordPress và MySQL bằng Docker Compose.
* Cung cấp phpMyAdmin để quản trị cơ sở dữ liệu.
* Sử dụng Nginx làm reverse proxy và chuyển hướng HTTP sang HTTPS.
* Giám sát tài nguyên container bằng Prometheus, cAdvisor và Grafana.
* Thu thập, truy vấn log bằng Loki, Promtail và Grafana.
* Áp dụng các biện pháp bảo mật cơ bản và quản lý mã nguồn bằng Git.

## 2. Công nghệ sử dụng

| Thành phần        | Công nghệ              | Chức năng                      |
| ----------------- | ---------------------- | ------------------------------ |
| Website           | WordPress              | Quản lý và hiển thị nội dung   |
| Database          | MySQL 8.0              | Lưu dữ liệu website            |
| Database GUI      | phpMyAdmin             | Quản trị cơ sở dữ liệu         |
| Reverse proxy     | Nginx                  | Tiếp nhận yêu cầu HTTP/HTTPS   |
| Container         | Docker, Docker Compose | Đóng gói và triển khai dịch vụ |
| Metrics           | Prometheus             | Thu thập và lưu trữ metrics    |
| Container metrics | cAdvisor               | Theo dõi tài nguyên container  |
| Dashboard         | Grafana                | Trực quan hóa metrics và log   |
| Log storage       | Loki                   | Lưu trữ và truy vấn log        |
| Log collector     | Promtail               | Thu thập log container         |
| Version control   | Git, GitHub            | Quản lý lịch sử mã nguồn       |

## 3. Kiến trúc hệ thống

### Luồng truy cập website

Người dùng → Nginx HTTPS → WordPress → MySQL.

Nginx chạy trên máy chủ và chuyển tiếp yêu cầu đến WordPress tại `127.0.0.1:8080`.

### Luồng giám sát

cAdvisor → Prometheus → Grafana.

Promtail thu thập log container và gửi đến Loki. Grafana sử dụng Loki để truy vấn, phân tích log.

### Các thành phần triển khai

* `docker-compose.yml`: cấu hình WordPress, MySQL và phpMyAdmin.
* `monitoring/compose.yml`: cấu hình Prometheus và cAdvisor.
* `monitoring/prometheus.yml`: cấu hình các mục tiêu thu thập metrics.

Nginx, Loki và Promtail được quản lý riêng trên máy chủ hoặc trong môi trường monitoring hiện có; không phải tất cả dịch vụ đều được khởi chạy từ cùng một file Compose.

## 4. Cấu trúc thư mục

```text
.
├── docker-compose.yml
├── monitoring/
│   ├── compose.yml
│   └── prometheus.yml
├── .gitignore
└── README.md
```

## 5. Yêu cầu môi trường

* Linux hoặc môi trường tương thích Docker.
* Docker Engine.
* Docker Compose plugin.
* Tài nguyên máy chủ phù hợp với số lượng dịch vụ được triển khai.

Phiên bản thực tế và cấu hình máy chủ cần được ghi nhận khi nghiệm thu dự án.

## 6. Cấu hình môi trường

Tạo file `.env` ở thư mục gốc theo cấu hình mà `docker-compose.yml` yêu cầu.

Các biến môi trường cần có:

```dotenv
MYSQL_ROOT_PASSWORD=<your_root_password>
MYSQL_DATABASE=<your_database_name>
MYSQL_USER=<your_database_user>
MYSQL_PASSWORD=<your_database_password>
```

Thay các giá trị mẫu bằng thông tin riêng của môi trường triển khai. Không đưa mật khẩu thật hoặc token vào Git.

## 7. Triển khai

Kiểm tra cấu hình trước khi khởi chạy:

```bash
docker compose config -q
```

Khởi chạy các dịch vụ ứng dụng trong file Compose:

```bash
docker compose up -d
```

Kiểm tra trạng thái:

```bash
docker compose ps
```

Xem log của dịch vụ:

```bash
docker compose logs --tail=100
```

Lưu ý: môi trường hiện tại có một số dịch vụ được quản lý độc lập. Kiểm tra các cổng đang sử dụng và cấu hình thực tế trước khi triển khai trên máy chủ khác.

## 8. Giám sát bằng Prometheus và Grafana

Khởi chạy cấu hình monitoring:

```bash
docker compose -f monitoring/compose.yml config -q
docker compose -f monitoring/compose.yml up -d
```

Prometheus được cấu hình thu thập metrics từ chính nó và cAdvisor.

Trong Grafana, thêm Prometheus datasource với URL phù hợp với mạng Docker, ví dụ:

```text
http://wp_prometheus:9090
```

Các truy vấn PromQL tham khảo:

**CPU usage (% của một core):**

```promql
rate(container_cpu_usage_seconds_total{name=~"wp_app|mysql_db",cpu="total"}[5m]) * 100
```

**Memory working set (MiB):**

```promql
container_memory_working_set_bytes{name=~"wp_app|mysql_db"} / 1024 / 1024
```

Cần xác nhận trạng thái target trong Prometheus và dữ liệu hiển thị trong Grafana sau khi triển khai.

## 9. Thu thập và truy vấn log

Grafana sử dụng Loki datasource để truy vấn log được Promtail thu thập.

Các truy vấn LogQL tham khảo:

**Log của WordPress:**

```logql
{container="wp_app"}
```

**Tìm yêu cầu HTTP có mã trạng thái 4xx hoặc 5xx trong access log:**

```logql
{container="wp_app"} |~ "\" [45][0-9][0-9] "
```

**Theo dõi yêu cầu đến trang quản trị WordPress:**

```logql
{container="wp_app"} |= "/wp-admin/"
```

Kết quả truy vấn phụ thuộc vào log thực tế trong khoảng thời gian được chọn.

## 10. Biện pháp bảo mật

Các biện pháp đã được cấu hình hoặc cần kiểm tra trong môi trường triển khai:

* Chuyển hướng HTTP sang HTTPS bằng Nginx.
* Sử dụng TLS certificate có Subject Alternative Name phù hợp với tên miền.
* Thiết lập các HTTP security headers: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` và `Permissions-Policy`.
* Bind cổng WordPress vào `127.0.0.1:8080` để hạn chế truy cập trực tiếp từ mạng bên ngoài.
* Lưu thông tin đăng nhập trong `.env` và loại trừ file này khỏi Git.
* Hạn chế quyền truy cập vào các dịch vụ giám sát và giao diện quản trị.

Chứng chỉ CA tự tạo chỉ được tin cậy trên thiết bị đã cài đặt CA tương ứng. Không xem chứng chỉ tự ký hoặc CA nội bộ là chứng chỉ công khai được mọi trình duyệt tin cậy.

cAdvisor cần quyền truy cập vào một số tài nguyên của máy chủ để thu thập metrics. Cấu hình này có rủi ro bảo mật và chỉ nên dùng trong môi trường lab được kiểm soát.

## 11. Kiểm thử và nghiệm thu

Các hạng mục cần kiểm tra:

* WordPress truy cập được qua HTTPS.
* HTTP được chuyển hướng sang HTTPS.
* MySQL hoạt động và WordPress kết nối được cơ sở dữ liệu.
* phpMyAdmin truy cập được và đăng nhập đúng quyền.
* Prometheus thu thập metrics từ cAdvisor.
* Dashboard Grafana hiển thị dữ liệu CPU và RAM.
* Loki truy vấn được log WordPress.
* Các security headers xuất hiện trong HTTP response.
* Git không chứa file `.env`, token hoặc private key.

Chỉ đánh dấu một hạng mục hoàn thành sau khi có kết quả kiểm thử thực tế.

## 12. Quản lý mã nguồn

Dự án được quản lý bằng Git và lưu trữ trên GitHub.

Một số lệnh thường dùng:

```bash
git status
git log --oneline
git add <file>
git commit -m "Describe the change"
git push origin master
```

Không commit mật khẩu, token, private key hoặc dữ liệu nhạy cảm.

## 13. Hướng phát triển

* Hoàn thiện hướng dẫn triển khai từ máy mới.
* Bổ sung ảnh chụp màn hình dashboard và kết quả truy vấn log.
* Bổ sung quy trình backup và khôi phục dữ liệu.
* Kiểm tra phân quyền, bảo mật mạng và độ ổn định của các dịch vụ.
* Ghi nhận kết quả kiểm thử và tài nguyên sử dụng trong báo cáo nghiệm thu.
