# Configuration

---

## Environment variables

All variables are optional. Set them in your Docker Compose `environment` block, your shell, or a `.env` file (depending on how you run Torrix).

| Variable | Default | Description |
|---|---|---|
| `DB_PATH` | `/data/torrix.sqlite` (Docker)<br>`~/.torrix/torrix.sqlite` (native) | Path to the SQLite database file |
| `PORT` | `8088` | HTTP port Torrix listens on |
| `TORRIX_EDITION` | `community` | Edition: `community`, `pro`, or `cloud`. Controls retention, run limits, and feature availability |
| `TORRIX_LICENSE_KEY` | — | RSA-signed license key for Pro or Enterprise. Verified locally at startup — no cloud call needed |
| `TORRIX_TELEMETRY` | `true` | Set `false` to disable the anonymous startup ping |
| `TORRIX_OPEN_BROWSER` | `true` (native only) | Set `false` to prevent Torrix from opening a browser tab on startup |
| `TORRIX_LOCAL_HOST` | `localhost` | Hostname Torrix uses when calling local LLMs. Docker users pointing at Ollama on the host should set this to `host.docker.internal` |
| `TORRIX_RESET_PASSWORD` | — | Set `true` once to reset the admin password at next login. Remove after use |
| `NODE_ENV` | — | Set `development` to raise all rate limits to 300 req/min (useful during local development) |

---

## Native binary

Download from [torrix.ai](https://torrix.ai) or install with:

```bash
curl -fsSL https://torrix.ai/install.sh | sh
```

Set environment variables in your shell before running:

```bash
PORT=9000 TORRIX_TELEMETRY=false torrix
```

On macOS, the launchd service reads variables from `~/.torrix/env`:

```bash
# ~/.torrix/env
PORT=9000
TORRIX_TELEMETRY=false
```

Reload the service after editing:

```bash
launchctl unload ~/Library/LaunchAgents/io.torrix.app.plist
launchctl load  ~/Library/LaunchAgents/io.torrix.app.plist
```

---

## Docker Compose

Full example with all common overrides:

```yaml
services:
  torrix:
    image: torrixai/torrix:latest
    ports:
      - "8088:8088"
    volumes:
      - torrix_data:/data
    environment:
      - DB_PATH=/data/torrix.sqlite
      - PORT=8088
      - TORRIX_EDITION=community
      - TORRIX_TELEMETRY=true
      # Uncomment to activate a Pro license:
      # - TORRIX_LICENSE_KEY=trxl_...
      # Uncomment if using Ollama on the Docker host:
      # - TORRIX_LOCAL_HOST=host.docker.internal
    restart: unless-stopped

volumes:
  torrix_data:
```

Update to the latest image:

```bash
docker compose pull
docker compose up -d
```

---

## Nginx reverse proxy

Put Torrix behind nginx for HTTPS or a custom domain:

```nginx
server {
    listen 443 ssl;
    server_name torrix.example.com;

    ssl_certificate     /etc/letsencrypt/live/torrix.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/torrix.example.com/privkey.pem;

    location / {
        proxy_pass         http://localhost:8088;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_read_timeout 300s;
    }
}
```

For streaming responses, `proxy_read_timeout 300s` prevents nginx from closing long-running LLM streams.

---

## Edition limits

| | Community | Pro | Cloud |
|---|---|---|---|
| Max runs stored | 10,000 | Unlimited | Unlimited |
| Retention | 7 days | 30 days | 90 days |
| Users | 1 | Unlimited | Unlimited |
| Full-text search | No | Yes | Yes |
| Online evals | No | Yes | Yes |

Set with `TORRIX_EDITION=pro` once you have a valid `TORRIX_LICENSE_KEY`. If the key is missing or invalid, Torrix falls back to Community and logs a warning.

---

## Common overrides

**Run on a different port:**
```bash
PORT=9000 torrix
# or in Docker: ports: ["9000:8088"] + PORT=9000
```

**Point at Ollama from Docker:**
```yaml
environment:
  - TORRIX_LOCAL_HOST=host.docker.internal
```

**Reset a lost admin password:**
```bash
TORRIX_RESET_PASSWORD=true torrix
# Log in to set a new password, then remove the variable and restart
```

**Raise rate limits during development:**
```bash
NODE_ENV=development torrix
```

**Custom SQLite path:**
```bash
DB_PATH=/mnt/fast-disk/torrix.sqlite torrix
```
