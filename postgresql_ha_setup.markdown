# Thiết Lập Cụm HA PostgreSQL Với PgBouncer Và HAProxy Trên Ubuntu

Tài liệu này cung cấp hướng dẫn chi tiết để thiết lập cụm High Availability (HA) cho PostgreSQL với:
- **1 node PostgreSQL**: hostname `vlo-pg01`, IP `192.168.10.20`
- **3 nodes PgBouncer**: hostnames `vlo-pgb01/02/03`, IPs `192.168.10.24/25/26`
- **1 node HAProxy**: hostname `vlo-ha01`, IP `192.168.10.27`
- **Hệ điều hành**: Ubuntu 24.04.3 LTS (Noble Numbat, released August 2025)
- **Yêu cầu**: Hỗ trợ 10.000 concurrent connections (CCU)

Cấu hình dựa trên best practices cho HA PostgreSQL với luồng: App -> HAProxy -> PgBouncer -> PostgreSQL. Phiên bản phần mềm:
- PostgreSQL 17 (stable, 18 đang ở RC1, GA dự kiến 25/09/2025)
- PgBouncer 1.24.1 (fix CVE-2025-2291)
- HAProxy 3.2 (LTS đến 2030)

## 1. Yêu Cầu Phần Cứng
Chạy trên máy ảo ESXi, cấu hình tối ưu cho 10k CCU, sử dụng SSD để giảm latency.

| Node Type | Số Node | CPU Cores | RAM | Disk | Lý Do |
|-----------|---------|-----------|-----|------|-------|
| PostgreSQL (vlo-pg01) | 1 | 8-16 cores | 32-64 GB | 500 GB SSD | Xử lý queries, caching. Pool size ~500-1000 connections. |
| PgBouncer (vlo-pgb01/02/03) | 3 | 2-4 cores mỗi | 4-8 GB mỗi | 50-100 GB HDD/SSD | Pooling nhẹ, mỗi node ~3k-5k connections. |
| HAProxy (vlo-ha01) | 1 | 1-2 cores | 2-4 GB | 50 GB HDD | Load balancer TCP nhẹ. |

- **Tổng ước tính**: 20-30 CPU cores, 50-100 GB RAM, 800-1000 GB disk.
- **Lưu ý**: Test với `pgbench`. Nếu workload nặng, tăng CPU/RAM cho PG. Dùng Prometheus để monitor.

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
  4. Tạo user/DB:
     ```bash
     sudo -u postgres psql
     CREATE USER kynguyenx WITH PASSWORD 'kynguyenx';
     GRANT CONNECT ON DATABASE postgres TO kynguyenx;
     ```

- **Cấu hình**:
  1. `/etc/postgresql/17/main/postgresql.conf`:
     ```bash
     listen_addresses = '*'
     max_connections = 500
     shared_buffers = 8GB
     work_mem = 16M
     ```
     Restart: `sudo systemctl restart postgresql`
  2. `/etc/postgresql/17/main/pg_hba.conf`:
     ```bash
     host all all 192.168.10.0/24 scram-sha-256
     ```
     Reload: `sudo systemctl reload postgresql`
  3. Firewall: `sudo ufw allow 5432/tcp`

### b. PgBouncer trên vlo-pgb01/02/03 (IPs: 192.168.10.24-26)
Lặp lại cho từng node.
- **Cài đặt PgBouncer 1.24.1**:
  1. Cài: `sudo apt install pgbouncer`
  2. User list: `/etc/pgbouncer/userlist.txt`:
     ```bash
     "myuser" "SCRAM-SHA-256$4096:B57k...$Cx7h...$Cplz..."
     ```
     (Lấy hash: `sudo -u postgres psql -c "SELECT rolname, rolpassword FROM pg_authid WHERE rolname='kynguyenx';"`)
  3. `/etc/pgbouncer/pgbouncer.ini`:
     ```ini
     [databases]
     mydb = host=192.168.10.20 port=5432 dbname=mydb
     
     [pgbouncer]
     listen_addr = *
     listen_port = 6432
     auth_type = md5
     auth_file = /etc/pgbouncer/userlist.txt
     pool_mode = transaction
     max_client_conn = 10000
     default_pool_size = 100
     reserve_pool_size = 50
     ```
  4. Enable/start: `sudo systemctl enable pgbouncer && sudo systemctl start pgbouncer`
  5. Test: `psql -h <IP_PgBouncer> -p 6432 -U myuser -d mydb`

### c. HAProxy trên vlo-ha01 (IP: 192.168.10.27)
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
         timeout client 50000
         timeout server 50000
     
     frontend pg_frontend
         bind *:5432
         default_backend pg_backend
     
     backend pg_backend
         mode tcp
         balance roundrobin
         server pgb01 192.168.10.24:6432 check
         server pgb02 192.168.10.25:6432 check
         server pgb03 192.168.10.26:6432 check
     ```
  4. Enable/restart: `sudo systemctl enable haproxy && sudo systemctl restart haproxy`
  5. Test: `psql -h 192.168.10.27 -p 5432 -U myuser -d mydb`

## 3. Tối Ưu Hóa Hệ Điều Hành
Áp dụng trên tất cả nodes để hỗ trợ 10k CCU. Các script tự động hóa chạy với quyền root (`sudo`).

### a. Script Kernel Parameters (Chạy trên tất cả: vlo-pg01, vlo-pgb01/02/03, vlo-ha01)
Tối ưu network stack, memory, I/O. File: `optimize_kernel.sh`

```bash
#!/bin/bash
SYSCTL_FILE="/etc/sysctl.conf"

# Sao lưu
cp $SYSCTL_FILE ${SYSCTL_FILE}.bak

# Thêm hoặc cập nhật params
cat <<EOT >> $SYSCTL_FILE

# Network tuning for DB HA
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_fin_timeout = 15

# Memory và I/O
vm.swappiness = 10
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.overcommit_memory = 2
vm.overcommit_ratio = 95
fs.file-max = 2097152
EOT

# Áp dụng
sysctl -p
echo "Kernel parameters optimized."
```

- **Chạy**: `chmod +x optimize_kernel.sh && sudo ./optimize_kernel.sh`
- **Tham số**: Network (somaxconn, tcp_max_syn_backlog, tcp_tw_reuse, rmem/wmem), memory (swappiness, overcommit), file-max.

### b. Script File Descriptors Và Systemd (Chạy trên vlo-pg01)
Tăng giới hạn file mở cho PostgreSQL. File: `optimize_pg_limits.sh`

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
echo "postgres soft nofile 65535" >> $LIMITS_FILE
echo "postgres hard nofile 65535" >> $LIMITS_FILE

# Systemd defaults
sed -i '/^#DefaultLimitNOFILE=/d' $SYSTEMD_SYS
echo "DefaultLimitNOFILE=65535" >> $SYSTEMD_SYS
sed -i '/^#DefaultLimitNOFILE=/d' $SYSTEMD_USER
echo "DefaultLimitNOFILE=65535" >> $SYSTEMD_USER

# Reload
systemctl daemon-reload
echo "Limits optimized for PostgreSQL."
```

- **Chạy**: `chmod +x optimize_pg_limits.sh && sudo ./optimize_pg_limits.sh`
- **Tham số**: nofile (65535), DefaultLimitNOFILE.

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

### e. Script File Descriptors Và Service (Chạy trên vlo-pgb01/02/03)
Tăng file descriptors cho PgBouncer. File: `optimize_pgb_limits.sh`

```bash
#!/bin/bash
LIMITS_FILE="/etc/security/limits.conf"

# Sao lưu
cp $LIMITS_FILE ${LIMITS_FILE}.bak

# Thêm limits cho pgbouncer
echo "pgbouncer soft nofile 65535" >> $LIMITS_FILE
echo "pgbouncer hard nofile 65535" >> $LIMITS_FILE

# Chỉnh service
systemctl edit pgbouncer --full --force
sed -i '/LimitNOFILE=/d' /etc/systemd/system/pgbouncer.service
echo "LimitNOFILE=65535" >> /etc/systemd/system/pgbouncer.service

# Reload và restart
systemctl daemon-reload
systemctl restart pgbouncer
echo "Limits optimized for PgBouncer."
```

- **Chạy**: `chmod +x optimize_pgb_limits.sh && sudo ./optimize_pgb_limits.sh`
- **Tham số**: nofile (65535), LimitNOFILE.

### f. Script File Descriptors Và Service (Chạy trên vlo-ha01)
Tăng file descriptors cho HAProxy. File: `optimize_ha_limits.sh`

```bash
#!/bin/bash
LIMITS_FILE="/etc/security/limits.conf"

# Sao lưu
cp $LIMITS_FILE ${LIMITS_FILE}.bak

# Thêm limits cho haproxy
echo "haproxy soft nofile 65535" >> $LIMITS_FILE
echo "haproxy hard nofile 65535" >> $LIMITS_FILE

# Chỉnh service
systemctl edit haproxy --full --force
sed -i '/LimitNOFILE=/d' /etc/systemd/system/haproxy.service
echo "LimitNOFILE=65535" >> /etc/systemd/system/haproxy.service

# Reload và restart
systemctl daemon-reload
systemctl restart haproxy
echo "Limits optimized for HAProxy."
```

- **Chạy**: `chmod +x optimize_ha_limits.sh && sudo ./optimize_ha_limits.sh`
- **Tham số**: nofile (65535), LimitNOFILE.

## 4. Lưu Ý Chung Và Test
- **Security**: Firewall chỉ mở port 5432 (PG/HAProxy), 6432 (PgBouncer). Dùng SSL/TLS.
- **Monitoring**: Cài Prometheus + Grafana để theo dõi.
- **Test HA**: Tắt một PgBouncer, kiểm tra kết nối qua `psql -h 192.168.10.27 -p 5432 -U myuser -d mydb`.
- **Benchmark**: `pgbench -h 192.168.10.27 -p 5432 -U myuser -c 100 -T 60 mydb` (tăng -c để simulate 10k CCU).
- **Script**: Chạy script sau khi cài đặt phần mềm. Sao lưu file cấu hình trước. Kiểm tra log nếu lỗi.
- **Nâng cấp**: Cập nhật PostgreSQL 18 sau 25/09/2025 nếu stable. Nếu cần replication, tích hợp Patroni.
