# Monitoring

CT 108 (192.168.1.240) holds about 11,100 time series with 24-month retention
in VictoriaMetrics, with Grafana dashboards on top. Both were installed from the
community-scripts VictoriaMetrics and Grafana scripts.

## Components (CT 108)

| Service | Port | Role |
| --- | --- | --- |
| victoriametrics | 8428 | Time-series database. `--retentionPeriod=24` (months), self-scrape every 30 s. Accepts InfluxDB line protocol from PVE |
| vmagent | 8429 | Scrapes the targets below; config `/etc/vmagent/scrape.yml` |
| vmalert | 8880 | Evaluates alert rules (see [Alerting](alerting.md)) |
| prometheus-alertmanager | 9093 | Routes alerts to email |
| grafana-server | 3000 | Dashboards, public at grafana.knowles.xyz. Datasources are file-provisioned: VictoriaMetrics (uid `victoriametrics`) and Alertmanager |

VictoriaMetrics binaries live in `/opt/victoriametrics`. Grafana's root URL is
set in a systemd drop-in (`GF_SERVER_ROOT_URL=https://grafana.knowles.xyz/`).

## What is collected

| Scrape job | Target | Source |
| --- | --- | --- |
| node | wernstrom 192.168.1.9:9100 | node_exporter, plus textfile collectors: smartmon, nvme, apt, `pve_cgroup.prom`, `backup.prom` |
| frigate | 192.168.1.4:5000/api/metrics | Frigate's built-in metrics: camera fps, Coral inference speed, storage |
| opnsense | 192.168.1.1:9100 | os-node_exporter plugin |
| opnsense-telegraf | 192.168.1.1:9273 | os-telegraf plugin, including pf firewall stats |
| vmagent, vmalert, alertmanager | localhost | Self-monitoring |
| (push) | PVE metric server `victoriametrics` | `/etc/pve/status.cfg`, InfluxDB protocol: per-guest CPU, memory, disk, network |

## Dashboards

Grafana folder **Homelab**, provisioned from `/var/lib/grafana/dashboards`:

| Dashboard | UID |
| --- | --- |
| Proxmox host | `pve-host` |
| Docker containers | `docker-containers` |
| Frigate | `frigate` |
| Guests (pick a VM with the `vmid` variable) | `guest-detail` |
| OPNsense | `opnsense-overview` |
| Stack health | `stack-health` |
| Storage health | `storage-health` |
| Node Exporter Full (community 1860) | `rYdddlPWk` |

Every custom panel is full width with a 300 px right-hand legend, so
timelines line up across panels.

## OPNsense pf metrics need sudo

The telegraf user gets degraded `pfctl` output without root. The fix:

- `/usr/local/etc/sudoers.d/telegraf-pf` lets telegraf run only
  `/sbin/pfctl -s info` as root.
- `/usr/local/etc/telegraf.d/pf.conf` defines `[[inputs.pf]]` with
  `use_sudo = true`. Files in `telegraf.d` survive GUI saves.
- The GUI's own pf input should be unticked (Services → Telegraf → Input). If
  it is ticked, a GUI save regenerates `telegraf.conf` without `use_sudo`, and
  that copy errors every interval.
