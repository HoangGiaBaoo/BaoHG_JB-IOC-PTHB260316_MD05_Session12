# Bài 2: Tối ưu Timeout cho Proxy

## 1. Nguyên nhân lỗi `504 Gateway Time-out`

API export báo cáo của Spring Boot cần khoảng **90 giây** để xử lý, trong thời gian đó backend chưa gửi về byte nào.
Mặc định `proxy_read_timeout` của Nginx là **60 giây**. Sau 60 giây chờ mà không nhận được dữ liệu,
Nginx ngắt kết nối tới upstream và trả về `504 Gateway Time-out` cho client.

## 2. Cấu hình sau khi cập nhật

File: [`api.conf`](api.conf)

```nginx
server {
    listen 80;
    server_name api.example.com;

    location /api/export/ {
        proxy_pass http://localhost:8080;

        # Timeout cho API export báo cáo (~90s xử lý) -> nâng lên 120s
        proxy_connect_timeout 60s;
        proxy_send_timeout    120s;
        proxy_read_timeout    120s;
        send_timeout          120s;
    }
}
```

Các timeout chỉ được nâng trong `location /api/export/`. Các API khác vẫn giữ mặc định 60s,
nên request bị treo ở các API thông thường vẫn bị cắt sớm, không ảnh hưởng toàn hệ thống.

## 3. Ý nghĩa các thông số

| Directive | Mặc định | Giá trị mới | Ý nghĩa |
|---|---|---|---|
| `proxy_connect_timeout` | 60s | 60s | Thời gian tối đa để Nginx **thiết lập kết nối TCP** tới backend. Backend chạy cùng máy nên kết nối gần như tức thì, không cần tăng (thường không nên vượt quá 75s). |
| `proxy_send_timeout` | 60s | 120s | Thời gian tối đa **giữa hai lần ghi liên tiếp** khi Nginx **gửi request** tới backend. Quá thời gian này mà backend không nhận dữ liệu thì kết nối bị đóng. |
| `proxy_read_timeout` | 60s | **120s** | Thời gian tối đa **giữa hai lần đọc liên tiếp** khi Nginx **chờ response** từ backend. **Đây là thông số gây ra lỗi 504**: backend mất 90s mới trả dữ liệu nên giá trị phải lớn hơn 90s. |
| `send_timeout` | 60s | 120s | Thời gian tối đa giữa hai lần ghi liên tiếp khi Nginx **gửi response về client**, giúp client tải file báo cáo lớn hoặc mạng chậm không bị ngắt. |

> Lưu ý: các timeout trên tính **giữa hai lần đọc/ghi liên tiếp**, không phải tổng thời gian của cả request.
> Nếu backend trả dữ liệu dạng stream liên tục thì mỗi lần nhận được dữ liệu, bộ đếm được tính lại từ đầu.

## 4. Áp dụng và kiểm tra

```bash
sudo nginx -t && sudo systemctl reload nginx

# Gọi thử API export, in ra mã trạng thái và tổng thời gian
curl -o report.xlsx -w "HTTP %{http_code} - %{time_total}s\n" http://api.example.com/api/export/report
# Kết quả mong muốn: HTTP 200 - ~90s (không còn 504 ở giây thứ 60)
```

## 5. Lưu ý thêm

- Nếu phía trước Nginx còn Load Balancer hoặc CDN (vd. AWS ALB có idle timeout mặc định 60s, Cloudflare 100s)
  thì cũng phải tăng timeout ở các tầng đó, nếu không request vẫn bị cắt.
- Về lâu dài, tác vụ tốn hơn 1 phút nên chuyển sang **xử lý bất đồng bộ**: API trả về `202 Accepted` kèm mã job,
  client hỏi trạng thái định kỳ và tải file khi job hoàn tất, thay vì giữ kết nối HTTP mở quá lâu.
