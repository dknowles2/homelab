# Guests

Six guests run on wernstrom, all on bridge `vmbr0` and all set to start at
boot. Every root disk lives on `nvmepool`.

| ID | Name | Type | Purpose | IP | vCPU | RAM | Root disk |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 100 | unifi | VM | UniFi network controller | 192.168.9.2 (VLAN 99) | 2 | 2 GiB (balloon 1 GiB) | 32 GB |
| 101 | elzar | VM | General-purpose Linux VM; heaviest disk writer (~67 GB/day) | 192.168.1.24 | 4 | 6 GiB | 60 GB |
| 105 | haos | VM | Home Assistant OS | 192.168.1.11 | 2 | 4 GiB | 50 GB |
| 106 | docker | LXC, unprivileged | Docker host for Frigate and other containers | 192.168.1.4 (DHCP) | 10 | 8 GiB, no swap | 36 GB |
| 107 | samba | LXC, unprivileged | SMB shares: backups, Time Machine, Duplicati, scans | 192.168.1.3 (static) | 2 | 1 GiB | 8 GB |
| 108 | metrics | LXC, unprivileged | VictoriaMetrics, vmagent, vmalert, Alertmanager, Grafana | 192.168.1.240 (DHCP) | 2 | 2 GiB | 32 GB |

## CT 106 docker

- Devices passed in: `/dev/apex_0` (Coral), `/dev/dri/renderD128` and
  `/dev/dri/card0` (iGPU for video decode), `/dev/kfd`.
- Bind mounts: `/mnt/frigate` (the mergerfs union) at both `/mnt/data` and
  `/frigate/data`; `/tank/data/scans` at `/scans`.
- Docker containers:

  | Container | Purpose |
  | --- | --- |
  | frigate (0.18.2) | NVR with Coral object detection |
  | nginx-proxy-manager | Internal reverse proxy and certificates (ports 80, 81, 443) |
  | portainer (EE) + portainer_agent | Docker management UI |
  | cloudflare-tunnel | Cloudflare tunnel connector |
  | watchtower | Automatic image updates |
  | rtl-433 | SDR receiver for 433 MHz sensors |
  | nut-upsd | UPS monitor, port 3493 |
  | gdrive_sync | Google Drive sync |

### Frigate

- Config: `/frigate/config/config.yml`. It holds an API key; treat the file as
  a secret.
- Detector: Coral via `edgetpu`, model from Frigate+.
- Retention: 30 days for continuous, motion, alert and detection recordings.
- Semantic search (small) and face recognition (large) are on. LPR, audio
  transcription and bird classification are off.
- Cameras at 5 fps: front-porch, backyard, back_yard_right, driveway,
  basement, garage. doorbell is disabled; haunted_house_a and haunted_house_b
  report 0 fps.

## CT 107 samba

Bind mounts from tank:

| Host path | Container path |
| --- | --- |
| `/tank/data/backup` | `/srv/backup` |
| `/tank/data/backup/TimeMachine` | `/srv/backup/TimeMachine` |
| `/tank/data/backup/duplicati` | `/srv/backup/duplicati` |
| `/tank/data/scans` | `/srv/scans` |

It took 192.168.1.3 from the decommissioned host leela on 2026-09-03.

## VMs

- **elzar** and **haos** run qemu-guest-agent and are also Tailscale nodes.
- **haos** has the USB device on host port `1-1` passed through.
- **unifi** uses public resolvers (`1.1.1.1`, `8.8.8.8`) rather than the
  router.
