# opserver: Redis chạy riêng

Redis cache cho backend, host trên máy riêng (tách khỏi máy chạy backend).

## Chạy

```bash
cd opserver
cp .env.example .env
nano .env                      # đặt REDIS_PASSWORD
docker compose up -d
docker compose ps              # healthy
docker exec th-proptech-redis redis-cli config get maxmemory   # 134217728
```

## Bảo mật (bắt buộc)

Redis nghe trên `0.0.0.0:6379` vì backend ở máy khác. Phải làm cả hai:

- Security Group của máy Redis: mở inbound `6379` **chỉ** từ Security Group/IP private của máy backend. Không mở `0.0.0.0/0`.
- `REDIS_PASSWORD` mạnh (compose từ chối chạy nếu thiếu).

## Nối backend

Trong `.env` của backend:

```
REDIS_URL=redis://:<REDIS_PASSWORD>@<private-ip-may-redis>:6379
```

Mật khẩu có ký tự đặc biệt phải URL-encode.

## Cấu hình

`maxmemory 128mb`, `allkeys-lru`, AOF `everysec`, không RDB, container giới hạn `192m`. Đổi qua `REDIS_MAXMEMORY` và `REDIS_MEM_LIMIT`.
