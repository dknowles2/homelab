# Runbooks

## Update packages

1. On wernstrom: `apt update && apt dist-upgrade`. New kernels install, but the
   pin keeps 6.17.13-21 booting.
2. Confirm DKMS built gasket for every kernel: `dkms status gasket`.
3. Reboot when convenient. Afterwards check `uname -r`, that `/dev/apex_0`
   exists, and Coral inference in Frigate (8–14 ms is healthy).

## Try a new kernel without risk

1. Confirm `dkms status gasket` lists the new kernel as installed.
2. `proxmox-boot-tool kernel pin <version> --next-boot` boots it once; the next
   reboot returns to the pinned kernel.
3. Watch Coral inference for at least a day. The 7.0 regression showed only
   under load and survived a Frigate restart.
4. If it is good: `proxmox-boot-tool kernel pin <version>`. If not: reboot.

## Restore a guest

- **To a point within the last day or week:** with the guest stopped, roll back
  its sanoid snapshot (`zfs rollback`), or clone the snapshot to inspect files.
  The replicas on tank hold the same snapshots.
- **To a weekly point, or if nvmepool is lost:** restore the vzdump archive
  from storage `tank-backups` in the PVE UI.
- **If nvmepool is lost but tank survives:** `zfs send` the replica from
  `tank/backup/<disk>` back to a new pool. It is up to 1 h behind.

## Rebuild the host (boot module dies)

1. Install PVE from the ISO on a new boot device.
2. `zpool import nvmepool` and `zpool import tank`.
3. Extract the newest `/tank/pve-backups/host/wernstrom-*.tar.zst` over `/`.
   Review `manifests/` first: packages, pool layout, boot info. If pmxcfs comes
   up empty, restore `/etc/pve` from `manifests/config.db`.
4. Reinstall packages from `manifests/dpkg-selections.txt`, including
   `gasket-dkms` and the kernel headers, then pin the kernel.
5. Reboot. Guests start from their existing disks.

## Add a guest to the 15-minute backups

1. In `/etc/sanoid/sanoid.conf`, add `[nvmepool/<disk>]` with
   `use_template = production` and `[tank/backup/<disk>]` with
   `use_template = backup`. Use `high_churn` and `backup_high_churn` for busy
   disks.
2. Add a matching line to `/etc/cron.d/syncoid`:
   `*/15 * * * * root /usr/sbin/syncoid nvmepool/<disk> tank/backup/<disk> --no-sync-snap --quiet`
3. Run `TZ=UTC sanoid --take-snapshots`, then one manual `syncoid` for the
   initial full send.
4. `backup-metrics` picks up the new datasets automatically, so the backup
   alerts cover them.

## Run commands on OPNsense

The root shell is csh, which mangles quoting. Send scripts through `sh`:

```bash
ssh root@192.168.1.1 /bin/sh -s <<'EOF2'
pfctl -sn
EOF2
```

Make firewall changes through the GUI or the MVC models
(`/usr/local/opnsense/mvc/app/models/OPNsense/Firewall/`), never by editing
`/conf/config.xml` directly. Back up `/conf/config.xml` first; System →
Configuration → History also keeps prior versions.
