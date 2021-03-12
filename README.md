# docker-home-nas
docker-compose for home server with adguard, sonarr, radarr, jackett, etc installed

## Services exposed

* Sonarr v3 https://github.com/linuxserver/docker-sonarr
* Radarr v3 https://github.com/linuxserver/docker-radarr
* Plex https://github.com/linuxserver/docker-plex
* deluge https://github.com/linuxserver/docker-deluge
* Jackett https://github.com/linuxserver/docker-jackett
* Portainer (Container management UI) https://github.com/portainer/portainer
* Ombi (managed aggregator for sonarr and radarr) https://github.com/linuxserver/docker-ombi
* Heimdall (links UI to container UIs) https://github.com/linuxserver/Heimdall
* AdGuard https://github.com/AdguardTeam/AdGuardHome


## Setting up docker networks on both ethernet ports 
Driver here is set to qnet, a native driver on QNAP NAS servers

```
docker network create --driver=qnet --ipam-driver=qnet --ipam-opt=iface=eth0 --subnet 192.168.X.0/24 --gateway 192.168.X.Z my-network

```

## Useful URLs
* Setting up bitwarden with https: https://scyzoryk.fubar.pl/bitwarden-on-qnap/
