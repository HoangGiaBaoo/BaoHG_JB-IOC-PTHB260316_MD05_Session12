# MD05 - Session 12: Nginx, Proxy & Bảo mật hệ thống

| Bài | Nội dung | Thư mục |
|---|---|---|
| 1 | Xử lý sự cố sập Nginx (lỗi cú pháp thiếu `;`) | [`bai1/`](bai1/README.md) |
| 2 | Tối ưu timeout cho proxy (khắc phục `504 Gateway Time-out`) | [`bai2/`](bai2/README.md) |
| 3 | Cấu hình HTTP Security Headers | [`bai3/`](bai3/README.md) |
| 4 | Tích hợp UFW và Docker (bind port vào `127.0.0.1`) | [`bai4/`](bai4/README.md) |

Mỗi thư mục gồm:
- File cấu hình đã sửa (`*.conf` hoặc `docker-compose.yml`)
- `README.md`: báo cáo phân tích nguyên nhân, cách sửa, giải thích và lệnh kiểm chứng

## Tóm tắt kết quả

**Bài 1**: Dòng 6 `proxy_pass http://localhost:8080` thiếu dấu `;` nên Nginx báo lỗi ở dòng 7. Sửa thành
`proxy_pass http://localhost:8080;` rồi chạy `sudo nginx -t && sudo systemctl reload nginx`.

**Bài 2**: Thêm `proxy_connect_timeout 60s; proxy_send_timeout 120s; proxy_read_timeout 120s; send_timeout 120s;`
vào `location /api/export/`. `proxy_read_timeout` (mặc định 60s) là nguyên nhân chính gây 504.

**Bài 3**: Thêm `add_header ... always;` cho `X-Frame-Options`, `X-Content-Type-Options`,
`Strict-Transport-Security`; kiểm chứng bằng `curl -I -k https://secure.example.com`.

**Bài 4**: Đổi port mapping `"8080:8080"` thành `"127.0.0.1:8080:8080"` để Docker chỉ publish cổng trên loopback.
Internet không thể truy cập trực tiếp, còn Nginx cùng máy vẫn proxy được.
