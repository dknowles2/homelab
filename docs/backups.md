# Backups

Every guest has a 15-minute recovery point and a weekly full archive, and the
host configuration is archived daily. All of it lives on tank, inside the same
machine. Nothing is copied off-site.

| Guest | Weekly vzdump | sanoid policy (source) | syncoid replica on tank | Worst-case loss |
| --- | --- | --- | --- | --- |
| 100 unifi | yes | production | `tank/backup/vm-100-disk-0` | ~15 min |
| 101 elzar | yes | high_churn | `tank/backup/vm-101-disk-0` | ~15 min |
| 105 haos | yes | production | `tank/backup/vm-105-disk-0` | ~15 min |
| 106 docker | yes, root disk only | production | `tank/backup/subvol-106-disk-0` | ~15 min for root; Frigate media not backed up |
| 107 samba | yes, root disk only | production | `tank/backup/subvol-107-disk-0` | ~15 min; share data lives on tank |
| 108 metrics | yes | production | `tank/backup/subvol-108-disk-0` | ~15 min |

sanoid templates:

| Template | Hourly | Daily | Monthly | Takes snapshots |
| --- | --- | --- | --- | --- |
| production | 12 | 7 | 2 | yes |
| high_churn | 6 | 4 | 1 | yes |
| backup | 12 | 7 | 2 | no (prune only, for replicas) |
| backup_high_churn | 6 | 4 | 1 | no (prune only, for replicas) |

## How each layer works

- **vzdump:** job `backup-7989ed81-d166`, every Saturday 02:00, all guests,
  snapshot mode, zstd, keep 8 (eight weeks). Target is storage `tank-backups`
  at `/tank/pve-backups/dump`, guarded by `is_mountpoint /tank/pve-backups`.
  Only failures are emailed.
- **sanoid:** runs every 15 min from `/etc/cron.d/sanoid` with `TZ=UTC`, so
  snapshot names are in UTC.
- **syncoid:** six lines in `/etc/cron.d/syncoid`, every 15 min, with
  `--no-sync-snap`. It sends only when sanoid has made a new snapshot, so
  replicas trail the source by up to one hour.
- **Host config:** `pve-host-backup.timer`, daily at 03:00, writes
  `/tank/pve-backups/host/wernstrom-<date>.tar.zst` (~650 KB, keep 30). Each
  archive holds `/etc/pve`, a consistent copy of `config.db`, the relevant
  `/etc` subtrees, `/usr/local`, `/root`, and manifests: package list, pool
  layout, blkid and PVE versions.
- **Other data on tank:** `tank/data/backup/TimeMachine` (Mac backups through
  samba) and `tank/data/backup/duplicati`.

Backup freshness is exported by `backup-metrics` and alerted on (see
[Alerting](alerting.md)).

## Gaps

- No off-site copy. Losing the machine loses every backup.
- Frigate recordings (~11 TB) are not backed up; 30-day retention makes them
  disposable by design.
- tank's two disks are more than 7 years old.
