# Transmission with PIA VPN

This repository contains a Docker Compose configuration for running Transmission with PIA VPN protection and RSS feed integration.

## Prerequisites

- Docker and Docker Compose
- PIA VPN subscription
- Valid RSS feed URL (provided by the RSS Feed App)

## Quick Start

- Clone the repository:

```bash
git clone https://github.com/tadeasf/transmission-pia-compose-public
cd transmission-pia-compose-public
```

- Create necessary directories and files:

```bash
mkdir -p config/transmission-rss
touch config/transmission-rss/seen.txt
chmod 666 config/transmission-rss/seen.txt  # Important for write permissions
```

- Create `.env` file:

```bash
cat << EOF > .env
DOWNLOADS_DIR=/path/to/your/downloads
OPENVPN_USERNAME=your_pia_username
OPENVPN_PASSWORD=your_pia_password
TRANSMISSION_USER=your_transmission_user
TRANSMISSION_PASSWORD=your_transmission_password
EOF
```

- Configure transmission-rss:

```bash
cat << EOF > config/transmission-rss/transmission-rss.conf
feeds:
  - url: "https://your-rss-app-domain/api/rss"
    download_path: "/data/completed"
    seen_by_guid: true

server:
  host: "transmission-pia"
  port: 9091
  rpc_path: "/transmission/rpc"
  tls: false

login:
  username: "${TRANSMISSION_USER}"
  password: "${TRANSMISSION_PASSWORD}"

update_interval: 60

log:
  level: debug
  target: STDERR

add_paused: false
seen_file: "/root/.config/transmission/seen"
EOF
```

- Start the services:

```bash
docker compose up -d
```

## Accessing the Interface

- Transmission Web UI: `http://localhost:9091`
- Default credentials: As set in your .env file

## Configuration Options

### VPN Settings

- Provider: PIA (Private Internet Access)
- Default Location: Frankfurt
- Change location by modifying `OPENVPN_CONFIG` in docker-compose.yml

### Web UI Options

Two UI options are available (uncomment your preference in docker-compose.yml):

- Flood for Transmission (default)
- Transmissionic

### RSS Feed Integration

The RSS service automatically:

- Checks for new torrents every 60 seconds
- Downloads them to the specified path
- Maintains a history of downloaded torrents in seen.txt

## Troubleshooting

1. VPN Connection Issues:

```bash
docker logs transmission-pia
```

- RSS Feed Issues:

```bash
docker logs transmission-rss
```

- Permission Issues:

```bash
# Fix seen.txt permissions
chmod 666 config/transmission-rss/seen.txt
```

## Security Notes

- Always use strong passwords
- Keep your .env file secure
- Regular updates recommended:

```bash
docker compose pull
docker compose up -d
```

## Network & Security Setup

### Exposed Ports

The service exposes these ports by default:

```yaml
ports:
  - "9091:9091"    # Web UI
  - "51413:51413"  # Peer listening (TCP)
  - "51413:51413/udp"  # Peer listening (UDP)
```

### Secure Access Setup

For secure remote access, we recommend using Caddy v2 as a reverse proxy. Here's how to set it up:

1. Install Caddy:

    ```bash
    sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
    curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
    curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
    sudo apt update
    sudo apt install caddy
    ```

2. Modify your docker-compose.yml to only expose ports locally:

    ```yaml
    ports:
    - "127.0.0.1:9091:9091"
    ```

3. Add this to your /etc/caddy/Caddyfile:

    ```caddyfile
    transmission.yourdomain.com {
        reverse_proxy localhost:9091
        tls {
            protocols tls1.3
        }
        encode gzip
        header {
            # Security headers
            Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
            X-Content-Type-Options "nosniff"
            X-Frame-Options "DENY"
            Referrer-Policy "strict-origin-when-cross-origin"
        }
    }
    ```

4. Restart Caddy:

    ```bash
    sudo systemctl reload caddy
    ```
