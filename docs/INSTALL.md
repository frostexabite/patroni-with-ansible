# Installing the PostgreSQL HA cluster on Rocky Linux / RHEL (copy & paste)

Supported: Rocky Linux / RHEL **8 (8.10), 9 (9.4+), 10** — all nodes must run the same major version.

## Your values

Have these ready before you start (the values in parentheses are only examples).

| | Value |
|---|---|
| nodes 1–3 | name + IP, e.g. `pg-node-1` (10.10.10.11), `pg-node-2` (10.10.10.12), `pg-node-3` (10.10.10.13) |
| cluster subnet | subnet all nodes live in, e.g. `10.10.10.0/24` |
| app subnet | subnet of the applications that connect to the database, e.g. `10.10.20.0/24` |
| VIP | a free IP in the cluster subnet, e.g. `10.10.10.10` |
| SSH user | `ansible` (passwordless sudo) |

The commands below use shell variables — **set them once per terminal**, then copy the commands as they are:

```bash
NODE1_IP=10.10.10.11      # ← replace with your real IPs
NODE2_IP=10.10.10.12
NODE3_IP=10.10.10.13
VIP=10.10.10.10
```

---

## Part A — every node (run as root on each machine)

Install Rocky Linux or RHEL (Minimal is enough) and configure a static IP.

### A1. (RHEL only) Register and enable repositories

```bash
subscription-manager register
```
```bash
subscription-manager repos --enable "rhel-$(rpm -E %rhel)-for-$(uname -m)-baseos-rpms" --enable "rhel-$(rpm -E %rhel)-for-$(uname -m)-appstream-rpms"
```

Skip this step on Rocky Linux.

### A2. Create the `ansible` user

```bash
useradd -m ansible
```
```bash
passwd ansible
```
```bash
echo 'ansible ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/ansible
```
```bash
chmod 440 /etc/sudoers.d/ansible
```

### A3. Check the OS and repositories

```bash
cat /etc/os-release | grep -E '^(NAME|VERSION_ID)='
```
```bash
dnf repolist
```

BaseOS and AppStream must be listed:

| OS | Expected repositories |
|---|---|
| Rocky Linux | `baseos`, `appstream` |
| RHEL | `rhel-<8/9/10>-for-x86_64-baseos-rpms`, `rhel-<8/9/10>-for-x86_64-appstream-rpms` |

EL8 must be 8.10 and EL9 must be 9.4 or newer (they provide `python3.12` in AppStream) — if older, run
`dnf upgrade -y && reboot` first.

> Do not install `postgresql-server` or run `initdb` yourself — Patroni creates the data directory.

---

## Part B — control node (the machine that runs Ansible)

The control node can be one of the cluster nodes (e.g. node 1) or a separate machine.

### B1. Install Ansible

```bash
sudo dnf install -y git python3.12 python3.12-pip openssh-clients unzip
```
```bash
python3.12 -m venv ~/ansible-venv
```
```bash
source ~/ansible-venv/bin/activate
```
```bash
pip install -U pip
```

Pick the ansible-core version based on the OS of the **cluster nodes** (not of the control node):

| Cluster nodes | Command |
|---|---|
| any EL8 (Rocky/RHEL 8) | `pip install 'ansible-core>=2.16,<2.17'` |
| EL9 or EL10 only | `pip install ansible-core ansible-lint` |

```bash
ansible --version
```

> In every new terminal run `source ~/ansible-venv/bin/activate` first.
>
> Why EL8 needs 2.16: on EL8 the only Python with dnf/SELinux/firewalld bindings is platform-python 3.6,
> which ansible-core 2.17 and later no longer support — preflight stops with instructions if the version is wrong.

### B2. Copy your SSH key to every node

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```
```bash
for ip in $NODE1_IP $NODE2_IP $NODE3_IP; do ssh-copy-id ansible@$ip; done
```

### B3. Get the project

```bash
git clone <your-repository-url> ~/pg-ha-ansible
```
```bash
cd ~/pg-ha-ansible
```
```bash
git config core.hooksPath scripts/git-hooks
```
```bash
ansible-galaxy collection install -r requirements.yml -p ~/.ansible/collections
```
```bash
ansible-galaxy collection list ansible.posix
```

If `ansible-galaxy` cannot reach galaxy.ansible.com, install `pip install -U ansible` instead (it bundles ansible.posix;
with EL8 nodes use `pip install 'ansible>=9,<10'`, which ships ansible-core 2.16).

### B4. Enter IPs / subnets / VIP

```bash
python3 scripts/setup-inventory.py
```

Answer the prompts (Enter = use the value in brackets). The script validates IPs, duplicates, subnets and the VIP,
then writes `inventory/hosts.yml`. Or pass everything on one line using the variables set above:

```bash
python3 scripts/setup-inventory.py --ips $NODE1_IP,$NODE2_IP,$NODE3_IP --vip $VIP --app-cidrs 10.10.20.0/24 --yes
```

Check the result:

```bash
cat inventory/hosts.yml
```

To change something later, re-run the script (current values become the defaults) or edit the file directly — the previous
file is backed up as `hosts.yml.bak-<timestamp>`.

### B5. Passwords (Vault)

Change the passwords before pasting — `vault_keepalived_auth_pass` must be 8 characters or fewer.

```bash
cat > ~/pg-ha-ansible/inventory/group_vars/all/vault.yml <<'EOF'
---
vault_pg_superuser_password: "Super#Pass2026"
vault_pg_replication_password: "Repl#Pass2026"
vault_pg_rewind_password: "Rewind#Pass2026"
vault_patroni_restapi_password: "Api#Pass2026"
vault_haproxy_stats_password: "Stats#Pass2026"
vault_keepalived_auth_pass: "Vrrp2026"
EOF
```
```bash
ansible-vault encrypt ~/pg-ha-ansible/inventory/group_vars/all/vault.yml
```

> Every `ansible` / `ansible-playbook` command needs `--ask-vault-pass` (even `-m ping`).

### B6. Test connectivity

```bash
ansible pg_cluster -m ping --ask-vault-pass
```
```bash
ansible pg_cluster -m command -a "cat /etc/os-release" --ask-vault-pass | grep -E 'SUCCESS|PRETTY_NAME'
```

### B7. Readiness checks (changes nothing)

```bash
ansible-playbook playbooks/preflight.yml --ask-vault-pass
```

Checks: the values in `hosts.yml` (IPs/subnets/VIP), node count, all secrets present, OS and identical major version on
every node, ansible-core version, repositories provide `python3.12` (EL8/9), each IP is configured on its node, the VIP is
free, download sources are reachable, time is synchronised. It ends with the plan for each node, for example:

```
Rocky 9.6 | IP 10.10.10.11 | VRRP on ens192 | Patroni Python /usr/bin/python3.12 | PostgreSQL 18 from pgdg (/usr/pgsql-18/bin)
```

### B8. Install

```bash
ansible-playbook site.yml --ask-vault-pass
```

Takes about 15–30 minutes (HAProxy/keepalived are compiled) and finishes with an automatic verify.
Expected result: `failed=0` on every node.

### B9. Commit your settings (private repository)

```bash
git add -A && git commit -m "Deploy cluster $NODE1_IP,$NODE2_IP,$NODE3_IP VIP $VIP"
```

---

## Part C — post-install checks

```bash
ansible-playbook playbooks/verify.yml --ask-vault-pass
```

Expect `Leader pg-node-1, replicas streaming: pg-node-2, pg-node-3` (with your node names),
`recovery=false` on the write port, `recovery=true` on the read port, and the VIP on exactly one node.

Detailed checks and failover tests: [OPERATIONS.md](OPERATIONS.md)

## Application connection

| | Value |
|---|---|
| Host | your VIP (`$VIP`) |
| Write (primary) | port `5000` |
| Read (replicas, round-robin) | port `5001` |
| HAProxy stats | `http://<VIP>:7000` (user `admin`) |
