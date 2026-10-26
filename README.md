# docker-home-nas
Docker Compose stacks for a home server running Unraid on an Asustor AS5404T (Intel N5105, 16GB RAM).

## Stacks

| File | Services |
|------|----------|
| `docker-compose.yml` | Portainer |
| `core-services.yml` | AdGuard Home, Backrest, Nginx Proxy Manager (+ MariaDB), Watchtower, Nextcloud (+ MariaDB), Redis, Dozzle, Cloudflare Tunnel, Fail2ban |
| `media-services.yml` | Plex, Radarr, Sonarr, Seerr, Deluge, Sportarr, Prowlarr, FlareSolverr |
| `iptv.yml` | Threadfin, WebGrab+, Jellyfin |
| `home-automation.yml` | Home Assistant, Matter Server |
| `immich.yml` | Immich (server, machine learning, Valkey, Postgres) |
| `network-monitoring.yml` | Speedtest Tracker, Uptime Kuma |

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
`core-services.yml` reads `CLOUDFLARE_TUNNEL_TOKEN`, `network-monitoring.yml` reads `SPEEDTEST_APP_KEY`, and `immich.yml` reads `UPLOAD_LOCATION`, `DB_*` and `IMMICH_VERSION`, from a `.env` file next to the compose file. Don't commit `.env`; [`.env.example`](.env.example) lists every variable.

## Immich
`immich.yml` extends [`hwaccel.transcoding.yml`](hwaccel.transcoding.yml) (QuickSync) and [`hwaccel.ml.yml`](hwaccel.ml.yml) (OpenVINO), so keep those next to it. Keep Immich's `.env` in its own folder, because `env_file` passes every variable in it to the Immich containers.

Its Redis and Postgres services are named `immich-redis` and `immich-database` so they don't share the name `redis` with the core stack's Redis (Nextcloud uses that one) on `bridge-net-0`. Moving from the old names takes a one-off stop and start, which keeps your data (it's in bind mounts and a named volume):

```
docker compose -f immich.yml down
docker compose -f immich.yml up -d
```

The external library mounts Nextcloud's `InstantUpload` folder at `/mnt/media`.

## Fail2ban
Config lives in [`fail2ban/`](fail2ban/).

| Jail | Watches | Bans at |
|------|---------|---------|
| `nextcloud-auth` | Nextcloud failed logins | Cloudflare |
| `seerr-auth` | Seerr failed logins | Cloudflare |
| `immich-auth` | Immich failed logins (via Unraid syslog) | Cloudflare |
| `unraid-sshd` | Unraid SSH failed logins | host firewall |
| `unraid-webgui` | Unraid web GUI failed logins | host firewall |

Nextcloud, Seerr and Immich are reached through Cloudflare Tunnel, which bypasses the home firewall, so their bans are added as Cloudflare IP Access Rules. LAN and Docker addresses are never banned. 5 failures in 10 minutes gives a 1 hour ban, getting longer for repeat offenders (up to a week).

Setup:

1. Copy the config into appdata:
   ```
   mkdir -p /mnt/appdata/docker/volumes/fail2ban/fail2ban
   cp -R fail2ban/* /mnt/appdata/docker/volumes/fail2ban/fail2ban/
   ```
2. Create a Cloudflare API token (permission *Zone → Firewall Services → Edit*, limited to your zone). Copy `action.d/cloudflare-token.local.example` to `action.d/cloudflare-token.local` in the appdata folder and fill in the token and zone ID. That file is git-ignored.
3. **Nextcloud:** add to `/mnt/appdata/docker/volumes/nextcloud/config/www/nextcloud/config/config.php` so it logs the visitor's IP rather than the tunnel's, to a file fail2ban can read:
   ```php
   'trusted_proxies' => ['172.16.1.0/24'],
   'forwarded_for_headers' => ['HTTP_CF_CONNECTING_IP', 'HTTP_X_FORWARDED_FOR'],
   'log_type' => 'file',
   'logfile' => '/config/log/nextcloud/nextcloud.log',
   ```
   Then run `mkdir -p /mnt/appdata/docker/volumes/nextcloud/config/log/nextcloud` and restart Nextcloud.
4. **Seerr:** Settings → Network → turn on *Enable Proxy Support*, so it logs the visitor's IP.
5. **Immich:** nothing to set. It already trusts `cloudflared`, and `immich.yml` sends its logs to Unraid's syslog. Recreate the stack once so the logging change applies.
6. Start fail2ban, then check with `docker exec fail2ban fail2ban-client status <jail>`.

If an app still logs the tunnel's IP (`172.16.1.x`), fail2ban ignores it instead of banning the tunnel.

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
