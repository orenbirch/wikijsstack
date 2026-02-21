# Log Rotation Configuration Guide

## Overview

All services in the Wiki.js Docker stack are configured with automatic log rotation to prevent logs from consuming excessive disk space. This uses Docker's built-in JSON file logging driver with rotation capabilities.

## Configuration

### Default Settings

- **Max Size**: 10MB per log file
- **Max Files**: 3 rotated files kept per service
- **Compression**: Enabled (rotated logs are compressed)

This means each service can use up to ~30MB of log storage (3 files × 10MB).

### Customization

Edit the `.env` file to change log rotation settings:

```env
# Maximum size of each log file before rotation
LOG_MAX_SIZE=10m

# Number of rotated log files to keep
LOG_MAX_FILE=3
```

#### Size Format Examples:
- `10m` = 10 megabytes
- `100m` = 100 megabytes
- `1g` = 1 gigabyte
- `500k` = 500 kilobytes

### Per-Service Configuration

If you need different log settings for specific services, you can override the environment variables in `docker-compose.yml`:

```yaml
services:
  wikijs:
    # ... other settings ...
    logging:
      driver: "json-file"
      options:
        max-size: "50m"    # Custom size for this service
        max-file: "5"      # Keep more files for this service
        compress: "true"
```

## Log File Locations

Docker stores log files on the host system. To find them:

```bash
# Get log file path for a specific container
docker inspect --format='{{.LogPath}}' wikijs-app
docker inspect --format='{{.LogPath}}' wikijs-mariadb
docker inspect --format='{{.LogPath}}' wikijs-filebrowser
```

Typical location: `/var/lib/docker/containers/<container-id>/<container-id>-json.log`

## Viewing Logs

### Using Docker Compose

```bash
# View all service logs
docker compose logs

# Follow logs in real-time
docker compose logs -f

# View logs for specific service
docker compose logs wikijs

# View last N lines
docker compose logs --tail=100 wikijs

# View logs with timestamps
docker compose logs -t wikijs

# View logs from specific time
docker compose logs --since 2024-01-01T00:00:00 wikijs
docker compose logs --since 1h wikijs
```

### Checking Log File Sizes

```bash
# Check size of all container logs
docker ps -q | xargs -I {} sh -c 'echo "Container: {}"; docker inspect --format="{{.LogPath}}" {} | xargs ls -lh'

# Or with container names
for container in wikijs-app wikijs-mariadb wikijs-filebrowser; do
    echo "=== $container ==="
    docker inspect --format='{{.LogPath}}' $container | xargs ls -lh
done
```

## Log Rotation Behavior

### How Rotation Works

1. When a log file reaches `LOG_MAX_SIZE`, Docker:
   - Renames the current log to include a sequence number
   - Compresses it (e.g., `container-json.log.1.gz`)
   - Starts a new empty log file

2. When the number of rotated files exceeds `LOG_MAX_FILE`:
   - The oldest rotated log is deleted
   - New rotation proceeds normally

### Rotation Example

With `LOG_MAX_SIZE=10m` and `LOG_MAX_FILE=3`:

```
container-json.log       (current, up to 10MB)
container-json.log.1.gz  (compressed, ~10MB)
container-json.log.2.gz  (compressed, ~10MB)
container-json.log.3.gz  (compressed, ~10MB)
```

When the current log reaches 10MB, the oldest file (`.3.gz`) is deleted.

## Alternative Logging Drivers

### Syslog

For centralized logging to syslog:

```yaml
logging:
  driver: "syslog"
  options:
    syslog-address: "tcp://192.168.0.100:514"
    tag: "wikijs-{{.Name}}"
```

### Journald

For systemd journald integration:

```yaml
logging:
  driver: "journald"
  options:
    tag: "wikijs-{{.Name}}"
```

### Fluentd

For advanced log aggregation:

```yaml
logging:
  driver: "fluentd"
  options:
    fluentd-address: "localhost:24224"
    tag: "wikijs.{{.Name}}"
```

### Disable Logging

To disable logging for a service (not recommended):

```yaml
logging:
  driver: "none"
```

## Monitoring Log Space Usage

### Create a monitoring script

```bash
#!/bin/bash
# monitor-logs.sh

echo "Docker Log Space Usage"
echo "======================"
echo ""

for container in wikijs-app wikijs-mariadb wikijs-filebrowser; do
    if docker ps -q -f name=$container > /dev/null 2>&1; then
        echo "Container: $container"
        logpath=$(docker inspect --format='{{.LogPath}}' $container)
        if [ -f "$logpath" ]; then
            ls -lh "$logpath"
            # Also check for rotated logs
            logdir=$(dirname "$logpath")
            logbase=$(basename "$logpath")
            echo "Rotated logs:"
            ls -lh "$logdir/$logbase"* 2>/dev/null || echo "  None"
        fi
        echo ""
    fi
done

echo "Total log space used:"
docker ps -q | xargs docker inspect --format='{{.LogPath}}' | xargs du -ch 2>/dev/null | tail -n1
```

### Automated cleanup

If you need to manually clean up old logs:

```bash
# Warning: This will delete log history!
docker compose down
rm -rf /var/lib/docker/containers/*/
docker compose up -d
```

Or for a specific container:
```bash
docker compose stop wikijs
truncate -s 0 $(docker inspect --format='{{.LogPath}}' wikijs-app)
docker compose start wikijs
```

## Recommendations

### Development Environment
- `LOG_MAX_SIZE=20m`
- `LOG_MAX_FILE=2`
- Keep logs minimal

### Production Environment
- `LOG_MAX_SIZE=50m` to `100m`
- `LOG_MAX_FILE=5` to `10`
- Consider external log aggregation (Elasticsearch, Splunk, etc.)

### High-Traffic Sites
- Use external logging service (Fluentd, Logstash)
- Set up log rotation with longer retention
- Monitor log growth regularly
- Consider log levels (reduce verbosity in production)

## Troubleshooting

### Logs not rotating

1. Check Docker version (rotation requires Docker 1.13+):
   ```bash
   docker --version
   ```

2. Verify logging configuration:
   ```bash
   docker inspect wikijs-app | grep -A 10 LogConfig
   ```

3. Check disk space:
   ```bash
   df -h
   ```

### High disk usage despite rotation

1. Check for multiple containers with same name (old containers):
   ```bash
   docker ps -a | grep wikijs
   ```

2. Clean up stopped containers:
   ```bash
   docker container prune
   ```

3. Check system logs:
   ```bash
   journalctl -u docker
   ```

## Best Practices

1. **Set appropriate sizes**: Balance between log history and disk space
2. **Monitor regularly**: Set up alerts for disk usage
3. **Use compression**: Always enable compression for rotated logs
4. **External aggregation**: For production, consider centralized logging
5. **Backup important logs**: If you need long-term retention, backup rotated logs
6. **Test rotation**: Verify rotation works by generating test logs
7. **Document changes**: Keep track of log rotation settings in documentation

## Additional Resources

- [Docker Logging Documentation](https://docs.docker.com/config/containers/logging/)
- [JSON File Logging Driver](https://docs.docker.com/config/containers/logging/json-file/)
- [Log Rotation Best Practices](https://docs.docker.com/config/containers/logging/configure/)
