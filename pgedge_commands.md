# 📘 Tài liệu tham số lệnh `pgedge`

## 1. Cú pháp tổng quát
```bash
pgedge [COMMAND] [OPTIONS]
```

---

## 2. Các lệnh chính và tham số

### 2.1 `init` – Khởi tạo cluster mới
```bash
pgedge init --cluster-name <name> --node-name <name> [OPTIONS]
```

| Tham số | Mô tả | Mặc định |
|---------|-------|----------|
| `--cluster-name <name>` | Tên cluster (bắt buộc) | – |
| `--node-name <name>` | Tên node đầu tiên (bắt buộc) | – |
| `--data-dir <path>` | Thư mục dữ liệu PostgreSQL | `/var/lib/postgresql/data` |
| `--host <ip/hostname>` | Địa chỉ node | `localhost` |
| `--port <port>` | Cổng PostgreSQL | `5432` |
| `--user <user>` | User quản trị | `postgres` |
| `--password <pwd>` | Mật khẩu quản trị | – |
| `--replication-user <user>` | User replication | `replicator` |
| `--replication-password <pwd>` | Mật khẩu replication | – |
| `--force` | Ghi đè dữ liệu cũ | – |
| `--yes` | Tự động xác nhận | – |

**Ví dụ**:  
```bash
pgedge init --cluster-name geo_cluster --node-name hanoi   --data-dir /var/lib/postgresql/17/main   --host 10.0.0.1 --port 5432   --user postgres --password 123456
```

---

### 2.2 `add-node` – Thêm node
```bash
pgedge add-node --cluster-name <name> --node-name <name> [OPTIONS]
```

| Tham số | Mô tả |
|---------|-------|
| `--cluster-name <name>` | Tên cluster |
| `--node-name <name>` | Node mới |
| `--host <ip>` | Host/IP node mới |
| `--port <port>` | Port PostgreSQL |
| `--data-dir <path>` | Thư mục dữ liệu node |
| `--sync-method <stream|logical>` | Kiểu đồng bộ (mặc định `stream`) |

**Ví dụ**:  
```bash
pgedge add-node --cluster-name geo_cluster --node-name hcm   --host 10.0.0.2 --port 5432   --user postgres --password 123456
```

---

### 2.3 `status` – Trạng thái
```bash
pgedge status [OPTIONS]
```

| Tham số | Mô tả |
|---------|-------|
| `--cluster-name <name>` | Trạng thái toàn cluster |
| `--node-name <name>` | Trạng thái node |
| `--output <text|json|yaml>` | Định dạng hiển thị |
| `--verbose` | Thông tin chi tiết |

**Ví dụ**:  
```bash
pgedge status --cluster-name geo_cluster --output json
```

---

### 2.4 `backup` – Sao lưu
```bash
pgedge backup --cluster-name <name> --backup-dir <path> [OPTIONS]
```

| Tham số | Mô tả |
|---------|-------|
| `--cluster-name <name>` | Cluster cần backup |
| `--node-name <name>` | Node thực hiện backup |
| `--backup-dir <path>` | Thư mục backup |
| `--backup-type <full|incremental>` | Kiểu backup |
| `--compress` | Bật nén dữ liệu |
| `--include-config` | Sao lưu cả file config |
| `--verbose` | Log chi tiết |

**Ví dụ**:  
```bash
pgedge backup --cluster-name geo_cluster   --node-name hanoi   --backup-type full   --backup-dir /mnt/backups/geo_cluster   --include-config --compress
```

---

### 2.5 `restore` – Phục hồi
```bash
pgedge restore --cluster-name <name> --restore-dir <path> [OPTIONS]
```

| Tham số | Mô tả |
|---------|-------|
| `--cluster-name <name>` | Cluster cần phục hồi |
| `--restore-dir <path>` | Thư mục dữ liệu phục hồi |
| `--backup-dir <path>` | Nguồn backup |
| `--timestamp <time>` | Phục hồi đến thời điểm cụ thể |
| `--force` | Ghi đè dữ liệu cũ |

**Ví dụ**:  
```bash
pgedge restore --cluster-name geo_cluster   --restore-dir /var/lib/postgresql/17/main   --backup-dir /mnt/backups/geo_cluster   --timestamp 2025-09-15T02:00:00Z   --force
```

---

### 2.6 `logs` – Nhật ký
```bash
pgedge logs [OPTIONS]
```

| Tham số | Mô tả |
|---------|-------|
| `--cluster-name <name>` | Lọc theo cluster |
| `--node-name <name>` | Lọc theo node |
| `--tail <N>` | Lấy N dòng cuối |
| `--follow` | Theo dõi realtime (giống `tail -f`) |

**Ví dụ**:  
```bash
pgedge logs --cluster-name geo_cluster --node-name hanoi --tail 100 --follow
```

---

### 2.7 `destroy` – Xóa
```bash
pgedge destroy --cluster-name <name> [OPTIONS]
```

| Tham số | Mô tả |
|---------|-------|
| `--cluster-name <name>` | Cluster cần xóa |
| `--node-name <name>` | Node cụ thể cần xóa |
| `--force` | Ép xóa, bỏ qua xác nhận |

**Ví dụ**:  
```bash
pgedge destroy --cluster-name geo_cluster --node-name hcm --force
```

---

## 3. Lệnh tiện ích khác

- `pgedge start|stop|restart` → Quản lý dịch vụ  
- `pgedge config get|set|list` → Quản lý cấu hình  
- `pgedge upgrade --version <ver>` → Nâng cấp  
- `pgedge version` → Kiểm tra phiên bản  
