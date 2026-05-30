---
title: "Docker từ A đến Z: Hướng dẫn thực hành cho người mới"
date: 2026-05-04 09:00:00 +0700
categories: [DevOps, Docker]
tags: [docker, container, devops, tutorial]
---

Docker là công cụ không thể thiếu trong hệ sinh thái DevOps hiện đại. Bài này sẽ đi từ khái niệm đến thực hành.

## Docker là gì?

Docker đóng gói ứng dụng và toàn bộ dependency vào một **container** — đảm bảo chạy giống nhau trên mọi môi trường.

```
[App Code] + [Runtime] + [Libraries] + [Config] = Docker Image
Docker Image chạy → Docker Container
```

## Cài đặt

```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

## Các lệnh cơ bản

### Images

```bash
docker pull nginx:latest           # tải image
docker images                      # liệt kê image
docker rmi nginx:latest            # xóa image
docker build -t myapp:1.0 .        # build từ Dockerfile
```

### Containers

```bash
docker run -d -p 8080:80 --name web nginx   # chạy nền, map port
docker ps                                    # container đang chạy
docker ps -a                                 # tất cả container
docker stop web                              # dừng
docker rm web                                # xóa
docker logs -f web                           # xem log realtime
docker exec -it web bash                     # vào trong container
```

## Dockerfile thực tế

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files trước để tận dụng layer cache
COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000
USER node

CMD ["node", "server.js"]
```

Mẹo quan trọng: đặt các lệnh ít thay đổi lên trên để tận dụng build cache.

## Docker Compose cho multi-container

```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 3s
      retries: 5
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

```bash
docker compose up -d      # chạy tất cả service
docker compose down -v    # dừng và xóa volume
docker compose logs -f    # xem log tất cả service
```

## Tối ưu image size

```dockerfile
# Dùng multi-stage build
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
CMD ["node", "dist/server.js"]
```

Kết quả: image giảm từ ~1GB xuống ~150MB.

## Các lỗi thường gặp

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `permission denied` | Chưa thêm user vào group docker | `sudo usermod -aG docker $USER` rồi logout |
| Port đã bị dùng | Port trên host đã có process khác | Đổi port hoặc dừng process đó |
| Container thoát ngay | CMD bị lỗi hoặc process thoát | Kiểm tra `docker logs <id>` |
| Image quá lớn | Dùng base image nặng | Chuyển sang `-alpine` variant |

---

Phần tiếp theo: **Docker networking và volumes** sẽ được đề cập trong bài sau.
