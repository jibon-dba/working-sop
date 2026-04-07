# 📜 Standard Operating Procedure (SOP)
## Scaling PostgreSQL HA Cluster — Adding a New Node to an Existing Patroni + etcd Cluster

**Scope:** This guide covers adding a new database node to an existing cluster running on RHEL 9 / Rocky Linux 9.

**Stack:** PostgreSQL 16 · Patroni · etcd · HAProxy (optional) · Keepalived

---

## Demo Environment Reference

| Role | Hostname | IP Address |
|---|---|---|
| Existing Leader | `db-node-01` | `192.168.10.11` |
| Existing Replica 1 | `db-node-02` | `192.168.10.12` |
| Existing Replica 2 | `db-node-03` | `192.168.10.13` |
| **New Node (to be added)** | **`db-node-04`** | **`192.168.10.14`** |
| VIP (HAProxy/Keepalived) | `db-vip` | `192.168.10.10` |

> All commands and configuration snippets in this document use the demo values above. Substitute with your actual values when deploying.

---

## Prerequisites

- An existing healthy cluster (Leader + at least 1 Replica).
- Root or sudo access to all nodes (existing and new).
- New VM provisioned with network connectivity to all existing nodes.
- `etcdctl` available on at least one existing node.

---

## Phase 0: Pre-Flight Checks & Planning

### Step 0.1 — Verify Existing Cluster Health

Run the following on any existing node before touching anything:

```bash
# Check etcd health
etcdctl endpoint health \
  --endpoints=http://192.168.10.11:2379,http://192.168.10.12:2379,http://192.168.10.13:2379

# Check Patroni cluster status
patronictl -c /etc/patroni/patroni.yml list
```

**Expected result:** All nodes must report `healthy` and all replicas must show `streaming` state. Do not proceed until the cluster is fully healthy.

---

## Phase 1: System Preparation (On the New Node — `db-node-04`)

### Step 1.1 — SSH Into the New Node and Set Hostname

```bash
sudo hostnamectl set-hostname db-node-04
```

### Step 1.2 — Update `/etc/hosts` on All Nodes

You must update `/etc/hosts` on **the new node** AND **every existing node**.

Add the following block to `/etc/hosts` on `db-node-04`:

```bash
sudo tee -a /etc/hosts << EOF
192.168.10.11   db-node-01
192.168.10.12   db-node-02
192.168.10.13   db-node-03
192.168.10.14   db-node-04
EOF
```

> Repeat this same addition on `db-node-01`, `db-node-02`, and `db-node-03` — adding only the new line `192.168.10.14 db-node-04` to each existing node's `/etc/hosts`.

### Step 1.3 — Disable Firewall and SELinux

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld

sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
sudo setenforce 0
```

> A full reboot is required for the SELinux config change to persist. `setenforce 0` makes it effective immediately for the current session.

### Step 1.4 — Create the `postgres` System User

```bash
sudo useradd postgres
sudo passwd postgres

echo "postgres ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/postgres
sudo chmod 440 /etc/sudoers.d/postgres
```

### Step 1.5 — Install PostgreSQL 16

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib
```

> ⚠️ **DO NOT run `postgresql-16-setup initdb`.** Patroni will initialize the data directory during its own startup.

### Step 1.6 — Install Patroni and Supporting Packages

```bash
sudo dnf install -y patroni patroni-etcd watchdog etcd
```

---

## Phase 2: etcd Configuration (Critical)

> ⚠️ **Critical Order:** You must add the new member to the existing etcd cluster **before** starting etcd on the new node. Skipping this causes a split-brain or startup failure.

### Step 2.1 — Add New Member to the Existing etcd Cluster

**Run this on an existing healthy node (e.g., `db-node-01`):**

```bash
etcdctl member add db-node-04 \
  --peer-urls=http://192.168.10.14:2380 \
  --endpoints=http://192.168.10.11:2379
```

**Copy the output carefully.** It will provide:
- An updated `ETCD_INITIAL_CLUSTER` string containing all old nodes plus the new node.
- The instruction to set `ETCD_INITIAL_CLUSTER_STATE="existing"`.

You will use both values in Step 2.2.

---

### Step 2.2 — Configure etcd on the New Node

Choose **one** of the two options below depending on how your existing nodes are configured.

---

#### Option A: Systemd Environment Variables *(Recommended for RHEL 9)*

Use this if your existing nodes configure etcd via `/etc/systemd/system/etcd.service`.

```bash
sudo nano /etc/systemd/system/etcd.service
```

Paste the following (substitute the `ETCD_INITIAL_CLUSTER` value from Step 2.1):

```ini
[Unit]
Description=etcd key-value store
After=network.target

[Service]
User=etcd
Type=simple

# Data directory
Environment=ETCD_DATA_DIR=/var/lib/etcd
Environment=ETCD_NAME=db-node-04

# Network listeners (0.0.0.0 allows connections from all interfaces)
Environment=ETCD_LISTEN_PEER_URLS=http://0.0.0.0:2380
Environment=ETCD_LISTEN_CLIENT_URLS=http://0.0.0.0:2379

# Advertised URLs (must use this node's specific IP)
Environment=ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.168.10.14:2380
Environment=ETCD_ADVERTISE_CLIENT_URLS=http://192.168.10.14:2379

# Cluster config — paste the ETCD_INITIAL_CLUSTER value from Step 2.1
Environment=ETCD_INITIAL_CLUSTER="db-node-01=http://192.168.10.11:2380,db-node-02=http://192.168.10.12:2380,db-node-03=http://192.168.10.13:2380,db-node-04=http://192.168.10.14:2380"

# Token — must match the existing cluster token exactly
Environment=ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster-demo

# State — MUST be "existing" when joining a running cluster
Environment=ETCD_INITIAL_CLUSTER_STATE="existing"

ExecStart=/usr/bin/etcd
Restart=always
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

Create the data directory and set ownership:

```bash
sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
```

Reload systemd and start etcd:

```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

---

#### Option B: File-Based Configuration (`/etc/etcd/etcd.conf`)

Use this if your existing nodes configure etcd via a config file.

```bash
sudo nano /etc/etcd/etcd.conf
```

Paste the following:

```ini
ETCD_NAME="db-node-04"
ETCD_DATA_DIR="/var/lib/etcd"

ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"

ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.10.14:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.10.14:2379"

# Paste the ETCD_INITIAL_CLUSTER value from Step 2.1
ETCD_INITIAL_CLUSTER="db-node-01=http://192.168.10.11:2380,db-node-02=http://192.168.10.12:2380,db-node-03=http://192.168.10.13:2380,db-node-04=http://192.168.10.14:2380"

ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-demo"

# CRITICAL: Must be "existing" — not "new"
ETCD_INITIAL_CLUSTER_STATE="existing"
ETCD_ENABLE_V2="true"
```

Create the data directory and set ownership:

```bash
sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
```

Ensure the systemd service unit references this config file via `EnvironmentFile=` or a command argument, then start:

```bash
sudo systemctl enable etcd
sudo systemctl start etcd
```

---

### Step 2.3 — Verify the New Node Joined etcd

Run from the new node or any existing node:

```bash
etcdctl endpoint health \
  --endpoints=http://192.168.10.14:2379,http://192.168.10.11:2379
```

All endpoints must return `healthy` before proceeding to Phase 3.

---

## Phase 3: Patroni Configuration

### Step 3.1 — Prepare Data Directories (On `db-node-04`)

```bash
sudo mkdir -p /data/pgsql/16/data
sudo chown -R postgres:postgres /data/pgsql/16
sudo chmod 700 /data/pgsql/16/data

sudo mkdir -p /run/postgresql
sudo chown postgres:postgres /run/postgresql
sudo chmod 775 /run/postgresql

# Ensure the runtime directory persists across reboots
echo 'd /run/postgresql 0775 postgres postgres -' | sudo tee /etc/tmpfiles.d/postgresql.conf
```

### Step 3.2 — Update `pg_hba` on the Existing Leader

Before starting Patroni on the new node, the cluster leader must be configured to allow replication connections from the new node's IP.

**Run on the existing leader (`db-node-01`):**

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml edit-config pg-demo-cluster
```

> Replace `pg-demo-cluster` with your actual Patroni `scope` name.

Add the new node's IP to the `pg_hba` list inside the config:

```yaml
postgresql:
  pg_hba:
    - host replication replicator 192.168.10.11/32 scram-sha-256
    - host replication replicator 192.168.10.12/32 scram-sha-256
    - host replication replicator 192.168.10.13/32 scram-sha-256
    - host replication replicator 192.168.10.14/32 scram-sha-256   # <-- ADD THIS LINE
    - host all all 0.0.0.0/0 scram-sha-256
```

Save and exit. Then reload the cluster config:

```bash
sudo -u postgres patronictl -c /etc/patroni/patroni.yml reload pg-demo-cluster db-node-01
```

### Step 3.3 — Create Patroni Configuration on the New Node

Create `/etc/patroni/patroni.yml` on `db-node-04`:

```yaml
scope: pg-demo-cluster
namespace: /db/
name: db-node-04

restapi:
  listen: 0.0.0.0:8008
  connect_address: 192.168.10.14:8008

etcd3:
  hosts: 192.168.10.11:2379,192.168.10.12:2379,192.168.10.13:2379,192.168.10.14:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
  initdb:
    - encoding: UTF8
    - data-checksums
  pg_hba:
    - host replication replicator 192.168.10.11/32 scram-sha-256
    - host replication replicator 192.168.10.12/32 scram-sha-256
    - host replication replicator 192.168.10.13/32 scram-sha-256
    - host replication replicator 192.168.10.14/32 scram-sha-256
    - host all all 0.0.0.0/0 scram-sha-256

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 192.168.10.14:5432
  data_dir: /data/pgsql/16/data
  bin_dir: /usr/pgsql-16/bin
  authentication:
    replication:
      username: replicator
      password: <replicator_password>
    superuser:
      username: postgres
      password: <postgres_password>
  parameters:
    unix_socket_directories: '/run/postgresql'

watchdog:
  mode: off
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
```

> 🔐 Replace `<replicator_password>` and `<postgres_password>` with your actual credentials. Ensure they match the values used on existing nodes.

### Step 3.4 — Disable the Standalone PostgreSQL Service

Patroni manages PostgreSQL directly. The native PostgreSQL service must not run alongside it.

```bash
sudo systemctl stop postgresql-16
sudo systemctl disable postgresql-16
```

### Step 3.5 — Start Patroni

```bash
sudo systemctl enable patroni
sudo systemctl start patroni

# Watch logs to confirm successful join
sudo journalctl -u patroni -f
```

Patroni will perform a base backup (`pg_basebackup`) from the leader automatically and start streaming replication. This may take several minutes depending on database size.

---

## Phase 4: Verification

### Step 4.1 — Check Cluster Status

Run on any node:

```bash
patronictl -c /etc/patroni/patroni.yml list
```

**Expected output:**

| Member | Host | Role | State | TL | Lag in MB |
|---|---|---|---|---|---|
| db-node-01 | 192.168.10.11:5432 | Leader | running | 1 | |
| db-node-02 | 192.168.10.12:5432 | Replica | streaming | 1 | 0 |
| db-node-03 | 192.168.10.13:5432 | Replica | streaming | 1 | 0 |
| db-node-04 | 192.168.10.14:5432 | Replica | streaming | 1 | 0 |

Confirm:
- `db-node-04` appears in the list.
- Role is `Replica`.
- State is `streaming`.
- Lag is `0`.

### Step 4.2 — Verify Replication on the Leader

Connect to the leader and confirm the new node appears in `pg_stat_replication`:

```bash
psql -h 192.168.10.11 -p 5432 -U postgres \
  -c "SELECT client_addr, state, sync_state FROM pg_stat_replication;"
```

You should see `192.168.10.14` listed with `state = streaming`.

### Step 4.3 — Test Read/Write via Proxy (If HAProxy Is in Use)

```bash
# Write test — routes to Leader via port 5000
psql -h 192.168.10.10 -p 5000 -U postgres -c "SELECT 1;"

# Read test — routes to replicas (round-robin) via port 5001
psql -h 192.168.10.10 -p 5001 -U postgres -c "SELECT 1;"
```

### Step 4.4 — Test Failover (Optional but Recommended)

```bash
patronictl -c /etc/patroni/patroni.yml switchover pg-demo-cluster
```

Observe that a replica is promoted cleanly, and the old leader rejoins as a replica. Confirm the new node (`db-node-04`) participates correctly throughout.

---

## Appendix: Configuring a Node as a Read-Only Standby (No Failover, No Traffic)

If you want `db-node-04` to be a silent standby — no writes routed to it, not eligible for promotion — update its `tags` section in `/etc/patroni/patroni.yml`:

```bash
sudo nano /etc/patroni/patroni.yml
```

Change the `tags` block as follows:

```yaml
tags:
  nofailover: true     # Excludes this node from leader election
  noloadbalance: true  # Excludes this node from read traffic rotation
  clonefrom: false
  nosync: false
```

Then reload Patroni to apply:

```bash
sudo systemctl reload patroni
# or via patronictl:
patronictl -c /etc/patroni/patroni.yml reload pg-demo-cluster db-node-04
```

| Tag | Effect |
|---|---|
| `nofailover: true` | Node will never be promoted to leader during a failover event. |
| `noloadbalance: true` | HAProxy or Patroni-aware clients will not route read queries to this node. |
| Both set to `true` | Node acts as a pure silent standby — replicates data but serves no traffic. |

---

*End of SOP*
