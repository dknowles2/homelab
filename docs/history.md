# History and lessons

## Timeline

Newest first.

| Date | Event |
| --- | --- |
| 2026-09-25 | Alerting built (vmalert + Alertmanager, 26 rules). Email delivery was never configured: the password file was left empty |
| 2026-09-25 | Removed leela's old WAN forwards (80, 443, 8555), a WAN SSH forward to 192.168.1.12, and leela's alias, DNS override, DHCP mapping and IoT rules. Fixed the NPM printer host (dead vouch auth). Reclaimed ~63 GB of ghost backups, ISOs and unused Frigate models |
| 2026-09-25 | Added the daily host config backup |
| 2026-09-20 | Extended sanoid/syncoid from 2 guests to all 6. Tightened vzdump's `is_mountpoint` to `/tank/pve-backups` |
| 2026-09-11 | Grafana exposed at grafana.knowles.xyz through the existing Cloudflare tunnel |
| 2026-09-09 | An OPNsense GUI save regenerated `telegraf.conf` and dropped `use_sudo`; pf graphs were silently empty for 2 days. Fixed durably with `telegraf.d/pf.conf` |
| 2026-09-06 | Metrics stack built: VictoriaMetrics, vmagent, node_exporter, cgroup collector, Grafana, OPNsense exporters, 8 dashboards |
| 2026-09 | Tried kernel 7.0.14-15. Coral inference went from ~8 ms to ~700 ms and stayed bad after a Frigate restart. Rolled back to 6.17.13-21 and pinned it. Root cause unknown; the PAT and PCIe messages seen on 7.0 also appear on healthy 6.17 |
| 2026-09-05 | Successful-backup emails suppressed; failures are still mailed |
| 2026-09-03 | Cleanups: docker prune, journal capped at 500 MB, old kernels and snapshots removed, frigate-tier-down hardened against files vanishing mid-run. The journal vacuum also destroyed the May 2026 crash logs |
| 2026-09-03 | leela decommissioned; the samba container took 192.168.1.3 |
| 2026-09-02 | tank rebuilt as a mirror (Toshiba attached to the HGST); the old zspin pool was evacuated |
| 2026-05 | Unexplained crashes on a newer kernel, possibly related to GPU passthrough to a VM (no longer used). Logs lost |

## Lessons

- **Measure before calling something a performance problem.** Swap used and
  load average are state, not cost. Check PSI (`/proc/pressure/*`) and
  swap-in/out rates first. High swap on CT 106 turned out to be cold pages
  with zero stalls.
- **Silent failures are the common failure.** pf metrics, the Coral regression
  and alert delivery all broke without anyone being told. Every new collector
  or job should come with an alert for its output going missing.
- **GUI-managed config gets regenerated.** On OPNsense, put custom settings in
  include directories such as `telegraf.d`, not in files the GUI writes.
- **sanoid snapshot names are UTC.** Read them as UTC, not local time, when
  judging freshness.
- **IPs get reused.** Rules written for one host (leela) quietly applied to the
  next host at that address (samba). Remove firewall rules when a host is
  retired.
