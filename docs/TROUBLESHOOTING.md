# Troubleshooting

## Preflight / playbook messages and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `... is not filled in (value: '<IP-node-1>')` | `inventory/hosts.yml` is still the template | `python3 scripts/setup-inventory.py`, or replace the `<...>` values in the file |
| `... is outside cluster_network_cidr` / `already used by` / `not an IPv4` | IPs, subnet and VIP do not fit together | re-run `scripts/setup-inventory.py` (validates each value) |
| `runs "CentOS Stream 9" ... Supported: rocky, rhel, almalinux, ol` | the OS is not on the supported list | use Rocky/RHEL 8–10, or `-e allow_unsupported_os=true` (settings follow the major version) |
| `Mixed EL major versions` | nodes run different major versions (e.g. 9 and 10) | install every node with the same major (glibc collations must match for replication) |
| `EL8 nodes need ansible-core 2.16` | ansible-core ≥ 2.17 cannot manage Python 3.6 (EL8's platform-python) | on the control node: `pip install 'ansible-core>=2.16,<2.17'` |
| `enabled repositories do not provide python3.12` | RHEL not registered / EL8 < 8.10 / EL9 < 9.4 | `subscription-manager register` + enable BaseOS/AppStream, or `dnf upgrade -y && reboot`, or point `patroni_python_by_major` at `python3.11` |
| `This system is not registered with an entitlement server` | RHEL is not registered | `subscription-manager register`, then `subscription-manager repos --list-enabled` |
| `AppStream ... has no postgresql:18 module stream` | `pg_install_source: appstream` on an EL8/9 release without that stream | use `pg_install_source: auto` (= pgdg on EL8/9) |
| `Could not install postgresql18 ... from appstream` | an older EL10 minor without `postgresql18` in AppStream | `dnf upgrade` to the latest minor, or `pg_install_source: pgdg` |
| `No match for argument: postgresql18-server` (PGDG on EL8/9) | the AppStream `postgresql` module hides the PGDG packages | the project disables the module — if it still happens: `sudo dnf -qy module disable postgresql` |
| `Failed to download metadata for repo 'pgdg...'` 404 `rhel-10.-x86_64` | EL10 has no `$releasever_minor`, breaking the PGDG URL | fixed (pinned to `rhel-10-$basearch`) — check `pgdg_pin_major_baseurl` |
| `firewalld and its python module are required` / `Failed to import the required Python library (libselinux-python)` | Ansible runs a non-system Python (e.g. python3.12) | do not set `ansible_python_interpreter` to python3.12 — let `detect_os.yml` choose (EL8 `/usr/libexec/platform-python`, EL9/10 `/usr/bin/python3`) |
| `Require Python >= 3.10 for Patroni` | `patroni_python` points to an older Python | adjust `patroni_python_by_major` in `group_vars/all/main.yml` |
| `uses Python 3.11 but patroni_python ... is 3.12` | Patroni's Python changed on a running cluster | follow the steps in the message (pause → stop → remove venv → `--tags patroni` → resume) |
| `EL8 AppStream ships HAProxy 1.8` | `haproxy_install_method: package` on EL8 | use `source` (the default) on EL8 |
| `Attempting to decrypt but no vault secrets found` | `vault.yml` is encrypted — every ansible command needs the vault password | add `--ask-vault-pass`, or set `vault_password_file` in `ansible.cfg` (see below) |
| `couldn't resolve module/action 'ansible.posix.selinux'` | the collection is not installed | `ansible-galaxy collection install -r requirements.yml -p ~/.ansible/collections` |
| Patroni validate: `Directory '/usr/pgsql-18/bin' does not exist` | packages came from the other source (AppStream binaries live in `/usr/bin`) | never switch `pg_install_source` on an installed cluster — `dnf info --installed postgresql18-server \| grep 'From repo'` |
| Replica loops on `no pg_hba.conf entry for replication connection from host "x.x.x.x"` | `cluster_network_cidr` does not cover the node's real IP | preflight catches this before install — fix with `scripts/setup-inventory.py` and re-run `site.yml` |
| Leader still uses the old `pg_hba.conf` after a subnet change | the reload was deferred to the end of the play | fixed (`flush_handlers` before waiting for replicas) — `sudo systemctl reload patroni` on the Leader |
| Your settings disappeared after extracting a new release over the old one | `hosts.yml` was replaced by the template | back up `inventory/hosts.yml` and `vault.yml` before updating, or use git — `scripts/setup-inventory.py` always keeps a `.bak-<timestamp>` copy |
| Changing VIP/router id in `group_vars/all/main.yml` or in `hosts.yml` has no effect | the site value is defined in both places (`group_vars/all` beats group vars in `hosts.yml`) | keep site values in `inventory/hosts.yml` only |

## Using a vault password file (no `--ask-vault-pass`)

Replace `your-vault-password` with the real password.

```bash
echo 'your-vault-password' > ~/.vault_pass_pgha
```
```bash
chmod 600 ~/.vault_pass_pgha
```
```bash
cd ~/pg-ha-ansible && sed -i 's|^# vault_password_file = .*|vault_password_file = ~/.vault_pass_pgha|' ansible.cfg
```

## By symptom

| Symptom | What to check |
|---|---|
| preflight: `NOT inside` / `outside cluster_network_cidr` | `ansible pg_cluster -m command -a "ip -brief addr show scope global" --ask-vault-pass`, then fix with `scripts/setup-inventory.py` |
| preflight: `is not an address of <node>` | `ansible_host` in `hosts.yml` is not the node's cluster-network IP (e.g. a NAT or management IP — set `node_ip` for that host) |
| preflight: `keepalived interface ... not found` | set `keepalived_interface: <name>` in `hosts.yml` (see `ip -brief link`) |
| preflight: `Another device already uses <VIP>` | pick a new VIP with `scripts/setup-inventory.py` |
| preflight: `cannot reach: https://...` | proxy (`/etc/dnf/dnf.conf`, environment), outbound firewall, DNS, SSL inspection (CA in `/etc/pki/ca-trust/source/anchors/` + `update-ca-trust`) |
| etcd not healthy | `sudo journalctl -u etcd -n 50`, ports 2379/2380, clocks in sync (`chronyc tracking`) |
| replica stuck in `creating replica` | `sudo journalctl -u patroni -f` on the replica; for large databases raise `patroni_replica_wait_retries` |
| leftover files in a replica's data dir | only on the broken replica (never on the Leader): `sudo systemctl stop patroni` → `sudo rm -rf /var/lib/pgsql/18/data/*` → `sudo systemctl start patroni` |
| all HAProxy servers DOWN | `curl -s -o /dev/null -w '%{http_code}' http://<node>:8008/primary`, `getsebool haproxy_connect_any` |
| VIP does not come up | `sudo journalctl -u keepalived -n 50`, `virtual_router_id` clashing with another system on the L2 (change it with `scripts/setup-inventory.py`), switches blocking gratuitous ARP / VRRP, cloud VMs that do not support this kind of VIP |
| `Maintenance mode` stuck in `pctl list` | `pctl resume` |
| want SELinux enforcing | set `selinux_state: enforcing`, test in UAT first, then check `sudo ausearch -m avc -ts recent` |
| which Python does Ansible use? | the `Detected OS` line at the start of the run, e.g. `Rocky Linux 8.10 → Ansible Python /usr/libexec/platform-python` |

## Logs worth reading

| Component | Command |
|---|---|
| Patroni | `sudo journalctl -u patroni -n 50 --no-pager` |
| PostgreSQL | `sudo tail -50 /var/lib/pgsql/18/data/log/*.log` |
| etcd | `sudo journalctl -u etcd -n 50 --no-pager` |
| HAProxy | `sudo journalctl -u haproxy -n 50 --no-pager` |
| keepalived | `sudo journalctl -u keepalived -n 50 --no-pager` |
| dnf (package install failures) | `sudo dnf repolist -v`, `sudo dnf module list postgresql` (EL8/9) |
