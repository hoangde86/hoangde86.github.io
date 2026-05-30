---
title: "CI/CD thực tế với GitHub Actions: Build, Test và Deploy tự động"
date: 2026-05-07 09:00:00 +0700
categories: [DevOps, CI/CD]
tags: [github-actions, cicd, automation, devops]
---

GitHub Actions cho phép tự động hóa toàn bộ quy trình từ khi push code đến khi deploy — miễn phí cho public repo.

## Khái niệm cơ bản

```
Trigger (push/PR/schedule)
  └── Workflow (.github/workflows/*.yml)
        └── Job (chạy trên runner)
              └── Steps (các lệnh thực thi)
```

## Workflow CI đơn giản cho Node.js

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18, 20, 22]

    steps:
      - uses: actions/checkout@v4

      - name: Setup Node ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm test -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v4
```

## Deploy lên server qua SSH

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: test   # chỉ deploy khi test pass

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Push to registry
        run: |
          echo ${{ secrets.REGISTRY_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            docker pull ghcr.io/${{ github.repository }}:${{ github.sha }}
            docker stop myapp || true
            docker rm myapp || true
            docker run -d --name myapp \
              -p 3000:3000 \
              --env-file /etc/myapp/.env \
              ghcr.io/${{ github.repository }}:${{ github.sha }}
```

## Secrets và Environment Variables

```yaml
# Dùng secrets từ Settings > Secrets and variables
- run: echo "${{ secrets.API_KEY }}"   # ❌ không làm vậy!

# Đúng cách — gán vào env
env:
  API_KEY: ${{ secrets.API_KEY }}
```

Thêm secrets tại: **Repo Settings → Secrets and variables → Actions**

## Cache để build nhanh hơn

```yaml
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-npm-
```

Tiết kiệm 1-2 phút mỗi lần chạy cho project Node.js trung bình.

## Workflow chạy theo lịch (Cron)

```yaml
on:
  schedule:
    - cron: '0 2 * * *'   # 2 AM UTC hàng ngày = 9 AM VN
```

Dùng cho: backup database, report tự động, dependency update check.

## Phê duyệt trước khi deploy production

```yaml
jobs:
  deploy-prod:
    environment:
      name: production   # cần approval trong Settings > Environments
    steps:
      - run: ./deploy-prod.sh
```

---

GitHub Actions là đủ cho hầu hết dự án nhỏ đến vừa. Chỉ cần chuyển sang Jenkins/GitLab CI khi cần self-hosted runner hoặc tích hợp on-premise.
</content>
