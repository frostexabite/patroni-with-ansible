# Operations

The commands here use shell variables — set them to match `inventory/hosts.yml` first (once per terminal):

```bash
NODE1_IP=10.10.10.11      # ← IP of node 1 (see inventory/hosts.yml)
NODE2_IP=10.10.10.12
NODE3_IP=10.10.10.13
VIP=10.10.10.10
```

The examples use the node names `pg-node-1..3` (the `scripts/setup-inventory.py` default) — substitute your own.
Commands starting with `ansible` run on the control node inside `~/pg-ha-ansible` (after `source ~/ansible-venv/bin/activate`).

`psql` is on the PATH with both PGDG and AppStream packages — if it is not found, use `/usr/pgsql-18/bin/psql` (PGDG).

After installation every node has these helpers (log in again once to load them):
- `pctl` = `sudo patronictl -c /etc/patroni/patroni.yml`, e.g. `pctl list`, `pctl history`
- `ETCDCTL_ENDPOINTS` is preset → `etcdctl endpoint health` works without flags

---

## 1. Cluster health check

```bash
ansible-playbook playbooks/verify.yml --ask-vault-pass
```

Checks: all 4 services running on every node, etcd healthy, exactly one Leader, every replica `streaming`,
no maintenance mode, write port → primary, read port → replica, VIP on exactly one node.

## 2. Inspecting Patroni

### 2.1 Cluster state

```bash
pctl list
```

| Look at | Healthy | Problem |
|---|---|---|
| Leader | exactly 1 | 0 or >1 → check etcd |
| Replica State | `streaming` | stuck in `starting` / `stopped` / `creating replica` → check the logs |
| TL (timeline) | same on every node | differs → a replica has not caught up after a failover |
| Lag | 0 or close to 0 | keeps growing → the replica cannot keep up |
| Header | no `Maintenance mode` | present → `pctl resume` |

Refresh every 2 seconds: `pctl list -W` (exit with Ctrl+C)

### 2.2 Service and logs

```bash
sudo systemctl status patroni --no-pager
```
```bash
sudo journalctl -u patroni -n 30 --no-pager
```
```bash
sudo journalctl -u patroni --since "1 hour ago" --no-pager | grep -iE "error|warning|fail"
```

Normal log lines every 10 seconds:
- Leader: `no action. I am (pg-node-1), the leader with the lock`
- Replica: `no action. I am (pg-node-2), a secondary, and following a leader (pg-node-1)`

### 2.3 REST API (what HAProxy uses)

```bash
curl -s http://$NODE1_IP:8008/cluster | python3 -m json.tool
```
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://$NODE1_IP:8008/primary
```
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://$NODE2_IP:8008/replica
```
```bash
curl -s http://$NODE2_IP:8008/patroni | python3 -m json.tool
```

`/primary` returns 200 only on the Leader, `/replica` returns 200 only on a healthy replica.

### 2.4 Replication on the PostgreSQL side (on the Leader)

```bash
sudo -u postgres psql -c "SELECT application_name, client_addr, state, sync_state, write_lag, replay_lag FROM pg_stat_replication;"
```
```bash
sudo -u postgres psql -c "SELECT slot_name, active, restart_lsn FROM pg_replication_slots;"
```

Expect 2 `streaming` rows and both slots with `active = t`.

### 2.5 Configuration and history

```bash
pctl show-config
```
```bash
pctl history
```
```bash
pctl list -e
```

`list -e` adds a `Pending restart` column — a value there means a restart is needed for a new parameter to take effect.

## 3. etcd / HAProxy / VIP

```bash
etcdctl endpoint health -w table
```
```bash
etcdctl member list -w table
```
```bash
ip -brief addr show to $VIP
```

HAProxy stats: `http://<VIP>:7000` (user `admin`, password `vault_haproxy_stats_password`)
- `postgres_write`: UP only on the Leader
- `postgres_read`: UP only on the replicas

## 4. Replication smoke test

```bash
psql -h $VIP -p 5000 -U postgres -c "CREATE TABLE ha_test(id int, note text); INSERT INTO ha_test VALUES (1, 'hello from leader');"
```
```bash
psql -h $VIP -p 5001 -U postgres -c "SELECT inet_server_addr(), * FROM ha_test;"
```
```bash
psql -h $VIP -p 5000 -U postgres -c "DROP TABLE ha_test;"
```

## 5. Failover tests (do these before going live)

### 5.1 Planned switchover

```bash
ansible-playbook playbooks/switchover.yml -e switchover_candidate=pg-node-2 --ask-vault-pass
```
```bash
psql -h $VIP -p 5000 -U postgres -Atc "SELECT inet_server_addr(), pg_is_in_recovery();"
```

Expect node 2's IP followed by `|f`.

### 5.2 Automatic failover (simulated Leader crash) — on the current Leader

```bash
sudo systemctl stop patroni
```

Wait ~30 seconds, then on another node:

```bash
pctl list
```

There must be a new Leader. Then, back on the original machine:

```bash
sudo systemctl start patroni
```

The old node rejoins as a Replica by itself (using pg_rewind when needed).

### 5.3 VIP failover — on the node holding the VIP

```bash
sudo systemctl stop haproxy
```
```bash
ip -brief addr show to $VIP
```

The VIP must disappear from this node and move to the next one. Restore with:

```bash
sudo systemctl start haproxy
```

## 6. Changes and upgrades

| Task | How |
|---|---|
| Preview what would change | `ansible-playbook site.yml --ask-vault-pass --check --diff` |
| PostgreSQL parameter | `pctl edit-config`, then `pctl restart postgres-cluster` if a restart is needed (`pg_dcs_parameters` is only used at the first bootstrap) |
| subnet / pg_hba / app subnet / VIP | `python3 scripts/setup-inventory.py` (or edit `inventory/hosts.yml`) → `ansible-playbook site.yml --ask-vault-pass` (Patroni reloads immediately, stale firewall rules are removed) |
| PostgreSQL minor | `package_state: latest` updates the RPMs → `pctl restart postgres-cluster pg-node-2 --force` one replica at a time, then switch over and restart the old leader |
| Patroni / psycopg | change `patroni_version` / `psycopg_version` → run `--tags patroni` (pause → restart one node at a time → resume, no failover) |
| etcd | change `etcd_version` → `--tags etcd` (restarts one node at a time, blocks minor-version skips) — take a snapshot first: `etcdctl snapshot save /root/etcd-$(date +%F).db` |
| HAProxy | change `haproxy_version` → `--tags haproxy` (rebuild + reload without dropping connections) |
| keepalived | change `keepalived_version` and `keepalived_git_commit` → `--tags keepalived` |
| OS minor (e.g. 9.5 → 9.6) | `os_update_all: true` updates during `site.yml` → reboot one node at a time as in the row below |
| OS major (8 → 9 → 10) | not in place — build a new cluster on the new major and migrate the data (all nodes must share one major) |
| OS reboot | one node at a time, replicas first: if it is the Leader, switch over first → `sudo reboot` → wait for `pctl list` to show it streaming → next node |

Finding the latest versions:

```bash
pip index versions patroni
```
```bash
git ls-remote --tags https://github.com/etcd-io/etcd.git 'v3.*' | grep -v '\^{}' | awk -F/ '{print $3}' | grep -vE 'rc|alpha|beta' | sort -V | tail -1
```
```bash
git ls-remote --tags https://github.com/acassen/keepalived.git | grep -v '\^{}' | awk -F/ '{print $3}' | sort -V | tail -1
```

HAProxy: check the latest LTS branch at https://www.haproxy.org/
