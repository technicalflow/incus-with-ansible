# Incus Container Provisioning with Ansible

This project automates the creation, network configuration, and service deployment of Incus containers using Ansible and a simple CSV-based inventory.

---

## Overview

The setup allows you to declaratively define containers in `incus.csv`. When you run the playbook:

1. **Discovery & Creation**: Reads `incus.csv`, verifies which containers already exist on the host, creates missing containers using Incus (`macvlan` profile), mounts host storage paths, and starts them.
2. **Dynamic Inventory**: Populates Ansible's in-memory inventory using the `community.general.incus` connection driver and assigns role tags.
3. **Base Configuration (`main` role)**: Configures static IP networking (Netplan), creates the administrative user, installs SSH authorized keys, sets timezone/locale, and upgrades packages.
4. **Role Provisioning**: Executes specific service roles on designated containers based on the `ROLE` column in `incus.csv`.

---


## Configuration

### 1. Container Definitions (`incus.csv`)

Add or modify rows in `incus.csv` to specify your containers:

| Column | Description | Example |
|---|---|---|
| `CONTAINERNAME` | Unique name of the Incus container | `msllaub24code01` |
| `SYSTEM` | Incus image to use | `ubuntu/noble/cloud` |
| `IP` | Static IP address to assign | `192.168.50.220` |
| `ROLE` | Role to apply (matches `role_<name>`) | `vscode`, `postgres`, `jellyfin` |
| `MEDIA_DEVICES_NAME` | Device names (pipe-separated `\|`) | `pgsql\|pgsql_etc` |
| `MEDIA_DEVICES_SOURCE`| Host source directory paths | `/data/Backup/pgsql` |
| `MEDIA_DEVICES_PATH`  | Target path inside the container | `/var/lib/postgresql` |
| `MEDIA_DEVICES_READONLY` | Read-only flag (`true` or `false`) | `false\|false` |

### 2. Global Variables (`group_vars/all.yml`)

Adjust default values such as:
- `admin_user`: Default container admin account name (e.g. `madmin`).
- `admin_password_hash`: Password hash for the admin account (for password: mandarynka).
- `gateway`: Default network gateway for the containers.
- `host_admin_user`: Incus host admin user required only to provision Incus with system role.

---

## Available Roles

- **`main`**: Applies to all containers. Performs `apt upgrade`, sets locale (`pl_PL.UTF-8`) and timezone (`Europe/Warsaw`), provisions admin user with passwordless sudo and SSH key, and configures static IP with Netplan.
- **`vscode`**: Deploys Code-Server on port 80, along with developer tooling (Azure CLI, PowerShell + Az module, Terraform, Ansible-core, Docker CLI).
- **`postgres`**: Installs PostgreSQL and automated `pgAdmin 4` web interface.
- **`mariadb`**: Installs MariaDB server, sets root password, creates initial database and user.
- **`mongodb`**: Configures MongoDB APT repository and starts the `mongod` service.
- **`jellyfin`**: Installs and enables the Jellyfin media server and prepares `/data`.
- **`syslog`**: Deploys `syslog-ng` listening on UDP 514 with UFW rules and Mikrotik syslog parsers.
- **`system`**: Host-level preparation role (installs Incus packages and creates `macvlan` profile).

---

## Prerequisites

- **Incus**: Incus installed and running on the host.
- **Network Profile**: An Incus profile named `macvlan` (can be created using `roles/system`).
- **Ansible Collections**:
  ```bash
  ansible-galaxy collection install community.general ansible.posix
  ```
- **SSH Key**: Controller public key at `/home/madmin/.ssh/ansible.pub` (copied to containers during setup).

---

## Usage

Run the playbook from this directory:

```bash
ansible-playbook playbook.yml
```

> **Note**: Existing containers are automatically detected and skipped during the creation stage, making subsequent runs safe when adding new entries to `incus.csv`.
