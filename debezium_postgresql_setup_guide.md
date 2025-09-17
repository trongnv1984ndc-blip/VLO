# Hướng dẫn Setup Debezium User cho PostgreSQL

## Tổng quan

Tài liệu này hướng dẫn tạo user `debezium_user` với đầy đủ quyền để thực hiện logical replication trên PostgreSQL. User này sẽ có khả năng:
- Đọc dữ liệu từ tất cả bảng hiện tại và tương lai
- Thực hiện logical replication 
- Tự động có quyền trên các bảng mới được tạo
- Thao tác dữ liệu (INSERT, UPDATE, DELETE) khi cần thiết

## Yêu cầu hệ thống

- PostgreSQL 10+ (khuyến nghị 12+)
- User có quyền SUPERUSER để tạo user và grant permissions
- Logical replication được bật trong PostgreSQL

## Bước 1: Cấu hình PostgreSQL

### 1.1 Cấu hình postgresql.conf

Thêm các dòng sau vào file `postgresql.conf`:

```ini
# Bật logical replication (BẮT BUỘC)
wal_level = logical

# Số lượng replication slots tối đa
max_replication_slots = 20

# Số lượng wal senders tối đa  
max_wal_senders = 20

# Số worker processes cho logical replication
max_logical_replication_workers = 10

# Số sync workers mỗi subscription
max_sync_workers_per_subscription = 4

# Tùy chọn: Load shared libraries
shared_preload_libraries = 'pg_stat_statements'

# Tùy chọn: Cấu hình checkpoint cho performance
checkpoint_timeout = 5min
max_wal_size = 1GB
min_wal_size = 80MB
```

### 1.2 Cấu hình pg_hba.conf

Thêm các dòng sau vào file `pg_hba.conf`:

```
# Cho phép replication connection
host    replication     debezium_user   127.0.0.1/32            md5
host    replication     debezium_user   ::1/128                 md5
host    replication     debezium_user   0.0.0.0/0               md5

# Cho phép database connection
host    all             debezium_user   127.0.0.1/32            md5
host    all             debezium_user   ::1/128                 md5
host    all             debezium_user   0.0.0.0/0               md5
```

### 1.3 Restart PostgreSQL

```bash
# Ubuntu/Debian
sudo systemctl restart postgresql

# CentOS/RHEL
sudo systemctl restart postgresql-12

# Hoặc
sudo pg_ctlcluster 12 main restart
```

## Bước 2: Tạo User và Grant Quyền

### 2.1 Script SQL hoàn chỉnh

Kết nối PostgreSQL với user có quyền superuser và chạy script sau:

```sql
-- ===============================================
-- TẠO USER DEBEZIUM VÀ GRANT QUYỀN HOÀN CHỈNH
-- ===============================================

-- 1. Kết nối với database postgres
\c postgres

-- 2. Tạo user debezium_user với quyền REPLICATION
CREATE USER debezium_user WITH 
    REPLICATION 
    LOGIN 
    PASSWORD 'kynguyenx';

-- 3. Grant quyền CONNECT cho tất cả database hiện tại
DO $$
DECLARE
    db_name TEXT;
BEGIN
    FOR db_name IN 
        SELECT datname FROM pg_database 
        WHERE datistemplate = false
    LOOP
        EXECUTE 'GRANT CONNECT ON DATABASE ' || quote_ident(db_name) || ' TO debezium_user';
    END LOOP;
END $$;

-- ===============================================
-- GRANT QUYỀN CHO TỪNG DATABASE CỤ THỂ
-- ===============================================

-- Danh sách databases cần grant quyền
-- Thay thế với tên database thực tế của bạn
\c vlo_search
GRANT ALL PRIVILEGES ON SCHEMA public TO debezium_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO debezium_user;
DROP PUBLICATION IF EXISTS debezium_publication;
CREATE PUBLICATION debezium_publication FOR ALL TABLES;

\c vlo_contact
GRANT ALL PRIVILEGES ON SCHEMA public TO debezium_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO debezium_user;
DROP PUBLICATION IF EXISTS debezium_publication;
CREATE PUBLICATION debezium_publication FOR ALL TABLES;

\c vlo_spam_evaluator
GRANT ALL PRIVILEGES ON SCHEMA public TO debezium_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO debezium_user;
DROP PUBLICATION IF EXISTS debezium_publication;
CREATE PUBLICATION debezium_publication FOR ALL TABLES;

\c vlo_logging
GRANT ALL PRIVILEGES ON SCHEMA public TO debezium_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO debezium_user;
DROP PUBLICATION IF EXISTS debezium_publication;
CREATE PUBLICATION debezium_publication FOR ALL TABLES;

\c vlo_notification
GRANT ALL PRIVILEGES ON SCHEMA public TO debezium_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO debezium_user;
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO debezium_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO debezium_user;
DROP PUBLICATION IF EXISTS debezium_publication;
CREATE PUBLICATION debezium_publication FOR ALL TABLES;

-- ===============================================
-- GRANT QUYỀN CHO TẤT CẢ SCHEMA KHÔNG PHẢI SYSTEM
-- ===============================================

-- Script tự động grant quyền cho tất cả schemas
DO $$
DECLARE
    db_record RECORD;
    schema_name TEXT;
BEGIN
    FOR db_record IN 
        SELECT datname FROM pg_database 
        WHERE datistemplate = false AND datname != 'postgres'
    LOOP
        -- Connect to each database (note: this won't work in DO block)
        -- You need to run this manually for each database
        RAISE NOTICE 'Please run the following for database: %', db_record.datname;
    END LOOP;
END $$;
```

### 2.2 Script Bash tự động

Tạo file `setup_debezium_user.sh`:

```bash
#!/bin/bash

# PostgreSQL connection parameters
PG_HOST="localhost"
PG_PORT="5432" 
PG_USER="postgres"
DEBEZIUM_USER="debezium_user"
DEBEZIUM_PASSWORD="your_strong_password_here"

# List of databases to grant permissions
DATABASES=("vlo_search" "vlo_contact" "vlo_spam_evaluator" "vlo_logging" "vlo_notification")

echo "=== CREATING DEBEZIUM USER FOR LOGICAL REPLICATION ==="

# 1. Create debezium user with REPLICATION privilege
psql -h $PG_HOST -p $PG_PORT -U $PG_USER -d postgres << EOF
-- Create user with REPLICATION privilege
CREATE USER $DEBEZIUM_USER WITH 
    REPLICATION 
    LOGIN 
    PASSWORD '$DEBEZIUM_PASSWORD';

-- Grant connect on all databases
DO \$\$
DECLARE
    db_name TEXT;
BEGIN
    FOR db_name IN 
        SELECT datname FROM pg_database 
        WHERE datistemplate = false
    LOOP
        EXECUTE 'GRANT CONNECT ON DATABASE ' || quote_ident(db_name) || ' TO $DEBEZIUM_USER';
    END LOOP;
END \$\$;
EOF

echo "✓ Created user $DEBEZIUM_USER with REPLICATION privilege"

# 2. Grant permissions for each database
for db in "${DATABASES[@]}"; do
    echo "Setting up permissions for database: $db"
    
    psql -h $PG_HOST -p $PG_PORT -U $PG_USER -d "$db" << EOF
-- Grant full privileges on schema public
GRANT ALL PRIVILEGES ON SCHEMA public TO $DEBEZIUM_USER;

-- Grant full privileges on all existing tables
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO $DEBEZIUM_USER;

-- Grant default privileges for future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO $DEBEZIUM_USER;

-- Grant full privileges on all sequences
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO $DEBEZIUM_USER;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO $DEBEZIUM_USER;

-- Grant privileges on all functions
GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA public TO $DEBEZIUM_USER;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON FUNCTIONS TO $DEBEZIUM_USER;

-- Grant usage on all types
GRANT USAGE ON ALL TYPES IN SCHEMA public TO $DEBEZIUM_USER;

-- Grant access to system catalogs
GRANT SELECT ON ALL TABLES IN SCHEMA information_schema TO $DEBEZIUM_USER;
GRANT SELECT ON ALL TABLES IN SCHEMA pg_catalog TO $DEBEZIUM_USER;

-- Create publication for logical replication
DROP PUBLICATION IF EXISTS debezium_publication;
CREATE PUBLICATION debezium_publication FOR ALL TABLES;

-- Grant additional schema permissions if they exist
DO \$\$
DECLARE
    schema_name TEXT;
BEGIN
    FOR schema_name IN 
        SELECT nspname FROM pg_namespace 
        WHERE nspname NOT LIKE 'pg_%' 
        AND nspname != 'information_schema'
        AND nspname != 'public'
    LOOP
        EXECUTE 'GRANT ALL PRIVILEGES ON SCHEMA ' || quote_ident(schema_name) || ' TO $DEBEZIUM_USER';
        EXECUTE 'GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA ' || quote_ident(schema_name) || ' TO $DEBEZIUM_USER';
        EXECUTE 'ALTER DEFAULT PRIVILEGES IN SCHEMA ' || quote_ident(schema_name) || ' GRANT ALL ON TABLES TO $DEBEZIUM_USER';
    END LOOP;
END \$\$;
EOF
    
    echo "✓ Completed permissions for database: $db"
done

echo "=== SETUP COMPLETED SUCCESSFULLY ==="
```

### 2.3 Chạy script

```bash
# Cấp quyền thực thi
chmod +x setup_debezium_user.sh

# Chạy script
./setup_debezium_user.sh
```

## Bước 3: Kiểm tra và Xác nhận

### 3.1 Kiểm tra user đã được tạo

```sql
-- Kiểm tra user và quyền replication
SELECT 
    usename,
    userepl,
    usesuper,
    usecreatedb
FROM pg_user 
WHERE usename = 'debezium_user';
```

### 3.2 Kiểm tra quyền trên bảng

```sql
-- Kiểm tra quyền trên database cụ thể
\c vlo_search

SELECT 
    schemaname,
    tablename,
    has_table_privilege('debezium_user', schemaname||'.'||tablename, 'SELECT') as can_select,
    has_table_privilege('debezium_user', schemaname||'.'||tablename, 'INSERT') as can_insert,
    has_table_privilege('debezium_user', schemaname||'.'||tablename, 'UPDATE') as can_update,
    has_table_privilege('debezium_user', schemaname||'.'||tablename, 'DELETE') as can_delete
FROM pg_tables 
WHERE schemaname = 'public'
ORDER BY tablename;
```

### 3.3 Kiểm tra publication

```sql
-- Kiểm tra publications
SELECT 
    pubname,
    puballtables,
    pubinsert,
    pubupdate,
    pubdelete,
    pubtruncate
FROM pg_publication 
WHERE pubname = 'debezium_publication';
```

### 3.4 Test kết nối

```bash
# Test kết nối với user debezium_user
psql -h localhost -U debezium_user -d vlo_search -c "\dt"
psql -h localhost -U debezium_user -d vlo_contact -c "SELECT count(*) FROM users;"
```

### 3.5 Test logical replication

```sql
-- Test tạo replication slot
SELECT pg_create_logical_replication_slot('test_slot', 'pgoutput');

-- Kiểm tra slot đã tạo
SELECT slot_name, plugin, slot_type, active FROM pg_replication_slots;

-- Xóa test slot
SELECT pg_drop_replication_slot('test_slot');
```

## Bước 4: Cấu hình Debezium Connector

### 4.1 Connector configuration example

```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "localhost",
    "database.port": "5432",
    "database.user": "debezium_user",
    "database.password": "your_strong_password_here",
    "database.dbname": "vlo_search",
    "database.server.name": "postgres-server",
    "plugin.name": "pgoutput",
    "publication.name": "debezium_publication",
    "table.include.list": "public.users,public.user_preferences,public.bank",
    "slot.name": "debezium_slot"
  }
}
```

## Troubleshooting

### Lỗi thường gặp

1. **"permission denied for table"**
   - Kiểm tra user có quyền trên bảng cụ thể
   - Chạy lại script grant permissions

2. **"replication slot does not exist"**
   - Kiểm tra cấu hình `wal_level = logical`
   - Restart PostgreSQL sau khi cấu hình

3. **"publication does not exist"**
   - Tạo lại publication trong database cần thiết
   - Kiểm tra user có quyền trên publication

### Commands hữu ích

```sql
-- Xem tất cả replication slots
SELECT * FROM pg_replication_slots;

-- Xem tất cả publications
SELECT * FROM pg_publication;

-- Xem quyền của user
\du debezium_user

-- Xem default privileges
\ddp
```