# Configuration

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `DB_PATH` | `/data/torrix.sqlite` | Path to SQLite database inside the container |
| `TORRIX_TELEMETRY` | `true` | Set to `false` to opt out of anonymous usage stats |

---

## Setting environment variables

Add an `environment` block to your `docker-compose.yml`:

```yaml
services:
  torrix:
    image: torrixai/torrix:latest
    ports:
      - "8088:8088"
    volumes:
      - ./data:/data
    environment:
      - TORRIX_TELEMETRY=false
    restart: unless-stopped
```

Then restart Torrix:

```bash
docker compose down
docker compose up -d
```
