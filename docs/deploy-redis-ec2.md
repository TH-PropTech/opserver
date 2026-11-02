# Deploy Redis lên EC2 riêng

Redis cache cho backend TH-PropTech, chạy trên một EC2 riêng. Backend nối vào qua IP private.

Máy gợi ý: Amazon Linux 2023, t3.micro (1 GiB RAM), 8 GiB gp3, user `ec2-user`. Cùng region (ap-southeast-2) và cùng VPC với máy backend.

## Bước 1. Tạo instance

Launch instance với:

| Mục | Giá trị |
|---|---|
| Name | `redis_th-proptech` |
| AMI | Amazon Linux 2023 (x86_64) |
| Instance type | t3.micro |
| Key pair | Dùng key hiện có hoặc tạo mới, lưu file `.pem` |
| VPC | Cùng VPC với máy backend |
| Auto-assign public IP | Enable (chỉ để SSH cài đặt) |
| Storage | 8 GiB gp3 |

Phần Security group, chọn **Create security group** và sửa luật mặc định của wizard:

- SSH (22): Source **My IP**. Không để `0.0.0.0/0`.
- Bỏ tick HTTP và HTTPS. Redis không cần.

## Bước 2. Security Group cho Redis

EC2 Console, chọn instance Redis, tab Security, bấm vào Security group, Edit inbound rules:

| Type | Port | Source | Ghi chú |
|---|---|---|---|
| SSH | 22 | My IP | Chỉ IP của bạn |
| Custom TCP | 6379 | Security group của máy backend | Hoặc `<private-ip-backend>/32` |

Không bao giờ mở 6379 cho `0.0.0.0/0`. Mật khẩu chỉ là lớp phụ; Security Group là lớp bảo vệ chính.

## Bước 3. SSH và cài Docker

```bash
ssh -i path/to/key.pem ec2-user@<public-ip-redis>
```

```bash
sudo dnf update -y
sudo dnf install -y docker git
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

Docker Compose plugin chưa có sẵn trên Amazon Linux 2023, cài thủ công:

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
```

Thoát SSH rồi vào lại để nhóm `docker` có hiệu lực, sau đó kiểm tra:

```bash
docker --version
docker compose version
```

## Bước 4. Chỉnh kernel cho Redis (khuyến nghị)

Tránh lỗi fork khi AOF rewrite trên máy RAM nhỏ:

```bash
echo 'vm.overcommit_memory = 1' | sudo tee /etc/sysctl.d/99-redis.conf
sudo sysctl --system
```

Tạo swap 1 GB phòng khi thiếu RAM:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=1024
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Bước 5. Lấy mã nguồn và tạo `.env`

```bash
git clone https://github.com/TH-PropTech/opserver.git
cd opserver
cp .env.example .env
```

Sinh mật khẩu rồi điền vào `.env`:

```bash
openssl rand -hex 24
nano .env        # REDIS_PASSWORD=<chuỗi vừa sinh>
```

Repo private thì dùng deploy key hoặc Personal Access Token khi clone.

Biến tuỳ chọn trong `.env`:

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `REDIS_PASSWORD` | bắt buộc | Mật khẩu Redis |
| `REDIS_MAXMEMORY` | `128mb` | Trần bộ nhớ dữ liệu |
| `REDIS_MEM_LIMIT` | `192m` | Trần bộ nhớ container |
| `REDIS_BIND_IP` | `0.0.0.0` | Đặt IP private của máy để chỉ nghe trên VPC |

## Bước 6. Chạy Redis

```bash
docker compose up -d
docker compose ps
```

Cột trạng thái phải là `healthy`. Kiểm tra cấu hình và xác thực:

```bash
docker exec th-proptech-redis redis-cli config get maxmemory          # 134217728
docker exec th-proptech-redis redis-cli config get maxmemory-policy   # allkeys-lru
docker exec th-proptech-redis redis-cli ping                          # PONG (container tự xác thực qua REDISCLI_AUTH)
docker run --rm --network host redis:7-alpine redis-cli -h 127.0.0.1 ping   # NOAUTH Authentication required
```

Lấy IP private của máy Redis để nối backend:

```bash
curl -s -H "X-aws-ec2-metadata-token: $(curl -s -X PUT http://169.254.169.254/latest/api/token -H 'X-aws-ec2-metadata-token-ttl-seconds: 60')" \
  http://169.254.169.254/latest/meta-data/local-ipv4
```

IP private giữ nguyên khi stop/start instance, nên không cần Elastic IP.

## Bước 7. Nối backend

Trên máy backend, sửa `.env`:

```
REDIS_URL=redis://:<REDIS_PASSWORD>@<private-ip-redis>:6379
```

Mật khẩu sinh bằng `openssl rand -hex` chỉ có ký tự an toàn. Nếu tự đặt mật khẩu có ký tự đặc biệt thì phải URL-encode.

Kiểm tra kết nối từ máy backend trước khi restart:

```bash
docker run --rm redis:7-alpine redis-cli -h <private-ip-redis> -a '<REDIS_PASSWORD>' ping
```

Trả `PONG` là thông. Treo hoặc timeout nghĩa là Security Group chưa mở đúng. Rồi restart backend:

```bash
pm2 restart all        # hoặc docker compose restart backend, tuỳ cách backend đang chạy
curl http://localhost:3001/api/v1/health
```

## Kiểm tra bảo mật

Từ một máy không thuộc Security Group được phép:

```bash
nc -vz -w 3 <public-ip-redis> 6379
```

Kết quả phải là timeout hoặc bị từ chối. Nếu kết nối được, sửa Security Group ngay.

## Vận hành

```bash
cd ~/opserver
git pull && docker compose up -d     # cập nhật cấu hình
docker compose logs --tail 50 redis  # xem log
docker stats th-proptech-redis       # RAM, phải dưới 192 MiB
```

Container có `restart: unless-stopped`, tự lên lại khi reboot hoặc crash. Dữ liệu nằm trong volume `redis-data` (AOF).

Đổi mật khẩu: sửa `REDIS_PASSWORD` trong `.env`, chạy `docker compose up -d` rồi cập nhật `REDIS_URL` của backend và restart backend.

## Rollback

Đặt lại `REDIS_URL` của backend về Redis cũ rồi restart backend. Cache và rate limit đều fail-open nên backend vẫn trả đúng (chậm hơn) khi Redis không sẵn sàng.

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
|---|---|---|
| `REDIS_PASSWORD is required` | `.env` thiếu hoặc rỗng | Điền `REDIS_PASSWORD` |
| Backend timeout khi nối | Security Group chưa mở 6379 cho backend | Sửa inbound ở Bước 2 |
| `NOAUTH` / `WRONGPASS` | Sai mật khẩu trong `REDIS_URL` | Kiểm tra lại mật khẩu, URL-encode nếu có ký tự đặc biệt |
| `Can't open or create append only dir` | Quyền volume | `docker compose down`, `docker volume rm opserver_redis-data`, chạy lại (mất dữ liệu cache) |
| Container bị kill (OOM) | Thiếu RAM | Giảm `REDIS_MAXMEMORY`, kiểm tra swap, hoặc nâng lên t3.small |
