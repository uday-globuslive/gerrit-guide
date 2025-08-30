# Docker Compose Troubleshooting for Gerrit

## 🔧 Common Issues and Solutions

### Issue 1: Plugin Installation Permission Denied

**Error:**
```
Exception in thread "main" java.io.IOException: Failed to install plugin codemirror-editor
Caused by: java.nio.file.AccessDeniedException: /var/gerrit/plugins/codemirror-editor.jar
```

**Root Cause:**
- Host-mounted plugin directory has incorrect permissions
- Plugin installation conflicts with existing files

**Solutions:**

#### Solution A: Remove plugin volume mount (Fixed in updated docker-compose.yml)
```bash
# The updated docker-compose.yml removes the plugin volume mount
# and downloads plugins inside the container
```

#### Solution B: Clean start (if still having issues)
```bash
# Stop and remove containers
sudo docker-compose down

# Remove volumes (WARNING: This deletes all Gerrit data)
sudo docker volume rm gerrit-docker_gerrit_data

# Remove any local plugin directory
sudo rm -rf ./plugins

# Start fresh
sudo docker-compose up
```

#### Solution C: Fix permissions (if you need to keep plugin directory)
```bash
# Create plugin directory with correct permissions
mkdir -p ./plugins
sudo chown -R 1000:1000 ./plugins
chmod -R 755 ./plugins

# Then use the original docker-compose.yml
```

### Issue 2: OAuth Plugin Download Fails

**Error:**
```
curl: (6) Could not resolve host: github.com
```

**Solutions:**

#### Solution A: Check network connectivity
```bash
# Test DNS resolution
sudo docker run --rm alpine nslookup github.com

# Test network connectivity
sudo docker run --rm alpine ping -c 3 github.com
```

#### Solution B: Use alternative plugin download method
```bash
# Download plugin locally first
wget https://github.com/davido/gerrit-oauth-provider/releases/download/v3.8.0/gerrit-oauth-provider.jar

# Then mount it in docker-compose.yml
volumes:
  - gerrit_data:/var/gerrit
  - ./gerrit-oauth-provider.jar:/var/gerrit/plugins/gerrit-oauth-provider.jar
```

### Issue 3: Gerrit Won't Start After Configuration

**Error:**
```
gerrit-1 exited with code 1
```

**Solutions:**

#### Solution A: Check Gerrit logs
```bash
# View container logs
sudo docker-compose logs gerrit

# Follow logs in real-time
sudo docker-compose logs -f gerrit
```

#### Solution B: Start with basic configuration
```bash
# Use minimal docker-compose.yml for testing
cat > docker-compose-minimal.yml << 'EOF'
version: '3'
services:
  gerrit:
    image: gerritcodereview/gerrit:latest
    ports:
      - "8080:8080"
      - "29418:29418"
    volumes:
      - gerrit_data:/var/gerrit
    environment:
      - CANONICAL_WEB_URL=http://10.11.53.35:8080

volumes:
  gerrit_data:
EOF

# Start minimal version
sudo docker-compose -f docker-compose-minimal.yml up
```

#### Solution C: Interactive debugging
```bash
# Run container interactively
sudo docker run -it --rm \
  -v gerrit_data:/var/gerrit \
  -p 8080:8080 -p 29418:29418 \
  gerritcodereview/gerrit:latest bash

# Inside container, manually run initialization
/var/gerrit/bin/gerrit.sh init -d /var/gerrit --batch
```

### Issue 4: Port Already in Use

**Error:**
```
Error starting userland proxy: listen tcp4 0.0.0.0:8080: bind: address already in use
```

**Solutions:**

#### Solution A: Check what's using the port
```bash
# Check what's using port 8080
sudo netstat -tulpn | grep 8080
# or
sudo ss -tulpn | grep 8080
# or
sudo lsof -i :8080
```

#### Solution B: Use different ports
```bash
# Modify docker-compose.yml to use different ports
ports:
  - "8081:8080"  # Access Gerrit at http://10.11.53.35:8081
  - "29419:29418"  # SSH on port 29419

# Update canonical URL accordingly
environment:
  - CANONICAL_WEB_URL=http://10.11.53.35:8081
```

#### Solution C: Stop conflicting services
```bash
# Find and stop the service using the port
sudo systemctl stop apache2  # if Apache is running
sudo systemctl stop nginx    # if Nginx is running
```

### Issue 5: OAuth Configuration Not Working

**Error:**
```
OAuth login fails or redirects incorrectly
```

**Solutions:**

#### Solution A: Verify OAuth application settings
```bash
# Check GitLab OAuth application settings:
# 1. Go to https://gitlab.com/-/profile/applications
# 2. Verify Redirect URI: http://10.11.53.35:8080/oauth
# 3. Check Application ID and Secret match docker-compose.yml
```

#### Solution B: Test OAuth endpoints
```bash
# Test OAuth URL manually
curl -v "https://gitlab.com/oauth/authorize?client_id=YOUR_CLIENT_ID&redirect_uri=http://10.11.53.35:8080/oauth&response_type=code&scope=read_user"
```

#### Solution C: Check Gerrit OAuth configuration
```bash
# Exec into running container
sudo docker exec -it gerrit-docker-gerrit-1 bash

# Check OAuth configuration
cat /var/gerrit/etc/gerrit.config | grep -A 10 oauth
```

## 🚀 Quick Recovery Steps

### Complete Reset (Nuclear Option)
```bash
# Stop everything
sudo docker-compose down

# Remove all volumes and data
sudo docker volume prune
sudo docker system prune -a

# Remove local files
sudo rm -rf ./plugins
rm -f docker-compose.yml

# Download fresh docker-compose.yml
# (use the updated version provided above)

# Start fresh
sudo docker-compose up
```

### Gradual Debugging
```bash
# Step 1: Start with minimal configuration
sudo docker-compose -f docker-compose-minimal.yml up

# Step 2: Access Gerrit at http://10.11.53.35:8080
# Should show basic Gerrit interface

# Step 3: If working, stop and use full configuration
sudo docker-compose -f docker-compose-minimal.yml down
sudo docker-compose up
```

### Health Check Commands
```bash
# Check if Gerrit is responding
curl -I http://10.11.53.35:8080

# Check Docker container status
sudo docker ps
sudo docker stats

# Check volume mounts
sudo docker inspect gerrit-docker-gerrit-1 | grep -A 20 Mounts

# Check container logs
sudo docker logs gerrit-docker-gerrit-1
```

## 📋 Useful Commands

### Container Management
```bash
# Stop containers
sudo docker-compose down

# Start containers
sudo docker-compose up -d

# Restart specific service
sudo docker-compose restart gerrit

# View logs
sudo docker-compose logs -f gerrit

# Execute commands in container
sudo docker exec -it gerrit-docker-gerrit-1 bash
```

### Volume Management
```bash
# List volumes
sudo docker volume ls

# Inspect volume
sudo docker volume inspect gerrit-docker_gerrit_data

# Backup volume
sudo docker run --rm -v gerrit-docker_gerrit_data:/data -v $(pwd):/backup alpine tar czf /backup/gerrit-backup.tar.gz /data

# Restore volume
sudo docker run --rm -v gerrit-docker_gerrit_data:/data -v $(pwd):/backup alpine tar xzf /backup/gerrit-backup.tar.gz -C /
```

### Network Troubleshooting
```bash
# Check container network
sudo docker network ls
sudo docker network inspect gerrit-docker_default

# Test connectivity from container
sudo docker exec gerrit-docker-gerrit-1 ping gitlab.com
sudo docker exec gerrit-docker-gerrit-1 curl -I https://github.com
```

# Docker Compose Troubleshooting Guide

## Critical Issues Fixed

### 1. "repository not found: /var/gerrit/git/All-Projects"

**Error Message:**
```
repository not found: /var/gerrit/git/All-Projects
Failed to read NoteDb schema version
```

**Root Cause:**
- Gerrit was not properly initialized before starting
- The `All-Projects` repository is created during initialization
- NoteDb schema is set up during initialization

**Solution:**
1. **Stop containers**: `docker-compose down`
2. **Remove volumes**: `docker volume rm gerrit_gerrit_data`
3. **Start with minimal version**: `docker-compose -f docker-compose-minimal.yml up`

**Prevention:**
- All compose files now check for existing initialization
- Use `--no-auto-start` flag during initialization
- Properly separate initialization from daemon startup

### 2. OAuth Plugin Download Failures

**Error Message:**
```
curl: (22) The requested URL returned error: 404 Not Found
```

**Root Cause:**
- Plugin URL changed or network issues
- Plugin not available for current Gerrit version

**Solution:**
- All compose files now include fallback to development authentication
- If OAuth plugin download fails, system automatically switches to `DEVELOPMENT_BECOME_ANY_ACCOUNT`

### 3. SSH Key Generation Issues

**Error Message:**
```
Could not load host key: /var/gerrit/etc/ssh_host_rsa_key
```

**Root Cause:**
- SSH host keys not generated during initialization
- Permission issues with key files

**Solution:**
- All compose files now automatically generate SSH keys
- Proper permissions set (600 for private keys, 644 for public keys)

## Compose File Hierarchy

### docker-compose-minimal.yml (Use First)
```yaml
# Minimal setup for testing
- Development authentication only
- Basic initialization
- No external dependencies
- Best for troubleshooting
```

### docker-compose-simple.yml (Use Second)
```yaml
# Simple development setup
- Development authentication
- SSH key generation
- Full plugin installation
- Good for learning
```

### docker-compose-gitlab.yml (Use Third)
```yaml
# GitLab.com OAuth integration
- OAuth plugin with fallback
- GitLab.com specific configuration
- Production-like testing
```

### docker-compose.yml (Use Last)
```yaml
# Full production setup
- Complete OAuth integration
- Error handling
- All features enabled
```

## Troubleshooting Steps

### Step 1: Clean Start
```bash
# Stop everything
docker-compose down

# Remove volumes
docker volume rm gerrit_gerrit_data

# Check for leftover containers
docker ps -a | grep gerrit
docker rm -f $(docker ps -a -q --filter "name=gerrit")
```

### Step 2: Test Minimal Setup
```bash
# Start minimal version
docker-compose -f docker-compose-minimal.yml up

# Check logs for errors
docker-compose -f docker-compose-minimal.yml logs -f

# Verify initialization
docker exec -it gerrit_gerrit_1 ls -la /var/gerrit/git/
```

### Step 3: Verify Initialization
```bash
# Check if All-Projects exists
docker exec -it gerrit_gerrit_1 ls -la /var/gerrit/git/All-Projects

# Check gerrit.config
docker exec -it gerrit_gerrit_1 cat /var/gerrit/etc/gerrit.config

# Check SSH keys
docker exec -it gerrit_gerrit_1 ls -la /var/gerrit/etc/ssh_host_*
```

### Step 4: Test OAuth Setup
```bash
# Stop minimal version
docker-compose -f docker-compose-minimal.yml down

# Start OAuth version
docker-compose -f docker-compose-gitlab.yml up

# Check OAuth plugin
docker exec -it gerrit_gerrit_1 ls -la /var/gerrit/plugins/
```

## Common Error Patterns

### Pattern 1: Initialization Errors
```
Exception in thread "main" java.lang.RuntimeException: Cannot initialize site at /var/gerrit
```
**Solution**: Remove volumes and start with minimal version

### Pattern 2: Plugin Errors
```
com.google.gerrit.server.plugins.PluginInstallException: Plugin installation failed
```
**Solution**: Check plugin download, use fallback authentication

### Pattern 3: Database Errors
```
org.eclipse.jgit.errors.RepositoryNotFoundException: repository not found
```
**Solution**: Verify initialization completed, check All-Projects repository

### Pattern 4: OAuth Errors
```
OAuth authentication failed: Invalid client credentials
```
**Solution**: Verify client ID/secret, check GitLab.com application settings

## Recovery Procedures

### Full Reset
```bash
# Stop all containers
docker-compose down

# Remove all volumes
docker volume prune -f

# Remove all containers
docker container prune -f

# Start fresh
docker-compose -f docker-compose-minimal.yml up
```

### Partial Reset (Keep Data)
```bash
# Stop containers
docker-compose down

# Start with different authentication
docker-compose -f docker-compose-simple.yml up
```

### Debug Mode
```bash
# Start with debug output
docker-compose -f docker-compose-minimal.yml up --verbose

# Check container logs
docker logs gerrit_gerrit_1 -f

# Enter container for debugging
docker exec -it gerrit_gerrit_1 /bin/bash
```

## Verification Checklist

- [ ] All-Projects repository exists: `/var/gerrit/git/All-Projects`
- [ ] Configuration file exists: `/var/gerrit/etc/gerrit.config`
- [ ] SSH keys generated: `/var/gerrit/etc/ssh_host_*`
- [ ] Web interface accessible: `http://10.11.53.35:8080`
- [ ] Authentication working: Can login and create projects
- [ ] OAuth plugin loaded: `/var/gerrit/plugins/gerrit-oauth-provider.jar`

## Getting Help

1. **Check logs**: `docker-compose logs -f`
2. **Verify initialization**: Follow verification checklist
3. **Test minimal setup**: Always start with `docker-compose-minimal.yml`
4. **Check network**: Ensure port 8080 and 29418 are accessible
5. **Review configuration**: Check `/var/gerrit/etc/gerrit.config`

## Next Steps

After successful troubleshooting:
1. Follow `gitlab-integration-test.md` for GitLab.com integration
2. Use `docker-compose-quickstart.md` for deployment options
3. Check `README.md` for complete learning resources
