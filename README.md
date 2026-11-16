# server-bootstrap

Reusable Ansible roles for bootstrapping Linux servers. The first role installs and manages **Prometheus node_exporter**. Future Docker and Nginx roles can be added under `roles/` without changing the existing role.

## Requirements

- Ansible installed on the control machine
- SSH access with sudo/become privileges on the target hosts
- Linux x86_64 (amd64) or aarch64 (arm64) hosts using systemd
- Outbound HTTPS access to GitHub Releases on target hosts

## Layout

```text
site.yml
inventory/example.ini
roles/
  node_exporter/
    defaults/main.yml
    tasks/main.yml
    handlers/main.yml
    templates/node_exporter.service.j2
```

## Install

Create an inventory (the included example is intentionally not a deployable host):

```ini
[servers]
server1 ansible_host=192.0.2.10 ansible_user=ubuntu
```

Run:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags node_exporter
```

Defaults:

| Setting | Value |
| --- | --- |
| Version | `1.12.1` |
| Binary | `/usr/local/bin/node_exporter` |
| systemd service | `node_exporter.service` |
| Account | `node_exporter:node_exporter` (system user/group) |
| Listen address | `:8220` |
| Startup | enabled at boot |

The role downloads the upstream release, verifies its official SHA256 checksum, installs the binary and configures systemd. It skips downloading when the requested version is already installed.

## Upgrade

Override the version; the same command works for installation and upgrade:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags node_exporter \
  -e node_exporter_version=1.12.2
```

Replace `1.12.2` with an **actually published** release version. The role reads `sha256sums.txt` from that release automatically; it does not assume that version exists.

To change the port:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags node_exporter \
  -e node_exporter_listen_address=:8221
```

## Verify

```bash
systemctl status node_exporter
/usr/local/bin/node_exporter --version
curl -fsS http://127.0.0.1:8220/metrics
```

**Security:** the default `:8220` listens on all interfaces, matching the requested configuration. Restrict inbound TCP 8220 to your Prometheus scrapers via the host firewall or cloud security groups; do not expose it publicly unless intended.
