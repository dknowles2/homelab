# Proxmox host: wernstrom

wernstrom runs PVE 9.2.32 on kernel **6.17.13-21-pve, pinned**, because 7.0
kernels break the Coral TPU (see [History](history.md)). As of 2026-11-21 it had
been up 10 weeks with no failed units and no pending updates.

## Hardware

- AMD Ryzen 7 255 (16 threads), Radeon 780M iGPU (Phoenix3), 29 GiB RAM,
  7.1 GiB swap
- Google Coral Edge TPU on PCIe (01:00.0), driver `apex` from `gasket-dkms`
- Boot: LVM volume group `pve` on a 58 GB BAYHUB/Samsung module (nvme1n1),
  booted by plain GRUB. It is a single device with no redundancy, and
  `proxmox-boot-tool` does not manage it.
- Data disks: see [Storage](storage.md).

## Kernel and Coral driver

- Pin: `proxmox-boot-tool kernel pin 6.17.13-21-pve`. `apt dist-upgrade` can
  install newer kernels; the pin keeps booting this one.
- `gasket-dkms` rebuilds automatically for every installed kernel through
  `/etc/kernel/postinst.d/dkms` and the matching headers hook, as long as the
  headers meta-packages are installed. `proxmox-headers-6.17` and
  `proxmox-headers-7.0` both are.
- Source for rebuilding the driver by hand is kept in `/root/gasket-driver`,
  `/root/gasket-dkms` and the built `.deb` beside them.
- Healthy Coral inference is about 8–14 ms.

## Cluster

Cluster `homelab`:

| Node | Quorum votes |
| --- | --- |
| wernstrom | 2 |
| farnsworth | 1 |

`corosync.conf` also sets `two_node: 1` and `wait_for_all: 0`
(config_version 6). wernstrom alone is quorate, so farnsworth can stay off.
`two_node: 1` is a split-brain risk when farnsworth is on; removing it is an
[open item](open-items.md).

## Host customizations

| Item | What it does |
| --- | --- |
| `/usr/local/sbin/frigate-tier-down` + `.timer` (hourly) | Moves Frigate media older than 24 h from SSD to HDD (see [Storage](storage.md)) |
| `/usr/local/sbin/pve-cgroup-metrics` + `.timer` (30 s) | Per-guest and per-container cgroup memory, CPU, PSI and swap metrics for node_exporter |
| `/usr/local/sbin/backup-metrics` + `.timer` (5 min) | Exports newest snapshot, replica, vzdump and host-backup timestamps |
| `/usr/local/sbin/pve-host-backup` + `.timer` (daily 03:00) | Host configuration archive (see [Backups](backups.md)) |
| `/etc/cron.d/syncoid` | Six replication jobs every 15 min |
| `/etc/sanoid/sanoid.conf` | Snapshot policy for six datasets and their replicas |
| `/etc/systemd/journald.conf.d/size.conf` | Caps the journal at 500 MB |
| `/etc/default/prometheus-node-exporter` | Removes Debian's `/mnt` exclusion so the Frigate disks are monitored |
| `/etc/pve/notifications.cfg` | Emails all PVE notifications via Gmail except successful backups |
| `/etc/pve/status.cfg` | Sends PVE metrics to VictoriaMetrics over the InfluxDB protocol |
| `scrub-watch.sh`, `resilver-watch.sh`, `zspin-*.sh`, `backup-copy.sh`, `blinkpull.sh` | Older helper scripts in `/usr/local/sbin`; purpose not reviewed |
