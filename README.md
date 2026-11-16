# server-bootstrap

Reusable Ansible roles for bootstrapping Linux servers. Roles currently manage **Prometheus node_exporter** and **Docker Engine**. Add future Nginx or other roles under `roles/`.

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

## Docker (Ubuntu 26.04, plus 22.04/24.04/26.10)

The Docker role uses the **official Docker APT stable repository** (not get.docker.com). It installs Docker Engine + CLI, containerd.io, Buildx and Compose plugins, configures docker/containerd services to start at boot, and adds the SSH login user to the `docker` group (equivalent to `sudo gpasswd -a <user> docker`).

**Recommended inventory:** specify `ansible_user` so the correct account is added:

```ini
[servers]
myserver ansible_host=192.0.2.10 ansible_user=ubuntu
```

Install or update to the latest **stable** package versions available for the Ubuntu release:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags docker
```

Install or upgrade Node Exporter and Docker together:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags node_exporter,docker
```

To **pin or upgrade to a specific Docker Engine version**, first run `apt-cache madison docker-ce` on a target server after the official Docker repository has been configured, then pass the exact complete Debian package version (including epoch and Ubuntu codename):

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags docker \
  -e 'docker_apt_version=5:29.9.3-1~ubuntu.26.04~resolute'
```

The version above is **an example format**, not a guarantee that the package exists in the current repository. With a pinned version, only Docker Engine and Docker CLI are pinned; the containerd, Buildx and Compose packages are installed from the repo, but aren't proactively upgraded if already present. Leave `docker_apt_version` empty to follow latest stable on later runs.

Override the group membership list if needed:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags docker \
  -e '{"docker_users":["ubuntu","deploy"]}'
```

Check after deployment:

```bash
docker --version
docker compose version
systemctl is-active docker containerd
id -nG ubuntu
```

**Important:**
- Group membership becomes effective for new login sessions; reconnect SSH or run `newgrp docker`.
- Members of the `docker` group effectively have root-level permissions; add only trusted users.
- If Ubuntu distribution Docker/containerd packages are installed, the role **stops** rather than forcibly uninstalling existing services. Review package conflicts manually first.
- A Docker package upgrade can restart the daemon; schedule changes on production hosts appropriately.
- Docker-published container ports can bypass parts of UFW/firewalld policy; review security groups and the `DOCKER-USER` firewall chain.
