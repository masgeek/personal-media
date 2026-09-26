# Pi-hole

Network-wide ad blocking via DNS.

## Endpoint

| Item | Value |
|------|-------|
| Web interface port | 8081 |
| DNS port | 53 (TCP and UDP) |
| Network | `host` |

## Why Host Networking

Pi-hole uses `network_mode: host` because it must bind port 53 on the host to serve DNS to LAN clients. A bridge network cannot publish a privileged-style DNS service the same way without extra configuration.

The tradeoffs are significant:

- The container is not attached to `dokploy-network`, so Dokploy cannot route to it by container name.
- The published `ports` block is commented out because host networking already exposes the ports. Uncommenting it has no effect and will confuse the configuration.
- `FTLCONF_dns_listeningMode: ALL` makes the DNS service reachable from outside the machine unless the host firewall blocks port 53. Restrict it to the LAN.

## Local Access

Pi-hole is an internal service and has no reverse proxy example. Reach the web interface on the LAN at `http://<host-ip>:8081`. Do not expose it beyond the LAN: there are no user accounts, and the interface exposes the full DNS query history.

## Required Environment

| Variable | Purpose |
|----------|---------|
| `PIHOLE_PASSWORD` | web interface and API password |

`FTLCONF_webserver_port` is set to `8081` to avoid clashing with Dokploy's nginx, which uses port 80.

## Volumes

`pihole_data` is a named volume mounted at `/etc/pihole`. It contains the blocking lists and query database. Back it up with Dokploy Volume Backups.

## Not Deployed

This stack is tracked in git but is **not** listed in the root `docker-compose.yml`. It does not start as part of a Dokploy deploy. Add the include line to enable it:

```yaml
- stacks/pihole/docker-compose.yml
```

## Notes

The `pihole-network` bridge network is declared but unused while `network_mode: host` is set. It can be removed to reduce confusion.
