# Project N.O.M.A.D - Troubleshooting Guide

This guide covers common issues and their solutions when running Project N.O.M.A.D on WSL with Snap Docker.

## Table of Contents

1. [Plugin Installation Issues](#plugin-installation-issues)
2. [Container Issues](#container-issues)
3. [Download Issues](#download-issues)
4. [Database Issues](#database-issues)
5. [Network/Access Issues](#networkaccess-issues)
6. [WSL-Specific Issues](#wsl-specific-issues)

---

## Plugin Installation Issues

### Symptom: Plugin installation fails with "read-only filesystem" error

**Cause:** Snap Docker has filesystem confinement and can only access `/home`, `/media`, `/mnt`, `/tmp`, and `/var/snap` directories. If Project N.O.M.A.D is installed outside these locations, containers cannot mount the required volumes.

**Solution:**
The installation script automatically fixes this. If you encounter this issue manually, update the database paths:

```bash
# Get database credentials
DB_USER=$(grep "DB_USER=" /root/project-nomad/compose.yml | head -1 | cut -d'=' -f2)
DB_PASSWORD=$(grep "DB_PASSWORD=" /root/project-nomad/compose.yml | head -1 | cut -d'=' -f2)
DB_NAME=$(grep "DB_DATABASE=" /root/project-nomad/compose.yml | head -1 | cut -d'=' -f2)
NOMAD_DIR="/root/project-nomad"  # Your actual installation path

# Update service paths
docker exec nomad_mysql mysql -u"${DB_USER}" -p"${DB_PASSWORD}" "${DB_NAME}" -e \
  "UPDATE services SET container_config = REPLACE(container_config, '/opt/project-nomad', '${NOMAD_DIR}') WHERE container_config LIKE '%/opt/project-nomad%';"
```

---

### Symptom: Kiwix Server (Information Library) plugin installs but container keeps restarting

**Cause:** Corrupted or invalid ZIM files in the library directory cause kiwix-serve to crash.

**Solution:**
Add the `--skipInvalid` flag to skip corrupted ZIM files:

```bash
# Get database credentials (as above)
docker exec nomad_mysql mysql -u"${DB_USER}" -p"${DB_PASSWORD}" "${DB_NAME}" -e \
  "UPDATE services SET container_command = '*.zim --address=all --skipInvalid' WHERE service_name = 'nomad_kiwix_server' AND (container_command LIKE '%--address=all%' AND container_command NOT LIKE '%--skipInvalid%');"

# Restart the container
docker restart nomad_kiwix_server
```

---

## Container Issues

### Symptom: Container in restart loop

**Diagnosis:**
```bash
# Check container status
docker ps -a --filter "name=nomad_"

# Check container logs
docker logs <container_name> --tail 100

# Check specific container logs in WSL
wsl -d Ubuntu -- sudo /snap/bin/docker logs <container_name> --tail 100
```

**Common causes:**
1. Missing or inaccessible volume mounts (see Plugin Installation Issues)
2. Invalid command arguments
3. Corrupted data files
4. Insufficient resources (memory/disk)

---

### Symptom: Container not accessible on expected port

**Diagnosis:**
```bash
# Check if container is running
docker ps --filter "name=<container_name>"

# Check port mappings
docker port <container_name>

# Check if port is listening
netstat -tlnp | grep <port>
```

**Solution:**
Ensure the container is running and ports are correctly mapped in the database `services` table.

---

## Download Issues

### Symptom: Download stuck in "downloading" status but no progress

**Cause:** The download URL may be invalid (404 error) or the file may have been removed from the source server.

**Diagnosis:**
```bash
# Check download status in database
docker exec nomad_mysql mysql -u"${DB_USER}" -p"${DB_PASSWORD}" "${DB_NAME}" -e \
  "SELECT * FROM wikipedia_selections;"

# Verify URL is accessible
curl -sI "<url_from_database>" | head -5
```

**Solution:**
Update to a valid URL:

```bash
# Check available files on Kiwix server
curl -s 'https://download.kiwix.org/zim/wikipedia/' | grep -oE 'wikipedia_en_all_maxi_[0-9]+-[0-9]+\.zim' | sort -u

# Update database with valid URL
docker exec nomad_mysql mysql -u"${DB_USER}" -p"${DB_PASSWORD}" "${DB_NAME}" -e \
  "UPDATE wikipedia_selections SET url='<new_valid_url>', filename='<new_filename>', status='none' WHERE option_id='all-maxi';"
```

---

### Symptom: Download fails with "aborted" or "404" errors

**Cause:** Network issues, server unavailability, or file not found.

**Diagnosis:**
```bash
# Check download logs
docker logs nomad_admin 2>&1 | grep -i "download\|error" | tail -50
```

**Solution:**
1. Verify network connectivity
2. Check if the source URL is valid
3. Retry the download from the web interface

---

## Database Issues

### Symptom: Cannot connect to MySQL database

**Diagnosis:**
```bash
# Check if MySQL container is running
docker ps --filter "name=nomad_mysql"

# Check MySQL logs
docker logs nomad_mysql --tail 50
```

**Solution:**
```bash
# Restart MySQL container
docker restart nomad_mysql

# Wait for MySQL to be ready
docker exec nomad_mysql mysqladmin ping -h localhost -u root -p
```

---

### Symptom: Database credentials not working

**Solution:**
Get credentials from the compose file:

```bash
grep -E "DB_USER|DB_PASSWORD|DB_DATABASE" /root/project-nomad/compose.yml
```

---

## Network/Access Issues

### Symptom: Cannot access web interface

**Diagnosis:**
```bash
# Check if admin container is running
docker ps --filter "name=nomad_admin"

# Check port mappings
docker port nomad_admin

# Check if port is listening in WSL
wsl -d Ubuntu -- netstat -tlnp | grep 8080
```

**Solution:**
1. Ensure the container is running: `docker start nomad_admin`
2. Check firewall settings
3. Verify the correct port is being used (default: 8080)

---

### Symptom: Services not accessible from Windows

**Cause:** WSL network configuration or port forwarding issues.

**Solution:**
1. Ensure WSL is running: `wsl -l -v`
2. Check Windows firewall allows the ports
3. Access using `localhost:<port>` from Windows browser

---

## WSL-Specific Issues

### Symptom: Docker command not found in WSL

**Cause:** Docker is installed via Snap, which requires the full path.

**Solution:**
```bash
# Use full path for Docker commands
sudo /snap/bin/docker <command>

# Or add to PATH
echo 'export PATH=$PATH:/snap/bin' >> ~/.bashrc
source ~/.bashrc
```

---

### Symptom: Services don't start automatically after Windows restart

**Cause:** WSL doesn't persist services across Windows reboots by default.

**Solution:**
```bash
# Start Project N.O.M.A.D
cd /root/project-nomad
./start_nomad.sh

# Or start Docker service first
sudo systemctl start snapd.service
sudo snap start docker
```

---

### Symptom: systemd services not working

**Cause:** systemd is not enabled in WSL.

**Solution:**
```bash
# Enable systemd in WSL
cat > /etc/wsl.conf <<EOF
[boot]
systemd=true
EOF

# Restart WSL from Windows PowerShell
wsl -t Ubuntu
wsl -d Ubuntu
```

---

## Useful Commands Reference

### Container Management
```bash
# List all Project N.O.M.A.D containers
docker ps --filter "name=nomad_"

# View container logs
docker logs <container_name> --tail 100 -f

# Restart a container
docker restart <container_name>

# Execute command in container
docker exec -it <container_name> /bin/sh
```

### Database Queries
```bash
# Get database credentials
grep -E "DB_USER|DB_PASSWORD|DB_DATABASE" /root/project-nomad/compose.yml

# Connect to database
docker exec -it nomad_mysql mysql -u<user> -p<password> <database>

# Show all tables
docker exec nomad_mysql mysql -u<user> -p<password> <database> -e "SHOW TABLES;"

# Check services table
docker exec nomad_mysql mysql -u<user> -p<password> <database> -e "SELECT service_name, container_name, status FROM services;"
```

### File System
```bash
# Check ZIM files
ls -lh /root/project-nomad/storage/zim/

# Check disk usage
df -h

# Check directory sizes
du -sh /root/project-nomad/*
```

### Redis Queue
```bash
# Check download queue
docker exec nomad_redis redis-cli KEYS '*download*'

# Check queue length
docker exec nomad_redis redis-cli LLEN 'bull:downloads:wait'
```

---

## Getting Help

If the issue persists after trying the solutions above:

1. Check the main logs: `docker logs nomad_admin --tail 200`
2. Check the container logs: `docker logs <problematic_container> --tail 200`
3. Check system resources: `docker stats`
4. Open an issue on the project repository with:
   - Description of the problem
   - Steps to reproduce
   - Relevant log output
   - System information (WSL version, Docker version, etc.)
