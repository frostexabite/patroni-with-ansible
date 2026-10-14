# PostgreSQL HA on Rocky Linux / RHEL 8 · 9 · 10

**Patroni + etcd + HAProxy + keepalived, deployed with Ansible**

An Ansible project that builds a highly available PostgreSQL cluster in a **stacked topology** (every node runs every component).
It is adapted from Foyez Ahmed Rony's article *Building a Highly Available PostgreSQL Cluster with Patroni, etcd, and HAProxy
(Oracle Linux 8 & PostgreSQL 16)* and brought up to date to run on **every supported release of Rocky Linux and Red Hat
Enterprise Linux (8, 9, 10)** with the latest version of each component. Node IPs, subnets and the VIP are entered by you —
nothing site-specific is hard-coded.

```
                         Application
                              │
                   VIP <VIP> (keepalived / VRRP)
                    :5000 write      :5001 read
        ┌─────────────────────┼─────────────────────┐
 ┌──────┴────────┐    ┌───────┴───────┐    ┌────────┴──────┐
 │ pg-node-1     │    │ pg-node-2     │    │ pg-node-3     │
 │ HAProxy       │    │ HAProxy       │    │ HAProxy       │  health check → Patroni REST :8008
 │ Patroni       │    │ Patroni       │    │ Patroni       │  (/primary, /replica)
 │ PostgreSQL 18 │═══▶│ PostgreSQL 18 │    │ PostgreSQL 18 │  streaming replication :5432
 │   (Leader)    │═══════════════════════▶ │   (Replica)   │
 │ etcd          │◀──▶│ etcd          │◀──▶│ etcd          │  quorum :2379 / :2380
 └───────────────┘    └───────────────┘    └───────────────┘
   <IP-node-1>          <IP-node-2>          <IP-node-3>
```

Node names and the node count (3, 5, 7, ...) are chosen when you enter your settings.

## Features

- One playbook (`site.yml`) from bare OS to a verified cluster: preflight → OS → PostgreSQL → etcd → Patroni → HAProxy → keepalived → verify
- OS-aware: detects EL 8 / 9 / 10 and picks the right Python, package source and repository handling automatically
- Interactive inventory setup with validation (`scripts/setup-inventory.py`) — no IPs to hunt down in config files
- Read-only preflight that catches bad IPs, VIP conflicts, missing repositories and wrong Ansible versions **before** touching a node
- Secure defaults: firewalld with per-subnet rules, `scram-sha-256`, Ansible Vault for every secret, SHA256/commit-verified downloads
- Safe re-runs on a live cluster: ordered bootstrap, one-node-at-a-time restarts, Patroni maintenance mode for daemon upgrades
- Health check (`verify.yml`) and planned switchover (`switchover.yml`) playbooks

## Supported operating systems

| OS | System Python (used by Ansible) | Python for Patroni | PostgreSQL 18 (`pg_install_source: auto`) | ansible-core on the control node |
|---|---|---|---|---|
| Rocky Linux / RHEL **8** (8.10) | platform-python 3.6 | `python3.12` from AppStream | PGDG (AppStream `postgresql` module disabled) | **2.16.x only** |
| Rocky Linux / RHEL **9** (9.4+) | python3 3.9 | `python3.12` from AppStream | PGDG (AppStream `postgresql` module disabled) | 2.16 – 2.21 |
| Rocky Linux / RHEL **10** | python3 3.12 | `python3` (3.12) | AppStream `postgresql18` | 2.16 – 2.21 |

- AlmaLinux and Oracle Linux 8/9/10 work the same way (IDs `almalinux`, `ol`).
- All nodes in a cluster must run the same major version (glibc collations must match for streaming replication).
- RHEL must be registered (`subscription-manager register`) with BaseOS and AppStream enabled.
- **EL8**: ansible-core 2.17+ cannot manage Python 3.6, and on EL8 the dnf bindings exist only for Python 3.6 — so use
  ansible-core 2.16. Preflight checks this and tells you how to fix it.
- RHEL/CentOS 7 is not supported: it is end-of-life and has no Python ≥ 3.10 for psycopg 3.3 (there is no Rocky Linux 7).

The per-OS choices are made automatically (`playbooks/detect_os.yml` + `inventory/group_vars/all/main.yml`) and can be
overridden with `os_system_python`, `patroni_python_by_major` and `pg_install_source`.

## Component versions (latest as of 2026-10-12)

| Component | Version | Installed from | Variable |
|---|---|---|---|
| OS packages | always latest | `dnf upgrade` + `state: latest` | `os_update_all`, `package_state` |
| PostgreSQL | **18.6** | EL8/9: PGDG · EL10: AppStream (configurable) | `pg_version`, `pg_install_source` |
| etcd | **3.7.2** | GitHub release, SHA256-verified | `etcd_version` |
| Patroni | **4.1.5** | PyPI → venv `/opt/patroni` | `patroni_version` |
| psycopg | **3.3.6** (needs Python ≥ 3.10) | PyPI | `psycopg_version` |
| HAProxy | **3.4.6** (LTS) | built from source (haproxy.org, SHA256-verified) | `haproxy_version` |
| keepalived | **2.4.3** | built from the Git tag, commit-verified | `keepalived_version` |
| Ansible | ansible-core 2.16 (with EL8) / 2.16–2.21 · ansible.posix ≥ 2.2.2 | control node | `requirements.yml` |

PostgreSQL 19 is still in beta at the time of writing, so 18 is used.

## Documentation

| File | Contents |
|---|---|
| [docs/INSTALL.md](docs/INSTALL.md) | Copy-and-paste installation from bare machines (Rocky / RHEL) |
| [docs/OPERATIONS.md](docs/OPERATIONS.md) | Health checks, Patroni inspection, failover tests, upgrades, routine tasks |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Common errors and how to fix them |
| [CHANGELOG.md](CHANGELOG.md) | Changes per release |

## Quick start

```bash
ansible-galaxy collection install -r requirements.yml
python3 scripts/setup-inventory.py                          # prompts for node IPs, subnets, VIP → writes inventory/hosts.yml
cp inventory/group_vars/all/vault.yml.example inventory/group_vars/all/vault.yml
vi inventory/group_vars/all/vault.yml && ansible-vault encrypt inventory/group_vars/all/vault.yml
ansible-playbook playbooks/preflight.yml --ask-vault-pass  # readiness checks (changes nothing)
ansible-playbook site.yml --ask-vault-pass                 # install + verify
```

## Entering IPs, subnets and the VIP

All site settings live in a single file, `inventory/hosts.yml`. The shipped file is a **template** (`<IP-node-1>`, `<VIP>`, ...).
Fill it in one of two ways.

**1. Interactive (recommended)** — every value is validated, the previous file is backed up, then the file is rewritten:

```bash
python3 scripts/setup-inventory.py
```

```
Number of nodes (odd, >= 3) [3]:
Node 1  host name [pg-node-1]:
Node 1  IP: 10.10.10.11
...
Cluster subnet (CIDR) [10.10.10.0/24]:
Application subnets allowed to connect (comma-separated) [10.10.10.0/24]: 10.10.20.0/24
Use a virtual IP for applications [Y/n]:
VIP (a free IP in 10.10.10.0/24): 10.10.10.10
```

Re-run it any time; the current values become the defaults. Or pass everything on one line (handy for scripts):

```bash
python3 scripts/setup-inventory.py --ips 10.10.10.11,10.10.10.12,10.10.10.13 --vip 10.10.10.10 --yes
python3 scripts/setup-inventory.py --nodes db1=10.0.0.11,db2=10.0.0.12,db3=10.0.0.13 \
    --cidr 10.0.0.0/24 --app-cidrs 10.0.1.0/24,10.0.2.0/24 --vip 10.0.0.10 --router-id 60 --yes
python3 scripts/setup-inventory.py --help
```

**2. Edit the file yourself** — replace every `<...>` placeholder in `inventory/hosts.yml`.

Either way, preflight validates the values on the control node before any SSH connection: leftover `<...>` placeholders,
malformed or duplicate IPs, IPs or VIP outside `cluster_network_cidr`, VIP colliding with a node, even node count.
It then checks on the real machines that each IP is configured on its node and that the VIP is still free.

**Only two files need editing for a new environment**

| File | Contents |
|---|---|
| `inventory/hosts.yml` | node names, IPs, `cluster_network_cidr`, `app_allowed_cidrs`, VIP (written by `scripts/setup-inventory.py`) |
| `inventory/group_vars/all/vault.yml` | all passwords (encrypted with ansible-vault) |

`inventory/group_vars/all/main.yml` holds defaults and tuning — leave it alone unless you want to tune something.
Do not repeat site settings there: `group_vars/all` takes precedence over group vars in `hosts.yml`.

## Repository layout

```
.
├── ansible.cfg
├── requirements.yml                  # ansible.posix
├── site.yml                          # validate inventory → detect OS → preflight → roles → verify
├── inventory/
│   ├── hosts.yml                     # site settings (template — fill in with scripts/setup-inventory.py)
│   └── group_vars/all/
│       ├── main.yml                  # defaults / tuning / versions / per-EL-major values
│       └── vault.yml.example         # password template
├── playbooks/
│   ├── detect_os.yml                 # read /etc/os-release, check the OS, pick the system Python (always runs)
│   ├── preflight.yml                 # validate the inventory + node readiness (read-only)
│   ├── verify.yml                    # cluster health check (read-only)
│   └── switchover.yml                # planned switchover
├── roles/
│   ├── common/      # OS update, hostname, /etc/hosts, chrony, SELinux, firewalld (+ stale rule cleanup)
│   ├── postgresql/  # PGDG or AppStream (EL8/9 module stream, EL10 postgresqlNN), no initdb
│   ├── etcd/        # binary + checksum, config, systemd, upgrade guard
│   ├── patroni/     # venv (python3.12 on EL8/9), patroni.yml, ordered bootstrap, safe rolling restart
│   ├── haproxy/     # source build of 3.4 LTS, write/read/stats listeners
│   └── keepalived/  # source build, VIP, unicast VRRP, tracks haproxy
├── scripts/
│   ├── setup-inventory.py            # enter IPs / subnets / VIP interactively → inventory/hosts.yml
│   └── git-hooks/pre-commit          # refuses to commit an unencrypted vault.yml
└── docs/
```

## Changes from the original article

| Topic | Original (OL8) | This project | Why |
|---|---|---|---|
| OS | Oracle Linux 8 | Rocky Linux / RHEL 8, 9, 10 (settings adapt to the major version) | |
| PostgreSQL | PGDG EL-8, PG 16 | PG 18.6 — EL8/9: PGDG (AppStream module disabled), EL10: AppStream | EL8/9 AppStream lacks PG 18 on many minors |
| PGDG on EL10 | — | baseurl pinned to `rhel-10-$basearch` + `priority=1` | no `$releasever_minor` → 404, and AppStream ships same-name `postgresql18` |
| Patroni | `pip3` into the system Python | venv + pinned versions + psycopg3, Python ≥ 3.10 per OS | keeps the system Python clean; psycopg 3.3 needs 3.10+ |
| Ansible Python | auto | the system Python (platform-python on EL8), set per node | dnf/SELinux/firewalld bindings exist only there |
| etcd | 3.5.17 | 3.7.2 + SHA256 check + minor-skip guard | |
| Firewall / SELinux | firewalld off, `setenforce 0` | firewalld on, only required ports, source-restricted | safer |
| Auth | `md5`, plaintext passwords | `scram-sha-256`, Ansible Vault, separate `rewind_user` | |
| pg_hba | under `bootstrap` | under `postgresql` + reloaded immediately on change | subnet changes apply without a re-bootstrap |
| HAProxy check | `OPTIONS /master` | `GET /primary` | `/master` is deprecated |
| keepalived | `killall -0`, `weight 2` | `vrrp_track_process`, `weight 0` | in the original the VIP never moved when haproxy died (150 > 140+2) |
| keepalived auth | `patroni_vip` (11 chars) | checked ≤ 8 chars | VRRP PASS silently truncates longer values |
| VRRP | multicast, `eth0` | unicast, interface that holds the node IP | VMs/clouds often block multicast; RHEL uses `ens*` names |
| IPs / VIP | hard-coded in configs | entered via `scripts/setup-inventory.py` + validated before SSH | nothing site-specific in the code |
| Pre-install checks | — | `preflight.yml`: inventory, OS, repos, subnet, free VIP, internet | catches problems before changing anything |

## Git workflow

- `vault.yml` may only be committed encrypted — enable the hook with `git config core.hooksPath scripts/git-hooks`
- Before pushing: `ansible-lint && ansible-playbook site.yml --syntax-check`
- Before changing a running cluster: `ansible-playbook site.yml --ask-vault-pass --check --diff`
- Multiple environments: copy the whole `inventory/` folder to e.g. `inventories/uat/` (group_vars must sit next to hosts.yml),
  run `python3 scripts/setup-inventory.py -o inventories/uat/hosts.yml`, then use `-i inventories/uat/hosts.yml`
- Never commit a filled-in `inventory/hosts.yml` to a public repository if your IPs are sensitive — keep it in a private repo or fork

## Credits

Based on *Building a Highly Available PostgreSQL Cluster with Patroni, etcd, and HAProxy (Oracle Linux 8 & PostgreSQL 16)*
by Foyez Ahmed Rony.
