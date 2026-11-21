# Open items

As of 2026-11-21.

- [ ] Put the Gmail app password in CT 108 so alerts are delivered (see
      [Alerting](alerting.md))
- [ ] Investigate GuestSwapFail on VM 101 elzar, firing since 2026-11-21.
      Host swap is at 4.6 of 7.1 GiB
- [ ] Set up an external dead-man's switch for the Watchdog alert (for example
      healthchecks.io), so a dead CT 108 or host still pages
- [ ] Copy backups off-site: the host config archive first (~650 KB/day), then
      the vzdump archives
- [ ] Restore admin SSH access to calculon (the key was rejected on
      2026-11-21), then reverify [Network](network.md)
- [ ] Untick the pf input in OPNsense Services → Telegraf → Input
- [ ] Remove `two_node: 1` from corosync the next time farnsworth is on, and
      bump `config_version` to 7
- [ ] Document which VLAN carries which subnet
- [ ] Plan a replacement for nvmepool's Intel 660p: 44% of rated endurance
      used, up from 28% in September
- [ ] Plan replacements for the 7+ year old tank disks
- [ ] Optional: scrape Home Assistant into VictoriaMetrics
