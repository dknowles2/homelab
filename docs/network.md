# Network

No service is reachable from the internet by port forward. Public hostnames go
through Cloudflare tunnels with Cloudflare Access in front.

The router details below were last verified on 2026-09-25. SSH to calculon
rejected the admin key on 2026-11-21, so they could not be rechecked.

## Router: calculon (192.168.1.1)

- WAN is `re1`, on Verizon FiOS.
- Internal VLAN interfaces: `vlan001`, `vlan010`, `vlan020`, `vlan030`, `vlan099`.
- The main LAN is 192.168.1.0/24. The UniFi controller lives on VLAN 99 at
  192.168.9.2.
- Other DHCP pools: 192.168.0.100–245, 192.168.2.100–254, 192.168.3.6–254.
  Which VLAN carries which pool is not documented yet.
- `opt1` carries the trusted IoT devices (`trusted_iot_devices` alias).
- DHCP: ISC dhcpd on vlan001, vlan010, vlan030 and vlan099.
- DNS: ctrld listens on :53 and forwards to Unbound on :5555. Local names live
  under `i.knowles.xyz`.
- Remaining WAN port forwards: UDP 45000–65535 to two Nintendo Switch consoles.
- calculon and wernstrom are both on Tailscale.

Plugins in use: `os-node_exporter`, `os-telegraf` (see
[Monitoring](monitoring.md)).

## Public access

| Hostname | Path | Backend |
| --- | --- | --- |
| frigate.knowles.xyz | Cloudflare tunnel in CT 106 | Frigate |
| docker2.knowles.xyz | Cloudflare tunnel in CT 106 | Portainer |
| grafana.knowles.xyz | Cloudflare tunnel in CT 106 | Grafana, 192.168.1.240:3000 |
| proxmox.knowles.xyz, genmon.knowles.xyz | a second tunnel, not managed from CT 106 | wernstrom :8006; generator monitor 192.168.3.71 |

The CT 106 tunnel runs as the `cloudflare-tunnel` container with host
networking and a token. Every public host requires a Cloudflare Access login.

## Internal reverse proxy

Nginx Proxy Manager runs in CT 106 (192.168.1.4, admin UI on :81).

| Hostname | Backend |
| --- | --- |
| casa.knowles.xyz | Home Assistant 192.168.1.11:8123 |
| proxmox.knowles.xyz, proxmox.i.knowles.xyz | wernstrom :8006 |
| frigate.knowles.xyz | 192.168.1.4:5000 |
| go2rtc.i.knowles.xyz | 192.168.1.4:1984 |
| docker.i.knowles.xyz | Portainer 192.168.1.4:9443 |
| nginx.i.knowles.xyz | NPM admin 192.168.1.4:81 |
| router.i.knowles.xyz | OPNsense 192.168.1.1:443 |
| printer.i.knowles.xyz | printer 192.168.1.90:443 |
| genmon.knowles.xyz | 192.168.3.71:443 |

All NPM certificates use DNS challenges (Cloudflare and Route 53), so nothing
needs inbound port 80.
