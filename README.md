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
* `logging/compose.yml`: cấu hình Loki và Promtail.
* `logging/loki-config.yml`: cấu hình Loki.
* `logging/promtail-config.yml`: cấu hình Promtail.

Nginx hiện chạy trực tiếp trên máy chủ Ubuntu. Loki và Promtail đang chạy trong môi trường monitoring hiện có. Các file trong `logging/` đã được tạo và kiểm tra cú pháp Compose nhưng chưa được dùng để tái triển khai các container đang chạy. Cần xác minh project, network và volume trước khi áp dụng cấu hình mới.

## 4. Cấu trúc thư mục

```text
.
├── docker-compose.yml
├── nginx/
│   └── demo.local.conf.example
├── monitoring/
│   ├── compose.yml
│   └── prometheus.yml
├── php-config/
│   └── 99-security.ini
├── .gitignore
└── README.md
```
## Cấu hình Nginx

File `nginx/demo.local.conf.example` là cấu hình mẫu cho Nginx trên máy chủ.

Cấu hình thực hiện:
- Chuyển hướng HTTP sang HTTPS.
- Reverse proxy đến WordPress tại `127.0.0.1:8080`.
- Thêm bốn HTTP security headers.

File mẫu tham chiếu chứng chỉ tại `/etc/nginx/ssl/`. Các đường dẫn này phải phù hợp với chứng chỉ đã cài trên máy triển khai. Không commit private key hoặc thông tin bí mật vào Git.

Nginx hiện chạy trực tiếp trên máy chủ Ubuntu, không thuộc dịch vụ Nginx trong Docker Compose của dự án.
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

Promtail phát hiện container Docker, gắn nhãn container/service và gửi log đến Loki. Grafana sử dụng Loki datasource để truy vấn log.

Kiểm tra cú pháp Compose logging mà không khởi động lại dịch vụ:

    docker compose -f logging/compose.yml config -q

Trong môi trường lab hiện tại, truy vấn Loki với selector `{container="wp_app"}` đã trả về log WordPress. Bằng chứng thực tế gồm yêu cầu `GET /` có HTTP 301 và yêu cầu `POST /wp-cron.php` có HTTP 200. Điều này xác nhận log WordPress đã được chuyển đến Loki và truy vấn được.

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

### Các biện pháp đã cấu hình và kiểm tra

* HTTP tự chuyển hướng sang HTTPS.
* TLS certificate có Subject Alternative Name (SAN) cho `demo.local`; đã kiểm tra truy cập HTTPS trong môi trường hiện tại.
* Bốn HTTP security headers đã được xác minh: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy` và `Permissions-Policy`.
* WordPress chỉ bind vào `127.0.0.1:8080`.
* phpMyAdmin chỉ bind vào `127.0.0.1:8081`; đã xác nhận giao diện truy cập được và nhìn thấy database `wordpress_db`.
* File `php-config/99-security.ini` đặt `expose_php = Off`; đã xác minh header `X-Powered-By` không còn xuất hiện trong phản hồi HTTPS.
* File `.env` được khai báo trong `.gitignore`. Cần tiếp tục kiểm tra lịch sử Git trước khi kết luận không có bí mật từng bị commit.
* Cần đánh giá thêm quyền truy cập các dịch vụ monitoring và bảo mật mạng khi triển khai trên máy khác.

Chứng chỉ CA tự tạo chỉ được tin cậy trên thiết bị đã cài đặt CA tương ứng. Không xem chứng chỉ tự ký hoặc CA nội bộ là chứng chỉ công khai được mọi trình duyệt tin cậy.

cAdvisor cần quyền truy cập vào một số tài nguyên của máy chủ để thu thập metrics. Cấu hình này có rủi ro bảo mật và chỉ nên dùng trong môi trường lab được kiểm soát.

## 11. Kiểm thử và nghiệm thu

### Kết quả đã xác minh trong môi trường hiện tại

| Hạng mục | Kết quả | Bằng chứng |
|---|---|---|
| WordPress qua HTTPS | PASS | HTTPS trả về HTTP 200 |
| HTTP chuyển sang HTTPS | PASS | HTTP trả về HTTP 301 |
| HTTP security headers | PASS | Đã xác minh đủ bốn header |
| Ẩn phiên bản PHP | PASS | `expose_php = Off`; không thấy `X-Powered-By` |
| WordPress và MySQL | PASS | Hai container đang chạy |
| phpMyAdmin | PASS | Truy cập được `wordpress_db` qua cổng localhost |
| Prometheus và cAdvisor | PASS | Prometheus ready; cả hai scrape targets ở trạng thái up |
| Grafana CPU/RAM | PASS | Dashboard hiển thị dữ liệu |
| Loki và LogQL | PASS | Đã truy vấn được log WordPress |
| Git | PASS (kiểm tra cơ bản) | Đã push lên GitHub; working tree sạch; không tìm thấy .env, SQL, .key hoặc .pem theo các mẫu tìm kiếm trong lịch sử Git đã kiểm tra |

Các kết quả trên phản ánh môi trường lab hiện tại, chưa chứng minh việc triển khai thành công từ một máy hoàn toàn mới.

### Các hạng mục cần kiểm tra thêm

* Kiểm tra đăng nhập WordPress quản trị và chức năng website sau khi khởi động lại.
* Thử quy trình triển khai từ máy mới bằng cấu hình và hướng dẫn trong README.
* Đánh giá quyền truy cập dịch vụ monitoring và bảo mật mạng trước khi triển khai ra ngoài môi trường lab.

### Kiểm thử backup và khôi phục dữ liệu

* Đã tạo bản sao lưu database `wordpress_db` bằng `mysqldump`, lưu ngoài repository tại `~/wordpress-backups/`.
* Đã khôi phục bản dump vào database thử nghiệm riêng `wordpress_restore_test`, không ghi đè database đang được WordPress sử dụng.
* Kết quả kiểm tra: 12 bảng sau khôi phục; số bài viết khớp 6/6 và số người dùng khớp 1/1 giữa database gốc và database thử nghiệm.
* Lệnh khôi phục kết thúc với exit code 0.
* Phạm vi kiểm thử mới đối chiếu cấu trúc bảng và số lượng bài viết, người dùng; chưa so sánh toàn bộ dữ liệu của mọi bảng.

Chỉ đánh dấu các hạng mục còn lại là hoàn thành khi đã có kết quả kiểm thử thực tế.

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

### Kết quả xác minh monitoring bổ sung (18/10/2026)

Lượt kiểm tra gần nhất ghi nhận các target Prometheus sau đều ở trạng thái `UP`:

- `prometheus`: giám sát Prometheus.
- `cadvisor`: thu thập metrics tài nguyên container.
- `mysql`: MySQL Exporter kết nối được tới database; truy vấn `mysql_up` trả về `1`.
- `nginx_https`: Blackbox Exporter kiểm tra `https://demo.local`; truy vấn `probe_success{job="nginx_https"}` trả về `1`.

**Phạm vi giám sát:** `nginx_https` kiểm tra khả năng truy cập HTTPS và mã phản hồi HTTP mong đợi, không phải metrics nội bộ chuyên biệt của Nginx.

**Lưu ý TLS:** Blackbox probe hiện đặt `insecure_skip_verify: true` để tương thích với chứng chỉ tự ký của môi trường lab. Thiết lập này không xác minh chuỗi chứng chỉ và cần được xem xét lại nếu triển khai production.

File credentials của MySQL Exporter nằm tại `monitoring/mysql-exporter.cnf`, được loại khỏi Git bằng `.gitignore`. Không commit file này hoặc bất kỳ file `.env`, private key hay mật khẩu nào.
