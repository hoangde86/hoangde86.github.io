---
title: "PostgreSQL: Những kỹ thuật tối ưu query cho Developer"
date: 2026-05-19 09:00:00 +0700
categories: [Database, PostgreSQL]
tags: [postgresql, database, sql, performance, tips]
---

PostgreSQL là database mạnh mẽ nhất trong hệ relational database open-source. Bài này tập trung vào các kỹ thuật thực tế.

## EXPLAIN ANALYZE — Hiểu query plan

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id) as order_count
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.name;
```

Những điều cần chú ý trong output:
- **Seq Scan** trên bảng lớn → cần index
- **cost=X..Y**: Y là estimated total cost
- **actual time**: thời gian thực
- **rows**: nếu estimated << actual → thống kê lỗi thời, chạy `ANALYZE`

## Index đúng chỗ

```sql
-- Index thông thường
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- Partial index (index chỉ phần dữ liệu cần)
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';

-- Index cho LIKE tìm kiếm
CREATE INDEX idx_users_email_pattern ON users USING gin(email gin_trgm_ops);

-- Composite index (thứ tự quan trọng: equality trước, range sau)
CREATE INDEX idx_orders_user_status_date ON orders(user_id, status, created_at);
```

## CTEs và Window Functions

```sql
-- Tính rank doanh thu theo tháng
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', created_at) AS month,
        SUM(amount) AS revenue
    FROM orders
    WHERE status = 'completed'
    GROUP BY 1
),
ranked AS (
    SELECT
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_month,
        RANK() OVER (ORDER BY revenue DESC) AS revenue_rank
    FROM monthly_revenue
)
SELECT
    TO_CHAR(month, 'MM/YYYY') AS thang,
    revenue,
    ROUND((revenue - prev_month) / prev_month * 100, 1) AS tang_truong_pct,
    revenue_rank
FROM ranked
ORDER BY month;
```

## UPSERT và Conflict Handling

```sql
-- Insert hoặc update nếu đã tồn tại
INSERT INTO user_stats (user_id, login_count, last_login)
VALUES ($1, 1, NOW())
ON CONFLICT (user_id) DO UPDATE SET
    login_count = user_stats.login_count + 1,
    last_login = EXCLUDED.last_login;

-- Insert chỉ khi chưa tồn tại
INSERT INTO email_subscriptions (email, created_at)
VALUES ($1, NOW())
ON CONFLICT (email) DO NOTHING;
```

## Connection Pooling với PgBouncer

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction      # transaction pooling hiệu quả nhất
max_client_conn = 1000
default_pool_size = 20
min_pool_size = 5
```

Ứng dụng kết nối đến PgBouncer (port 6432), không phải Postgres trực tiếp. Giải quyết vấn đề "too many connections".

## Partitioning cho bảng lớn

```sql
-- Tạo bảng parent với partition by range
CREATE TABLE logs (
    id BIGSERIAL,
    created_at TIMESTAMPTZ NOT NULL,
    level TEXT,
    message TEXT
) PARTITION BY RANGE (created_at);

-- Tạo partition theo tháng
CREATE TABLE logs_2026_05 PARTITION OF logs
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE TABLE logs_2026_06 PARTITION OF logs
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

-- Index trên partition
CREATE INDEX ON logs_2026_05 (created_at, level);
```

## Backup và Restore

```bash
# Backup một database
pg_dump -Fc -j 4 mydb > mydb_$(date +%Y%m%d).dump

# Restore
pg_restore -Fc -j 4 -d mydb_new mydb_20260523.dump

# Backup all databases
pg_dumpall > all_databases.sql

# Point-in-time recovery → cần WAL archiving (topic riêng)
```

---

Query chậm nhất thường không phải do thiếu index mà do **N+1 problem** ở application layer — kiểm tra với `pg_stat_statements`.
