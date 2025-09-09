# Thiết Lập Cụm HA PostgreSQL Với PgBouncer Và HAProxy Trên Ubuntu (Cập Nhật 09/09/2025)

Tài liệu này cung cấp hướng dẫn chi tiết để thiết lập cụm High Availability (HA) cho PostgreSQL với:
- **1 node PostgreSQL**: hostname `vlo-pg01`, IP `192.168.10.20`
- **5 nodes PgBouncer**: hostnames `vlo-pgb01/02/03/04/05`, IPs `192.168.10.24/25/26/27/28`
- **1 node HAProxy**: hostname `vlo-ha01`, IP `192.168.10.29`
- **Hệ điều hành**: Ubuntu 24.04.3 LTS (Noble Numbat, released August 2025)
- **Yêu cầu**: Hỗ trợ 100.000 concurrent connections (CCU)
- **Xác thực**: Sử dụng SCRAM-SHA-256 thay vì MD5 cho bảo mật cao hơn

Cấu hình dựa trên best practices cho HA PostgreSQL với luồng: App -> HAProxy -> PgBouncer -> PostgreSQL. Phiên bản phần mềm (cập nhật đến 09/09/2025):
- PostgreSQL 17 (stable, PostgreSQL 18 đang ở RC1, GA dự kiến 25/09/2025)
- PgBouncer 1.24.1 (hỗ trợ SCRAM-SHA-256 với `auth_type = scram-sha-256`)
- HAProxy 3.2 (LTS đến 2030)

## 1. Yêu Cầu Phần Cứng
Chạy trên máy ảo ESXi, cấu hình tối ưu cho 100k CCU, sử dụng SSD để giảm latency.

| Node Type | Số Node | CPU Cores | RAM | Disk | Lý Do |
|-----------|---------|-----------|-----|------|-------|
| PostgreSQL (vlo-pg01) | 1 | 16-32 cores | 64-128 GB | 1 TB SSD (RAID 10) | Xử lý queries nặng, cần CPU/RAM cao. SSD giảm IO latency. |
| PgBouncer (vlo-pgb01/02/03/04/05) | 5 | 4-8 cores mỗi | 8-16 GB mỗi | 100 GB SSD | Xử lý ~20k connections mỗi node để chia tải. |
| HAProxy (vlo-ha01) | 1 (gợi ý thêm node thứ 2) | 2-4 cores | 4-8 GB | 50 GB SSD | Xử lý 100k connections TCP. |

- **Tổng ước tính**: 38-64 CPU cores, 108-192 GB RAM, 1.5-1.7 TB SSD.
- **Network**: NIC 10Gbps trên tất cả nodes.
- **Lưu ý**: Test với `pgbench`. Nếu workload nặng, tăng CPU/RAM. Dùng Prometheus để monitor.

## 2. Cài Đặt Và Cấu Hình
Cập nhật hệ thống trước: `sudo apt update && sudo apt upgrade -y`.

### a. PostgreSQL trên vlo-pg01 (IP: 192.168.10.20)
- **Cài đặt PostgreSQL 17**:
  1. Thêm repository:
     ```bash
     sudo apt install wget ca-certificates
     wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
     sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
     sudo apt update
     ```
  2. Cài: `sudo apt install postgresql-17`
  3. Kiểm tra: `sudo systemctl status postgresql`
  4. Tạo user/DB với SCRAM-SHA-256:
     ```bash
     sudo -u postgres psql
     ALTER SYSTEM SET password_encryption = 'scram-sha-256';
     \password myuser  # Nhập mật khẩu, e.g., 'mypassword'
     CREATE DATABASE mydb OWNER myuser;
     ```

- **Cấu hình**:
  1. `/etc/postgresql/17/main/postgresql.conf`:
     ```bash
     listen_addresses = '*'
     max_connections = 1000  # Tăng cho 100k CCU (với pooling)
     shared_buffers = 16GB
     work_mem = 32MB
     ```
     Restart: `sudo systemctl restart postgresql`
  2. `/etc/postgresql/17/main/pg_hba.conf`:
     ```bash
     host all all 192.168.10.0/24 scram-sha-256
     ```
     Reload: `sudo systemctl reload postgresql`
  3. Firewall: `sudo ufw allow 5432/tcp`

### b. PgBouncer trên vlo-pgb01/02/03/04/05 (IPs: 192.168.10.24-28)
Lặp lại cho từng node.
- **Cài đặt PgBouncer 1.24.1**:
  1. Cài: `sudo apt install pgbouncer`
  2. User list: `/etc/pgbouncer/userlist.txt`:
     ```bash
     "myuser" "SCRAM-SHA-256$4096:<salt>$<stored_key>:<server_key>"
     ```
     - Lấy SCRAM-SHA-256 hash:
       ```bash
       sudo -u postgres psql -c "SELECT passwd FROM pg_shadow WHERE usename = 'myuser';"
       ```
       Kết quả sẽ có dạng: `SCRAM-SHA-256$4096:<salt>$<stored_key>:<server_key>`. Sao chép vào `userlist.txt`.
  3. `/etc/pgbouncer/pgbouncer.ini`:
     ```ini
     [databases]
     mydb = host=192.168.10.20 port=5432 dbname=mydb
     
     [pgbouncer]
     listen_addr = *
     listen_port = 6432
     auth_type = scram-sha-256
     auth_file = /etc/pgbouncer/userlist.txt
     pool_mode = transaction
     max_client_conn = 100000
     default_pool_size = 200
     reserve_pool_size = 100
     ```
  4. Enable/start: `sudo systemctl enable pgbouncer && sudo systemctl start pgbouncer`
  5. Test: `psql -h <IP_PgBouncer> -p 6432 -U myuser -d mydb`

### c. HAProxy trên vlo-ha01 (IP: 192.168.10.29)
- **Cài đặt HAProxy 3.2**:
  1. Thêm PPA: `sudo add-apt-repository ppa:vbernat/haproxy-3.2 -y`
     `sudo apt update`
  2. Cài: `sudo apt install haproxy`
  3. `/etc/haproxy/haproxy.cfg`:
     ```haproxy
     global
         log /dev/log local0
         log /dev/log local1 notice
         chroot /var/lib/haproxy
         stats socket /run/haproxy/admin.sock mode 660 level admin
         stats timeout 30s
         user haproxy
         group haproxy
         daemon
     
     defaults
         log global
         mode tcp
         option dontlognull
         timeout connect 5000
         timeout client 100000
         timeout server 100000
     
     frontend pg_frontend
         bind *:5432
         default_backend pg_backend
     
     backend pg_backend
         mode tcp
         balance leastconn
         server pgb01 192.168.10.24:6432 check
         server pgb02 192.168.10.25:6432 check
         server pgb03 192.168.10.26:6432 check
         server pgb04 192.168.10.27:6432 check
         server pgb05 192.168.10.28:6432 check
     ```
  4. Enable/restart: `sudo systemctl enable haproxy && sudo systemctl restart haproxy`
  5. Test: `psql -h 192.168.10.29 -p 5432 -U myuser -d mydb`

## 3. Tối Ưu Hóa Hệ Điều Hành
Áp dụng trên tất cả nodes để hỗ trợ 100k CCU. Các script chạy với quyền root (`sudo`).

### a. Script Kernel Parameters (Chạy trên tất cả: vlo-pg01, vlo-pgb01/02/03/04/05, vlo-ha01)
Tối ưu network, memory, I/O. File: `optimize_kernel_100k.sh`

```bash
#!/bin/bash
SYSCTL_FILE="/etc/sysctl.conf"

# Sao lưu
cp $SYSCTL_FILE ${SYSCTL_FILE}.bak

# Thêm hoặc cập nhật params
cat <<EOT >> $SYSCTL_FILE

# Network tuning for 100k CCU
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 32768
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 262144
net.core.netdev_max_backlog = 5000
net.core.rmem_max = 33554432
net.core.wmem_max = 33554432
net.ipv4.tcp_rmem = 4096 87380 33554432
net.ipv4.tcp_wmem = 4096 65536 33554432
net.ipv4.tcp_fin_timeout = 10

# Memory và I/O
vm.swappiness = 5
vm.dirty_ratio = 5
vm.dirty_background_ratio = 2
vm.overcommit_memory = 2
vm.overcommit_ratio = 95
fs.file-max = 4194304
EOT

# Áp dụng
sysctl -p
echo "Kernel parameters optimized for 100k CCU."
```

- **Chạy**: `chmod +x optimize_kernel_100k.sh && sudo ./optimize_kernel_100k.sh`
- **Tham số**: Network (somaxconn, tcp_max_syn_backlog, tcp_tw_reuse, rmem/wmem), memory (swappiness, overcommit), file-max.

### b. Script File Descriptors Và Systemd (Chạy trên vlo-pg01)
Tăng giới hạn file mở. File: `optimize_pg_limits.sh`

```bash
#!/bin/bash
LIMITS_FILE="/etc/security/limits.conf"
SYSTEMD_SYS="/etc/systemd/system.conf"
SYSTEMD_USER="/etc/systemd/user.conf"

# Sao lưu
cp $LIMITS_FILE ${LIMITS_FILE}.bak
cp $SYSTEMD_SYS ${SYSTEMD_SYS}.bak
cp $SYSTEMD_USER ${SYSTEMD_USER}.bak

# Thêm limits cho postgres
echo "postgres soft nofile 131072" >> $LIMITS_FILE
echo "postgres hard nofile 131072" >> $LIMITS_FILE

# Systemd defaults
sed -i '/^#DefaultLimitNOFILE=/d' $SYSTEMD_SYS
echo "DefaultLimitNOFILE=131072" >> $SYSTEMD_SYS
sed -i '/^#DefaultLimitNOFILE=/d' $SYSTEMD_USER
echo "DefaultLimitNOFILE=131072" >> $SYSTEMD_USER

# Reload
systemctl daemon-reload
echo "Limits optimized for PostgreSQL."
```

- **Chạy**: `chmod +x optimize_pg_limits.sh && sudo ./optimize_pg_limits.sh`
- **Tham số**: nofile (131072), DefaultLimitNOFILE.

### c. Script I/O Scheduler (Chạy trên vlo-pg01, disk /dev/sda)
Tối ưu I/O cho SSD. File: `optimize_io.sh`

```bash
#!/bin/bash
DISK="sda"
UDEV_RULE="/etc/udev/rules.d/60-scheduler.rules"

# Tạm thời
echo deadline > /sys/block/$DISK/queue/scheduler

# Vĩnh viễn
echo 'ACTION=="add|change", KERNEL=="'$DISK'", ATTR{queue/scheduler}="deadline"' > $UDEV_RULE

# Reload
udevadm control --reload-rules && udevadm trigger
echo "I/O scheduler optimized."
```

- **Chạy**: `chmod +x optimize_io.sh && sudo ./optimize_io.sh`
- **Tham số**: I/O scheduler (deadline).

### d. Script Tắt Transparent Huge Pages (Chạy trên vlo-pg01)
Tắt THP để giảm latency. File: `disable_thp.sh`

```bash
#!/bin/bash
SERVICE_FILE="/etc/systemd/system/disable-thp.service"

# Tạm thời
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# Vĩnh viễn
cat <<EOT > $SERVICE_FILE
[Unit]
Description=Disable Transparent Huge Pages
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOT

# Enable
systemctl enable disable-thp
systemctl start disable-thp
echo "THP disabled."
```

- **Chạy**: `chmod +x disable_thp.sh && sudo ./disable_thp.sh`
- **Tham số**: transparent_hugepage (never).

### e. Script File Descriptors Và Service (Chạy trên vlo-pgb01/02/03/04/05)
Tăng file descriptors cho PgBouncer. File: `optimize_pgb_limits.sh`

```bash
#!/bin/bash
LIMITS_FILE="/etc/security/limits.conf"

# Sao lưu
cp $LIMITS_FILE ${LIMITS_FILE}.bak

# Thêm limits cho pgbouncer
echo "pgbouncer soft nofile 131072" >> $LIMITS_FILE
echo "pgbouncer hard nofile 131072" >> $LIMITS_FILE

# Chỉnh service
systemctl edit pgbouncer --full --force
sed -i '/LimitNOFILE=/d' /etc/systemd/system/pgbouncer.service
echo "LimitNOFILE=131072" >> /etc/systemd/system/pgbouncer.service

# Reload và restart
systemctl daemon-reload
systemctl restart pgbouncer
echo "Limits optimized for PgBouncer."
```

- **Chạy**: `chmod +x optimize_pgb_limits.sh && sudo ./optimize_pgb_limits.sh`
- **Tham số**: nofile (131072), LimitNOFILE.

### f. Script File Descriptors Và Service (Chạy trên vlo-ha01)
Tăng file descriptors cho HAProxy. File: `optimize_ha_limits.sh`

```bash
#!/bin/bash
LIMITS_FILE="/etc/security/limits.conf"

# Sao lưu
cp $LIMITS_FILE ${LIMITS_FILE}.bak

# Thêm limits cho haproxy
echo "haproxy soft nofile 131072" >> $LIMITS_FILE
echo "haproxy hard nofile 131072" >> $LIMITS_FILE

# Chỉnh service
systemctl edit haproxy --full --force
sed -i '/LimitNOFILE=/d' /etc/systemd/system/haproxy.service
echo "LimitNOFILE=131072" >> /etc/systemd/system/haproxy.service

# Reload và restart
systemctl daemon-reload
systemctl restart haproxy
echo "Limits optimized for HAProxy."
```

- **Chạy**: `chmod +x optimize_ha_limits.sh && sudo ./optimize_ha_limits.sh`
- **Tham số**: nofile (131072), LimitNOFILE.

### g. Script Tạo `/etc/pgbouncer/userlist.txt` (Chạy trên vlo-pgb01/02/03/04/05)
Tạo file userlist với SCRAM-SHA-256 hash. File: `create_pgb_userlist_scram.sh`

```bash
#!/bin/bash
USERLIST_FILE="/etc/pgbouncer/userlist.txt"
PG_USER="myuser"

# Lấy SCRAM-SHA-256 hash từ PostgreSQL
SCRAM_HASH=$(sudo -u postgres psql -t -c "SELECT passwd FROM pg_shadow WHERE usename = '$PG_USER';" | tr -d '[:space:]')

# Tạo file userlist.txt
cat <<EOT > $USERLIST_FILE
"$PG_USER" "$SCRAM_HASH"
EOT

# Chỉnh quyền
chown pgbouncer:pgbouncer $USERLIST_FILE
chmod 600 $USERLIST_FILE

echo "Created $USERLIST_FILE with SCRAM-SHA-256 hash for $PG_USER."
```

- **Chạy**: `chmod +x create_pgb_userlist_scram.sh && sudo ./create_pgb_userlist_scram.sh`

## 4. Lưu Ý Chung Và Test
- **Security**:
  - Firewall chỉ mở port 5432 (PG/HAProxy), 6432 (PgBouncer):
    ```bash
    sudo ufw allow 5432/tcp
    sudo ufw allow 6432/tcp
    ```
  - Sử dụng SSL/TLS cho kết nối (cấu hình trong `postgresql.conf` và `pgbouncer.ini`).
- **Monitoring**: Cài Prometheus + Grafana để theo dõi connections, CPU, RAM, network (e.g., TIME_WAIT sockets, file-nr).
- **Test HA**: Tắt một PgBouncer, kiểm tra kết nối:
  ```bash
  psql -h 192.168.10.29 -p 5432 -U myuser -d mydb
  ```
- **Benchmark**: Mô phỏng 100k CCU:
  ```bash
  pgbench -h 192.168.10.29 -p 5432 -U myuser -c 1000 -T 60 mydb
  ```
  Tăng `-c` dần để kiểm tra bottlenecks.
- **Script**: Sao lưu file cấu hình trước khi chạy script. Kiểm tra log nếu lỗi.
- **Nâng cấp**: Cập nhật PostgreSQL 18 sau 25/09/2025 nếu stable. Thêm node HAProxy với Keepalived nếu cần.
- **SCRAM-SHA-256**: Đảm bảo PostgreSQL và PgBouncer đều dùng SCRAM-SHA-256. Hash lấy từ `pg_shadow` phải khớp.