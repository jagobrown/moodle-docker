# Moodle-Docker: Local Development Notes

## Table of Contents

- [Environment Setup](#environment-setup)
- [Docker Compose Commands](#docker-compose-commands)
- [HTTPS Configuration](#https-configuration)
- [Useful URLs](#useful-urls)
- [Troubleshooting](#troubleshooting)

---

## Environment Setup

### Required Environment Variables

Add these to your `~/.bashrc` to persist them across sessions:

```bash
export MOODLE_DOCKER_WWWROOT=/home/jago/repos/moodle-docker/MOODLE_LTS
export MOODLE_DOCKER_DB=pgsql

# For HTTPS/SSL connections
export MOODLE_DOCKER_SSL=1
export MOODLE_DOCKER_WEB_HOST=127.0.0.1:8443
```

### Verify Environment

```bash
# Check current environment variables
printenv | grep MOODLE_DOCKER

# Check Docker Compose version
docker compose version
```

---

## Docker Compose Commands

### Start/Stop Containers

```bash
# Start containers (detached)
bin/moodle-docker-compose up -d

# Stop containers (preserves data)
bin/moodle-docker-compose stop

# Restart stopped containers
bin/moodle-docker-compose start

# Stop and remove containers
bin/moodle-docker-compose down
```

### Database Management

```bash
# ⚠️ DANGER: Remove PostgreSQL volume (DELETES ALL DATA)
docker volume rm moodle-docker_pgsql-data
```

---

## HTTPS Configuration

### Check Container Status

```bash
# List all moodle-docker containers
docker ps -a --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# View HTTPS proxy logs
docker logs moodle-docker-https-proxy-1 2>&1 | tail -20

# Restart HTTPS proxy
docker restart moodle-docker-https-proxy-1
```

### Verify SSL Configuration

```bash
# Check SSL certificates exist
ls -la /home/jago/repos/moodle-docker/ssl/certs/

# Check local.yml exists
ls -la /home/jago/repos/moodle-docker/local.yml

# View nginx config
cat /home/jago/repos/moodle-docker/ssl/nginx.conf

# Inspect proxy volume mounts
docker inspect moodle-docker-https-proxy-1 --format '{{json .HostConfig.Binds}}' | python3 -m json.tool
```

### Test HTTPS Connection

```bash
# Quick test
curl -k -I https://127.0.0.1:8443/

# Check redirect chain
curl -k -L --max-redirs 5 -I https://127.0.0.1:8443/ 2>&1 | grep -E "HTTP|Location"
```

### Verify Moodle Configuration

```bash
# Check environment variables in container
docker exec moodle-docker-webserver-1 printenv | grep -E "MOODLE_DOCKER_SSL|MOODLE_DOCKER_WEB_HOST"

# Check wwwroot configuration
docker exec moodle-docker-webserver-1 php -r "
define('CLI_SCRIPT', true);
require('/var/www/html/config.php');
echo 'wwwroot: ' . \$CFG->wwwroot . PHP_EOL;
echo 'sslproxy: ' . (\$CFG->sslproxy ?? 'not set') . PHP_EOL;
"

# Check database wwwroot value
docker exec moodle-docker-db-1 psql -U moodle -d moodle -t -c "SELECT value FROM m_config WHERE name='wwwroot';"
```

---

## Useful URLs

| Service | URL |
|---------|-----|
| Moodle (HTTP) | http://localhost:8000/ |
| Moodle (HTTPS) | https://127.0.0.1:8443/ |
| Mailpit (emails) | http://localhost:8000/_/mail/ |

---

## Troubleshooting

### Git Permission Error

**Error:** `Git: unable to create file ssl/certs/server.crt: Permission denied`

**Solution:**
```bash
sudo chown -R $USER:$USER /home/jago/repos/moodle-docker/ssl/
```

### HTTPS Proxy Not Starting

1. Check if SSL certs exist in `ssl/certs/`
2. Ensure `local.yml` exists and defines the https-proxy service
3. Restart containers: `bin/moodle-docker-compose down && bin/moodle-docker-compose up -d`

### Redirect Loop (Too Many Redirects)

Ensure all these are set correctly:
- `MOODLE_DOCKER_SSL=1` environment variable
- `MOODLE_DOCKER_WEB_HOST=127.0.0.1:8443` environment variable
- Nginx config uses `$http_host` (not `$host`) for the Host header

curl -k -L --max-redirs 5 -I https://127.0.0.1:8443/ 2>&1 | grep -E "HTTP|Location"