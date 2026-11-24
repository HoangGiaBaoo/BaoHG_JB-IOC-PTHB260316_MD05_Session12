# Bài 1: Xử lý sự cố sập Nginx

## 1. Cấu hình lỗi

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://localhost:8080          # <-- dòng 6: THIẾU dấu ";"
        proxy_set_header Host $host;              # <-- dòng 7: nơi Nginx báo lỗi
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Thông báo lỗi:

```
nginx: [emerg] unexpected "proxy_set_header" in /etc/nginx/sites-enabled/app.conf:7
```

## 2. Phân tích nguyên nhân

- Trong Nginx, **mỗi directive phải kết thúc bằng dấu `;`**. Xuống dòng chỉ là khoảng trắng, không kết thúc directive.
- Dòng 6 `proxy_pass http://localhost:8080` bị **thiếu dấu `;`**, nên Nginx đọc tiếp sang dòng 7 và coi
  `proxy_set_header Host $host` là phần tiếp theo của directive `proxy_pass`.
- Vì vậy Nginx báo lỗi tại **dòng 7** (nơi nó gặp `proxy_set_header` ở vị trí không hợp lệ),
  nhưng **lỗi thực sự nằm ở dòng 6**.

> Lưu ý: tùy phiên bản, Nginx có thể báo lỗi dạng
> `invalid number of arguments in "proxy_pass" directive in ...app.conf:7`. Nguyên nhân vẫn như trên:
> `proxy_pass` chỉ nhận 1 tham số nhưng lại "nuốt" thêm các từ của dòng 7 do thiếu `;`.
>
> Kinh nghiệm: khi Nginx báo lỗi tại dòng N, hãy kiểm tra cả **dòng N-1** (thiếu `;`, thiếu `{` hoặc `}`).

## 3. Cách sửa

Thêm dấu `;` vào cuối dòng 6:

```diff
-        proxy_pass http://localhost:8080
+        proxy_pass http://localhost:8080;
```

File cấu hình hoàn chỉnh: [`app.conf`](app.conf)

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## 4. Lệnh kiểm tra cấu hình và khởi động lại Nginx

```bash
# 1. Kiểm tra cú pháp toàn bộ cấu hình
sudo nginx -t
# Kết quả mong muốn:
# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
# nginx: configuration file /etc/nginx/nginx.conf test is successful

# 2. Reload cấu hình (không làm gián đoạn các kết nối đang phục vụ)
sudo systemctl reload nginx
# hoặc
sudo nginx -s reload

# 3. Nếu dịch vụ đã bị dừng hẳn thì khởi động lại
sudo systemctl restart nginx

# 4. Kiểm tra trạng thái dịch vụ
sudo systemctl status nginx
```

Cách an toàn nhất là chỉ reload khi test cấu hình thành công:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Nếu vẫn lỗi, xem log để tìm nguyên nhân:

```bash
sudo journalctl -u nginx --no-pager -n 50
sudo tail -n 50 /var/log/nginx/error.log
```
