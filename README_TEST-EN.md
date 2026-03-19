# Red Hat 9 High Availability Cluster — Test Playbook

**`ha_cluster_test.yml`** · RHEL 9 · Pacemaker 2.1.x · Corosync 3.x · PostgreSQL 18

> This document describes all tests performed by `ha_cluster_test.yml`, how to run them, what each test verifies, and how to interpret results. The test playbook must be run **after** a successful cluster installation with `ha_cluster_setup.yml`.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Directory Structure](#3-directory-structure)
4. [Usage](#4-usage)
5. [Test Categories](#5-test-categories)
6. [Test Details](#6-test-details)
7. [Test Report](#7-test-report)
8. [Safety and Cleanup](#8-safety-and-cleanup)
9. [Troubleshooting Test Failures](#9-troubleshooting-test-failures)

---

## 1. Overview

The test playbook automatically verifies that the cluster behaves correctly across all failure scenarios. Tests are divided into two groups:

### Safe Tests (always run)

| Test | Description |
|---|---|
| T01 | Basic cluster health check |
| T02 | VIP failover (maintenance simulation) |
| T06 | PostgreSQL service crash (kill -9) |
| T07 | GFS2 concurrent write (both nodes) |
| T08 | LVM lock validation |
| T11 | Failover under load |
| T12 | Multiple consecutive failovers (resilience) |

### Destructive Tests (require `-e run_destructive=true`)

| Test | Description | Risk |
|---|---|---|
| T03 | Active node sudden shutdown | Node is stopped and restarted |
| T04 | Corosync ring network interruption | iptables rules applied |
| T05 | Split-brain prevention (qdevice blocked) | iptables rules applied |
| T10 | iSCSI connection interruption | iptables rules applied |
| T13 | Full cluster restart | All nodes stopped and started |

### STONITH Test (requires `-e run_stonith_test=true`)

| Test | Description | Risk |
|---|---|---|
| T09 | STONITH fence agent test | **Node is REBOOTED via IPMI!** |

---

## 2. Prerequisites

- Cluster installation successfully completed with `ha_cluster_setup.yml`
- Both nodes online: `pcs status` shows no OFFLINE or FAILED resources
- PostgreSQL accessible via VIP: `psql -h 192.168.0.63 -U postgres -c "SELECT 1;"`
- `pgsql_password` correctly set in `inventory/hosts.yml`
- `tasks/failover_cycle.yml` present in the project directory (required by T12)

### Pre-Test Cluster Health Check

Before running the test playbook, verify the cluster is clean:

```bash
pcs status
# Expected: all resources Started, no FAILED or OFFLINE

pcs resource cleanup
# Resets any stale failure counters

psql -h 192.168.0.63 -U postgres -c "SELECT version();"
# Expected: PostgreSQL version string returned
```

---

## 3. Directory Structure

```
project-directory/
├── ha_cluster_test.yml           # Main test playbook
└── tasks/
    └── failover_cycle.yml        # Failover cycle task file (used by T12)
```

The `tasks/failover_cycle.yml` file is called by T12 via `include_tasks` for each failover loop iteration. It must be present in the `tasks/` subdirectory relative to `ha_cluster_test.yml`.

---

## 4. Usage

### Run All Tests (Safe Mode)

```bash
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml
```

Destructive tests (T03, T04, T05, T10, T13) and the STONITH test (T09) are **skipped** in this mode.

### Run All Tests Including Destructive

```bash
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -e run_destructive=true
```

### Run All Tests Including STONITH (Node Will Reboot!)

```bash
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -e run_destructive=true \
  -e run_stonith_test=true
```

### Run a Single Test

```bash
# Run only T01
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml -t test_T01

# Run only T06
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml -t test_T06
```

### Run a Specific Set of Tests

```bash
# Run T01 and T02 only
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -t "test_T01,test_T02"

# Run only safe tests
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml \
  -t "test_T01,test_T02,test_T06,test_T07,test_T08,test_T11,test_T12"
```

### Variable Reference

| Variable | Default | Description |
|---|---|---|
| `run_destructive` | `false` | Enable destructive tests (T03, T04, T05, T10, T13) |
| `run_stonith_test` | `false` | Enable STONITH test (T09) — node will reboot! |
| `failover_wait` | `60` | Seconds to wait for failover to complete |
| `settle_wait` | `30` | Seconds to wait for resource stabilization |
| `pgsql_password` | (from hosts.yml) | PostgreSQL postgres user password |

---

## 5. Test Categories

### PREP — Pre-Test Environment Validation

Before any test runs, the PREP play performs the following cleanup and validation:

1. **iptables cleanup:** Removes any leftover rules from previous test runs on all cluster nodes
2. **PostgreSQL table cleanup:** Drops any leftover test tables (`ha_test_t02`, `ha_test_t03`, `ha_test_t06`, `ha_load_t11`, `ha_test_t12`)
3. **Location constraint cleanup:** Clears any residual VIP location constraints (`pcs resource clear vip`)
4. **Resource cleanup:** Runs `pcs resource cleanup` to reset stale failure counters
5. **Stabilization wait:** 15-second pause for the cluster to stabilize
6. **Cluster health check:** Verifies no OFFLINE or FAILED resources exist
7. **PostgreSQL connectivity test:** Verifies the `pgsql_password` works against the VIP before running any tests — fails early with a clear error if the password is wrong
8. **Active/passive node determination:** Identifies which node holds the VIP

### RAPOR — Final Report and Environment Restore

After all tests complete, the RAPOR play always runs (tagged `always`):

1. **iptables cleanup:** Removes all test-related firewall rules from all nodes
2. **PostgreSQL table cleanup:** Drops all test tables
3. **GFS2 file cleanup:** Removes test files from `/shared/webfs`
4. **Location constraint cleanup:** Clears all VIP location constraints
5. **Resource cleanup:** Final `pcs resource cleanup`
6. **Stabilization wait:** 20 seconds
7. **Final status collection:** `pcs status`, `pcs constraint`, `pcs quorum status`
8. **Mount status:** Verifies GFS2 and PostgreSQL mounts on both nodes
9. **Final PostgreSQL connection test:** Verifies connectivity after all tests
10. **Console summary report:** ASCII table showing test results
11. **Markdown report file:** Detailed report written to `/tmp/ha_cluster_test/ha_cluster_test_report_<timestamp>.md`

---

## 6. Test Details

### T01 — Basic Cluster Health Check

**Tag:** `test_T01` | **Type:** Safe | **Always runs:** Yes

**What it verifies:**

| Check | Command | Expected |
|---|---|---|
| Corosync ring | `corosync-cfgtool -s` | All peers connected, no disconnected/lost |
| Quorum device | `pcs quorum device status` | State: connected |
| Cluster quorate | `pcs quorum status` | Quorate: Yes |
| All resources running | `pcs resource status` | No FAILED or Stopped |
| GFS2 on both nodes | `pcs resource status gfs2_fs-clone` | Both node names present |
| GFS2 mount | `mountpoint -q /shared/webfs` | rc=0 on both nodes |
| PostgreSQL mount | `mountpoint -q /var/lib/pgsql` | rc=0 on active node |
| PostgreSQL connection | `psql -h VIP -c "SELECT version()"` | Successful |
| STONITH enabled | `pcs property config stonith-enabled` | stonith-enabled=true |
| IPMI reachable | `ipmitool chassis power status` | rc=0 for each node |
| Configuration valid | `crm_verify -L -V` | rc=0 |
| Service status | `systemctl is-active` | corosync, pacemaker, qdevice, multipathd, iscsid all active |

**Failure action:** Fix the underlying cluster issue before proceeding with other tests.

---

### T02 — VIP Failover (Maintenance Simulation)

**Tag:** `test_T02` | **Type:** Safe | **Always runs:** Yes

**What it does:**

1. Records which node currently holds the VIP
2. Writes a test record to PostgreSQL (`ha_test_t02` table)
3. Moves the VIP to the other node using `pcs resource move vip <target>`
4. Waits for the VIP to appear on the target node (up to 60s)
5. Waits for PostgreSQL to start on the new node
6. Inserts another record after failover
7. Verifies the pre-failover record is still present (data consistency)
8. Removes the location constraint (`pcs resource clear vip`)
9. Drops the test table

**What it proves:**
- VIP failover completes successfully within the configured timeout
- PostgreSQL remains accessible via the VIP throughout the failover
- Data written before failover is preserved after failover

**Cleanup:** Always runs (in `always` block) — location constraint and test table are removed even if the test fails.

---

### T03 — Active Node Sudden Shutdown

**Tag:** `test_T03` | **Type:** Destructive | **Requires:** `-e run_destructive=true`

**What it does:**

1. Identifies the active node (VIP holder) and passive node
2. Writes a test record to PostgreSQL
3. Stops the active node: `pcs cluster stop <active_node> --force` (async)
4. Waits on the passive node for PostgreSQL to appear (up to 120s)
5. Verifies the VIP moved to the passive node
6. Tests PostgreSQL connectivity via VIP on the passive node
7. Verifies the pre-shutdown data record is still present
8. Restarts the stopped node: `pcs cluster start <active_node>`
9. Waits for the stopped node to rejoin the cluster

**What it proves:**
- Cluster correctly detects node failure and performs failover
- PostgreSQL becomes available on the passive node automatically
- Data is preserved across an unexpected node shutdown

**Cleanup:** Always runs — stopped node is restarted, resource cleanup is performed, test table is dropped.

---

### T04 — Corosync Ring Network Interruption

**Tag:** `test_T04` | **Type:** Destructive | **Requires:** `-e run_destructive=true`

**What it does:**

1. Identifies active and passive nodes
2. On the passive node, blocks corosync traffic (UDP/TCP 5405) via iptables for 45 seconds (async)
3. Monitors cluster behavior during the interruption from the active node
4. Waits for the interruption to end and iptables rules to be removed
5. Waits for cluster stabilization (up to 120s)
6. Runs `pcs resource cleanup`
7. Tests PostgreSQL connectivity via VIP
8. Verifies corosync ring is fully recovered (`corosync-cfgtool -s`)

**Why 45 seconds?** This value must be greater than the STONITH delay (30s on clstr02) to trigger quorum/fencing logic.

**What it proves:**
- Cluster handles a temporary corosync ring interruption gracefully
- After network recovery, corosync ring reconnects
- VIP and PostgreSQL remain accessible throughout

**Cleanup:** Always runs — iptables rules are removed from both nodes.

---

### T05 — Split-Brain Prevention (Quorum Device Blocked)

**Tag:** `test_T05` | **Type:** Destructive | **Requires:** `-e run_destructive=true`

**What it does:**

1. Blocks qdevice port (TCP 5403) from both cluster nodes via iptables for 30 seconds (async)
2. Monitors quorum status during the interruption
3. Waits for the interruption to end
4. Verifies qdevice reconnects (`pcs quorum device status`)

**What it proves:**
- When the qdevice connection is lost, the cluster applies `no-quorum-policy=freeze` (resources are frozen, not stopped)
- After qdevice reconnects, the cluster returns to a fully quorate state
- The ffsplit algorithm correctly handles qdevice loss without split-brain

**Cleanup:** Always runs — iptables rules are removed from both nodes.

---

### T06 — PostgreSQL Service Crash (kill -9)

**Tag:** `test_T06` | **Type:** Safe | **Always runs:** Yes

**What it does:**

1. Identifies which node is running PostgreSQL
2. Writes a test record to PostgreSQL (`ha_test_t06` table)
3. Detects the postmaster PID: `pgrep -o -f 'postgres.*postmaster'`
4. Sends `kill -9` to the postmaster process
5. Waits 5 seconds (for Pacemaker to detect the failure)
6. Waits for Pacemaker to restart PostgreSQL (up to 60s)
7. Checks the failure count (`pcs resource failcount show postgresql`)
8. Inserts a record after restart
9. Verifies the pre-kill record is still present

**What it proves:**
- Pacemaker's `on-fail=restart` monitor action correctly restarts a crashed PostgreSQL
- The migration threshold (3 failures) is not exceeded in a single crash
- Data written before the crash is preserved after restart

**Cleanup:** Always runs — failure counters reset, test table dropped.

---

### T07 — GFS2 Concurrent Write (Both Nodes)

**Tag:** `test_T07` | **Type:** Safe | **Always runs:** Yes | **Hosts:** `cluster_nodes` (both)

**What it does:**

1. Verifies GFS2 is mounted on both nodes
2. Checks available disk space on `/shared/webfs`
3. Simultaneously writes 100 lines from each node to separate files in `/shared/webfs` (async)
4. Waits for both write operations to complete
5. Verifies both files are visible from both nodes (cross-read test)
6. Verifies each file contains the expected number of lines (100+)
7. Verifies GFS2 is mounted read-write

**What it proves:**
- GFS2 handles concurrent writes from both nodes without corruption
- Files written by one node are immediately visible on the other node
- GFS2 distributed locking (via DLM) works correctly

**Cleanup:** Always runs — test files removed from `/shared/webfs`.

---

### T08 — LVM Lock Validation

**Tag:** `test_T08` | **Type:** Safe | **Always runs:** Yes

**What it does:**

1. Verifies `lvmlockd` is running (checks PID)
2. Runs `lvmlockctl --dump` to show current lockspace state
3. Verifies `shared_vg` lockspace is active
4. Verifies `db_vg` lockspace is active
5. Checks `pcs resource failcount show lvmlockd`
6. Reads and displays `use_lvmlockd` setting from `lvm.conf`
7. Tests `vgchange --lock-start` idempotency for both VGs (running it again should not cause errors)
8. Verifies both VGs are accessible with `vgs`
9. Reports LV activation status with `lvs`

**What it proves:**
- lvmlockd is running and managing distributed locks correctly
- Both shared VGs have active lockspaces
- lock-start can be called multiple times without errors (idempotent)
- Both VGs and their LVs are accessible

---

### T09 — STONITH Fence Agent Test

**Tag:** `test_T09` | **Type:** STONITH | **Requires:** `-e run_stonith_test=true`

> **WARNING:** This test will REBOOT the passive cluster node via IPMI. Make sure this is acceptable in your environment before running.

**What it does:**

1. Identifies the passive node (the one NOT holding the VIP)
2. Tests IPMI connectivity to the passive node via `ipmitool chassis power status`
3. Fences the passive node: `pcs stonith fence <passive_node> --off`
4. Waits for the node to appear OFFLINE in cluster status
5. Powers the node back on via IPMI: `ipmitool chassis power on`
6. Waits for SSH to become available on the node (up to 180s)
7. Starts the cluster service on the node: `pcs cluster start`
8. Waits for the node to rejoin the cluster

**What it proves:**
- The STONITH agent (`fence_ipmilan`) can successfully fence a node
- IPMI credentials and connectivity are correct
- The fenced node can be recovered and rejoin the cluster

**Cleanup:** Always runs — `pcs resource cleanup` is performed after the node rejoins.

---

### T10 — iSCSI Connection Interruption

**Tag:** `test_T10` | **Type:** Destructive | **Requires:** `-e run_destructive=true`

**What it does:**

1. Records current iSCSI sessions and multipath device state
2. Blocks iSCSI target (TCP 3260) from both cluster nodes via iptables for 30 seconds (async)
3. Monitors cluster behavior during the interruption
4. Waits for the interruption to end
5. Runs `pcs resource cleanup`
6. Waits for cluster stabilization
7. Verifies GFS2 is still mounted on both nodes
8. Reports multipath recovery status

**What it proves:**
- Cluster handles a temporary iSCSI path interruption
- Multipath recovers when connectivity is restored
- GFS2 mounts recover after iSCSI reconnection

**Cleanup:** Always runs — iptables rules are removed from both nodes.

---

### T11 — Failover Under Load

**Tag:** `test_T11` | **Type:** Safe | **Always runs:** Yes

**What it does:**

1. Creates the `ha_load_t11` table in PostgreSQL
2. Starts a background load generator: 1000 INSERT operations with a 30ms interval (async)
3. Waits 10 seconds for the load to build up
4. Triggers a VIP failover while the load is running
5. Waits for PostgreSQL to start on the new node (up to 90s)
6. Queries the record count after failover
7. Reports how many records were committed before the failover

**What it proves:**
- The cluster can perform a VIP failover while PostgreSQL is under write load
- Pacemaker correctly manages the resource transition under load conditions
- Records committed before the failover are preserved

**Cleanup:** Always runs — location constraint removed, test table dropped.

---

### T12 — Multiple Consecutive Failovers (Resilience)

**Tag:** `test_T12` | **Type:** Safe | **Always runs:** Yes

**What it does:**

Runs 3 consecutive VIP failover cycles (configurable via `failover_cycles`). Each cycle (handled by `tasks/failover_cycle.yml`) performs:

1. Records which node currently holds the VIP
2. Writes a record to `ha_test_t12` (cycle number and source node)
3. Moves the VIP to the other node: `pcs resource move vip <target>`
4. Waits for failover to complete (up to 60s)
5. Waits for PostgreSQL to start on the new node
6. Verifies PostgreSQL is accessible via VIP
7. Queries the record written in step 2 to verify data consistency
8. Removes the location constraint for the next cycle
9. Waits 30 seconds for stabilization

After all cycles, verifies the total record count equals the number of cycles.

**What it proves:**
- The cluster can sustain multiple consecutive failovers without degradation
- Each failover completes successfully
- Data written during each cycle is preserved through all subsequent failovers

**Cleanup:** Always runs — location constraint removed, test table dropped.

---

### T13 — Full Cluster Restart

**Tag:** `test_T13` | **Type:** Destructive | **Requires:** `-e run_destructive=true`

**What it does:**

1. Stops the entire cluster: `pcs cluster stop --all --wait=120`
2. Verifies corosync is stopped on all nodes
3. Waits 10 seconds
4. Starts the entire cluster: `pcs cluster start --all --wait=120`
5. Waits for both nodes to come online and all resources to start (up to 180s)
6. Runs `pcs resource cleanup`
7. Waits for GFS2 to start on both nodes
8. Waits for PostgreSQL to start
9. Tests PostgreSQL connectivity via VIP

**What it proves:**
- The cluster can perform a controlled full stop and restart
- All resources start correctly in the proper order after a full restart
- PostgreSQL is accessible via VIP after the restart

**Cleanup:** Always runs — `pcs resource cleanup` is performed.

---

## 7. Test Report

### Console Report

At the end of each test run, a summary report is printed to the console:

```
╔══════════════════════════════════════════════════════════════════════╗
║                  HA CLUSTER TEST REPORT                              ║
╠══════════════════════════════════════════════════════════════════════╣
║  Date      : 2026-03-19 15:30:00
║  Cluster   : ha_webpgsql
║  VIP       : 192.168.0.63:5432
╠══════════════════════════════════════════════════════════════════════╣
║  TEST RESULTS:
║
║  T01  Basic Health Check        PASSED (always runs)
║  T02  VIP Failover              PASSED (always runs)
║  T03  Active Node Shutdown      PASSED (ran)        or SKIPPED
║  ...
╠══════════════════════════════════════════════════════════════════════╣
║  FINAL CLUSTER STATUS:
║  [pcs status output]
╚══════════════════════════════════════════════════════════════════════╝
```

### Markdown Report File

A detailed Markdown report is written to:

```
/tmp/ha_cluster_test/ha_cluster_test_report_<TIMESTAMP>.md
```

The report contains:

| Section | Content |
|---|---|
| General Information | Date, cluster name, nodes, VIP, mount points, test mode |
| System Information | Kernel, OS, pcs/corosync/psql versions |
| Test Results Summary | Table with pass/skip status for each test |
| pcs status | Full cluster status output |
| Quorum Status | `corosync-quorumtool -s` output |
| Corosync Ring | Ring connection status |
| Constraints | `pcs constraint show --full` output |
| iSCSI Sessions | Active session list |
| Multipath Devices | `multipath -ll` output |
| LVM Lock Status | `lvmlockctl --dump` output |
| Disk / Mount Usage | `df -h` output from both nodes |
| PostgreSQL | VIP connection status and version |
| Resource Failcount | `pcs resource failcount show` output |
| Test Details | Description of what each test performed |

**Download the report from the cluster node:**

```bash
scp root@192.168.0.61:/tmp/ha_cluster_test/ha_cluster_test_report_*.md ./ha_cluster_report.md
```

---

## 8. Safety and Cleanup

### block / rescue / always Pattern

Every destructive test is wrapped in a `block / rescue / always` structure:

```yaml
block:
  # Test steps
rescue:
  # Actions on failure (e.g., remove iptables rules, restart stopped nodes)
  - fail: msg: "Test failed"
always:
  # Always runs regardless of pass/fail:
  # - Remove iptables rules
  # - Restart stopped nodes
  # - pcs resource cleanup
  # - Drop test tables
  # - pcs resource clear vip
```

This guarantees that even if a test fails halfway through, iptables rules are removed, stopped nodes are restarted, and test tables are cleaned up. The cluster is always left in a usable state.

### Automatic Pre-Test Cleanup

The PREP play removes leftovers from any previous test run before starting:

```bash
# Removes all test-related iptables rules from all nodes
# Drops all test tables (ha_test_t02, ha_test_t03, etc.)
# Clears all VIP location constraints
# Runs pcs resource cleanup
# Waits 15 seconds for stabilization
```

This means the test playbook is safe to run multiple times in succession.

### What Gets Cleaned Up

| Artifact | Cleanup Location |
|---|---|
| iptables rules (corosync, qdevice, iSCSI ports) | PREP + each test `always` block + RAPOR |
| VIP location constraints | Each test `always` block + RAPOR |
| PostgreSQL test tables | Each test `always` block + RAPOR |
| GFS2 test files (`ha_test_t07_*.txt`) | T07 `always` block + RAPOR |
| Resource failure counters | Each test + RAPOR final cleanup |
| Stopped nodes | T03, T09 `always` blocks |

---

## 9. Troubleshooting Test Failures

### T01 Failures

#### `Corosync ring error`
```bash
corosync-cfgtool -s
# Look for disconnected or lost entries

# Check corosync log
journalctl -u corosync -n 50

# Verify cluster network connectivity
ping -I enp2s0 172.16.16.62
```

#### `Quorum device not connected`
```bash
pcs quorum device status
systemctl status corosync-qdevice

# Re-add qdevice if needed
pcs quorum device remove
pcs quorum device add model net \
  host=clstr-qrm01.lab.akyuz.tech \
  algorithm=ffsplit port=5403
```

#### `Failed or stopped resources`
```bash
pcs status
pcs resource cleanup
journalctl -u pacemaker -n 50
```

---

### T02 / T11 / T12 Failures

#### `VIP did not move within timeout`
```bash
# Check for blocking constraints
pcs constraint

# Check resource status
pcs resource status vip

# Remove constraint and retry manually
pcs resource clear vip
pcs resource move vip clstr02.lab.akyuz.tech
```

#### `PostgreSQL not accessible after failover`
```bash
# Check where PostgreSQL is running
pcs resource status postgresql
pcs resource status vip

# They must be on the same node (colocation)
pcs constraint | grep postgresql

# If they're on different nodes, add missing colocation
pcs constraint colocation add postgresql with pgsql_fs INFINITY
pcs resource cleanup
```

---

### T03 Failures

#### `Failover did not complete`
```bash
# Check passive node
pcs status   # run from passive node

# Force resource cleanup
pcs resource cleanup

# Restart the stopped node if needed
pcs cluster start <node>
```

#### `Data consistency check failed`
```bash
# Verify PostgreSQL is running and accessible
psql -h 192.168.0.63 -U postgres -c "SELECT * FROM ha_test_t03;"
```

---

### T06 Failures

#### `PostgreSQL did not restart after kill -9`
```bash
# Check resource status
pcs resource status postgresql

# Check failure count (must be < migration-threshold=3)
pcs resource failcount show postgresql

# Clean up and retry
pcs resource cleanup postgresql
```

#### `Postmaster PID not found`
```bash
# Verify PostgreSQL is running
pcs resource status postgresql

# Find the process manually
pgrep -a postgres
```

---

### T09 Failures

#### `IPMI not reachable`
```bash
# Test manually
ipmitool -I lanplus \
  -H <ipmi_ip> -p <ipmi_port> \
  -U <ipmi_user> -P <ipmi_pass> \
  chassis power status

# Check hosts.yml: ipmi_ip, ipmi_port, ipmi_user, ipmi_password
```

#### `Node did not come back after fence`
```bash
# Check power status
ipmitool -I lanplus \
  -H <ipmi_ip> -p <ipmi_port> \
  -U <ipmi_user> -P <ipmi_pass> \
  chassis power status

# Power on if needed
ipmitool -I lanplus ... chassis power on

# Wait for boot and start cluster manually
pcs cluster start <node>
```

---

### General Test Failures

#### iptables rules stuck after a failed test
```bash
# Remove all test-related rules manually
iptables -D INPUT  -p udp --dport 5405 -j DROP 2>/dev/null
iptables -D OUTPUT -p udp --dport 5405 -j DROP 2>/dev/null
iptables -D INPUT  -p tcp --dport 5405 -j DROP 2>/dev/null
iptables -D OUTPUT -p tcp --dport 5405 -j DROP 2>/dev/null
iptables -D OUTPUT -p tcp --dport 5403 -j DROP 2>/dev/null
iptables -D INPUT  -p tcp --dport 5403 -j DROP 2>/dev/null
iptables -D OUTPUT -p tcp -d 172.16.16.248 --dport 3260 -j DROP 2>/dev/null
iptables -D INPUT  -p tcp -s 172.16.16.248 --dport 3260 -j DROP 2>/dev/null

# Or simply rerun the test playbook -- PREP always cleans up
ansible-playbook -i inventory/hosts.yml ha_cluster_test.yml
```

#### Location constraint stuck after a failed test
```bash
# Remove all VIP constraints
pcs resource clear vip
pcs constraint | grep vip
```

#### Test tables stuck in PostgreSQL
```bash
psql -h 192.168.0.63 -U postgres -c \
  "DROP TABLE IF EXISTS ha_test_t02, ha_test_t03, ha_test_t06,
   ha_load_t11, ha_test_t12 CASCADE;"
```

#### Cluster not clean before re-running tests
```bash
# Run cleanup steps manually
pcs resource cleanup
pcs resource clear vip
# Then rerun the test playbook -- PREP handles the rest
```

---

*Test playbook version: last updated 19 March 2026 · RHEL 9.7 · Pacemaker 2.1.x · pcs 0.11.x · PostgreSQL 18 · Ansible 2.14.x*
