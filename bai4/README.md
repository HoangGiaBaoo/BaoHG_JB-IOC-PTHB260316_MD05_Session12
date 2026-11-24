# Bài 4: Tích hợp UFW và Docker

## 1. Vấn đề

`docker-compose.yml` ban đầu:

```yaml
version: '3'
services:
  backend:
    image: my-spring-boot-app:latest
    ports:
      - "8080:8080"
```

`"8080:8080"` tương đương `"0.0.0.0:8080:8080"`: Docker publish cổng 8080 trên **mọi network interface** của host,
kể cả IP public. Dù đã chạy `sudo ufw deny 8080`, từ Internet vẫn truy cập được `http://<IP-public>:8080`.

## 2. Cấu hình `docker-compose.yml` mới

File: [`docker-compose.yml`](docker-compose.yml)

```yaml
services:
  backend:
    image: my-spring-boot-app:latest
    ports:
      - "127.0.0.1:8080:8080"
```

Cú pháp: `"<IP_host>:<port_host>:<port_container>"`.
Thay đổi cần thiết là thêm `127.0.0.1:` vào trước port mapping.

> Dòng `version: '3'` đã được bỏ vì thuộc tính này lỗi thời trong Docker Compose v2 (chỉ sinh cảnh báo
> `the attribute 'version' is obsolete`). Nếu môi trường dùng `docker-compose` v1 cũ thì có thể giữ lại, không ảnh hưởng kết quả.

Nginx trên cùng máy chủ proxy vào backend qua loopback:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
}
```

## 3. Vì sao Docker "vượt mặt" UFW?

- UFW chỉ là công cụ quản lý **iptables**. Các rule của UFW (`ufw-user-input`) nằm trong chain **`INPUT`**,
  chain xử lý gói tin gửi tới **chính host**.
- Khi publish port, Docker tự thêm rule **DNAT** vào bảng `nat` (chain `PREROUTING` → `DOCKER`):
  gói tin tới `<IP-host>:8080` bị đổi đích thành `<IP-container>:8080` **ngay khi vừa vào máy**.
- Sau khi DNAT, gói tin không còn gửi tới host mà được **chuyển tiếp (forward)** sang container,
  nên đi qua chain **`FORWARD`** (nơi Docker đã tự cho phép trong chain `DOCKER`) chứ **không đi qua `INPUT`**.
  Do đó rule chặn của UFW không bao giờ được áp dụng.

```
Gói tin từ Internet → <IP-public>:8080
        │
   nat PREROUTING ──► DOCKER: DNAT → 172.18.0.2:8080      (Docker tự thêm)
        │
     FORWARD ──► DOCKER: ACCEPT ──► container             (không qua INPUT ⇒ UFW bị bỏ qua)
```

## 4. Port mapping `127.0.0.1` ngăn truy cập trái phép như thế nào?

- Với `"127.0.0.1:8080:8080"`, Docker chỉ tạo rule DNAT **khớp với địa chỉ đích `127.0.0.1`**
  (và `docker-proxy` chỉ lắng nghe trên `127.0.0.1:8080`).
- `127.0.0.1` là địa chỉ **loopback**, chỉ tồn tại bên trong máy. Gói tin từ mạng ngoài **không thể có đích là
  `127.0.0.1`** của máy chủ (kernel loại bỏ gói tin như vậy khi nó đến từ interface khác `lo`).
- Gói tin từ Internet gửi tới `<IP-public>:8080` **không khớp rule DNAT nào**, nên không được forward vào container
  mà đi vào chain `INPUT` như bình thường: bị UFW chặn, và cũng không có tiến trình nào lắng nghe. **Kết nối thất bại.**
- Nginx chạy trên cùng host gọi `127.0.0.1:8080`, khớp rule, nên vào container bình thường.

Kết quả: cổng 8080 chỉ truy cập được **từ bên trong máy chủ**. Mọi request từ bên ngoài bắt buộc phải đi qua
Nginx (cổng 80/443), nơi kiểm soát SSL, security headers, timeout, rate limit...

## 5. Áp dụng và kiểm chứng

```bash
# Tạo lại container với port mapping mới
docker compose up -d --force-recreate

# 1. Port chỉ bind trên loopback (mong muốn: 127.0.0.1:8080, KHÔNG phải 0.0.0.0:8080)
docker compose ps
sudo ss -tlnp | grep 8080

# 2. Rule DNAT của Docker chỉ áp dụng cho đích 127.0.0.1
sudo iptables -t nat -L DOCKER -n
# DNAT  tcp  --  0.0.0.0/0  127.0.0.1  tcp dpt:8080 to:172.18.0.2:8080

# 3. Trên chính máy chủ: truy cập được
curl -I http://127.0.0.1:8080

# 4. Từ máy bên ngoài: bị chặn (Connection refused / timeout)
curl -I --max-time 5 http://<IP-public>:8080

# 5. Từ máy bên ngoài qua Nginx: truy cập bình thường
curl -I http://<domain>
```

## 6. Các cách bổ sung hoặc thay thế

- **Không publish port**: cho Nginx chạy dạng container trong cùng Docker network và dùng `proxy_pass http://backend:8080;`.
  Backend hoàn toàn không lộ ra host.
- Đặt mặc định cho toàn bộ Docker daemon trong `/etc/docker/daemon.json`: `{ "ip": "127.0.0.1" }`.
  Mọi port publish không ghi IP sẽ chỉ bind vào loopback.
- Thêm rule vào chain `DOCKER-USER` (chain Docker dành riêng cho người dùng, được xử lý trước các rule của Docker)
  nếu cần lọc truy cập vào container theo IP nguồn.
