# setup-prometheus

An Ansible role that deploys [Prometheus](https://prometheus.io/) as a Docker container using Docker Compose. It integrates with [NetBox](https://netbox.dev/) as an HTTP-based service discovery (SD) backend, automatically pulling scrape targets from NetBox's Prometheus SD plugin.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Repository Structure](#repository-structure)
- [Role Variables](#role-variables)
- [How It Works](#how-it-works)
- [Usage](#usage)
- [Scrape Jobs](#scrape-jobs)
- [Customization](#customization)

---

## Overview

This role:

1. Installs Docker (if not already present) on Debian/Ubuntu hosts.
2. Creates the Prometheus working directory.
3. Renders a `docker-compose.yml` and a `prometheus.yml` configuration from Jinja2 templates.
4. Starts the Prometheus stack using `docker compose`.
5. Restarts the stack automatically whenever the configuration changes.

Targets are discovered dynamically from NetBox via its [Prometheus SD plugin](https://github.com/FlxPeters/netbox-plugin-prometheus-sd), so there is no need to hardcode host IPs.

---

## Requirements

| Requirement | Notes |
|---|---|
| Ansible | `>= 2.14` |
| Target OS | Debian / Ubuntu |
| Docker | Installed automatically if missing |
| Ansible collection | `community.docker` |
| NetBox | Must have the Prometheus SD plugin installed and a valid API token |

Install the required Ansible collection:

```bash
ansible-galaxy collection install community.docker
```

---

## Repository Structure

```
setup-prometheus/
├── prometheus.yml                  # Ansible playbook entry point
└── prometheus/                     # Ansible role
    ├── defaults/
    │   └── main.yaml               # Default variables
    ├── handlers/
    │   └── main.yaml               # Handler: restart Prometheus stack
    ├── meta/
    │   └── main.yaml               # Role metadata
    ├── tasks/
    │   └── main.yaml               # Main task list
    └── templates/
        ├── docker-compose.yml.j2   # Docker Compose template
        └── prometheus.yml.j2       # Prometheus configuration template
```

---

## Role Variables

All variables are defined in `prometheus/defaults/main.yaml` and can be overridden in your inventory or playbook.

| Variable | Default | Description |
|---|---|---|
| `prometheus_dir` | `/opt/prometheus` | Directory where config files and volumes are stored |
| `prometheus_image` | `prom/prometheus:latest` | Docker image to use |
| `prometheus_port` | `9090` | Host port to expose Prometheus on |
| `prometheus_retention_time` | `30d` | TSDB data retention period |
| `prometheus_retention_size` | `10GB` | TSDB data retention size limit |
| `scrape_interval` | `15s` | Global Prometheus scrape interval |
| `netbox_base_url` | `http://172.17.0.1:8000` | Base URL of your NetBox instance |
| `netbox_token` | `CHANGE_ME` | NetBox API token for authentication |
| `jobs` | *(see below)* | List of scrape job definitions |

> ⚠️ **Important:** Always override `netbox_token` with a real token. Never commit secrets to version control.

---

## How It Works

### Service Discovery via NetBox

Each scrape job is configured with `http_sd_configs`, pointing to a NetBox API endpoint that returns a list of scrape targets in Prometheus HTTP SD format.

Relabeling rules extract the following metadata from NetBox:

| Prometheus Label | NetBox Source |
|---|---|
| `__address__` | Primary IPv4 + service port |
| `instance` | NetBox device/VM name |
| `site` | NetBox site slug |
| `services` | NetBox services field |

### Docker Compose

Prometheus runs as a single container with:
- A bind-mounted `prometheus.yml` configuration file.
- A named Docker volume (`prometheus_data`) for persistent TSDB storage.
- WAL compression and configurable retention enabled via CLI flags.
- The `--web.enable-lifecycle` flag, allowing hot-reload via `POST /-/reload`.

---

## Usage

### 1. Configure Your Inventory

Create an inventory file (e.g. `inventory.ini`):

```ini
[monitoring]
your-server-ip ansible_user=ubuntu
```

### 2. Override Variables

Override sensitive or environment-specific variables in `group_vars/monitoring.yaml`:

```yaml
netbox_base_url: "https://netbox.example.com"
netbox_token: "your-real-netbox-api-token"
prometheus_port: 9090
```

### 3. Run the Playbook

```bash
ansible-playbook -i inventory.ini prometheus.yml
```

To do a dry-run first:

```bash
ansible-playbook -i inventory.ini prometheus.yml --check
```

---

## Scrape Jobs

The default `jobs` list defines the following scrape jobs, all discovered from NetBox:

| Job Name | NetBox Path | Description |
|---|---|---|
| `nodes-exporter` | `/api/plugins/prometheus-sd/virtual-machines/` | All VMs in NetBox |
| `node-exporter` | `/api/plugins/prometheus-sd/services/?name=node-exporter` | Node Exporter services |
| `redis-exporter` | `/api/plugins/prometheus-sd/services/?name=redis-exporter` | Redis Exporter services |
| `nginx-exporter` | `/api/plugins/prometheus-sd/services/?name=nginx-exporter` | Nginx Exporter services |
| `postgres-exporter` | `/api/plugins/prometheus-sd/services/?name=postgres-exporter` | PostgreSQL Exporter services |

To add a new job, append an entry to the `jobs` list in your vars:

```yaml
jobs:
  - name: my-custom-exporter
    path: /api/plugins/prometheus-sd/services/?name=my-custom-exporter
    refresh_interval: 300s
```

---

## Customization

- **Retention:** Adjust `prometheus_retention_time` and `prometheus_retention_size` to fit your storage capacity.
- **Scrape interval:** Change `scrape_interval` globally; per-job overrides can be added directly in `prometheus.yml.j2`.
- **Port:** Change `prometheus_port` if port `9090` is already in use.
- **Image pinning:** Replace `prom/prometheus:latest` with a specific version tag (e.g. `prom/prometheus:v2.53.0`) for reproducible deployments.
