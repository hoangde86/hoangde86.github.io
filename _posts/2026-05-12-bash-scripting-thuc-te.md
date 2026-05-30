---
title: "Bash Scripting thực tế: Viết script đáng tin cậy"
date: 2026-05-12 09:00:00 +0700
categories: [Linux, Scripting]
tags: [bash, shell, scripting, automation, linux]
---

Bash script viết sai rất dễ gây ra lỗi âm thầm. Bài này tập trung vào cách viết script **an toàn** và **dễ bảo trì**.

## Luôn bắt đầu với safe mode

```bash
#!/usr/bin/env bash
set -euo pipefail
IFS=$'\n\t'
```

- `set -e`: thoát ngay khi có lệnh lỗi
- `set -u`: lỗi khi dùng biến chưa khai báo
- `set -o pipefail`: pipe lỗi nếu bất kỳ command nào trong pipe lỗi
- `IFS=$'\n\t'`: tránh lỗi khi tên file có dấu cách

## Logging đúng cách

```bash
readonly LOG_FILE="/var/log/myscript.log"
readonly SCRIPT_NAME=$(basename "$0")

log()   { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO]  $*" | tee -a "$LOG_FILE"; }
warn()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WARN]  $*" | tee -a "$LOG_FILE" >&2; }
error() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [ERROR] $*" | tee -a "$LOG_FILE" >&2; }
die()   { error "$*"; exit 1; }
```

## Kiểm tra dependency trước khi chạy

```bash
check_deps() {
    local deps=("docker" "kubectl" "jq" "curl")
    for dep in "${deps[@]}"; do
        command -v "$dep" >/dev/null 2>&1 || die "$dep không được cài đặt"
    done
}

check_deps
```

## Xử lý tham số dòng lệnh

```bash
usage() {
    cat <<EOF
Usage: $SCRIPT_NAME [OPTIONS] <environment>

Options:
  -t, --tag     Docker image tag (mặc định: latest)
  -d, --dry-run Chạy thử không thực hiện thật
  -h, --help    Hiển thị hướng dẫn

Ví dụ:
  $SCRIPT_NAME -t v1.2.3 production
  $SCRIPT_NAME --dry-run staging
EOF
}

DOCKER_TAG="latest"
DRY_RUN=false

while [[ $# -gt 0 ]]; do
    case $1 in
        -t|--tag)     DOCKER_TAG="$2"; shift 2 ;;
        -d|--dry-run) DRY_RUN=true; shift ;;
        -h|--help)    usage; exit 0 ;;
        -*)           die "Option không hợp lệ: $1" ;;
        *)            ENVIRONMENT="$1"; shift ;;
    esac
done

[[ -z "${ENVIRONMENT:-}" ]] && { usage; die "Thiếu environment"; }
```

## Retry với exponential backoff

```bash
retry() {
    local max_attempts=5
    local delay=1
    local attempt=1

    while true; do
        if "$@"; then
            return 0
        fi

        if (( attempt >= max_attempts )); then
            error "Thất bại sau $max_attempts lần thử: $*"
            return 1
        fi

        warn "Lần $attempt thất bại. Thử lại sau ${delay}s..."
        sleep "$delay"
        (( attempt++ ))
        (( delay *= 2 ))
    done
}

# Dùng:
retry curl -sf https://api.example.com/health
```

## Xử lý file an toàn

```bash
# Luôn dùng mktemp cho file tạm
TMPFILE=$(mktemp /tmp/myscript.XXXXXX)
trap 'rm -f "$TMPFILE"' EXIT

# Kiểm tra file tồn tại trước khi dùng
[[ -f "$CONFIG_FILE" ]] || die "File không tồn tại: $CONFIG_FILE"
[[ -r "$CONFIG_FILE" ]] || die "Không có quyền đọc: $CONFIG_FILE"

# Backup trước khi ghi đè
cp "$CONFIG_FILE" "${CONFIG_FILE}.bak.$(date +%Y%m%d_%H%M%S)"
```

## Script deploy thực tế

```bash
#!/usr/bin/env bash
set -euo pipefail

readonly APP_NAME="myapp"
readonly COMPOSE_FILE="/opt/${APP_NAME}/docker-compose.yml"

deploy() {
    local tag="$1"
    log "Bắt đầu deploy ${APP_NAME}:${tag}"

    log "Pull image mới..."
    docker pull "ghcr.io/myorg/${APP_NAME}:${tag}"

    log "Cập nhật tag trong compose file..."
    sed -i "s|image:.*|image: ghcr.io/myorg/${APP_NAME}:${tag}|" "$COMPOSE_FILE"

    log "Rolling restart..."
    docker compose -f "$COMPOSE_FILE" up -d --no-deps --remove-orphans "${APP_NAME}"

    log "Kiểm tra health..."
    retry curl -sf http://localhost:3000/health

    log "Deploy ${tag} thành công!"
}

deploy "${1:?Tag là bắt buộc}"
```

---

Script tốt không phải script chạy được — mà là script **dễ debug khi gặp lỗi lúc 2 giờ sáng**.
