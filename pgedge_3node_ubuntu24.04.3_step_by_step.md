# pgEdge Distributed Postgres (Multi-Master) — 3 Nodes on Ubuntu 24.04.3

**Profile:** `--pg_ver 17` • DB user `admin` password **`Adm1n#2024`** • Nodes: `pgedge01`, `pgedge02`, `pgedge03`

---

## 0) Assumptions & Topology

- Hosts (adjust to your environment):
  - `pgedge01` = `192.168.10.150`
  - `pgedge02` = `192.168.10.151`
  - `pgedge03` = `192.168.10.152`
- OS user for deployment: **`pgops`** (non-root with passwordless sudo).
- Database name: **`appdb`**
- Database superuser: **`admin`** with password **`Adm1n#2024`**
- PostgreSQL port: **5432**

> You can use another non-root user (e.g., `ubuntu`). Replace the username consistently in all commands.

---

## 1) OS Preparation (run on **each** node)

```bash
# 1. Update packages & install basics
sudo apt update && sudo apt -y upgrade
sudo apt -y install curl openssh-client ufw python3 python3-venv

# 2. (Recommended) Create an ops user with passwordless sudo
sudo adduser --disabled-password --gecos "" pgops
echo "pgops ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/pgops

# 3. (Recommended) SSH key for pgops (generate once per node, copy to the others)
sudo -iu pgops bash <<'EOF'
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
for h in pgedge01 pgedge02 pgedge03; do
  ssh-copy-id -i ~/.ssh/id_ed25519.pub pgops@$h
done
EOF

# 4. /etc/hosts for name resolution (run on all nodes)
sudo tee -a /etc/hosts >/dev/null <<'EOF'
192.168.10.150  pgedge01
192.168.10.151  pgedge02
192.168.10.152  pgedge03
EOF

# 5. Firewall (UFW) — allow Postgres among nodes
sudo ufw allow from 192.168.10.150 to any port 5432 proto tcp
sudo ufw allow from 192.168.10.151 to any port 5432 proto tcp
sudo ufw allow from 192.168.10.152 to any port 5432 proto tcp
sudo ufw enable
```

---

## 2) Install **pgEdge CLI** (on **each** node)

> This installs the CLI into `~/pgedge` of the current user (here, `pgops`).

```bash
sudo -iu pgops bash -lc "python3 -c \"\$(curl -fsSL https://downloads.pgedge.com/platform/repos/download/install.py)\""
```
## set environment varible 
echo 'export PATH='/opt/pgedge/bin:
$PATH >> ~/.bashrc
source ~/.bashrc

---

## 3) Initialize Postgres + pgEdge (on **each** node)

> Create DB `appdb`, superuser `admin` with **`Adm1n#2024`**, using **PostgreSQL 17**, and enable autostart.

```bash
# Switch to pgops and run
sudo -iu pgops bash
cd ~/pgedge
./pgedge setup -U admin -P 'Adm1n#2024' -d appdb --pg_ver 17 --autostart True

# Quick checks
./pgedge info
./pgedge service status
```

**GUC suggestions:**

```bash
./pgedge db guc-set listen_addresses '*'
./pgedge db guc-set track_commit_timestamp on
./pgedge service reload
```

> If you enforce host-based auth, add appropriate `pg_hba.conf` entries for the three node IPs (use `md5`/`scram`).

---

## 4) Create **Spock nodes** (multi-master identity)

Run these on the respective machines. DSN uses the OS user (`pgops`) for seamless local auth.

```bash
# On pgedge01
cd ~/pgedge
./pgedge spock node-create n1 'host=192.168.10.150 port=5432 user=pgops dbname=appdb' appdb

# On pgedge02
./pgedge spock node-create n2 'host=192.168.10.151 port=5432 user=pgops dbname=appdb' appdb

# On pgedge03
./pgedge spock node-create n3 'host=192.168.10.152 port=5432 user=pgops dbname=appdb' appdb
```

> Naming nodes `n1/n2/n3` plays nicely with pgEdge Snowflake sequences.

---

## 5) Create **subscriptions** (full-mesh 3-way)

Each node both publishes and subscribes. For 3 nodes, you need **6 subscriptions** (two per pair).

```bash
# On n1 (pgedge01) — subscribe to n2 & n3
./pgedge spock sub-create sub_n1n2 'host=192.168.10.151 port=5432 user=pgops dbname=appdb' appdb
./pgedge spock sub-create sub_n1n3 'host=192.168.10.152 port=5432 user=pgops dbname=appdb' appdb

# On n2 (pgedge02) — subscribe to n1 & n3
./pgedge spock sub-create sub_n2n1 'host=192.168.10.150 port=5432 user=pgops dbname=appdb' appdb
./pgedge spock sub-create sub_n2n3 'host=192.168.10.152 port=5432 user=pgops dbname=appdb' appdb

# On n3 (pgedge03) — subscribe to n1 & n2
./pgedge spock sub-create sub_n3n1 'host=192.168.10.150 port=5432 user=pgops dbname=appdb' appdb
./pgedge spock sub-create sub_n3n2 'host=192.168.10.151 port=5432 user=pgops dbname=appdb' appdb
```

> When adding a new node to an existing cluster with data, consider `--synchronize_structure` and/or `--synchronize_data` flags in `sub-create`.

---

## 6) Declare tables to replicate (replication sets)

Spock only replicates tables included in replication sets. Add by pattern, schema, or specific table(s).

```bash
# Recommended: run on all three nodes for symmetric replication
./pgedge spock repset-add-table default 'public.*' appdb

# Or add a specific table later
./pgedge spock repset-add-table default 'public.demo' appdb
```

> Replication starts **after** the table is in the repset. Pre-existing rows are **not** auto-synced unless you seeded data or used the synchronize flags on subscription creation.

---

## 7) Quick replication test

```bash
# List Spock nodes (any node)
./pgedge spock node-list appdb

# On n1: create table & add to repset
psql -U admin -d appdb -h 192.168.10.150 -c "CREATE TABLE IF NOT EXISTS demo(id int primary key, note text);"
./pgedge spock repset-add-table default 'public.demo' appdb

# Insert a NEW row after adding to repset
psql -U admin -d appdb -h 192.168.10.150 -c "INSERT INTO demo VALUES (1, 'hello from n1');"

# Check on n2 & n3
psql -U admin -d appdb -h 192.168.10.151 -c "TABLE demo;"
psql -U admin -d appdb -h 192.168.10.152 -c "TABLE demo;"
```

If new rows don’t appear, inspect status and replication plumbing:

```bash
# CLI (per-subscription status)
./pgedge spock sub-show-status sub_n1n2 appdb
./pgedge spock sub-show-status sub_n1n3 appdb
./pgedge spock sub-show-status sub_n2n1 appdb
./pgedge spock sub-show-status sub_n2n3 appdb
./pgedge spock sub-show-status sub_n3n1 appdb
./pgedge spock sub-show-status sub_n3n2 appdb

# SQL (all subscriptions status)
psql -U admin -d appdb -c "SELECT * FROM spock.sub_show_status();"

# SQL (basic attributes — note: no sub_status column)
psql -U admin -d appdb -c "SELECT sub_name, sub_enabled, sub_replication_sets FROM spock.subscription;"

# Optionally wait for initial sync of a given subscription
psql -U admin -d appdb -c "SELECT spock.sub_wait_for_sync('sub_n1n2');"
```

---

## 8) Useful administration commands

```bash
# Install info
./pgedge info

# Manage services (Postgres/Spock/etc. under pgEdge)
./pgedge service status
./pgedge service restart
./pgedge service enable

# Upgrade CLI if needed
python3 -c "$(curl -fsSL https://downloads.pgedge.com/platform/repos/download/upgrade-cli.py)"
```

---

## 9) Practical notes

- DSNs in `node-create` / `sub-create` use the **OS user** (e.g., `pgops`) for convenience and consistent permissions.
- Enable `track_commit_timestamp on` to support conflict resolution modes like “last update wins.”
- **Single-primary HA vs. Multi-master**:
  - If you only need **1 writer with automatic failover**, use **Patroni + etcd + HAProxy**.
  - **pgEdge/Spock** is for **multi-master** (many writers) across sites/regions.
- Adding a new node to a live cluster? Prefer `--synchronize_structure` / `--synchronize_data` on `sub-create` to seed schema/data cleanly.

---

### Appendix — Reset DB user `admin` password

```bash
psql -U admin -d appdb -h 127.0.0.1 -c "ALTER USER admin WITH PASSWORD 'Adm1n#2024';"
```

**Done.** Copy/paste the commands above in order to bring up a fully-working 3-node pgEdge multi-master cluster on Ubuntu 24.04.3 with PostgreSQL 17.
