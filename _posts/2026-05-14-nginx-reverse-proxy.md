---
title: "Cấu hình Nginx làm Reverse Proxy với SSL miễn phí"
date: 2026-05-14 09:00:00 +0700
categories: [DevOps, Web Server]
tags: [nginx, ssl, reverse-proxy, certbot, linux]
---

Nginx + Let's Encrypt là combo phổ biến nhất để expose ứng dụng ra internet với HTTPS hoàn toàn miễn phí.

## Cài đặt

```bash
# Ubuntu
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

## Cấu hình cơ bản: HTTP → App

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        # Headers cần thiết
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";

        # Timeout
        proxy_connect_timeout 60s;
        proxy_read_timeout    300s;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

## Cấp SSL với Certbot

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot tự động:
1. Xác thực domain qua HTTP challenge
2. Tạo certificate
3. Chỉnh sửa nginx config để dùng HTTPS
4. Thiết lập cron tự động gia hạn

## Cấu hình SSL tối ưu (sau khi có cert)

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL certificates từ Certbot
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Cấu hình SSL an toàn
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # HSTS (bật sau khi đã test kỹ)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header Referrer-Policy no-referrer-when-downgrade;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Redirect HTTP → HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$host$request_uri;
}
```

## Load Balancing nhiều backend

```nginx
upstream app_backend {
    least_conn;   # gửi đến server ít kết nối nhất

    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000 backup;  # dùng khi 2 server kia down
}

server {
    location / {
        proxy_pass http://app_backend;
    }
}
```

## Rate Limiting để chống DDoS

```nginx
# Khai báo ở http block
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

# Áp dụng ở location
location /api/ {
    limit_req zone=api burst=20 nodelay;
    limit_req_status 429;
    proxy_pass http://127.0.0.1:3000;
}
```

## Kiểm tra cấu hình

```bash
sudo nginx -t                     # test config
sudo nginx -T | grep ssl          # xem config hiện tại
sudo certbot renew --dry-run      # test gia hạn cert
curl -I https://example.com       # kiểm tra headers
```

---

Certificate Let's Encrypt có hiệu lực 90 ngày, Certbot sẽ tự gia hạn trước 30 ngày — không cần lo.
