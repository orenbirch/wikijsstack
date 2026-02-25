# Wiki.js Docker Stack

A complete Docker Compose stack for running Wiki.js with MariaDB database and Filebrowser for file management.

## Services Included

- **Wiki.js**: Modern, powerful wiki application
- **MariaDB**: Database backend for Wiki.js
- **Filebrowser**: Web-based file manager for backups and file uploads
- **Persistent Volumes**: All data is stored in Docker volumes

## Quick Start

1. **Copy the environment file:**
   ```bash
   cp .env.example .env
   ```

2. **Edit the `.env` file and set required values:**
   ```bash
   nano .env
   ```
   ⚠️ **Important**: Set strong values for `DB_ROOT_PASSWORD`, `DB_PASSWORD`, and `FB_ADMIN_PASSWORD` before deploying.

3. **Start the stack:**
   ```bash
   docker compose up -d
   ```

4. **Access the applications:**
   - Wiki.js: http://localhost:3000
   - Filebrowser: http://localhost:8080

## Initial Setup

### Wiki.js Setup
1. Navigate to http://localhost:3000
2. Create your administrator account
3. Configure your wiki settings

### Filebrowser Setup
1. Navigate to http://localhost:8080
2. Login credentials (from your `.env` file):
   - Username: `FB_ADMIN_USER`
   - Password: `FB_ADMIN_PASSWORD`
3. **Change the password immediately** after first login if using a temporary value
4. Configure users and permissions as needed

## File Management

Filebrowser provides access to:
- `/srv/mariadb-data` - MariaDB database files (read-only)
- `/srv/wikijs-data` - Wiki.js application data
- `/srv/wikijs-content` - Wiki.js content/repository
- `/srv/backups` - Local backup directory (maps to `./backups` on host)

## Backup Strategy

### Manual Backup via Filebrowser
1. Access Filebrowser at http://localhost:8080
2. Navigate to the volumes you want to backup
3. Select files/folders and download them

### Database Backup
Create a database backup with:
```bash
docker compose exec mariadb mysqldump -u root -p wikijs > backups/wikijs-backup-$(date +%Y%m%d).sql
```

### Automated Backup Script
Create a backup script:
```bash
#!/bin/bash
BACKUP_DIR="./backups"
DATE=$(date +%Y%m%d-%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup database
docker compose exec -T mariadb mysqldump -u root -p$DB_ROOT_PASSWORD wikijs > $BACKUP_DIR/db-backup-$DATE.sql

# Backup Wiki.js data
docker run --rm -v wikijs-docker-stack_wikijs-data:/data -v $(pwd)/backups:/backup alpine tar czf /backup/wikijs-data-$DATE.tar.gz -C /data .
docker run --rm -v wikijs-docker-stack_wikijs-content:/data -v $(pwd)/backups:/backup alpine tar czf /backup/wikijs-content-$DATE.tar.gz -C /data .

echo "Backup completed: $DATE"
```

## Restore from Backup

### Restore Database
```bash
docker compose exec -T mariadb mysql -u root -p wikijs < backups/db-backup-YYYYMMDD.sql
```

### Restore Volumes
```bash
docker run --rm -v wikijs-docker-stack_wikijs-data:/data -v $(pwd)/backups:/backup alpine tar xzf /backup/wikijs-data-YYYYMMDD.tar.gz -C /data
```

## Useful Commands

### View logs
```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f wikijs
docker compose logs -f mariadb
docker compose logs -f filebrowser

# View last 100 lines
docker compose logs --tail=100 wikijs
```

### Log Rotation
All services are configured with automatic log rotation to prevent disk space issues:
- **Max size**: 10MB per log file (configurable via `LOG_MAX_SIZE` in `.env`)
- **Max files**: 3 rotated files kept (configurable via `LOG_MAX_FILE` in `.env`)
- **Compression**: Rotated logs are automatically compressed

To change log rotation settings, edit `.env`:
```env
LOG_MAX_SIZE=20m    # Max size before rotation (e.g., 10m, 100m, 1g)
LOG_MAX_FILE=5      # Number of rotated files to keep
```

To view current log file sizes:
```bash
docker inspect --format='{{.LogPath}}' wikijs-app
docker inspect --format='{{.LogPath}}' wikijs-mariadb
docker inspect --format='{{.LogPath}}' wikijs-filebrowser
```

### Stop the stack
```bash
docker compose down
```

### Stop and remove volumes (⚠️ DELETES ALL DATA)
```bash
docker compose down -v
```

### Restart a service
```bash
docker compose restart wikijs
```

### Update images
```bash
docker compose pull
docker compose up -d
```

## Volume Information

All data is stored in Docker volumes:
- `mariadb-data`: MariaDB database files
- `wikijs-data`: Wiki.js application data
- `wikijs-content`: Wiki.js content repository
- `filebrowser-config`: Filebrowser configuration
- `filebrowser-database`: Filebrowser user database

## Security Recommendations

1. **Set strong values for all required passwords** in the `.env` file
2. **Change Filebrowser admin password** after first login
3. Use strong, unique passwords for all services
4. Enable authentication in Filebrowser (`FB_NOAUTH=false`)
5. Use a reverse proxy (nginx, Traefik) with SSL in production
6. Regularly backup your data
7. Pin image tags (or digests) in `.env` and keep them updated
8. Keep container hardening enabled (`no-new-privileges`, reduced Linux capabilities, read-only rootfs for app containers)

## Ports

- `3000` - Wiki.js web interface
- `8080` - Filebrowser web interface

You can change these ports in the `.env` file.

## Troubleshooting

### Wiki.js won't start
- Check if MariaDB is healthy: `docker compose ps`
- View logs: `docker compose logs mariadb`
- Ensure database credentials match in `.env`

### Cannot access Filebrowser
- Check if container is running: `docker compose ps filebrowser`
- Verify port isn't in use: `netstat -tulpn | grep 8080`

### Database connection issues
- Verify database service is running
- Check environment variables in `.env`
- Look at Wiki.js logs: `docker compose logs wikijs`

## Additional Resources

- [Wiki.js Documentation](https://docs.requarks.io/)
- [Filebrowser Documentation](https://filebrowser.org/)
- [MariaDB Documentation](https://mariadb.org/documentation/)
