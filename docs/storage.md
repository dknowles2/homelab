# Storage

Two ZFS pools hold guests and backups. Frigate recordings sit on two separate
ext4 disks joined by mergerfs. Both pools scrubbed clean on 2026-11-08, and
every disk showed zero reallocated or pending sectors on 2026-11-21.

## Pools and filesystems

| Name | Type | Devices | Size | Used | Holds |
| --- | --- | --- | --- | --- | --- |
| nvmepool | ZFS, single disk | Intel 660p 512 GB (nvme2n1) | 476 GB | 35% | All guest root disks, sanoid snapshots |
| tank | ZFS mirror | HGST HUH721010 10 TB (sdc) + Toshiba MG06 10 TB (sda) | 9.1 TB | 15% | vzdump archives, syncoid replicas, Time Machine, Duplicati, scans |
| /mnt/frigate-ssd | ext4 | WD Black SN850X 1 TB (nvme0n1) | 916 GB | 37% | Frigate media, newest 24 h |
| /mnt/frigate-hdd | ext4 | WD 16 TB (sdb) | 15 TB | 75% | Frigate media older than 24 h |
| / | ext4 on LVM | 58 GB module (nvme1n1) | 50 GB | 24% | Proxmox OS |

The ARC is capped at 2.92 GiB (`zfs_arc_max=3129999360`) with a 97.6% hit
ratio. The cap stays because the heaviest IO, Frigate, is on ext4 and never
touches the ARC.

## Disk health and age (2026-11-21)

| Disk | Power-on | Wear or bad sectors | Note |
| --- | --- | --- | --- |
| sdc HGST 10 TB | 66,854 h (7.6 yr) | 0 | Mirror half |
| sda Toshiba 10 TB | 64,677 h (7.4 yr) | 0 | Mirror half. 23,641 ATA errors in its log, all from one burst on 2026-08-27 before it was installed; not growing |
| sdb WD 16 TB | 29,248 h (3.3 yr) | 0 | Write-only Frigate tier |
| nvme2 Intel 660p | — | 44% of rated endurance used, 117 TB written | Heaviest wear; was 28% in September |
| nvme0 SN850X | — | 1%, 33 TB written | |
| nvme1 boot module | — | 0% | |

## Frigate tiering

- `/mnt/frigate` is a mergerfs union of `/mnt/frigate-ssd/media` and
  `/mnt/frigate-hdd/media` with `category.create=ff`, `minfreespace=100G` and
  `moveonenospc=true`. New files land on the SSD.
- `frigate-tier-down.timer` runs hourly. It moves recordings, clips and exports
  older than 24 h (`TIER_AGE_MINUTES=1440`) to the HDD with
  `rsync --remove-source-files`. Paths inside the union never change, so
  Frigate's database is unaffected.
- Results are published to MQTT for Home Assistant. A failed run triggers
  `frigate-tier-down-failed.service`.
- Only the `media` subdirectory of each disk is in the union, so host-only data
  beside it stays invisible to CT 106.
- Frigate's own 30-day retention decides how full the HDD gets; tiering only
  decides where files sit.
