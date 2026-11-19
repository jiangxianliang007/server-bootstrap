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

## Docker (Ubuntu 26.04 or newer)

The Docker role uses the **official Docker APT stable repository** (not get.docker.com). It installs Docker Engine + CLI, containerd.io, Buildx and Compose plugins, configures docker/containerd services to start at boot, and adds the SSH login user to the `docker` group (equivalent to `sudo gpasswd -a <user> docker`).

**Recommended inventory:** specify `ansible_user` so the correct account is added:

```ini
[servers]
myserver ansible_host=192.0.2.10 ansible_user=ubuntu
```

First-time installation uses the latest **stable** package versions available for that Ubuntu release. Re-running the playbook **does not upgrade already-installed Docker packages** unless a specific Engine/CLI version is supplied:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags docker
```

Install or configure Node Exporter and Docker together (without automatically upgrading existing Docker packages):

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags node_exporter,docker
```

To **pin or upgrade to a specific Docker Engine version**, first run `apt-cache madison docker-ce` on a target server after the official Docker repository has been configured, then pass the exact complete Debian package version (including epoch and Ubuntu codename):

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags docker \
  -e 'docker_apt_version=5:29.9.3-1~ubuntu.26.04~resolute'
```

The version above is **an example format**, not a guarantee that the package exists in the current repository. With a pinned version, only Docker Engine and Docker CLI are pinned; the containerd, Buildx and Compose packages are installed from the repo, but aren't proactively upgraded if already present. Leave `docker_apt_version` empty to keep already-installed versions unchanged on later runs. Note that `apt upgrade` run outside this playbook can still upgrade Docker unless you hold/pin its packages.

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
- Explicit Docker Engine upgrades can restart the daemon; schedule changes on production hosts appropriately.\n- Ubuntu releases newer than 26.04 must have a published Docker stable repository for their codename; otherwise APT will fail rather than falling back to an older Ubuntu release.
- Docker-published container ports can bypass parts of UFW/firewalld policy; review security groups and the `DOCKER-USER` firewall chain.

## Nginx source build (Ubuntu 26.04+)

Installs **nginx 1.30.7** from the official nginx.org source tarball into `/usr/local/nginx`. Builds support for HTTP TLS, HTTP/2, real IP, gzip_static, status and TCP/UDP stream proxying with TLS/SNI preread.

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags nginx
```

The role installs `build-essential`, `libpcre2-dev`, `libssl-dev`, and `zlib1g-dev`, compiles Nginx, creates a systemd service and enables it at boot. The default Nginx configuration lives in `/usr/local/nginx/conf/nginx.conf`. **Existing Nginx installations are not upgraded by default.** An initial installation creates the stock upstream configuration. Changing the binary does not automatically edit your existing configuration.

To explicitly upgrade to a published version, set both the target version and `nginx_upgrade=true`:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags nginx \
  -e nginx_version=1.30.7 -e nginx_upgrade=true
```

Use a *newer published version* for actual upgrades; the command above demonstrates the syntax. The role checks the installed binary version and only recompiles if different. During an upgrade it tests the new binary against the existing config before replacing the executable, keeps an Ansible backup of the previous binary, and restarts nginx via systemd to activate the new version. The upgrade deliberately does **not** run `make install`, so it does not overwrite existing `nginx.conf` or other installation configuration. Plan a maintenance window for any restart.

Additional compile flags can be supplied via `nginx_configure_extra_args`. For example, add `--with-http_auth_request_module` only if your configuration uses `auth_request`.

For strict source integrity checking, provide the independently verified source archive SHA256 with `nginx_source_sha256`. Otherwise the archive is downloaded over HTTPS without a pinned digest. The initial Nginx configuration may use port 80; ensure it does not conflict with an existing web server or Docker port mappings.

```bash
/usr/local/nginx/sbin/nginx -V
/usr/local/nginx/sbin/nginx -t
systemctl status nginx
```

To run selected roles together:

```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags node_exporter,docker,nginx
```
