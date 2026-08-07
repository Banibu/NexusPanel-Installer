<div align="center">

<img src="https://raw.githubusercontent.com/Banibu/NexusPanel-Installer/main/homeserver/logo.png" alt="Nexus Panel" width="96" height="96">

# Nexus Panel

**Control panel for Linux servers, containerised applications and game servers.**

Official distribution repository — installers, native packages, appliance images and home-server integrations.

[![Release](https://img.shields.io/github/v/release/Banibu/NexusPanel-Installer?style=flat-square&logo=github&label=release)](https://github.com/Banibu/NexusPanel-Installer/releases/latest)
[![Debian](https://img.shields.io/badge/APT-Debian%20%7C%20Ubuntu-A81D33?style=flat-square&logo=debian&logoColor=white)](#debian-ubuntu-and-derivatives-apt)
[![Fedora](https://img.shields.io/badge/DNF-Fedora%20%7C%20RHEL-51A2DA?style=flat-square&logo=fedora&logoColor=white)](#fedora-rhel-and-derivatives-dnfyum)
[![Snap](https://img.shields.io/badge/Snap-nexuspanel-82BEA0?style=flat-square&logo=snapcraft&logoColor=white)](#snap)
[![Kubernetes](https://img.shields.io/badge/Helm-chart-0F1689?style=flat-square&logo=helm&logoColor=white)](#kubernetes-helm)
[![NexusOS](https://img.shields.io/badge/NexusOS-ISO%20%7C%20QCOW2%20%7C%20RAW-06B6D4?style=flat-square&logo=linux&logoColor=white)](#nexusos-appliance)

</div>

---

## Table of contents

- [What Nexus Panel is](#what-nexus-panel-is)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
  - [Standalone installer (recommended)](#standalone-installer-recommended)
  - [Debian, Ubuntu and derivatives (APT)](#debian-ubuntu-and-derivatives-apt)
  - [Fedora, RHEL and derivatives (DNF/YUM)](#fedora-rhel-and-derivatives-dnfyum)
  - [Snap](#snap)
  - [Kubernetes (Helm)](#kubernetes-helm)
  - [CasaOS](#casaos)
  - [Unraid](#unraid)
  - [NexusOS appliance](#nexusos-appliance)
- [First run](#first-run)
- [Adding machines (nodes)](#adding-machines-nodes)
- [Updating](#updating)
- [Command line reference](#command-line-reference)
- [Verifying downloads](#verifying-downloads)
- [Uninstalling](#uninstalling)
- [Support](#support)

---

## What Nexus Panel is

Nexus Panel is a self-hosted control plane for workloads that run on Linux. You point it at
one or more machines and it manages the whole lifecycle of what runs on them: creating
workloads, giving each one CPU, memory and disk limits, exposing ports, taking backups,
running scheduled tasks and handing controlled access to other people.

Everything runs in containers on the machines you own. The panel itself is a small stack
(API, worker, proxy, object storage) plus an agent installed on each managed machine. The
agent talks to the panel over gRPC with mutual TLS, so adding a second machine does not
require opening a database or SSH access between them.

It is commonly used for game servers, but nothing in the design is specific to games: an
application, a service or a database container is managed exactly the same way, from the
same templates and the same interface.

---

## Features

**Workload management**

| Area | What you get |
| --- | --- |
| Console | Live terminal attached to the running process, with command history and log streaming |
| Files | Browser with editor and syntax highlighting, upload, multi-select, compress and extract |
| Databases | Provisions a MySQL/MariaDB database and user per workload, with password rotation |
| Backups | Snapshots to S3-compatible storage (bundled MinIO or your own bucket), restore and download |
| Schedules | Cron-style tasks: restarts, backups, arbitrary commands |
| Networking | Port allocations, additional ports, subdomains and reverse-proxy rules |
| Add-ons | Search and install from Modrinth, SpigotMC, Hangar and Polymart |
| Metrics | CPU, memory, network and disk history with charts |
| Mounts | Attach shared host directories to a workload |

**Access and organisation**

| Area | What you get |
| --- | --- |
| Projects | Group workloads, invite members with roles, register webhooks |
| Sub-users | Per-workload permissions (viewer, editor, manager) |
| Accounts | Two-factor authentication (TOTP), API keys, active session management |
| Audit | Signed audit trail of administrative and workload actions |
| Quotas | Limits per project for workloads, memory, disk and databases |

**Administration**

| Area | What you get |
| --- | --- |
| Nodes | Register machines, watch live CPU/memory, set capacity and maintenance mode |
| Templates | Define images, startup commands and variables; imports Pterodactyl eggs |
| Database hosts | Register external MySQL/MariaDB servers, or provision one during installation |
| Appearance | Theme designer with full colour control, light and dark |
| Plugins | Sandboxed extensions (WebAssembly) with an event API |
| API | Documented REST API with an OpenAPI specification served by the panel |
| Languages | English and Brazilian Portuguese |

---

## Requirements

| | Minimum | Recommended |
| --- | --- | --- |
| CPU | 2 cores | 4 cores or more |
| Memory | 2 GB | 8 GB or more |
| Disk | 20 GB | 60 GB or more, SSD |
| OS | Any current Linux distribution with systemd | Ubuntu 24.04, Debian 12/13, Fedora |
| Architecture | x86_64 or ARM64 | — |

Docker is installed automatically if it is not present. The machine that hosts the panel can
also run workloads, but for production a dedicated panel host with separate nodes is the more
predictable arrangement. Installation requires root.

---

## Installation

Pick **one** method. They all end at the same place; they differ in how updates reach you.

### Standalone installer (recommended)

Works on every supported distribution and is the method the project tests most. The
bootstrap script detects your architecture, downloads the installer, verifies its SHA-256
checksum and starts the interactive wizard:

```bash
curl -fsSL https://gist.github.com/Banibu/10d6b9e6737a693e36b1cf6392bc7b59/raw | sudo bash
```

<details>
<summary>Download it manually instead</summary>

```bash
arch="$(uname -m)"
case "$arch" in
  x86_64|amd64)  artifact=Nexus-Installer-amd64 ;;
  aarch64|arm64) artifact=Nexus-Installer-arm64 ;;
  *) echo "Unsupported architecture: $arch" >&2; exit 1 ;;
esac
curl -fL "https://github.com/Banibu/NexusPanel-Installer/releases/latest/download/$artifact" -o /tmp/nexusctl
sudo install -m 0755 /tmp/nexusctl /usr/local/bin/nexusctl
sudo nexusctl install
```

</details>

Nothing on the host changes until you run `install` and answer the wizard.

### Debian, Ubuntu and derivatives (APT)

Adds a signed repository, so `apt upgrade` keeps the panel current along with the rest of the
system.

```bash
# 1. Trust the signing key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://raw.githubusercontent.com/Banibu/NexusPanel-Installer/main/apt/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/nexus.gpg

# 2. Add the repository
echo "deb [signed-by=/etc/apt/keyrings/nexus.gpg] https://raw.githubusercontent.com/Banibu/NexusPanel-Installer/main/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/nexus-panel.list

# 3. Install and configure
sudo apt update && sudo apt install nexus-panel
sudo nexusctl install
```

### Fedora, RHEL and derivatives (DNF/YUM)

```bash
sudo curl -fsSL https://raw.githubusercontent.com/Banibu/NexusPanel-Installer/main/rpm/nexus-panel.repo \
  -o /etc/yum.repos.d/nexus-panel.repo
sudo dnf install nexus-panel
sudo nexusctl install
```

openSUSE uses the same repository with `sudo zypper install nexus-panel`.

### Snap

> The Snap Store channel is **not published yet**. Snaps that use classic confinement require
> manual review by Canonical, and that review is still pending. Until it clears, install the
> signed `.snap` file from the release page:

```bash
# Download nexus-panel_<version>_amd64.snap from the release page, then:
sudo snap install ./nexus-panel_<version>_amd64.snap --classic --dangerous
sudo nexuspanel.nexusctl install
```

`--dangerous` is required only because the file is installed from disk rather than from the
store; its checksum is published in `SHA256SUMS` and can be verified beforehand.

### Kubernetes (Helm)

```bash
# Download nexus-panel-<version>.tgz from the release page, then:
helm install nexus-panel ./nexus-panel-<version>.tgz \
  --namespace nexus --create-namespace
```

Review `values.yaml` before installing: it controls ingress, persistence and the credentials
used by the bundled PostgreSQL, Redis and object storage.

### CasaOS

1. Copy the Compose manifest URL:
   ```
   https://raw.githubusercontent.com/Banibu/NexusPanel-Installer/main/homeserver/casaos-compose.yml
   ```
2. In CasaOS, choose **Custom Install → Import Compose** and provide the file.
3. Approve the Docker socket access the bootstrap container requests, then install.

### Unraid

1. Open the Unraid web interface and go to **Apps**.
2. Under **Template Repositories**, add:
   ```
   https://raw.githubusercontent.com/Banibu/NexusPanel-Installer/main/homeserver/unraid.xml
   ```
3. Search for **NexusPanel** and install it.

### NexusOS appliance

NexusOS is a prepared Linux image with the panel already installed, for people who would
rather not manage a base system. Three formats are published:

| Format | File | Use it for |
| --- | --- | --- |
| QCOW2 | `NexusOS-<version>-x86_64.qcow2` | Proxmox VE, KVM/QEMU, libvirt |
| RAW | `NexusOS-<version>-x86_64.raw.tar.gz` | Bare metal, cloud imports, `dd` to disk |
| ISO | `NexusOS-<version>-x86_64.iso.part-*` | Installing on a physical machine from USB |

Importing into Proxmox VE:

```bash
wget https://github.com/Banibu/NexusPanel-Installer/releases/latest/download/NexusOS-<version>-x86_64.qcow2
qm importdisk 100 NexusOS-<version>-x86_64.qcow2 local-lvm
```

The ISO is published in parts because GitHub rejects release assets larger than 2 GiB.
Reassemble it before writing to a USB stick:

```bash
cat NexusOS-<version>-x86_64.iso.part-* > NexusOS-<version>-x86_64.iso
sha256sum --ignore-missing -c SHA256SUMS
```

---

## First run

The installer asks for the address, storage and administrator account, then brings the stack
up. When it finishes, open the panel at the address you configured and sign in at `/login`.

Appliance and app-store installations skip the terminal wizard. They start an onboarding page
instead, where the first administrator is created in the browser:

```
http://<server-address>:8080/onboarding
```

Only one of the two paths runs: once an administrator exists, onboarding closes.

---

## Adding machines (nodes)

A node is any Linux machine that runs your workloads. The panel host can be a node itself, and
you can add more at any time.

1. In the panel, go to **Administration → Nodes → New node**.
2. Fill in the address and the capacity you want to make available.
3. The panel shows a one-line command. Run it as root on the target machine.

The command installs the agent and enrols it using a single-use token. On a fresh machine it
also installs Docker and Caddy, creates the service user and directories, registers the
systemd unit and configures the firewall. The agent connects back over gRPC with mutual TLS,
using a certificate issued by the panel's own authority and renewed automatically — there is
no public certificate to obtain for this connection.

The enrolment token is shown once and cannot be recovered: if you leave the page before
running the command, delete the node record and create a new one.

**Before creating the node**, point the hostname at the machine and open the agent port:

- an `A`/`AAAA` record for the FQDN resolving to the machine's address. The panel dials the
  agent at that exact name and issues the certificate for it, so the record must exist first —
  node DNS is not created for you;
- if the record is behind Cloudflare, keep it **DNS only** (grey cloud). The orange proxy
  terminates HTTP/HTTPS and would break the gRPC connection;
- the agent port (9091 by default) reachable from the panel.

An IP address works instead of a hostname, but the certificate is then bound to that address
and changing it means re-registering the node.

---

## Updating

Update through the same channel you installed from — mixing them leaves the package manager's
file list inconsistent.

| Installed with | Update command |
| --- | --- |
| Standalone installer | `sudo nexusctl upgrade` |
| APT | `sudo apt update && sudo apt install --only-upgrade nexus-panel` |
| DNF / YUM | `sudo dnf upgrade nexus-panel` |
| zypper | `sudo zypper update nexus-panel` |
| Snap | `sudo snap refresh nexuspanel` |
| Helm | `helm upgrade nexus-panel ./nexus-panel-<version>.tgz` |
| CasaOS / Unraid | Update from the app store |
| NexusOS | `sudo nexusctl upgrade` inside the appliance |

`nexusctl upgrade` detects package-managed installations and refuses to replace files owned by
`dpkg`, `rpm` or `snapd`, pointing you at the correct command instead.

---

## Command line reference

`nexusctl` manages the installation from the shell.

| Command | Purpose |
| --- | --- |
| `nexusctl install` | Interactive installation wizard |
| `nexusctl status` | Service state, containers and health checks |
| `nexusctl start` / `stop` / `restart` | Control the panel stack |
| `nexusctl logs` | Stream aggregated logs |
| `nexusctl upgrade` | Update to the latest release |
| `nexusctl doctor` | Diagnose a broken installation |
| `nexusctl repair` | Rebuild the stack from the stored configuration |
| `nexusctl agent` | Install and enrol the node agent |
| `nexusctl node` / `server` / `project` | Manage resources without the web interface |
| `nexusctl config` | Read and change panel settings |
| `nexusctl audit verify` | Check the integrity of the audit trail |
| `nexusctl plugin` | Manage installed plugins |
| `nexusctl uninstall` | Remove the panel |
| `nexusctl version` | Version and build commit |

Run `nexusctl <command> --help` for the options of any command.

---

## Verifying downloads

Every release ships a `SHA256SUMS` manifest and a `RELEASE-PROVENANCE` file recording the
audited commit the artefacts were built from. The APT and RPM repository metadata is GPG
signed.

```bash
curl -fsSLO https://github.com/Banibu/NexusPanel-Installer/releases/latest/download/SHA256SUMS
sha256sum --ignore-missing -c SHA256SUMS
```

Container images are published to the GitHub Container Registry
(`ghcr.io/banibu/nexus-api`, `nexus-worker`, `nexus-caddy`, `nexus-minio`) and are pulled
automatically during installation.

---

## Uninstalling

```bash
sudo nexusctl uninstall
```

The command stops the stack and removes the panel's services and containers. It asks before
touching your data; keep a backup if you intend to reinstall later.

---

## Support

- **Issues and bug reports:** [github.com/Banibu/NexusPanel-Installer/issues](https://github.com/Banibu/NexusPanel-Installer/issues)
- **Security reports:** send them privately to the address in the release metadata rather than
  opening a public issue.

<div align="center">
<sub>Built by Banibu.</sub>
</div>
