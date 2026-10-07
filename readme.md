# Bài 3: Thiết lập Cơ sở dữ liệu và Tự cấu hình dịch vụ Systemd cho Spring Boot

## 1. Mục tiêu

* Khởi tạo cơ sở dữ liệu MySQL cho ứng dụng Spring Boot.
* Tạo user MySQL và phân quyền trên database.
* Tạo user Linux `spring-runner` để chạy ứng dụng với quyền hạn chế.
* Tự viết file Systemd Service để quản lý ứng dụng Spring Boot.
* Cấu hình ứng dụng tự động khởi động cùng hệ thống và tự động restart khi ứng dụng bị crash.
* Kiểm tra ứng dụng chạy trên port `8082`.

---

## 2. Thông tin bài tập

| Thành phần       | Giá trị                   |
| ---------------- | ------------------------- |
| Database         | `springboot_db`           |
| MySQL User       | `spring-admin`            |
| MySQL Password   | `SpringSecure@123`        |
| Linux User       | `spring-runner`           |
| Application      | `/opt/spring-app/app.jar` |
| Application Port | `8082`                    |
| Systemd Service  | `spring-app.service`      |

---

## 3. Tạo Database MySQL

Đăng nhập MySQL:

```bash
sudo mysql
```

Tạo database:

```sql
CREATE DATABASE springboot_db;
```

Tạo user:

```sql
CREATE USER 'spring-admin'@'localhost'
IDENTIFIED BY 'SpringSecure@123';
```

Cấp toàn quyền trên database:

```sql
GRANT ALL PRIVILEGES ON springboot_db.*
TO 'spring-admin'@'localhost';
```

Áp dụng lại quyền:

```sql
FLUSH PRIVILEGES;
```

Thoát MySQL:

```sql
EXIT;
```

### Kiểm tra

Kiểm tra database:

```bash
sudo mysql -e "SHOW DATABASES;"
```

Kiểm tra quyền:

```bash
sudo mysql -e "SHOW GRANTS FOR 'spring-admin'@'localhost';"
```

---

## 4. Tạo Linux User `spring-runner`

Tạo user hệ thống không có shell login:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin spring-runner
```

Kiểm tra:

```bash
id spring-runner
```

User `spring-runner` được sử dụng để chạy ứng dụng Spring Boot thay vì chạy ứng dụng bằng quyền `root`.

---

## 5. Chuẩn bị ứng dụng Spring Boot

Tạo thư mục:

```bash
sudo mkdir -p /opt/spring-app
```

Copy file ứng dụng:

```bash
sudo cp app.jar /opt/spring-app/app.jar
```

Thay đổi quyền sở hữu:

```bash
sudo chown -R spring-runner:spring-runner /opt/spring-app
```

Kiểm tra:

```bash
ls -l /opt/spring-app/
```

Kết quả:

```text
-rw-r--r-- 1 spring-runner spring-runner ... app.jar
```

---

## 6. Tạo Systemd Service

Tạo file:

```bash
sudo nano /etc/systemd/system/spring-app.service
```

Nội dung:

```ini
[Unit]
Description=Spring Boot Application
After=network.target mysql.service

[Service]
User=spring-runner
Group=spring-runner
WorkingDirectory=/opt/spring-app
ExecStart=/usr/bin/java -jar /opt/spring-app/app.jar
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Giải thích

* `User=spring-runner`: chạy ứng dụng bằng user giới hạn.
* `Group=spring-runner`: sử dụng group tương ứng.
* `WorkingDirectory=/opt/spring-app`: thư mục làm việc của ứng dụng.
* `ExecStart`: lệnh khởi chạy Spring Boot.
* `Restart=on-failure`: tự động restart khi ứng dụng bị crash.
* `RestartSec=10`: chờ 10 giây trước khi restart.
* `WantedBy=multi-user.target`: cho phép service khởi động cùng hệ thống.

---

## 7. Kiểm tra cấu hình Systemd

Kiểm tra file service:

```bash
sudo systemd-analyze verify /etc/systemd/system/spring-app.service
```

Sau khi tạo hoặc chỉnh sửa file service, reload Systemd:

```bash
sudo systemctl daemon-reload
```

Cho phép service tự động khởi động cùng VPS:

```bash
sudo systemctl enable spring-app.service
```

Khởi động service:

```bash
sudo systemctl start spring-app.service
```

---

## 8. Kiểm tra trạng thái Service

Kiểm tra:

```bash
sudo systemctl status spring-app.service
```

Kết quả mong đợi:

```text
● spring-app.service - Spring Boot Application
     Loaded: loaded (...)
     Active: active (running)
```

Trạng thái cần đạt:

```text
Active: active (running)
```

---

## 9. Kiểm tra Port 8082

Sử dụng lệnh:

```bash
ss -tlnp | grep 8082
```

Kết quả mong đợi là port `8082` đang ở trạng thái `LISTEN` và được sử dụng bởi tiến trình Java.

Ví dụ:

```text
LISTEN 0 128 0.0.0.0:8082 0.0.0.0:* users:(("java",pid=1234,fd=123))
```

---

## 10. Kiểm tra User chạy ứng dụng

Kiểm tra tiến trình Java:

```bash
ps -ef | grep '[j]ava'
```

Kết quả mong đợi:

```text
spring-runner ... /usr/bin/java -jar /opt/spring-app/app.jar
```

Điều này chứng minh ứng dụng Spring Boot đang chạy dưới user:

```text
spring-runner
```

thay vì `root`.

---

## 11. Kiểm tra Log của ứng dụng

Nếu service không chạy, xem log:

```bash
sudo journalctl -u spring-app.service -n 100 --no-pager
```

Xem log realtime:

```bash
sudo journalctl -u spring-app.service -f
```

---

## 12. Kết quả

Các yêu cầu của bài tập:

* [x] Tạo database `springboot_db`.
* [x] Tạo MySQL user `spring-admin`.
* [x] Cấp quyền cho `spring-admin` trên database `springboot_db`.
* [x] Tạo Linux user `spring-runner`.
* [x] Không cho phép `spring-runner` đăng nhập shell.
* [x] Đặt ứng dụng tại `/opt/spring-app/app.jar`.
* [x] Tạo file `/etc/systemd/system/spring-app.service`.
* [x] Chạy ứng dụng bằng user `spring-runner`.
* [x] Cấu hình `Restart=on-failure`.
* [x] Cấu hình `RestartSec=10`.
* [x] Cấu hình ứng dụng chạy trên port `8082`.
* [x] Enable service để tự động khởi động cùng hệ thống.
* [x] Kiểm tra trạng thái service bằng `systemctl status`.
* [x] Kiểm tra port bằng `ss -tlnp`.

---

## 13. Các lệnh kiểm tra cuối cùng

```bash
sudo systemctl status spring-app.service
```

```bash
ss -tlnp | grep 8082
```

```bash
ps -ef | grep '[j]ava'
```

```bash
sudo journalctl -u spring-app.service -n 50 --no-pager
```

---

## 14. Cấu trúc bài nộp

```text
ex3/
├── README.md
└── spring-app.service
```

File `spring-app.service`:

```text
/etc/systemd/system/spring-app.service
```

File báo cáo:

```text
homework/session_07/ex3/README.md
```

---

root@vps-2:~# sudo systemctl status spring-app.service --no-pager
○ spring-app.service - Spring Boot Application
     Loaded: loaded (/etc/systemd/system/spring-app.service; disabled; preset: enabled)
     Active: inactive (dead)
root@vps-2:~# ss -tlnp | grep 8082
LISTEN 0      5            0.0.0.0:8082       0.0.0.0:*    users:(("python3",pid=1689,fd=3))
root@vps-2:~#


