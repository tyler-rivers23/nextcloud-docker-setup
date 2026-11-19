# Nextcloud Migration: Raspberry Pi → Ubuntu (November 2026)

**Project:** Self-hosted Nextcloud infrastructure, backup, and recovery  
**Technology:** Linux, Docker Compose, MariaDB 10.11, Redis, SSH, rsync, Nginx Proxy Manager  
**Outcome:** Restored the existing Nextcloud installation onto an Ubuntu x86-64 host while keeping HTTPS routing through the original Raspberry Pi proxy.

> **Status:** Migration completed and application operational. Scheduled off-device backups and a full disaster-recovery drill are **planned but not yet implemented**.

## Architecture

\`\`\`mermaid
flowchart LR
    A[Remote client] --> B[Dynamic DNS and home router]
    B --> C[Raspberry Pi: Nginx Proxy Manager]
    C -->|HTTP over trusted LAN| D[Ubuntu: Nextcloud Apache]
    D --> E[(MariaDB 10.11)]
    D --> F[(Redis)]
    G[Nextcloud cron] --> D
\`\`\`

I kept Nginx Proxy Manager on the Pi because it also serves other home-lab applications. I moved only the Nextcloud app and its database to the larger Ubuntu host, reducing the impact on unrelated services.

## Results

| Check | Before | After |
|---|---|---|
| Root filesystem | Raspberry Pi: 117 GB, reached 100% usage | Ubuntu: 468 GB, **383 GB available** at last check |
| Nextcloud data | Approximately 32 GB | Approximately 32 GB transferred |
| Nextcloud release | 32.0.11.1 | **32.0.11.1**, matched during the migration |
| Database | MariaDB 10.11 | Final SQL restore succeeded, **197 tables** |
| Public access | Pi Nginx Proxy Manager and HTTPS | Same proxy and hostname; Ubuntu backend |

The 383 GB figure is free space on the Ubuntu filesystem, not an individual user's Nextcloud quota.

## Engineering work

### 1. Diagnose and recover from storage exhaustion

The Pi's root filesystem had filled. Using \`df\`, \`du\`, and Docker storage inspection, I traced much of the usage to Docker, including a repeatedly restarting, unused coturn container with approximately **22.34 GB** of JSON logs. I cleared the package cache, removed the unused coturn service, updated Docker Compose, and recovered about **30 GB** of free space on the Pi.

I avoided broad Docker volume pruning, which could have removed persistent application data.

### 2. Recover Docker management access

The installed Docker CLI failed with \`SIGILL\`. I confirmed that the daemon was responsive independently, checked the installed package, and found the CLI binary failed package verification. I extracted a clean ARM64 Docker client to a separate directory and used it to inspect containers without reinstalling Docker or touching live volumes.

### 3. Inventory data and discover a volume mismatch

The running Nextcloud application mounted a roughly **33 GB anonymous Docker volume** at \`/var/www/html\`, while the old cron service used a separate ~**3.7 GB named volume**. Rather than assuming both volumes were interchangeable, I inspected their contents, preserved the older volume separately, and corrected the destination app and cron configuration to use the same primary data mount.

### 4. Transfer files and configuration securely

I used encrypted SSH transfers and \`rsync\` to stage the active Nextcloud directory on Ubuntu. A tar stream transferred Compose configuration without creating another large archive on the Pi. I took a preliminary database dump to test the restore process before scheduling downtime.

### 5. Complete a consistent cutover

I stopped the old cron process, enabled Nextcloud maintenance mode, stopped the Pi application, and exported a final MariaDB SQL snapshot. I then synchronized the file tree again and performed a dry run.

**Final validation:** the file dry run showed **zero regular files requiring transfer** under the selected \`rsync\` options. A new Ubuntu database was initialized from the final SQL dump, which imported with exit status \`0\` and contained **197 tables**.

### 6. Restore application and reconfigure traffic

I pinned the destination to the matching Nextcloud version and kept MariaDB on 10.11. The restored files were assigned to the container's \`www-data\` UID/GID, the application started in maintenance mode, and \`php occ status\` confirmed \`installed=true\` and \`needsDbUpgrade=false\`.

I exposed an HTTP backend port to the private LAN only, updated the existing Nginx Proxy Manager upstream to the Ubuntu machine, and retained the existing HTTPS certificate and dynamic DNS hostname. After validation, I disabled maintenance mode and started the Ubuntu cron container. The migrated Nextcloud instance was confirmed operational.

## Reproducible command patterns

These examples use **placeholders**, omit sensitive data, and are **not** a standalone recovery script. Confirm paths and service names before running them.

\`\`\`bash
# Storage diagnosis
df -h /
sudo du -xhd1 /var/lib 2>/dev/null | sort -h | tail -15

# Discover the active application volume from a running container
SRC=$(docker inspect --format \
  '{{range .Mounts}}{{if eq .Destination "/var/www/html"}}{{.Source}}{{end}}{{end}}' \
  nextcloud_app)
printf '%s\n' "$SRC"

# Stage files over SSH
sudo rsync -rltH --partial --info=progress2 \
  -e "sudo -u $(id -un) ssh" \
  "$SRC/" USER@DESTINATION:/srv/nextcloud-migration/nextcloud-primary/

# MariaDB export, streaming to a private destination file
set -o pipefail
docker exec nextcloud_db sh -c \
  'exec mariadb-dump --single-transaction --quick \
   --user="$MYSQL_USER" --password="$MYSQL_PASSWORD" "$MYSQL_DATABASE"' \
  | ssh USER@DESTINATION 'umask 077; cat > /private/backup/nextcloud.sql'

# Validate the restored application
docker compose exec -u www-data nextcloud php occ status
\`\`\`

## Verification and limitations

| Evidence | Observation | What it does *not* prove |
|---|---|---|
| Database restore | Exit status 0; 197 tables | Integrity of every row and application workflow |
| File synchronization | No pending files in final dry run | Full ACL/xattr/permission equivalence; checks used \`-rltH\` |
| Nextcloud runtime | Version 32.0.11.1; maintenance disabled; no DB upgrade required | Full disaster-recovery validation |
| Cron | Container running | That recent scheduled jobs completed successfully |
| Storage | 383 GB free on the 468 GB filesystem | Guaranteed per-account quota or independent backup space |

## Security and backup follow-ups

- [ ] Add **encrypted, scheduled, off-device backups** of files, MariaDB, and application/proxy configuration.
- [ ] Run a complete restore drill on an isolated environment.
- [ ] Audit trusted-proxy headers and restrict the Ubuntu backend port to the expected proxy host.
- [ ] Verify scheduled Nextcloud jobs from the administration dashboard.
- [ ] Configure log rotation, health monitoring, and laptop power settings.

**Backup distinction:** The SQL dumps and copied files used during migration were validation and rollback artifacts. They are **not** a tested ongoing backup system for changes made after the cutover.

## Public documentation hygiene

This report deliberately omits real private IP addresses, domain names, user records, keys, passwords, and credentials. Do not commit live \`.env\` files, \`config.php\`, database dumps, user files, Nginx Proxy Manager data, TLS certificates, or unredacted terminal screenshots.

---

**Key takeaway:** The project exercised Linux troubleshooting, Docker persistence and lifecycle management, encrypted data transfer, SQL backup/restore, change control, reverse-proxy networking, and operational validation—not just deploying an app.
