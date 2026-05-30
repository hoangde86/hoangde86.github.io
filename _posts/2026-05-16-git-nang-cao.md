---
title: "Git nâng cao: Những kỹ thuật ít người biết"
date: 2026-05-16 09:00:00 +0700
categories: [DevOps, Git]
tags: [git, version-control, workflow, tips]
---

Hầu hết developer chỉ dùng `git add`, `commit`, `push`. Bài này sẽ giúp bạn làm chủ Git ở mức sâu hơn.

## Git Worktrees — Làm việc nhiều branch cùng lúc

```bash
# Tạo worktree mới (không cần stash hay checkout)
git worktree add ../feature-x feature/new-feature
git worktree add ../hotfix-123 hotfix/critical-bug

# Liệt kê
git worktree list

# Xóa khi xong
git worktree remove ../hotfix-123
```

Rất hữu ích khi đang code dở feature nhưng cần fix bug khẩn cấp.

## Interactive Rebase — Dọn dẹp commit history

```bash
git rebase -i HEAD~5   # chỉnh 5 commit gần nhất
```

Trong editor xuất hiện:
```
pick a1b2c3 Thêm login form
pick d4e5f6 Fix typo
pick g7h8i9 WIP: đang làm
pick j1k2l3 Thêm validation
pick m4n5o6 Fix validation bug

# Lệnh:
# p, pick = giữ nguyên
# r, reword = đổi message
# s, squash = gộp vào commit trước
# f, fixup = gộp, bỏ message
# d, drop = xóa commit
```

Kết quả sau rebase sạch hơn:
```
Thêm login form với validation
```

## Git Bisect — Tìm commit gây ra bug

```bash
git bisect start
git bisect bad                    # commit hiện tại là bug
git bisect good v1.2.0            # commit này chưa có bug

# Git tự checkout commit giữa, bạn test rồi đánh dấu:
git bisect good    # hoặc
git bisect bad

# Sau vài lần → Git tìm ra commit gây bug
git bisect reset   # kết thúc
```

## Stash có tên và linh hoạt

```bash
git stash push -m "WIP: form validation" -- src/components/
git stash list
git stash pop stash@{0}
git stash apply stash@{1}    # apply mà không xóa khỏi stash list
git stash drop stash@{1}
```

## Reflog — Cứu commit bị xóa nhầm

```bash
# Lỡ tay reset --hard? Không sao:
git reflog                     # xem lịch sử HEAD
git checkout -b recovery abc1234  # tạo branch từ commit đó
```

`git reflog` giữ history trong 90 ngày — safety net mạnh nhất của Git.

## Hooks tự động

```bash
# .git/hooks/pre-commit (chmod +x)
#!/bin/bash
npm run lint || exit 1
npm test || exit 1
```

```bash
# .git/hooks/commit-msg
#!/bin/bash
MSG=$(cat "$1")
PATTERN="^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,72}$"
[[ "$MSG" =~ $PATTERN ]] || {
    echo "Commit message không đúng format: feat(scope): description"
    exit 1
}
```

## Alias hữu ích

```bash
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.st "status -sb"
git config --global alias.undo "reset HEAD~1 --mixed"
git config --global alias.wip "!git add -A && git commit -m 'WIP'"
```

## .gitattributes — Chuẩn hóa line ending

```
# .gitattributes
* text=auto
*.sh text eol=lf
*.bat text eol=crlf
*.png binary
*.jpg binary
```

Đặc biệt quan trọng khi team có cả Windows lẫn Linux/Mac.

---

Kỹ thuật nào bạn thấy hữu ích nhất? Cá nhân tôi dùng `worktree` và `reflog` nhiều nhất.
