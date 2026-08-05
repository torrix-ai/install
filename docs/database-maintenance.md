# Database Maintenance

Torrix stores all data in a single SQLite file. This page covers backup, restore, monitoring, and high-availability options for production deployments.

## Storage location

By default the database file is at `./torrix.sqlite` relative to where you run Docker Compose. In the standard Docker deployment it is at `/data/torrix.sqlite` inside the container, mapped to `./data/torrix.sqlite` on your host.

```
torrix.sqlite       # main database
torrix.sqlite-wal   # write-ahead log (created automatically, normal)
torrix.sqlite-shm   # shared memory file (created automatically, normal)
```

Do not delete the `-wal` and `-shm` files while the server is running. They are part of the WAL journal and are safe to delete only when the server is stopped.

## Performance characteristics

Torrix enables WAL (Write-Ahead Logging) mode on startup. WAL mode provides:

- Concurrent reads during writes (no read locks during inserts)
- 3 to 5x higher write throughput compared to the SQLite default
- Automatic checkpointing every 1000 pages

For most production workloads (up to 10,000 AI calls per minute), SQLite with WAL mode is sufficient. If you are running more than 10,000 calls per minute, contact us about PostgreSQL support.

## Backup

### Manual backup via API

The admin backup endpoint streams a consistent point-in-time copy of the database using SQLite's online backup API. No downtime required.

```bash
curl -o torrix-backup-$(date +%Y-%m-%d).sqlite \
  http://localhost:8088/api/admin/backup \
  -H "Authorization: Bearer <your-admin-api-key>"
```

This endpoint is admin-only. The backup is logged in the audit trail.

### Manual file backup (server stopped)

If you have direct server access and can stop Torrix briefly:

```bash
docker compose stop torrix
cp ./data/torrix.sqlite ./backups/torrix-$(date +%Y-%m-%d-%H%M).sqlite
docker compose start torrix
```

### Automated backup with cron

Add a cron job on your host to call the backup API daily:

```bash
# /etc/cron.d/torrix-backup
0 2 * * * root curl -s -o /backups/torrix-$(date +\%Y-\%m-\%d).sqlite \
  http://localhost:8088/api/admin/backup \
  -H "Authorization: Bearer YOUR_ADMIN_KEY" && \
  find /backups -name "torrix-*.sqlite" -mtime +30 -delete
```

This runs at 2am daily and keeps 30 days of backups.

## Continuous replication with Litestream

[Litestream](https://litestream.io) is an open source tool that streams SQLite WAL changes to S3, Google Cloud Storage, or Azure Blob Storage in real time. It gives you:

- Continuous backup (not just nightly snapshots)
- Point-in-time restore to any second
- Disaster recovery to a new server in under a minute
- Zero application code changes

### Setup with S3

1. Install Litestream on your server:

```bash
wget https://github.com/benbjohnson/litestream/releases/download/v0.3.13/litestream-v0.3.13-linux-amd64.tar.gz
tar -xzf litestream-*.tar.gz
sudo mv litestream /usr/local/bin/
```

2. Create `/etc/litestream.yml`:

```yaml
dbs:
  - path: /path/to/data/torrix.sqlite
    replicas:
      - url: s3://your-bucket/torrix
        access-key-id: YOUR_AWS_KEY
        secret-access-key: YOUR_AWS_SECRET
        region: eu-west-1
```

3. Run Litestream as a service:

```bash
sudo litestream replicate -config /etc/litestream.yml
```

4. Restore from S3 to a new server:

```bash
litestream restore -config /etc/litestream.yml /path/to/data/torrix.sqlite
```

Litestream works alongside a running Torrix instance. No downtime, no application changes.

## Restore

### From a backup file

Stop Torrix, replace the database file, restart:

```bash
docker compose stop torrix
cp torrix-backup-2026-08-01.sqlite ./data/torrix.sqlite
docker compose start torrix
```

### From Litestream

```bash
docker compose stop torrix
litestream restore -config /etc/litestream.yml ./data/torrix.sqlite
docker compose start torrix
```

## Monitoring

### Database file size

```bash
du -sh ./data/torrix.sqlite
```

A typical deployment grows at roughly 1 to 5 MB per 10,000 runs depending on prompt length. With 7-day retention (Community) the database stabilises at a predictable size once the retention wipe runs.

### Check WAL size

A large WAL file (greater than 100 MB) indicates the checkpoint is not running. This is unusual with Torrix's workload but can happen if the server was stopped abnormally.

```bash
ls -lh ./data/torrix.sqlite*
```

If the `-wal` file is large, restart Torrix — the WAL checkpoint will run automatically on startup.

### Integrity check

Run SQLite's built-in integrity check periodically (takes a few seconds on typical databases):

```bash
sqlite3 ./data/torrix.sqlite "PRAGMA integrity_check;"
```

Expected output: `ok`

## Data retention

Torrix automatically deletes runs older than the retention window:

| Edition | Retention | When deleted |
|---|---|---|
| Community | 7 days | Checked every hour, wiped weekly |
| Pro | 30 days | Checked daily |
| Enterprise | 90 days | Checked daily |

The wipe keeps the database size bounded without any manual maintenance.

## Frequently asked questions

**Can I run Torrix with the database on a network file system (NFS)?**

No. SQLite requires local or fast block storage. NFS causes locking issues. Use a local SSD or Litestream to replicate to object storage instead.

**Can I have multiple Torrix instances sharing one database?**

No. SQLite allows only one writer process. Run one Torrix instance per database. If you need horizontal scaling, contact us about PostgreSQL support.

**How do I move the database to a different server?**

Stop Torrix, copy `torrix.sqlite` (and `torrix.sqlite-wal` if it exists) to the new server, update your `docker-compose.yml` volume path, start Torrix on the new server.

**Can I inspect the database directly?**

Yes. The SQL query interface at `/ui/query` lets admins run SELECT queries against the live database from the browser. For direct SQLite access: `sqlite3 ./data/torrix.sqlite`.
