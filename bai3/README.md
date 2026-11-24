# Bài 3: Cấu hình Header Security

## 1. Cấu hình hoàn chỉnh

File: [`secure.conf`](secure.conf)

```nginx
server {
    listen 443 ssl;
    server_name secure.example.com;

    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    # ===== HTTP Security Headers =====
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        proxy_pass http://localhost:8080;

        # Ẩn các header cùng tên do backend (vd. Spring Security) tự gửi
        proxy_hide_header X-Frame-Options;
        proxy_hide_header X-Content-Type-Options;
        proxy_hide_header Strict-Transport-Security;
    }
}
```

## 2. Ý nghĩa từng header

| Header | Giá trị | Chống lại | Ý nghĩa |
|---|---|---|---|
| `X-Frame-Options` | `SAMEORIGIN` | **Clickjacking** | Chỉ cho phép trang được nhúng vào `<iframe>` của **cùng origin**. Trang độc hại không thể nhúng trang của ta vào một iframe trong suốt để lừa người dùng click. |
| `X-Content-Type-Options` | `nosniff` | **MIME-sniffing / XSS** | Bắt trình duyệt dùng đúng `Content-Type` server trả về, không tự "đoán" kiểu file. Ngăn file upload (vd. `.txt`, ảnh) chứa mã JS bị trình duyệt thực thi như script. |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | **SSL stripping / MITM** | HSTS: trong **1 năm (31.536.000 giây)** trình duyệt chỉ truy cập domain này và **mọi subdomain** qua HTTPS, tự chuyển `http://` thành `https://` và không cho người dùng bỏ qua lỗi chứng chỉ. |

## 3. Giải thích các điểm quan trọng trong cấu hình

- **Tham số `always`**: mặc định `add_header` chỉ thêm header vào response có mã 200, 201, 204, 206, 301, 302, 303, 304, 307, 308.
  Thêm `always` để header được trả về cả với response lỗi (4xx, 5xx), vì trang lỗi cũng có thể bị khai thác.
- **Đặt `add_header` ở cấp `server`**: áp dụng cho mọi `location`. Lưu ý cơ chế kế thừa của Nginx:
  nếu một `location` khai báo **bất kỳ** `add_header` nào của riêng nó thì sẽ **không kế thừa** các `add_header`
  từ cấp `server` nữa. Khi đó phải khai báo lại đủ 3 header trong `location` đó.
- **`proxy_hide_header`**: Spring Security mặc định đã gửi `X-Frame-Options: DENY` và `X-Content-Type-Options: nosniff`.
  Nếu không ẩn header của backend, response sẽ chứa **2 giá trị** cho cùng một header (`DENY` và `SAMEORIGIN`),
  gây mâu thuẫn và bị công cụ scan đánh giá là cấu hình sai. Ẩn đi để Nginx là nơi duy nhất quản lý các header này.
- **HSTS chỉ có hiệu lực qua HTTPS**: trình duyệt bỏ qua header này nếu nhận qua HTTP, vì vậy đặt trong server block `listen 443 ssl` là đúng.
  Chỉ bật `includeSubDomains` khi chắc chắn **mọi subdomain** đều đã hỗ trợ HTTPS.

## 4. Áp dụng cấu hình

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## 5. Kiểm chứng bằng `curl -I`

```bash
# -I : chỉ lấy HTTP headers (gửi request HEAD)
# -k : bỏ qua kiểm tra chứng chỉ (dùng khi test với chứng chỉ tự ký)
curl -I https://secure.example.com
curl -I -k https://secure.example.com

# Chỉ lọc ra 3 header cần kiểm tra
curl -sI -k https://secure.example.com | grep -iE "x-frame-options|x-content-type-options|strict-transport-security"

# Test ngay trên server khi domain chưa trỏ DNS
curl -I -k --resolve secure.example.com:443:127.0.0.1 https://secure.example.com
```

Kết quả mong muốn:

```
HTTP/1.1 200
Server: nginx
Content-Type: text/html;charset=UTF-8
...
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

Mỗi header phải xuất hiện **đúng 1 lần** (không bị trùng với header của backend).
