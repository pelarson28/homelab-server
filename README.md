# homelab-server

# darbys-den

Self-hosted media server running on a refurbished SFF business desktop. Automated acquisition, transcoding, and remote access, fully containerized.

## Stack

| Category | Tool |
|---|---|
| OS | Ubuntu Server 24.04 LTS |
| Container management | Docker, Docker Compose, CasaOS |
| Media server | Jellyfin (Intel QSV hardware transcoding) |
| Indexer management | Prowlarr |
| Movie/TV automation | Radarr, Sonarr |
| Book/audiobook automation | LazyLibrarian |
| Subtitles | Bazarr |
| Download client | qBittorrent |
| VPN (download isolation) | Gluetun (WireGuard) |
| Ebook library | Calibre-Web |
| Requests (movies/TV) | Jellyseerr |
| Requests (books) | Libreseerr |
| Remote access (public) | Cloudflare Tunnel |
| Remote access (private) | Tailscale |
| DNS | Cloudflare |

## Hardware

| Component | Spec |
|---|---|
| Host | HP EliteDesk 800 G5 SFF, i5-9500, UHD 630 |
| RAM | 16GB DDR4 |
| Boot | 256GB NVMe |
| Storage | 4TB Seagate Exos, 3.5", room for a second internal drive |

Picked for two internal 3.5" bays (most SFF desktops only fit one) and a QSV-capable iGPU, well under the cost of a comparable pre-built NAS.

## Architecture

Services are split into two tiers based on how much they should trust the network they're on.

```
Internet
   |
   v
Cloudflare Tunnel (outbound only, no open ports on the router)
   |
   v
public / real auth
  - jellyfin
  - jellyseerr
  - libreseerr
  - calibre-web

Tailscale (private mesh)
   |
   v
private / admin only
  - casaos
  - prowlarr
  - qbittorrent
  - radarr / sonarr
  - lazylibrarian
```

The *arr apps and download client have thin built-in auth, fine for a personal admin tool, not fine to put on the public internet. Jellyfin, Jellyseerr, Libreseerr, and Calibre-Web have real account systems, so those are the only things routed through the public tunnel. Everything else is Tailscale-only.

qBittorrent, Prowlarr, Radarr, and Sonarr run with `network_mode: service:gluetun`, so only their traffic goes through the WireGuard tunnel, with a network-level kill switch if the tunnel drops. Everything else runs on the normal Docker bridge network and reaches the VPN-wrapped services through the host's exposed ports, since containers sharing a network namespace with `service:x` mode can't resolve each other by container name.

## Directory structure

```
/mnt/media/data/
├── torrents/
│   ├── movies/
│   ├── tv/
│   ├── books/
│   └── incomplete/
└── media/
    ├── movies/
    ├── tv/
    ├── music/
    ├── books/
    └── audiobooks/

/opt/appdata/
├── gluetun/
├── qbittorrent/
├── prowlarr/
├── radarr/
├── sonarr/
├── bazarr/
├── lazylibrarian/
└── calibre-web/
```

Downloads and finished media live under the same root so Radarr/Sonarr can hardlink instead of copying: qBittorrent keeps seeding from `torrents/`, Jellyfin serves from `media/`, no duplicate storage.

## Notes

Things that broke during setup, in case they save someone else time:

- A stray duplicate netplan config in `/etc/netplan/` caused two conflicting default route declarations. Netplan reads every `.yaml` file in that directory, not just the one you're editing.
- Containers on `network_mode: service:x` share a network namespace but not a process namespace, so `localhost` doesn't resolve between them like it would on a normal bridge network. Route through the host's LAN IP and the VPN container's exposed ports instead.
- Bazarr path mapping errors came down to reconciling three different views of the same files: host filesystem, the media manager's container path, and Bazarr's own container path. Debug logging showed the exact malformed path being requested.
- Migrating to a new router changed the LAN subnet, which silently broke Gluetun's `FIREWALL_OUTBOUND_SUBNETS` allowlist since it was scoped to the old range. Anything that hardcodes a subnet needs revisiting after a network change.
- Some public subtitle/torrent providers require a paid tier for programmatic API access even when the free tier works fine in a browser.

## TODO

- [ ] Second internal 3.5" drive
- [ ] Set up Home Assistant for home lighting and audio automation
- [ ] Host local AI for troubleshooting and free offline knowledge resource
- [ ] Audiobookshelf for audiobook-specific playback
- [ ] Move off managed Tailscale to self-hosted Headscale
- [ ] Offsite backup rotation via Duplicati

## Credits

Folder structure and hardlink approach based on [TRaSH Guides](https://trash-guides.info/). Stack shape informed by the usual r/selfhosted and r/homelab patterns.

## Scope

This repo documents infrastructure and automation architecture. It doesn't cover specific indexer or content-source configuration, that part is well-trodden ground elsewhere.

## License

MIT
