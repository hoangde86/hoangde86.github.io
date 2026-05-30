---
title: "10 lệnh Linux bạn nên biết khi làm DevOps"
date: 2026-05-01 09:00:00 +0700
categories: [Linux, Command Line]
tags: [linux, bash, terminal, devops]
---

Dù bạn làm dev hay ops, những lệnh này sẽ tiết kiệm cho bạn rất nhiều thời gian trong công việc hàng ngày.

## 1. `tmux` — Quản lý nhiều terminal trong một cửa sổ

```bash
tmux new -s mysession       # tạo session mới
tmux attach -t mysession    # kết nối lại session
Ctrl+b d                    # detach khỏi session
Ctrl+b %                    # chia đôi theo chiều dọc
Ctrl+b "                    # chia đôi theo chiều ngang
```

Dùng `tmux` khi SSH vào server — nếu mất kết nối, session vẫn còn chạy.

## 2. `rsync` — Đồng bộ file hiệu quả

```bash
rsync -avz --progress source/ user@server:/dest/
rsync -avz --delete src/ dst/   # xóa file không còn ở nguồn
```

Nhanh hơn `scp` vì chỉ truyền phần thay đổi.

## 3. `jq` — Parse JSON trên terminal

```bash
curl -s https://api.example.com/data | jq '.users[].name'
jq -r '.items[] | select(.status == "active") | .id' data.json
```

Không thể thiếu khi làm việc với API.

## 4. `awk` — Xử lý text theo cột

```bash
awk '{print $1, $3}' file.txt          # in cột 1 và 3
awk -F: '$3 >= 1000 {print $1}' /etc/passwd  # user có UID >= 1000
```

## 5. `ss` — Xem kết nối mạng (thay netstat)

```bash
ss -tlnp          # TCP đang listen, kèm process
ss -s             # thống kê tổng quan
```

## 6. `find` + `xargs` — Tìm và xử lý hàng loạt

```bash
find . -name "*.log" -mtime +30 | xargs rm -f     # xóa log cũ hơn 30 ngày
find . -name "*.py" | xargs grep -l "import os"   # tìm file python dùng os
```

## 7. `lsof` — Xem file/port đang được dùng

```bash
lsof -i :8080         # process nào đang dùng port 8080
lsof -u username      # file được mở bởi user
```

## 8. `strace` — Debug system call

```bash
strace -p <PID>                    # trace process đang chạy
strace -e trace=network ./app      # chỉ theo dõi network call
```

Hữu ích khi ứng dụng bị treo mà không rõ nguyên nhân.

## 9. `watch` — Theo dõi output lặp lại

```bash
watch -n 2 'docker ps'        # cập nhật mỗi 2 giây
watch -d 'cat /proc/meminfo'  # highlight phần thay đổi
```

## 10. `trap` trong shell script — Xử lý cleanup khi thoát

```bash
#!/bin/bash
cleanup() {
    echo "Đang dọn dẹp..."
    rm -f /tmp/myapp.lock
}
trap cleanup EXIT INT TERM

# code chính ở đây
```

Đảm bảo script luôn dọn dẹp dù thoát bình thường hay bị Ctrl+C.

---

Bạn dùng lệnh nào nhiều nhất trong danh sách trên? Để lại comment bên dưới!
