# Red Hat 9 High Availability Cluster — Ansible Playbook

**`ha_cluster_setup.yml`** · RHEL 9 · Pacemaker 2.1.x · Corosync 3.x · PostgreSQL 18

> This document is written for someone who has never used `ha_cluster_setup.yml` before and needs to build a cluster from scratch. Every step, variable, warning, and troubleshooting method is covered in full.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Inventory and Variable Configuration](#3-inventory-and-variable-configuration)
4. [Pre-Installation Steps](#4-pre-installation-steps)
5. [Installation Steps (Play Flow)](#5-installation-steps-play-flow)
6. [Cluster Resource Architecture](#6-cluster-resource-architecture)
7. [Usage](#7-usage)
8. [Verification and Monitoring](#8-verification-and-monitoring)
9. [Troubleshooting](#9-troubleshooting)
10. [Maintenance Operations](#10-maintenance-operations)
11. [Known Limitations](#11-known-limitations)

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Cluster: ha_webpgsql                          │
│                                                                      │
│  ┌──────────────────────┐          ┌──────────────────────┐          │
│  │       clstr01        │◄────────►│       clstr02        │          │
│  │   192.168.0.61       │ corosync │   192.168.0.62       │          │
│  │   172.16.16.61       │   ring   │   172.16.16.62       │          │
│  │  (enp1s0/enp2s0)     │          │  (enp1s0/enp2s0)     │          │
│  └──────────┬───────────┘          └───────────┬──────────┘          │
│             │                                  │                      │
│             └─────────────┬────────────────────┘                     │
│                           │ iSCSI (172.16.16.248:3260)               │
│                    ┌──────┴───────┐                                  │
│                    │  iSCSI LUNs  │                                  │
│                    │              │                                   │
│  /dev/mapper/pv_ha_shared01 (100G)│ → shared_vg → gfs2_lv           │
│                    │   (GFS2)     │   /shared/webfs  [Active/Active]  │
│                    │              │                                   │
│  /dev/mapper/pv_ha_shared02 (200G)│ → db_vg → pgsql_lv              │
│                    │   (ext4)     │   /var/lib/pgsql [Active/Passive] │
│                    └──────────────┘                                  │
│                                                                      │
│  Virtual IP: 192.168.0.63/23  (always on the active node)           │
│                                                                      │
│  ┌──────────────────────┐                                            │
│  │    clstr-qrm01       │  Quorum Device (ffsplit, port 5403)        │
│  │    192.168.0.67      │  NOT a cluster member — arbitrator only    │
│  │   (qnetd arbiter)    │                                            │
│  └──────────────────────┘                                            │
└──────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Value | Description |
|---|---|---|
| Cluster name | `ha_webpgsql` | Pacemaker/Corosync cluster name |
| Node 1 | `clstr01.lab.akyuz.tech` | 192.168.0.61 (public) / 172.16.16.61 (corosync) |
| Node 2 | `clstr02.lab.akyuz.tech` | 192.168.0.62 (public) / 172.16.16.62 (corosync) |
| Quorum device | `clstr-qrm01.lab.akyuz.tech` | 192.168.0.67 — ffsplit arbitrator |
| Virtual IP | `192.168.0.63/23` | Lives on the active node, automatic failover |
| iSCSI target | `172.16.16.248` | Shared storage server |
| GFS2 disk | `/dev/mapper/pv_ha_shared01` → `shared_vg` | `/shared/webfs` — Active/Active (both nodes read/write) |
| PostgreSQL disk | `/dev/mapper/pv_ha_shared02` → `db_vg` | `/var/lib/pgsql` — Active/Passive (VIP node only) |
| PostgreSQL | v18 (PGDG) | Managed as a Pacemaker cluster resource |
| STONITH | `fence_ipmilan` | Out-of-band fencing via IPMI/iDRAC/iLO |
| Multipath | `device-mapper-multipath` | Provides a stable `/dev/mapper/` name for iSCSI disks |

### Design Rationale

**Why two separate iSCSI disks?**
- The GFS2 disk (`shared_vg`) is accessed simultaneously by both nodes via DLM + lvmlockd. It is suitable for shared data such as web content.
- The PostgreSQL disk (`db_vg`) requires exclusive single-node access. Using a separate disk prevents VG lock conflicts and I/O isolation issues.

**Why multipath (`/dev/mapper/pv_ha_shared0x`)?**
- iSCSI paths receive different kernel device names on each node (`sda`, `sdb`, `sdc`, `sdd`). Multipath generates a WWID-based stable alias to avoid this.
- In case of path failure (e.g., multiple HBAs/paths), it provides automatic load balancing and failover.
- The WWID → alias mapping is managed centrally via `multipath_aliases` in `hosts.yml`.

**Why a Quorum Device (qnetd)?**
- In a 2-node cluster, each node holds 1 vote. If one node goes down, the remaining node cannot achieve quorum (`1/2 < majority`). qnetd provides a neutral 3rd vote: `2/3 > majority` → cluster keeps running.
- `ffsplit` algorithm: in a network split, if both sides lose connectivity to the qdevice, the smaller side stops itself.

**Why is STONITH mandatory?**
- In a split-brain scenario (network partition), both nodes believe they are active and write to the shared disk simultaneously → data corruption.
- STONITH (Shoot The Other Node In The Head): the node to be evicted is rebooted at the hardware level via IPMI/iDRAC/iLO, guaranteeing it loses access to the shared disk.

---

## 2. Prerequisites

### 2.1 Control Machine (Ansible / Bastion Host)

The machine that will run the playbook (bastion or management server):

```bash
# Check Ansible version (>= 2.12 required)
ansible --version

# Install the required Red Hat System Roles collection
ansible-galaxy collection install redhat.rhel_system_roles

# Distribute the SSH key to all nodes
ssh-copy-id root@192.168.0.61
ssh-copy-id root@192.168.0.62
ssh-copy-id root@192.168.0.67

# Connectivity test
ansible -i inventory/hosts.yml all -m ping
```

> **Note:** `psql` binary is **not required** on the bastion. The password-setting step (PLAY 13b) runs directly on a cluster node.

### 2.2 Cluster Nodes (clstr01, clstr02)

Required for both nodes:

- **Operating System:** RHEL 9.x (tested on 9.4, 9.5, 9.6, 9.7)
- **Registration:** Active RHSM (`subscription-manager status`) or a local repo
- **SSH:** Direct `root` access or a user with `sudo NOPASSWD`
- **RAM:** Minimum 2 GB (checked by the playbook)
- **Disk (OS):** Minimum 20 GB free on `/` (checked by the playbook)
- **NICs:** At least 2:
  - `enp1s0` → public/VIP network (192.168.0.0/23)
  - `enp2s0` → corosync ring network (172.16.16.0/24)
- **iSCSI LUN:** Access to the iSCSI target (172.16.16.248:3260)
- **IPMI/iDRAC/iLO:** **Mandatory** for STONITH; must be verified before installation

> **Critical Warning:** Without STONITH (`fence_ipmilan`), Pacemaker cannot protect data integrity in a split-brain situation. The playbook verifies IPMI access; installation stops if IPMI is unreachable.

### 2.3 Quorum Device (clstr-qrm01)

- **Operating System:** RHEL 9.x
- **Role:** NOT a cluster member — only runs `corosync-qnetd`
- **SSH:** `root` access
- **Network:** TCP 5403 must be reachable from cluster nodes
- **Repo:** Active repo for `corosync-qnetd` and `pcs` packages

### 2.4 Network Requirements

| Source | Destination | Port | Protocol | Purpose |
|---|---|---|---|---|
| cluster nodes ↔ cluster nodes | — | 5405, 5406 | UDP | Corosync ring |
| cluster nodes ↔ cluster nodes | — | 2224, 2225 | TCP | pcsd |
| cluster nodes ↔ cluster nodes | — | 3121 | TCP | Pacemaker |
| cluster nodes ↔ cluster nodes | — | 21064 | TCP | DLM |
| cluster nodes → quorum device | — | 5403 | TCP | qnetd connection |
| cluster nodes → iSCSI target | — | 3260 | TCP | iSCSI |
| cluster nodes → IPMI | — | 623 (or custom) | UDP/TCP | STONITH fencing |
| bastion → cluster nodes | — | 22 | TCP | SSH (Ansible) |
| bastion → quorum device | — | 22 | TCP | SSH (Ansible) |
| clients → VIP | — | 5432 | TCP | PostgreSQL access |

### 2.5 iSCSI and Multipath Pre-Configuration

The iSCSI target must be configured and LUNs presented before running the playbook.

**WWID Detection** (must be done before installation):

```bash
# Connect via iscsiadm first
iscsiadm -m discovery -t st -p 172.16.16.248
iscsiadm -m node -T iqn.2025-01.com.sirket:storage01 -p 172.16.16.248 --login

# Start multipathd
systemctl start multipathd
multipath -v2

# View WWIDs (the value inside parentheses is the WWID)
multipath -ll
```

Example output:
```
mpatha (360014050b56232984714db8a4bdd0ea9) dm-2 LIO-ORG,lv_ha_shared01_
size=100G ...
mpathb (360014056d2949ca6a2b455d82d2c6b37) dm-3 LIO-ORG,lv_ha_shared02_
size=200G ...
```

These WWID values are entered into the `multipath_aliases` section of `hosts.yml` (see [Section 3.3](#33-multipath-wwid-definitions)).

---

## 3. Inventory and Variable Configuration

### 3.1 Directory Structure

```
project-directory/
├── ha_cluster_setup.yml          # Main playbook — DO NOT MODIFY
├── inventory/
│   ├── hosts.yml                 # ALL configuration goes here
│   └── group_vars/
│       └── all.yml               # Computed variables — DO NOT MODIFY
```

> **Golden Rule:** All configuration is done in `inventory/hosts.yml`. **No changes** are made to `ha_cluster_setup.yml` or `group_vars/all.yml`.

### 3.2 `hosts.yml` — Full Reference

#### Cluster Nodes (host-level variables)

Defined separately for each node:

```yaml
all:
  children:
    cluster_nodes:
      hosts:
        clstr01.lab.akyuz.tech:
          ansible_host: 192.168.0.61          # SSH connection IP
          ansible_user: root                   # SSH user
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
          cluster_ip: 172.16.16.61             # Corosync ring IP (enp2s0 interface)
          # ---- IPMI / iDRAC / iLO ----
          ipmi_ip: 192.168.1.248               # IPMI management IP
          ipmi_port: 6330                      # IPMI port (standard: 623)
          ipmi_user: "admin"                   # IPMI username
          ipmi_password: "CHANGE_ME"           # !! MUST BE CHANGED !!
          ipmi_lanplus: true                   # true for IPMI v2/iDRAC/iLO

        clstr02.lab.akyuz.tech:
          ansible_host: 192.168.0.62
          ansible_user: root
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
          cluster_ip: 172.16.16.62
          ipmi_ip: 192.168.1.248
          ipmi_port: 6331                      # Different port from clstr01 (shared IPMI gateway)
          ipmi_user: "admin"
          ipmi_password: "CHANGE_ME"
          ipmi_lanplus: true

    quorum_device:
      hosts:
        clstr-qrm01.lab.akyuz.tech:
          ansible_host: 192.168.0.67
          ansible_user: root
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
          cluster_ip: 172.16.16.67             # Management IP
```

#### Global Variables (`vars` section)

```yaml
  vars:
    # ---- ANSIBLE SSH -------------------------------------------------------
    ansible_ssh_common_args: '-o StrictHostKeyChecking=no -o ConnectTimeout=10'

    # ---- CLUSTER BASICS ----------------------------------------------------
    cluster_name: "ha_webpgsql"              # Corosync/GFS2 cluster name
    cluster_password: "CHANGE_ME"           # hacluster user password !! MUST BE CHANGED !!
    cluster_domain: "lab.akyuz.tech"         # FQDN domain portion

    # ---- NETWORK -----------------------------------------------------------
    public_nic: "enp1s0"                     # VIP and public access interface
    cluster_nic: "enp2s0"                    # Corosync ring interface
    virtual_ip: "192.168.0.63"               # Cluster Virtual IP
    virtual_ip_cidr: "23"                    # Subnet mask (192.168.0.0/23)
    public_network: "192.168.0.0/23"         # Public network address

    # ---- iSCSI -------------------------------------------------------------
    iscsi_target_ip: "172.16.16.248"         # iSCSI target server IP
    iscsi_iqn: "iqn.2025-01.com.sirket:storage01"  # iSCSI IQN (get from target)
    iscsi_initiator_base: "iqn.1994-05.lab.local"  # Initiator IQN prefix

    # ---- STORAGE: GFS2 Disk ------------------------------------------------
    shared_disk: "/dev/mapper/pv_ha_shared01"  # Multipath device name
    shared_vg: "shared_vg"                   # GFS2 Volume Group name
    gfs2_lv: "gfs2_lv"                       # GFS2 Logical Volume name
    gfs2_lv_size: "50G"                      # GFS2 LV size
    gfs2_mount: "/shared/webfs"              # GFS2 mount point (Active/Active)
    gfs2_journals: 2                         # MUST EQUAL the number of cluster nodes
    gfs2_lock_table: "webfs"                 # GFS2 lock table name

    # ---- STORAGE: PostgreSQL Disk ------------------------------------------
    db_disk: "/dev/mapper/pv_ha_shared02"    # Multipath device name
    db_vg: "db_vg"                           # PostgreSQL Volume Group name
    pgsql_lv: "pgsql_lv"                     # PostgreSQL Logical Volume name
    pgsql_lv_size: "40G"                     # PostgreSQL LV size
    pgsql_mount: "/var/lib/pgsql"            # PostgreSQL mount point (Active/Passive)

    # ---- MULTIPATH WWID DEFINITIONS ----------------------------------------
    # WWID detection: multipath -ll | grep -E '^\S.*\('
    # !! Update WWIDs for each environment !!
    multipath_aliases:
      - alias: "pv_ha_shared01"              # /dev/mapper/pv_ha_shared01
        wwid: "360014050b56232984714db8a4bdd0ea9"   # GFS2 disk WWID
      - alias: "pv_ha_shared02"              # /dev/mapper/pv_ha_shared02
        wwid: "360014056d2949ca6a2b455d82d2c6b37"   # PostgreSQL disk WWID

    # ---- POSTGRESQL --------------------------------------------------------
    pgsql_version: "18"                      # PostgreSQL major version
    pgsql_port: 5432                         # PostgreSQL listening port
    pgsql_data_dir: "/var/lib/pgsql/data"    # PostgreSQL data directory
    pgsql_allowed_networks:                  # pg_hba.conf access list
      - "192.168.0.0/23"                     # Access from public network
      - "10.10.10.0/24"                      # Access from additional network
      - "127.0.0.1/32"                       # Localhost
    pgsql_password: "CHANGE_ME"             # postgres user password !! MUST BE CHANGED !!

    # ---- QUORUM DEVICE -----------------------------------------------------
    qdevice_port: 5403                       # corosync-qnetd listening port
    qdevice_algorithm: "ffsplit"             # ffsplit recommended for 2-node clusters

    # ---- STONITH -----------------------------------------------------------
    stonith_enabled: true
    stonith_action: "reboot"                 # reboot (recommended) | off

    # ---- CLUSTER RESOURCE TIMEOUTS -----------------------------------------
    resource_start_timeout: "60s"
    resource_stop_timeout: "60s"
    resource_monitor_interval: "30s"
    no_quorum_policy: "freeze"               # "freeze" recommended for 2-node clusters
    resource_stickiness: "100"               # No automatic failback (manual required)
    migration_threshold: "3"                 # 3 failures → exclude node

    # ---- FIREWALL PORTS ----------------------------------------------------
    ha_firewall_ports:
      - "2224/tcp"                           # pcsd
      - "3121/tcp"                           # pacemaker
      - "5405/udp"                           # corosync
      - "5406/udp"                           # corosync
      - "21064/tcp"                          # dlm
      - "9929/tcp"                           # corosync-qnetd
      - "9929/udp"                           # corosync-qnetd
      - "5432/tcp"                           # PostgreSQL
```

### 3.3 Multipath WWID Definitions

The `multipath_aliases` variable differs for each environment. To detect WWIDs:

```bash
# After the iSCSI session is established (or if already connected)
# and multipathd is running:
multipath -ll

# Output format:
# <name> (<WWID>) dm-X <vendor,model>
# size=<size> ...
# |-+- policy=... status=active
# | `- 6:0:0:0 sda 8:0 active ready running
```

The `alias` value must match the `basename` of `shared_disk` and `db_disk`:
- `shared_disk: "/dev/mapper/pv_ha_shared01"` → `alias: "pv_ha_shared01"`
- `db_disk: "/dev/mapper/pv_ha_shared02"` → `alias: "pv_ha_shared02"`

### 3.4 `group_vars/all.yml` — Description

This file contains variables automatically derived from `hosts.yml` values:

| Variable | Description |
|---|---|
| `ha_packages` | Package list to install on cluster nodes |
| `qdevice_packages` | Package list to install on the quorum device |
| `cluster_nodes_config` | Node configuration for the `ha_cluster` role (derived via Jinja2) |
| `fence_resources` | STONITH resource definition for each node (IPMI details from inventory) |
| `base_cluster_resources` | DLM, lvmlockd, VIP resource definitions |
| `base_resource_clones` | Clone configuration for DLM and lvmlockd |
| `base_constraints_order` | `dlm → lvmlockd` start ordering |
| `cluster_properties` | Cluster properties such as `stonith-enabled`, `no-quorum-policy` |
| `fence_location_constraints` | Constraint preventing a node from running its own fence agent |

> **Do not edit this file.** If a change is needed, update the source values in `hosts.yml`.

### 3.5 Security — Sensitive Variables

The following variables should be encrypted with **Ansible Vault** instead of storing them in plain text:

```bash
# Encrypt individual values
ansible-vault encrypt_string 'secret_password' --name 'cluster_password'
ansible-vault encrypt_string 'secret_password' --name 'ipmi_password'
ansible-vault encrypt_string 'secret_password' --name 'pgsql_password'

# Paste the generated output into hosts.yml:
# cluster_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   ...

# Run the playbook with vault encryption
ansible-playbook ha_cluster_setup.yml --ask-vault-pass
# or with a vault password file
ansible-playbook ha_cluster_setup.yml --vault-password-file ~/.vault_pass
```

---

## 4. Pre-Installation Steps

Complete the following checklist before running the playbook:

### Checklist

```
[ ] RHEL 9.x installed and up to date on all nodes
[ ] Active RHSM or local repo accessible on all nodes
[ ] root SSH access working (tested for all nodes)
[ ] Public NIC (enp1s0) and Corosync NIC (enp2s0) configured on both nodes
[ ] Ping working across corosync NICs (172.16.16.61 <-> 172.16.16.62)
[ ] iSCSI target reachable (172.16.16.248:3260)
[ ] Two separate iSCSI LUNs presented (one for GFS2, one for PostgreSQL)
[ ] IPMI/iDRAC/iLO accessible and tested for both nodes
[ ] WWIDs detected and entered into hosts.yml
[ ] All passwords in hosts.yml changed (cluster_password, ipmi_password, pgsql_password)
[ ] redhat.rhel_system_roles collection installed
```

### 4.1 IPMI Access Test

```bash
# Test clstr01's IPMI
ipmitool -I lanplus \
  -H 192.168.1.248 \
  -p 6330 \
  -U admin \
  -P <password> \
  chassis power status
# Expected: "Chassis Power is on"

# Test clstr02's IPMI
ipmitool -I lanplus \
  -H 192.168.1.248 \
  -p 6331 \
  -U admin \
  -P <password> \
  chassis power status
```

### 4.2 Corosync Network Test

```bash
# Ping from clstr01 to clstr02 over the corosync network
ping -I enp2s0 172.16.16.62 -c 4

# Ping from clstr02 to clstr01
ping -I enp2s0 172.16.16.61 -c 4
```

### 4.3 iSCSI Pre-Test

```bash
# Discover the iSCSI target
iscsiadm -m discovery -t st -p 172.16.16.248

# Expected:
# 172.16.16.248:3260,1 iqn.2025-01.com.sirket:storage01
```

### 4.4 Ansible Connectivity Test

```bash
# Ping all nodes
ansible -i inventory/hosts.yml all -m ping

# Expected:
# clstr01.lab.akyuz.tech | SUCCESS
# clstr02.lab.akyuz.tech | SUCCESS
# clstr-qrm01.lab.akyuz.tech | SUCCESS

# Syntax check
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml --syntax-check
```

---

## 5. Installation Steps (Play Flow)

The playbook consists of **19 plays** and takes approximately **8–12 minutes**. Every play runs with `any_errors_fatal: true`; a single failure stops the entire installation.

### Play 1 — Prerequisite Check and Preparation
**Target:** `all` (cluster nodes + quorum device)

This play verifies that the installation can proceed safely:

- **OS check:** Verifies RHEL 9.x; stops on other versions
- **RAM check:** Verifies minimum 2 GB
- **Disk check:** Verifies 20 GB free on `/`
- **SELinux status:** Reported (Enforcing recommended but not mandatory)
- **Hostname:** Set as FQDN via `systemd`
- **`/etc/hosts` fix:** Removes hostname conflicts from the `::1` loopback line

  > Without this fix, `pcs auth` and corosync resolve node names to `::1` and cannot communicate.

- **`/etc/hosts` update:** Adds all cluster node and quorum device entries (derived from inventory)
- **Hostname resolution test:** Verifies all nodes via `getent hosts`

### Play 1b — IPMI Connectivity Check
**Target:** `cluster_nodes`

- Installs `ipmitool`
- Tests TCP connectivity to each node's IPMI IP:port
- Tests authentication via `ipmitool chassis power status`
- **Installation stops if this fails** — a cluster without STONITH is unsafe

### Play 2 — Package Installation (Cluster Nodes)
**Target:** `cluster_nodes`

1. Conflicting packages removed (`python3-google-auth`, old `postgresql` packages)
2. System update performed (`dnf update`)
3. PostgreSQL PGDG `dnf module` disabled (prevents version conflicts)
4. All HA packages installed:

```
resource-agents     resource-agents-paf   pacemaker         pcs
fence-agents-all    corosync              corosync-qdevice  corosync-qnetd
dlm                 lvm2-lockd            gfs2-utils        ipmitool
iscsi-initiator-utils  lvm2              device-mapper-multipath
postgresql18-server postgresql18          postgresql18-contrib
```

### Play 3 — Package Installation (Quorum Device)
**Target:** `quorum_device`

- Installs quorum device packages: `corosync-qnetd`, `corosync-qdevice`, `pcs`
- `pcs` is mandatory — `pcsd` service comes with this package

### Play 4 — Firewall Configuration
**Target:** `all`

| Node Group | Configuration |
|---|---|
| Cluster nodes | `high-availability` firewalld service (2224, 3121/tcp; 5405, 5406/udp) |
| Cluster nodes | 5403/tcp (qdevice), 21064/tcp (DLM), 5432/tcp (PostgreSQL) |
| Quorum device | 5403/tcp (qnetd listening port) |

### Play 5 — Quorum Device Service Setup
**Target:** `quorum_device`

- Checks whether the NSS certificate DB exists
- If not: `pcs qdevice setup model net --enable --start`
- Verifies `corosync-qnetd` service (retries until active)
- With `force_reinstall=true`, the NSS DB and all configuration is reset

### Play 6 — iSCSI and Multipath Configuration
**Target:** `cluster_nodes`

This play combines two critical steps:

**iSCSI:**
1. Initiator name configured: `iqn.1994-05.lab.local:<hostname>`
2. `iscsid` service started and enabled
3. Target discovery: `iscsiadm -m discovery -t st -p 172.16.16.248`
4. LUN login: `iscsiadm -m node -T <iqn> --login`
5. Persistent login: `node.startup = automatic`

**Multipath:**
1. `device-mapper-multipath` installed
2. LUNs rescanned via `iscsiadm -m session --rescan`
3. `/etc/multipath.conf` written with WWID → alias mappings from `hosts.yml → multipath_aliases`:

```
multipaths {
    multipath {
        wwid  360014050b56232984714db8a4bdd0ea9
        alias pv_ha_shared01
    }
    multipath {
        wwid  360014056d2949ca6a2b455d82d2c6b37
        alias pv_ha_shared02
    }
}
```

4. `multipathd` restarted → `multipath -v2` → `udevadm settle`
5. Existence of `/dev/mapper/pv_ha_shared01` and `/dev/mapper/pv_ha_shared02` verified (120s timeout)

> **Critical:** WWID values in `multipath_aliases` are environment-specific and must be detected with `multipath -ll` before installation (see [Section 4.3](#43-iscsi-pre-test)).

### Play 7 — hacluster User and PCSD
**Target:** `cluster_nodes`

- `cluster_password` hashed and assigned to the `hacluster` user
- `pcsd` service started and enabled
- All cluster nodes authenticate each other via `pcs host auth`
- Quorum device also authenticated (required for cluster setup)

### Play 7b — Corosync Pre-Check and Cleanup
**Target:** `cluster_nodes`

Steps that prevent 90% of Corosync startup failures:

1. Verifies `cluster_ip` is assigned to the correct interface
2. Node-to-node ping test over the cluster network
3. Old `corosync.conf`, `authkey`, CIB, ring state files cleared
4. `pcsd` token cache cleared
5. Existing corosync/pacemaker services stopped
6. `pcsd` restarted (with clean token cache)
7. Re-authentication performed

With `force_reinstall=true`, additionally:
- `pcs cluster destroy --all` completely destroys the cluster
- All Pacemaker CIB and state files removed
- Critical directories recreated with correct ownership/permissions

### Play 8 — HA Cluster Setup (Red Hat System Roles)
**Target:** `cluster_nodes`

The `redhat.rhel_system_roles.ha_cluster` role is run:
- `corosync.conf` created (Kronosnet transport, ring: `enp2s0`)
- Pacemaker started
- Cluster membership established

**Post-task diagnostics:** If Corosync fails, an automatic diagnostic report is generated:
- Last 60 lines of `journalctl -u corosync`
- `corosync.conf` contents
- Cluster IP ring ping test
- SELinux AVC audit records

### Play 8-diag — Corosync/Pacemaker Validation
**Target:** `cluster_nodes`

- `systemctl is-active corosync pacemaker`
- `pcs status` output
- If corosync is not running, full diagnosis:
  - `journalctl -u corosync -n 80`
  - `corosync.conf` contents
  - Cluster IP ping test
  - SELinux AVC records
  - NetworkManager connection status
  - `iptables -L INPUT` (checks for rules outside firewalld)

### Play 8b — Quorum Device Addition
**Target:** `cluster_nodes[0]` (clstr01)

```bash
pcs quorum device add model net \
  host=clstr-qrm01.lab.akyuz.tech \
  algorithm=ffsplit \
  port=5403
```

- Quorum device presence checked via `pcs quorum device status`
- `corosync-qdevice` service started on each node
- Quorum status verified (`Quorate: Yes`)

Expected quorum output:
```
Nodeid   Votes   Qdevice   Name
     1       1   A,V,NMW   clstr01.lab.akyuz.tech
     2       1   A,V,NMW   clstr02.lab.akyuz.tech
     0       1             Qdevice
```

> **Critical:** The word "Qdevice" appears in the `pcs quorum status` table header, so `pcs quorum device status` must be used to verify qdevice presence.

### Play 8c — STONITH and Base Cluster Resources
**Target:** `cluster_nodes[0]` (clstr01)

**Cluster properties:**
```
stonith-enabled          = true
stonith-action           = reboot
no-quorum-policy         = freeze
start-failure-is-fatal   = false
resource-stickiness      = 100
migration-threshold      = 3
failure-timeout          = 3600s
```

**STONITH resources (one per node):**
```bash
pcs stonith create fence_ipmi_clstr01 fence_ipmilan \
  pcmk_host_list=clstr01.lab.akyuz.tech \
  ip=192.168.1.248 ipport=6330 \
  pcmk_delay_base=0s \
  meta pcmk_reboot_action=reboot

pcs stonith create fence_ipmi_clstr02 fence_ipmilan \
  pcmk_host_list=clstr02.lab.akyuz.tech \
  ip=192.168.1.248 ipport=6331 \
  pcmk_delay_base=30s \
  meta pcmk_reboot_action=reboot
```

> **Asymmetric delay:** In a split-brain situation where both nodes attempt to fence each other, clstr01's delay is 0s and clstr02's is 30s, so clstr01 wins first. This prevents a symmetric fencing loop.

**Location constraint:** A node cannot run its own fence agent:
```bash
pcs constraint location fence_ipmi_clstr01 avoids clstr01
pcs constraint location fence_ipmi_clstr02 avoids clstr02
```

**VIP resource:**
```bash
pcs resource create vip ocf:heartbeat:IPaddr2 \
  ip=192.168.0.63 cidr_netmask=23 nic=enp1s0
```

**DLM and lvmlockd (runs on both nodes — clone):**
```bash
pcs resource create dlm ocf:pacemaker:controld \
  op monitor interval=30s timeout=30s
pcs resource clone dlm interleave=true ordered=true

pcs resource create lvmlockd ocf:heartbeat:lvmlockd \
  op monitor interval=30s timeout=30s
pcs resource clone lvmlockd interleave=true ordered=true

pcs constraint order start dlm-clone then lvmlockd-clone
```

### Play 9 — LVM and Storage Configuration
**Target:** `cluster_nodes`

The order of steps in this play is critical:

```
1.  lvm.conf -> use_lvmlockd = 1 (lineinfile + regexp)
2.  Verify lvm2-lockd package is installed
3.  Mask lvmlockd systemd unit (Pacemaker will manage it)
4.  Wait for DLM cluster resource to start (retry: 20 x 10s)
5.  Wait for lvmlockd cluster resource to start (retry: 20 x 10s)
--- On clstr01 only ---
6.  pvcreate /dev/mapper/pv_ha_shared01
7.  vgcreate --shared shared_vg /dev/mapper/pv_ha_shared01
8.  vgchange --lock-start shared_vg   <- lock-start FIRST
9.  vgs shared_vg                     <- verify AFTER
10. lvcreate -L 50G -n gfs2_lv shared_vg
11. pvcreate /dev/mapper/pv_ha_shared02
12. vgcreate --shared db_vg /dev/mapper/pv_ha_shared02
13. vgchange --lock-start db_vg
14. vgs db_vg
15. lvcreate -L 40G -n pgsql_lv db_vg
16. lvchange -aey --nolocking /dev/shared_vg/gfs2_lv   <- exclusive (for mkfs)
17. lvchange -aey --nolocking /dev/db_vg/pgsql_lv
18. mkfs.gfs2 -j2 -p lock_dlm -t ha_webpgsql:gfs2_lv /dev/shared_vg/gfs2_lv
19. lvchange -an --nolocking /dev/shared_vg/gfs2_lv    <- DEACTIVATE
20. mkfs.ext4 /dev/db_vg/pgsql_lv
21. lvchange -an --nolocking /dev/db_vg/pgsql_lv
--- On each node ---
22. pvscan --cache && vgscan
23. vgchange --lock-start shared_vg   <- each node
24. vgchange --lock-start db_vg       <- each node
25. Verify vgs shared_vg
```

**How `use_lvmlockd = 1` is set:**

The `use_lvmlockd` line in `lvm.conf` is usually commented out (`# use_lvmlockd = 0`). The `lineinfile` module handles this:

```yaml
- name: "LVM | lvm.conf - set use_lvmlockd = 1"
  lineinfile:
    path: /etc/lvm/lvm.conf
    regexp: '^\s*#?\s*use_lvmlockd\s*='
    line: "\tuse_lvmlockd = 1"
    backup: yes
```

The `regexp` matches all of these formats:
- `#   use_lvmlockd = 0` (standard comment)
- `    use_lvmlockd = 0` (uncommented, value 0)
- `use_lvmlockd = 0` (no leading whitespace)

> **Critical:** VGs created with `vgcreate --shared` (`vg_lock_type=dlm`) cannot be seen by `vgs` until `vgchange --lock-start` is run first. The `--nolocking` flag does not resolve this.

> **Critical:** If the LV is not deactivated with `lvchange -an` after `mkfs`, Pacemaker will get `LV locked by other host` when it tries to activate the LV on the other node with `activation_mode=shared`.

### Play 10 — Mount Points
**Target:** `cluster_nodes`

- `/shared/webfs` directory created (`root:root`, 0755)
- `/var/lib/pgsql` directory created (`postgres:postgres`, 0700)
- Ownership set based on whether the `postgres` user exists at install time

### Play 11 — Cluster Resources
**Target:** `cluster_nodes[0]` (clstr01)

All Pacemaker resources and constraints are created:

```bash
# GFS2 LV activate (both nodes -- clone)
pcs resource create gfs2_lv_activate ocf:heartbeat:LVM-activate \
  vgname=shared_vg lvname=gfs2_lv \
  vg_access_mode=lvmlockd activation_mode=shared \
  op monitor interval=30s on-fail=fence
pcs resource clone gfs2_lv_activate interleave=true

# GFS2 Filesystem (both nodes -- clone)
pcs resource create gfs2_fs ocf:heartbeat:Filesystem \
  device=/dev/shared_vg/gfs2_lv directory=/shared/webfs \
  fstype=gfs2 options=noatime,nodiratime \
  op monitor interval=30s on-fail=fence
pcs resource clone gfs2_fs interleave=true

# PostgreSQL LV activate (active node only -- exclusive)
pcs resource create pgsql_lv_activate ocf:heartbeat:LVM-activate \
  vgname=db_vg lvname=pgsql_lv \
  vg_access_mode=lvmlockd activation_mode=exclusive \
  op monitor interval=30s on-fail=fence

# PostgreSQL Filesystem (active node only)
pcs resource create pgsql_fs ocf:heartbeat:Filesystem \
  device=/dev/db_vg/pgsql_lv directory=/var/lib/pgsql \
  fstype=ext4 \
  op stop on-fail=block meta failure-timeout=3600s

# Order constraints (start sequence):
pcs constraint order start dlm-clone         then lvmlockd-clone
pcs constraint order start lvmlockd-clone    then gfs2_lv_activate-clone
pcs constraint order start gfs2_lv_activate-clone then gfs2_fs-clone
pcs constraint order start lvmlockd-clone    then pgsql_lv_activate
pcs constraint order start pgsql_lv_activate then pgsql_fs

# Colocation constraints (same-node requirement):
pcs constraint colocation add pgsql_lv_activate with vip INFINITY
pcs constraint colocation add pgsql_fs with pgsql_lv_activate INFINITY
```

### Play 12 — PostgreSQL Installation and Configuration
**Target:** `cluster_nodes[0]` (clstr01)

1. Waits for `pgsql_fs` to be mounted (retry: 18 x 10s)
2. Initializes PostgreSQL with `initdb` (skipped if data directory already exists)
3. Basic `postgresql.conf` settings:
   - `listen_addresses = '*'`
   - `port = 5432`
   - `max_connections = 200`
   - `log_destination = 'stderr'`
   - `logging_collector = on`
4. `pg_hba.conf` network access rules (`hosts.yml → pgsql_allowed_networks`)
5. PostgreSQL systemd service **masked on all nodes** (Pacemaker will manage it)
6. PostgreSQL cluster resource created with `target-role=Stopped`
7. Constraints added (order: `pgsql_fs → postgresql`; colocation: `postgresql with pgsql_fs INFINITY`)
8. Stale probe records cleared (`pcs resource cleanup postgresql`)
9. `target-role=Started` set
10. Waits for PostgreSQL to start (retry: 24 x 10s)

> **Why create with `Stopped`?** After `pcs resource create`, Pacemaker probes all nodes. If probed before constraints are added, clstr02 cannot find `postgresql.conf` because `pgsql_fs` is not mounted there, triggering `on-fail=block` which results in BLOCKED. The sequence `target-role=Stopped` → add constraints → cleanup → `Started` prevents this.

### Play 13b — PostgreSQL User Password
**Target:** `cluster_nodes[0]` (clstr01)

After cluster setup is complete, a password is set for the `postgres` user:

1. Waits for PostgreSQL port to be accessible via VIP (120s timeout)
2. Detects `psql` binary path
3. Connects via **local UNIX socket + peer auth** (`become_user: postgres`)
4. Runs `ALTER ROLE postgres WITH PASSWORD '...'`

```bash
# Why local socket?
# PostgreSQL has no password set on first install.
# pg_hba.conf: local all postgres peer -> connects without a password.
# After the password is set, VIP connections with md5 auth work normally.
```

### Play 13 — Final Verification and Tests
**Target:** `cluster_nodes[0]` (clstr01)

- `pcs resource cleanup` (resets all failure counters)
- 45-second wait (for all resources to stabilize)
- `pcs status`, `pcs resource status`, `pcs stonith`, `pcs constraint show --full`
- `mountpoint -q` checks on both nodes for GFS2 and PostgreSQL mounts
- VIP active check (`ip addr show`)
- `corosync-quorumtool -s` — quorum status
- `pcs resource failcount show` — failed resource count
- Installation VIP preference constraint removed (`loc-vip-prefer-node0`)
- Installation completion report (ASCII table)

---

## 6. Cluster Resource Architecture

### 6.1 Resource Hierarchy

```
STONITH (each node fences the other):
  fence_ipmi_clstr01  -> Started: clstr02  (runs on clstr02 to fence clstr01)
  fence_ipmi_clstr02  -> Started: clstr01

SHARED (runs on both nodes -- clone):
  dlm-clone           -> Started: [ clstr01, clstr02 ]
    +-- lvmlockd-clone -> Started: [ clstr01, clstr02 ]
          +-- gfs2_lv_activate-clone -> Started: [ clstr01, clstr02 ]
          |     +-- gfs2_fs-clone   -> Started: [ clstr01, clstr02 ]
          |           /shared/webfs (GFS2, Active/Active)
          |
          +-- pgsql_lv_activate  -> Started: clstr01 (same node as vip)
                +-- pgsql_fs     -> Started: clstr01
                      +-- postgresql -> Started: clstr01

VIP:
  vip (192.168.0.63)  -> Started: clstr01
```

### 6.2 Order Constraints (Start Sequence)

```
dlm-clone -> lvmlockd-clone -> gfs2_lv_activate-clone -> gfs2_fs-clone
                            \
                             pgsql_lv_activate -> pgsql_fs -> postgresql
```

### 6.3 Colocation Constraints (Same-Node Requirement)

```
pgsql_lv_activate  must be on same node as  vip
pgsql_fs           must be on same node as  pgsql_lv_activate
postgresql         must be on same node as  pgsql_fs
```

This chain ensures `postgresql`, `pgsql_fs`, `pgsql_lv_activate`, and `vip` always run on the same node.

### 6.4 on-fail Behaviors

| Resource | Operation | on-fail | Description |
|---|---|---|---|
| DLM / lvmlockd | monitor | fence | Lock daemon access loss → fence node |
| gfs2_lv_activate | monitor | fence | Shared LV access loss → fence node |
| gfs2_fs | monitor | fence | GFS2 mount access loss → fence node |
| pgsql_fs | start | fence | Mount start failure → fence node |
| pgsql_fs | stop | **block** | Umount failure → NOT fenced, waits |
| postgresql | start | restart | Start failure → retry |
| postgresql | stop | **block** | Service stop failure → NOT fenced |
| postgresql | monitor | restart | Service died → restart |
| fence_ipmi | monitor | — | IPMI unreachable → reported |

> The `on-fail=block` strategy prevents fencing loops: if stop fails, the node is not fenced — the operator intervenes with `pcs resource cleanup`.

---

## 7. Usage

### 7.1 Normal Installation

```bash
# Syntax check
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml --syntax-check

# Dry-run (simulate without making changes)
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml --check

# Installation
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml

# With vault encryption
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml --ask-vault-pass
```

### 7.2 Reset (Clean Reinstallation)

The existing cluster is completely destroyed and rebuilt from scratch:

```bash
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml -e force_reinstall=true
```

`force_reinstall=true` does the following:
- `pcs cluster destroy --all`
- Clears `corosync.conf`, `authkey`, CIB, ring state files
- Deletes and recreates the `corosync-qnetd` NSS DB
- Removes existing qdevice configuration
- Recreates Pacemaker directories with correct ownership/permissions
- Rebuilds from scratch

### 7.3 Variable Override

```bash
# Test with a different PostgreSQL version
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml -e pgsql_version=16

# Different stonith action
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml -e stonith_action=off

# Start from a specific task
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml \
  --start-at-task "PLAY 9/13 | LVM ve Paylasilmis Depolama Yapilandirmasi"
```

---

## 8. Verification and Monitoring

### 8.1 Post-Installation Check

```bash
# Overall cluster status
pcs status

# Expected successful output:
# Cluster name: ha_webpgsql
# Status of pacemakerd: 'Pacemaker is running'
# Current DC: clstr01.lab.akyuz.tech - partition with quorum
# 2 nodes configured
#
# Node List:
#   * Online: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#
# Full List of Resources:
#   * fence_ipmi_clstr01  (stonith:fence_ipmilan): Started clstr02.lab.akyuz.tech
#   * fence_ipmi_clstr02  (stonith:fence_ipmilan): Started clstr01.lab.akyuz.tech
#   * Clone Set: dlm-clone [dlm]:
#       Started: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#   * Clone Set: lvmlockd-clone [lvmlockd]:
#       Started: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#   * Clone Set: gfs2_lv_activate-clone [gfs2_lv_activate]:
#       Started: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#   * Clone Set: gfs2_fs-clone [gfs2_fs]:
#       Started: [ clstr01.lab.akyuz.tech clstr02.lab.akyuz.tech ]
#   * pgsql_lv_activate   Started clstr01.lab.akyuz.tech
#   * pgsql_fs             Started clstr01.lab.akyuz.tech
#   * vip                  Started clstr01.lab.akyuz.tech
#   * postgresql           Started clstr01.lab.akyuz.tech

# Constraints
pcs constraint

# Quorum status
pcs quorum status
corosync-quorumtool -s

# Quorum device status
pcs quorum device status
# Algorithm: ffsplit  State: connected

# Configuration validation
crm_verify -L -V

# Mount points
df -h /shared/webfs /var/lib/pgsql

# PostgreSQL connection (via VIP)
psql -h 192.168.0.63 -U postgres -c "SELECT version();"

# STONITH status
pcs stonith
```

### 8.2 Continuous Monitoring

```bash
# Real-time cluster status (every 5 seconds)
watch -n 5 pcs status

# Pacemaker logs
journalctl -u pacemaker -f

# Corosync logs
journalctl -u corosync -f

# STONITH events
journalctl -u pacemaker | grep -i "fence\|stonith"

# Resource start/stop events
journalctl -u pacemaker | grep "Result of"

# GFS2 status
gfs2_tool df /shared/webfs

# LVM lock status
lvmlockctl --dump
```

---

## 9. Troubleshooting

### 9.1 General Cluster Issues

#### Cluster not starting / nodes not seeing each other

```bash
# Corosync ring check
corosync-cfgtool -s
# Expected: ring0: active with no faults

# Is the cluster network reachable?
ping -I enp2s0 172.16.16.62   # from clstr01 to clstr02

# Corosync log
journalctl -u corosync -n 80

# Do the authkeys match?
md5sum /etc/corosync/authkey   # must be identical on both nodes
```

**If authkeys differ:**
```bash
corosync-keygen   # generate on clstr01
scp /etc/corosync/authkey clstr02:/etc/corosync/
systemctl restart corosync   # on both nodes
```

---

#### No quorum, cluster frozen

```bash
pcs quorum status
# Quorate: YES must appear

# Is the qdevice connected?
pcs quorum device status
# State: connected must appear

# Is qnetd running? (on the quorum device server)
ssh root@192.168.0.67 systemctl status corosync-qnetd

# Is corosync-qdevice running? (on each cluster node)
systemctl status corosync-qdevice
```

**Re-add quorum device:**
```bash
pcs quorum device remove
pcs host auth clstr-qrm01.lab.akyuz.tech addr=192.168.0.67 \
  -u hacluster -p <cluster_password>
pcs quorum device add model net \
  host=clstr-qrm01.lab.akyuz.tech \
  algorithm=ffsplit port=5403
systemctl enable --now corosync-qdevice
```

---

### 9.2 Multipath / iSCSI Issues

#### `/dev/mapper/pv_ha_shared01` not found

```bash
# Multipath status
multipath -ll

# If mpatha/mpathb appear but pv_ha_shared01 does not:
# The alias definition in multipath.conf is missing or has the wrong WWID

# Check WWIDs
multipath -ll | grep -E '^\S.*\('

# Check multipath.conf
cat /etc/multipath.conf

# Restart multipathd
systemctl restart multipathd
multipath -v2
udevadm settle --timeout=30

# Is the device present?
ls -la /dev/mapper/pv_ha_shared*
```

**If WWID is wrong:**
1. Verify the `wwid` values in `hosts.yml → multipath_aliases` against `multipath -ll` output
2. Rerun the playbook

---

#### iSCSI login failed

```bash
# iSCSI sessions
iscsiadm -m session

# Rerun discovery
iscsiadm -m discovery -t st -p 172.16.16.248

# Login
iscsiadm -m node -T iqn.2025-01.com.sirket:storage01 \
  -p 172.16.16.248 --login

# iscsid service
systemctl status iscsid
```

---

### 9.3 STONITH Issues

#### Infinite fencing loop

```
reboot of clstr02 ... last failed at ...
reboot of clstr01 ... last failed at ...
```

**Cause:** Both nodes are trying to fence each other simultaneously. Asymmetric delay is not correctly set.

```bash
# Check delay values
pcs stonith show fence_ipmi_clstr01.lab.akyuz.tech | grep delay
pcs stonith show fence_ipmi_clstr02.lab.akyuz.tech | grep delay

# clstr01 fence agent: pcmk_delay_base=0s
# clstr02 fence agent: pcmk_delay_base=30s

# Fix
pcs stonith update fence_ipmi_clstr01.lab.akyuz.tech pcmk_delay_base=0s
pcs stonith update fence_ipmi_clstr02.lab.akyuz.tech pcmk_delay_base=30s
```

---

#### STONITH failed: IPMI unreachable

```bash
# IPMI access test
ipmitool -I lanplus -H 192.168.1.248 -p 6330 \
  -U admin -P <ipmi_password> chassis power status

# STONITH agent test
pcs stonith test fence_ipmi_clstr02.lab.akyuz.tech
```

---

### 9.4 LVM / GFS2 Issues

#### `use_lvmlockd = 1` not taking effect

```bash
# Check active value
grep 'use_lvmlockd' /etc/lvm/lvm.conf | grep -v '^[[:space:]]*#'
# Expected: use_lvmlockd = 1

# Is the lvmlockd process running? (Pacemaker should have started it)
pgrep -a lvmlockd
pcs resource status lvmlockd-clone
```

---

#### `Volume group "shared_vg" not found`

**Cause:** `vgs` was run before the DLM lockspace was started, or `lvmlockd` is not running.

```bash
# Is DLM running?
pcs resource status dlm-clone

# Is lvmlockd running?
pcs resource status lvmlockd-clone
pgrep -a lvmlockd

# lock-start
vgchange --lock-start shared_vg
vgchange --lock-start db_vg
vgs shared_vg
vgs db_vg

# If the problem persists
lvmlockctl --dump
journalctl -u pacemaker | grep lvmlockd | tail -30
```

---

#### GFS2 LV activate clone not starting on clstr02 — `LV locked by other host`

```bash
# An exclusive lock is still held on clstr01
journalctl -u pacemaker | grep gfs2_lv_activate | tail -20

# Deactivate on clstr01
lvchange -an --nolocking /dev/shared_vg/gfs2_lv

# Cleanup
pcs resource cleanup gfs2_lv_activate-clone
```

---

#### GFS2 not mounting

```bash
# Does the GFS2 cluster name match?
blkid /dev/shared_vg/gfs2_lv
# TYPE="gfs2" LABEL="ha_webpgsql:webfs" must appear

# DLM lock domain
dlm_tool ls
dlm_tool status

# Is the journal count sufficient? (must be >= number of nodes)
gfs2_tool jindex /dev/shared_vg/gfs2_lv
```

---

### 9.5 PostgreSQL Issues

#### `postgresql FAILED clstr02 (blocked)` — `stop 'not installed'`

```
postgresql stop on clstr02 returned 'not installed'
(Configuration file /var/lib/pgsql/data/postgresql.conf doesn't exist)
```

**Cause:** `pgsql_fs` is not mounted on clstr02, but Pacemaker tried to stop postgresql there.

```bash
# Reset failure counters
pcs resource cleanup postgresql

# Are constraints correct?
pcs constraint | grep postgresql
# Expected:
# postgresql with pgsql_fs  INFINITY  (colo-postgresql-psqlfs)
# start pgsql_fs then postgresql (order-psqlfs-postgresql)

# Add missing constraints if needed
pcs constraint colocation add postgresql with pgsql_fs INFINITY \
  id=colo-postgresql-psqlfs
pcs constraint order start pgsql_fs then postgresql \
  id=order-psqlfs-postgresql

pcs resource cleanup postgresql
```

---

#### PostgreSQL not starting — `initdb` not done

```bash
# Is the data directory empty?
ls -la /var/lib/pgsql/data/

# Manual initdb
sudo -u postgres /usr/pgsql-18/bin/initdb -D /var/lib/pgsql/data

pcs resource cleanup postgresql
```

---

#### PostgreSQL started but cannot connect

```bash
# Which node is it running on?
pcs resource status postgresql

# Is it on the same node as the VIP?
pcs resource status vip

# If on different nodes -- constraint problem
pcs constraint | grep postgresql

# Check pg_hba.conf access rules
grep -v '^#' /var/lib/pgsql/data/pg_hba.conf

# Connection test (via VIP)
psql -h 192.168.0.63 -U postgres -c "SELECT 1;"
```

---

#### postgres user password authentication failure

```
FATAL: password authentication failed for user "postgres"
```

**Cause:** PostgreSQL has no password set on first install. md5 auth over VIP requires a password to be set via local socket first.

```bash
# Set password via local socket (peer auth, no password required)
sudo -u postgres psql -p 5432 \
  -c "ALTER ROLE postgres WITH PASSWORD 'new_password';"

# Then test via VIP
PGPASSWORD="new_password" psql \
  -h 192.168.0.63 -p 5432 -U postgres -c "SELECT version();"
```

---

### 9.6 General Resource Issues

#### Resource not starting, failure count exhausted

```bash
# Failure details
pcs resource failcount show postgresql

# Reset failure counters
pcs resource cleanup postgresql

# All resources
pcs resource cleanup

# Put failing node in standby and clean up
pcs node standby clstr02.lab.akyuz.tech
pcs resource cleanup
pcs node unstandby clstr02.lab.akyuz.tech
```

---

#### Resource debug start

```bash
# Start resource in debug mode (verbose log)
pcs resource debug-start postgresql

# Run OCF agent directly
OCF_ROOT=/usr/lib/ocf \
OCF_RESKEY_pgctl=/usr/pgsql-18/bin/pg_ctl \
OCF_RESKEY_pgdata=/var/lib/pgsql/data \
/usr/lib/ocf/resource.d/heartbeat/pgsql monitor
```

---

### 9.7 Fencing Issues

#### Node does not come back after being fenced

```bash
# Check node status via IPMI
ipmitool -I lanplus -H 192.168.1.248 -p 6330 \
  -U admin -P <ipmi_password> chassis power status

# Power on
ipmitool -I lanplus -H 192.168.1.248 -p 6330 \
  -U admin -P <ipmi_password> chassis power on

# Remove from standby after node returns (if needed)
pcs node unstandby clstr01.lab.akyuz.tech
pcs resource cleanup
```

---

## 10. Maintenance Operations

### 10.1 Node Maintenance (Rolling Maintenance)

```bash
# Put the node to be maintained into standby (resources move to the other)
pcs node standby clstr01.lab.akyuz.tech

# Verify all resources have moved to clstr02
pcs status

# Perform maintenance (update, kernel reboot, etc.)
# ...

# Bring the node back
pcs node unstandby clstr01.lab.akyuz.tech

# Resources do not automatically move back (resource-stickiness=100)
# Optional manual move back:
pcs resource move postgresql clstr01.lab.akyuz.tech
pcs resource clear postgresql   # remove the temporary location constraint
```

---

### 10.2 VIP Failover Test

```bash
# Determine the active node
pcs resource status vip

# Trigger failover
pcs resource move vip clstr02.lab.akyuz.tech

# Remove constraint after test (to allow automatic failover again)
pcs resource clear vip
pcs status
```

---

### 10.3 PostgreSQL Manual Failover Test

```bash
# Move PostgreSQL to the other node
pcs resource move postgresql clstr02.lab.akyuz.tech

# vip and pgsql_fs must also move to clstr02 (colocation)
pcs status

# Remove constraint after test
pcs resource clear postgresql
```

---

### 10.4 Cluster Stop / Start

```bash
# Stop the entire cluster (planned maintenance)
pcs cluster stop --all

# Start the entire cluster
pcs cluster start --all

# Single node
pcs cluster stop clstr01.lab.akyuz.tech
pcs cluster start clstr01.lab.akyuz.tech

# Enable automatic start at boot
pcs cluster enable --all
```

---

### 10.5 Configuration Backup

```bash
# CIB (Cluster Information Base) backup
pcs cluster cib > cluster_cib_backup_$(date +%Y%m%d).xml

# Corosync configuration
cp /etc/corosync/corosync.conf corosync_backup_$(date +%Y%m%d).conf

# multipath.conf
cp /etc/multipath.conf multipath_backup_$(date +%Y%m%d).conf

# Restore CIB
pcs cluster cib-push cluster_cib_backup.xml --config
```

---

### 10.6 PostgreSQL Data Backup

```bash
# pg_dump via VIP (cluster-independent)
pg_dump -h 192.168.0.63 -U postgres -Fc mydb > mydb_$(date +%Y%m%d).dump

# Full cluster backup
pg_dumpall -h 192.168.0.63 -U postgres > dumpall_$(date +%Y%m%d).sql

# For continuous archiving, add to postgresql.conf:
# archive_mode = on
# archive_command = 'cp %p /backup/wal/%f'
```

---

### 10.7 Adding a New Node to the Cluster

> **Warning:** This playbook is optimized for 2 nodes. To add a new node, update `gfs2_journals` and `qdevice_algorithm`, then rerun `ha_cluster_setup.yml`.

```bash
# First add the new node to hosts.yml
# Increase gfs2_journals to 3
# Then run the playbook (force_reinstall is not required; the playbook is idempotent)
ansible-playbook -i inventory/hosts.yml ha_cluster_setup.yml
```

---

## 11. Known Limitations

| Topic | Detail |
|---|---|
| **Fixed at 2 nodes** | `gfs2_journals=2` and `qdevice_algorithm=ffsplit` are optimized for 2 nodes. For 3+ nodes, `gfs2_journals`, `qdevice_algorithm`, and resource constraints must be updated. |
| **Single iSCSI target** | The iSCSI target is a single point of failure. For target-side HA, consider DRBD or Ceph. |
| **No PostgreSQL replication** | `rep_mode=none` — in an active/passive configuration, only one node writes. For replication, consider `rep_mode=sync` or `async` with `resource-agents-paf`. |
| **IPMI mandatory** | Without STONITH, Pacemaker cannot protect the cluster in a split-brain situation. For environments without IPMI, consider `fence_vmware_soap` or `fence_kdump`. |
| **RHEL 9 / pcs 0.11+** | Due to syntax changes such as `pcs resource defaults update` (formerly `pcs resource defaults`) and `meta pcmk_reboot_action` (formerly `action=`), this playbook is not compatible with RHEL 8 or older. |
| **No automatic failback** | Due to `resource-stickiness=100`, resources do not automatically move back when a node returns. Manual movement via `pcs resource move` is required. |
| **Ansible 2.14** | The `ansible.posix` collection may emit `[WARNING]: Collection ansible.posix does not support Ansible version 2.14.18` with Ansible 2.14.x. This warning is not critical and installation completes successfully. |
| **no_log and debugging** | `no_log: false` is set in PLAY 13b. In production environments, set `no_log: true` and encrypt passwords with Ansible Vault. |
| **psql not required on bastion** | Since PLAY 13b runs on a cluster node, `psql` does not need to be installed on the bastion. |

---

*Playbook version: last updated 19 March 2026 · RHEL 9.7 · Pacemaker 2.1.x · pcs 0.11.x · PostgreSQL 18 · Ansible 2.14.x*
