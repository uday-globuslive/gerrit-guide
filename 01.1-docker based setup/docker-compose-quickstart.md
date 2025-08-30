# Gerrit Docker Compose Quick Start Guide

## 🚀 Available Configurations

### 1. Simple Configuration (Recommended for First Time)
**File:** `docker-compose-simple.yml`
- ✅ **Works immediately** - No external dependencies
- ✅ **Development authentication** - Just click "Become" and enter username
- ✅ **All core features** - Full Gerrit functionality
- ❌ **No OAuth** - Manual login only

```bash
# Start simple version
sudo docker-compose -f docker-compose-simple.yml up

# Access at: http://10.11.53.35:8080
# Login: Click "Sign In" → "Become" → Enter any username
```

### 2. Robust Configuration (Best for Production Testing)
**File:** `docker-compose-robust.yml`
- ✅ **Automatic fallback** - OAuth if available, development auth if not
- ✅ **Multiple download methods** - curl, wget, alternative URLs
- ✅ **Detailed logging** - See exactly what's happening
- ✅ **Self-healing** - Handles network issues gracefully

```bash
# Start robust version
sudo docker-compose -f docker-compose-robust.yml up

# Will try OAuth first, fallback to development auth if needed
```

### 3. Alternative Configuration (OAuth Focus)
**File:** `docker-compose-alternative.yml`
- ✅ **OAuth integration** - GitLab.com authentication
- ✅ **Conditional initialization** - Smart restart handling
- ⚠️ **Network dependent** - Requires internet for OAuth plugin
- ⚠️ **May fail** - If OAuth plugin download fails

```bash
# Start alternative version (fixed)
sudo docker-compose -f docker-compose-alternative.yml up

# Requires successful OAuth plugin download
```

### 4. Minimal Configuration (Recommended for Testing)
**File:** `docker-compose-minimal.yml`
- ✅ **Quick testing** - Starts Gerrit with minimal features
- ✅ **Development authentication** - Just click "Become" and enter username
- ❌ **No persistent data** - Data is stored in temporary volumes
- ❌ **Limited features** - Basic Gerrit functionality only

```bash
# Start minimal version
sudo docker-compose -f docker-compose-minimal.yml up

# Access at: http://10.11.53.35:8080
# Login: Click "Sign In" → "Become" → Enter any username
```

## 🔧 Troubleshooting Your Current Issue

### Problem Analysis:
1. **OAuth Plugin Download Failed** - Only 9 bytes downloaded (HTML redirect page)
2. **Missing SSH Keys** - Required for SSH daemon to start
3. **Configuration Before Initialization** - Tried to configure before proper init

### Quick Fix:
```bash
# Stop current containers
sudo docker-compose down

# Clean up (optional - removes all data)
sudo docker volume rm gerrit-docker_gerrit_data

# Start with simple configuration first
sudo docker-compose -f docker-compose-simple.yml up

# Should work immediately!
```

## 🎯 Recommended Workflow

### Step 1: Test Basic Functionality
```bash
# Use simple configuration
sudo docker-compose -f docker-compose-simple.yml up -d

# Test access
curl -I http://10.11.53.35:8080
# Should return HTTP 200

# Test in browser
# Go to http://10.11.53.35:8080
# Click "Sign In" → "Become" → Enter "testuser"
```

### Step 2: Test with OAuth (if needed)
```bash
# Stop simple version
sudo docker-compose -f docker-compose-simple.yml down

# Try robust version (handles OAuth failures)
sudo docker-compose -f docker-compose-robust.yml up

# Check logs for OAuth status
sudo docker-compose -f docker-compose-robust.yml logs
```

### Step 3: Production Configuration
```bash
# Once working, use the robust version for production
# It automatically handles OAuth or falls back gracefully
```

## 📋 Expected Outputs

### Simple Configuration Success:
```
gerrit-1  | Initializing Gerrit...
gerrit-1  | Generating SSH host keys...
gerrit-1  | ✓ SSH host keys generated
gerrit-1  | Starting Gerrit with development authentication...
gerrit-1  | Access: http://10.11.53.35:8080
gerrit-1  | [INFO] Gerrit Code Review ready
```

### Robust Configuration Success (with OAuth):
```
gerrit-1  | === Setting up OAuth ===
gerrit-1  | Downloading OAuth plugin...
gerrit-1  | ✓ OAuth plugin downloaded successfully (1234567 bytes)
gerrit-1  | ✓ OAuth configured for GitLab.com
gerrit-1  | === Configuration Summary ===
gerrit-1  | Auth Type: OAUTH
gerrit-1  | === Starting Gerrit ===
```

### Robust Configuration Fallback (no OAuth):
```
gerrit-1  | === Setting up OAuth ===
gerrit-1  | ⚠ OAuth plugin download failed, falling back to development auth
gerrit-1  | ✓ Development authentication enabled
gerrit-1  | === Configuration Summary ===
gerrit-1  | Auth Type: DEVELOPMENT_BECOME_ANY_ACCOUNT
```

## 🔄 Quick Commands

### Start/Stop Different Versions:
```bash
# Simple version
sudo docker-compose -f docker-compose-simple.yml up -d
sudo docker-compose -f docker-compose-simple.yml down

# Robust version
sudo docker-compose -f docker-compose-robust.yml up -d
sudo docker-compose -f docker-compose-robust.yml down

# Alternative version
sudo docker-compose -f docker-compose-alternative.yml up -d
sudo docker-compose -f docker-compose-alternative.yml down

# Minimal version
sudo docker-compose -f docker-compose-minimal.yml up -d
sudo docker-compose -f docker-compose-minimal.yml down
```

### Check Status:
```bash
# View logs
sudo docker-compose logs -f

# Check if Gerrit is responding
curl -I http://10.11.53.35:8080

# Check container status
sudo docker ps
```

### Clean Reset:
```bash
# Stop all
sudo docker-compose down

# Remove all data (fresh start)
sudo docker volume rm gerrit-docker_gerrit_data

# Start fresh
sudo docker-compose -f docker-compose-simple.yml up
```

## 🎯 Recommendation

**Start with `docker-compose-simple.yml`** - it will work immediately and let you test Gerrit functionality. Once you confirm everything works, you can switch to `docker-compose-robust.yml` for OAuth integration.

The robust version is production-ready and handles failures gracefully!

## Understanding the Files

### docker-compose-minimal.yml
- **Purpose**: Minimal setup for testing
- **Authentication**: Development (any username)
- **Features**: Basic Gerrit with no external dependencies
- **Best for**: First-time testing, troubleshooting

### docker-compose-simple.yml
- **Purpose**: Simple development setup
- **Authentication**: Development (any username)
- **Features**: SSH keys, full plugin installation
- **Best for**: Local development, learning

### docker-compose-gitlab.yml
- **Purpose**: GitLab.com OAuth integration
- **Authentication**: GitLab.com OAuth
- **Features**: OAuth plugin, fallback to development auth
- **Best for**: Production-like testing with GitLab.com

### docker-compose.yml
- **Purpose**: Main setup with OAuth
- **Authentication**: GitLab.com OAuth
- **Features**: Full OAuth integration, error handling
- **Best for**: Production deployment

## Common Issues and Solutions

### Issue: "repository not found: /var/gerrit/git/All-Projects"
**Cause**: Gerrit not properly initialized before starting
**Solution**: 
1. Stop containers: `docker-compose down`
2. Remove volumes: `docker volume rm gerrit_gerrit_data`
3. Use minimal version first: `docker-compose -f docker-compose-minimal.yml up`

### Issue: OAuth plugin fails to download
**Cause**: Network issues or plugin URL problems
**Solution**: The compose files automatically fall back to development authentication

### Issue: Permission errors
**Cause**: Docker volume permissions
**Solution**: 
```bash
# Reset volumes
docker-compose down
docker volume rm gerrit_gerrit_data
docker-compose up
```

## Initialization Process

All Docker Compose files now properly handle Gerrit initialization:

1. **Check if initialized**: Look for `/var/gerrit/etc/gerrit.config`
2. **Initialize if needed**: Run `gerrit.sh init` with `--no-auto-start`
3. **Generate SSH keys**: Create host keys if missing
4. **Download plugins**: Get OAuth plugin (with fallback)
5. **Configure settings**: Set canonical URL and authentication
6. **Start Gerrit**: Run `gerrit.sh run` in foreground

## Testing Steps

1. **Start with minimal**:
   ```bash
   docker-compose -f docker-compose-minimal.yml up
   ```

2. **Verify basic functionality**:
   - Visit http://10.11.53.35:8080
   - Login with any username
   - Create a test project

3. **Test OAuth integration**:
   ```bash
   docker-compose down
   docker-compose -f docker-compose-gitlab.yml up
   ```

4. **Verify GitLab.com OAuth**:
   - Visit http://10.11.53.35:8080
   - Login with GitLab.com credentials
   - Import GitLab.com repository

## Volume Management

### Reset Everything
```bash
# Stop all containers
docker-compose down

# Remove all volumes
docker volume rm gerrit_gerrit_data

# Start fresh
docker-compose -f docker-compose-minimal.yml up
```

### Backup Data
```bash
# Create backup
docker run --rm -v gerrit_gerrit_data:/data -v $(pwd):/backup ubuntu tar czf /backup/gerrit-backup.tar.gz /data

# Restore backup
docker run --rm -v gerrit_gerrit_data:/data -v $(pwd):/backup ubuntu tar xzf /backup/gerrit-backup.tar.gz -C /
```

## Next Steps

1. **Test basic functionality** with `docker-compose-minimal.yml`
2. **Enable OAuth** with `docker-compose-gitlab.yml`
3. **Follow integration guide** in `gitlab-integration-test.md`
4. **Configure production** settings as needed

See `docker-compose-troubleshooting.md` for detailed error solutions.
