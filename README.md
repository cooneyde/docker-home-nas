# docker-home-nas
Docker Compose stacks for a home server running Unraid on an Asustor AS5404T (Intel N5105, 16GB RAM).

## Stacks

| File | Services |
|------|----------|
| `docker-compose.yml` | Portainer |
| `core-services.yml` | AdGuard Home, Backrest, Nginx Proxy Manager (+ MariaDB), Watchtower, Nextcloud (+ MariaDB), Redis, Dozzle, Cloudflare Tunnel |
| `media-services.yml` | Plex, Radarr, Sonarr, Seerr, Deluge, Sportarr, Prowlarr, FlareSolverr |
| `iptv.yml` | Threadfin, WebGrab+, Jellyfin |
| `home-automation.yml` | Home Assistant, Matter Server |
| `immich.yml` | Immich (server, machine learning, Valkey, Postgres) |
| `network-monitoring.yml` | Speedtest Tracker, Uptime Kuma, Beszel (hub + agent) |

Bring a stack up with:

```
docker compose -f core-services.yml up -d
```

## Paths
* `/mnt/appdata/docker/volumes/<service>` – container config (the `appdata` pool)
* `/mnt/user/...` – media and downloads on the array

## Networks
Both networks are external, so they must exist before starting a stack.

* `br0` – Unraid's built-in custom network on the LAN (`192.168.114.0/24`), used for containers that need their own LAN IP (AdGuard on `.4`, Nginx Proxy Manager on `.101`). Enable it under *Settings → Docker → Custom network on interface br0*.
* `bridge-net-0` – shared user-defined bridge so containers can reach each other by name:

```
docker network create --subnet 172.16.1.0/24 bridge-net-0
```

## Secrets
`core-services.yml` reads `CLOUDFLARE_TUNNEL_TOKEN`, `network-monitoring.yml` reads `SPEEDTEST_APP_KEY` and `BESZEL_AGENT_KEY`, and `immich.yml` reads `UPLOAD_LOCATION`, `DB_*` and `IMMICH_VERSION`, from a `.env` file next to the compose file. Don't commit `.env`.

## Uptime Kuma ping monitors
Add these as **Ping** monitors (interval 20s) to see where latency or packet loss starts:

| Name | Host | Tells you |
|------|------|-----------|
| Router | `192.168.114.1` | Problems inside the home network |
| Virgin Hub | `192.168.100.1` | Router ↔ Hub link (Hub in modem mode only) |
| Virgin DNS 1 | `89.101.160.4` | Virgin's own network |
| Virgin DNS 2 | `89.101.160.5` | Virgin's own network |
| Quad9 | `9.9.9.11` | AdGuard upstream, beyond Virgin |
| Cloudflare | `1.1.1.1` | AdGuard upstream, beyond Virgin |

Bad on Router → home network; fine on Router but bad on Virgin targets → Virgin; only bad on Quad9/Cloudflare → beyond Virgin.
