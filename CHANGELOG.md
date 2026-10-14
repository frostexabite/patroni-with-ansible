# Changelog

## v2.0.0 — Rocky Linux / RHEL 8, 9, 10 and user-entered IPs

### Operating systems
- Supports Rocky Linux and RHEL 8, 9, 10 (AlmaLinux / Oracle Linux work too) instead of Oracle Linux 10 only.
- New `playbooks/detect_os.yml` (always runs): reads `/etc/os-release` with `raw`, checks the OS is supported,
  requires all nodes to share one major version, checks that EL8 is managed with ansible-core 2.16, and sets
  `ansible_python_interpreter` to the system Python (EL8 `/usr/libexec/platform-python`, EL9/10 `/usr/bin/python3`).
- The Patroni venv uses a Python per major version (`patroni_python_by_major`): EL8/9 = `python3.12`, EL10 = `python3`,
  because psycopg 3.3 needs Python ≥ 3.10. The version is checked, and a venv built with a different Python is detected.
- New default `pg_install_source: auto`: EL8/9 → PGDG, EL10 → AppStream.
  - PGDG: disables the AppStream `postgresql` module on EL8/9; uses repo `priority=1` instead of `excludepkgs` on the
    OS repo (repo ids differ per distro and subscription-manager rewrites them on RHEL); pins the baseurl only on EL10;
    imports the GPG key matching the architecture.
  - AppStream: EL8/9 enable module stream `postgresql:<ver>`, EL10 uses the `postgresql<ver>` packages.
  - Clear errors with a fix when a package is missing from the repositories.
- Base packages are no longer Oracle-specific (system-Python setuptools per major version, `iputils` added).
- keepalived build adds `pkgconf-pkg-config`; HAProxy package mode requires EL9+ (EL8 ships 1.8).
- etcd also supports ppc64le / s390x.
- `ansible.cfg`: `interpreter_python = auto_silent`.

### User-entered IPs / VIP
- `inventory/hosts.yml` is a template (`<IP-node-1>`, `<CIDR-cluster>`, `<VIP>`) with no site IPs baked in.
- New `scripts/setup-inventory.py`: interactive or argument-driven (`--ips`, `--nodes`, `--vip`, ...), validates the values,
  writes `hosts.yml` and backs up the previous file. Re-runs use current values as defaults and warn about keys the form
  does not manage. Standard library only (Python 3.6+).
- Preflight's first play validates the inventory on the control node before any SSH: leftover placeholders, IPv4 syntax,
  duplicate IPs, subnet membership, VIP not colliding with a node, network/broadcast addresses, prefix,
  virtual_router_id, host names.
- `keepalived_enabled`, `keepalived_router_id`, `keepalived_interface` are now site settings in `hosts.yml`
  (no longer defined in `group_vars/all`, so overriding them actually works).
- The VIP interface is the one that holds the node IP (previously the default-route interface).
- Preflight checks that the VIP interface exists and that the OS repositories provide the required packages.
- Default node names are `pg-node-N`; docs use shell variables (`$NODE1_IP`, `$VIP`) instead of fixed IPs.
- All documentation, comments and messages are in English.

## v1.0.0
- Oracle Linux 10, PostgreSQL 18 (AppStream/PGDG), etcd 3.7, Patroni 4.1, HAProxy 3.4 / keepalived 2.4 built from source.
