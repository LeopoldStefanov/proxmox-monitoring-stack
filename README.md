# Proxmox Monitoring Stack

A production-ready monitoring stack for Proxmox VE clusters using Prometheus, Grafana, and a suite of exporters. Covers hypervisor health, VM states, disk SMART data, MegaRAID controllers, network probes, and firewall metrics — all routing to alerting channels via Grafana.

---

## Stack Overview

```
Proxmox nodes (bare metal)
  ├── node_exporter        :9100  — OS & hardware metrics
  └── smartctl_exporter    :9633  — disk SMART health

Proxmox API ──────────────────── proxmox_exporter  :9221
FortiGate SNMP ────────────────── snmp_exporter
Blackbox targets ──────────────── blackbox_exporter :9115
MegaRAID controller (prox-sbnd) ─ textfile collector (cron)
                                          │
                                   Prometheus :9090
                                          │
                                    Grafana :3001
                                          │
                              ┌───────────┴───────────┐
                         it-ops-critical        it-ops-warnings
                         (Google Chat)          (Google Chat)
```

All monitoring services run as Docker containers on a dedicated monitoring VM. Exporters run as systemd services directly on each Proxmox node for direct hardware access.

---

## What Gets Monitored

### Proxmox Nodes
- CPU usage per node (alert >85%)
- RAM exhaustion (alert >92%)
- Node online/offline status
- Root disk space (alert <10% free)
- Disk IO utilisation (alert >20%)
- Network interface drops
- OOM kill events

### Virtual Machines & Containers
- VM/LXC up/down state (crash detection)
- Per-VM CPU usage (alert >90%)
- Per-VM memory usage (alert >90%)

### Disk Health
- SMART overall health pass/fail
- Reallocated sectors (early failure warning)
- SSD wearout indicator (alert >90% life used)
- Disk temperature (alert >55°C for SSDs)
- MegaRAID controller drives (see dedicated repo below)

### Network & Firewall (FortiGate)
- ICMP reachability probe
- TCP/443 probe
- DNS resolution probe (8.8.8.8)
- Interface traffic throughput and errors via SNMP

---

## Components

| Container | Port | Purpose |
|-----------|------|---------|
| grafana | 3001 | Dashboards & alerting |
| prometheus | 9090 | Metrics storage & query engine |
| proxmox_exporter | 9221 | Proxmox API metrics |
| snmp_exporter | — | FortiGate SNMP metrics |
| blackbox_exporter | 9115 | ICMP/TCP/DNS probes |

| Service (per node) | Port | Purpose |
|-------------------|------|---------|
| node_exporter | 9100 | OS & hardware metrics |
| smartctl_exporter | 9633 | Disk SMART data |

---

## Alert Rules

All rules are provisioned via YAML (`proxmox_alerts.yaml`) and loaded by Grafana on startup. No manual UI configuration required.

Rules use `noDataState: OK` to prevent false alerts during temporary scrape failures.

See [`proxmox_alerts.yaml`](proxmox_alerts.yaml) for the full set of rules.

### Routing

```
severity: critical  →  immediate action channel
severity: warning   →  business hours channel
```

Notification policy:
- `group_wait`: 2 minutes
- `group_interval`: 15 minutes
- `repeat_interval`: 6 hours

---

## Dashboards

Community dashboards used (import by ID in Grafana):

| Dashboard | ID | What it shows |
|-----------|----|--------------|
| Node Exporter Full | 1860 | CPU, RAM, disk IO, network per node |
| Proxmox Cluster | 10347 | VM states, storage pools, cluster health |
| smartctl_exporter | 20204 | Disk SMART health and temperatures |

---

## Directory Structure

```
monitoring-vm (Docker host)
├── docker-compose.yml
├── prometheus_config/
│   └── prometheus.yml              # scrape jobs
├── grafana_data/
│   ├── grafana.db                  # dashboards, users, contact points
│   └── provisioning/
│       └── alerting/
│           ├── proxmox_alerts.yaml # all alert rules
│           └── notifications.yaml  # routing & contact points
└── scripts/
    └── megaraid_textfile_collector.sh

proxmox-nodes (systemd services)
├── /usr/local/bin/node_exporter
├── /usr/local/bin/smartctl_exporter
├── /etc/systemd/system/node_exporter.service
├── /etc/systemd/system/smartctl_exporter.service
└── /etc/iptables/rules.v4          # firewall rules
```

---

## Adding a New Node

1. Install `node_exporter` and `smartctl_exporter` binaries on the new node
2. Deploy the systemd service files (same as existing nodes)
3. Add iptables rules to allow scraping from the monitoring VM
4. Add the node's IP to the Prometheus scrape jobs
5. Update FortiGate policy to allow TCP 9100/9633 from the monitoring VM

---

## Related

- [megaraid-prometheus-exporter](https://github.com/YOUR_USERNAME/megaraid-prometheus-exporter) — textfile collector for drives behind a MegaRAID controller

---

## Author

Leopold Stefanov
