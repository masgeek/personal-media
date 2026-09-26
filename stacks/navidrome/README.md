# Navidrome

Music streaming server with a Subsonic-compatible API.

## Endpoint

| Item | Value |
|------|-------|
| Container port | 4533 |
| Network | `dokploy-network` |
| Web UI path | `/app` |

## Local Access

Navidrome is an internal service and has no reverse proxy example. Reach it on the LAN at `http://<host-ip>:4533`, with the web UI under `/app` and the Subsonic API at `/rest` and `/stream`.

## Volumes

| Host path | Container path | Mode |
|-----------|----------------|------|
| `../files/config/navidrome` | `/data` | read/write |
| `/srv/media/music` | `/music` | read-only |

`/data` is a bind mount because the configuration and database should be directly accessible and backed up from the host. Back up `files/config/navidrome` accordingly.

## First Run

Navidrome has no default credentials. Open the UI and create the admin account before exposing the domain publicly.

## Notes

`ND_BASEURL` is set to an empty string because the app is served from the domain root. If you move it under a subpath such as `/music`, set `ND_BASEURL` to that prefix and adjust the Caddy site block accordingly.
