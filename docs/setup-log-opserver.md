# Nhật ký setup máy opserver (Redis)

Ghi lại những gì đã làm trên EC2 `opserver` ngày 2026-11-02. Làm theo [deploy-redis-ec2.md](deploy-redis-ec2.md). Không ghi mật khẩu, public IP và instance ID vào file này.

## Máy

| Mục | Giá trị |
|---|---|
| Name | `opserver` |
| Region | ap-southeast-2 (Sydney) |
| AMI | Amazon Linux 2023 (kernel 6.18, x86_64) |
| Instance type | t3.micro (2 vCPU, 1 GiB RAM) |
| Storage | 8 GiB gp3 |
| User | `ec2-user` |
| Private IP | `172.31.40.26` |
| Kết nối | EC2 Instance Connect (trình duyệt) |

Public IP đổi mỗi lần stop/start instance. Backend chỉ dùng private IP nên không ảnh hưởng.

## Phiên bản đã cài

| Thành phần | Phiên bản |
|---|---|
| Docker | 25.0.18 |
| Docker Compose plugin | v5.6.1 (cài thủ công vào `/usr/local/lib/docker/cli-plugins/`) |
| Git | 2.50.1 |
| Redis | `redis:7-alpine` |

## Đã làm

1. Cài `docker` và `git` bằng `dnf`, bật `docker` khi khởi động, thêm `ec2-user` vào nhóm `docker`.
2. Cài Docker Compose plugin thủ công, `newgrp docker`, `docker compose version` chạy được.
3. Đặt `vm.overcommit_memory = 1` trong `/etc/sysctl.d/99-redis.conf`.
4. Tạo swap 1 GiB `/swapfile`, thêm vào `/etc/fstab`.
5. Clone `https://github.com/TH-PropTech/opserver.git` vào `~/opserver`.
6. Tạo `.env` từ `.env.example` với `REDIS_PASSWORD` sinh bằng `openssl rand -hex 24`.
7. `docker compose up -d`: tạo network `opserver_default`, volume `opserver_redis-data`, container `th-proptech-redis`.

## Kết quả kiểm tra

- `docker compose ps`: `th-proptech-redis` ở trạng thái `Up (healthy)`, cổng `0.0.0.0:6379->6379/tcp`.
- `redis-cli config get maxmemory`: `134217728` (128 MiB).

## Còn lại

- [ ] Security Group: thêm inbound Custom TCP `6379` với source là Security Group của máy backend.
- [ ] Security Group: đổi SSH (22) từ `0.0.0.0/0` sang My IP, bỏ HTTP/HTTPS nếu còn.
- [ ] Kiểm tra `maxmemory-policy` trả `allkeys-lru`.
- [ ] Kiểm tra truy cập không mật khẩu trả `NOAUTH`.
- [ ] Kiểm tra dữ liệu còn sau `docker compose restart redis`.
- [ ] Backend: đặt `REDIS_URL=redis://:<REDIS_PASSWORD>@172.31.40.26:6379`, restart, `GET /api/v1/health` báo Redis ok.
- [ ] Từ máy ngoài Security Group, `nc -vz -w 3 <public-ip> 6379` phải timeout hoặc bị từ chối.
