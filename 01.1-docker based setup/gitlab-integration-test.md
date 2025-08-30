# GitLab.com Integration Test Guide

## 🚀 Quick Start

### 1. Start Gerrit with GitLab.com OAuth
```bash
# Stop any existing containers
sudo docker-compose down

# Start with GitLab integration
sudo docker-compose -f docker-compose-gitlab.yml up

# Should see:
# ✓ Initializing Gerrit...
# ✓ Generating SSH host keys...
# ✓ OAuth plugin downloaded successfully
# ✓ Gerrit with GitLab.com OAuth is starting
```

### 2. Access Gerrit
```bash
# Open browser and go to:
http://10.11.53.35:8080

# Click "Sign In" 
# You'll be redirected to GitLab.com
# Login with your GitLab.com credentials
# Authorize the application
# You'll be redirected back to Gerrit
```

## 🔧 Test GitLab.com Repository Integration

### 1. Prepare Your GitLab.com Repository
```bash
# Clone your GitLab.com repository
git clone https://gitlab.com/your-username/your-repo.git
cd your-repo

# Add Gerrit as remote
git remote add gerrit ssh://your-gitlab-username@10.11.53.35:29418/your-username/your-repo
```

### 2. Install Gerrit Commit Hook
```bash
# Download commit-msg hook
curl -Lo .git/hooks/commit-msg http://10.11.53.35:8080/tools/hooks/commit-msg
chmod +x .git/hooks/commit-msg
```

### 3. Create a Test Change
```bash
# Make a test change
echo "Test Gerrit integration with GitLab.com" > test-gerrit.txt
git add test-gerrit.txt
git commit -m "Test Gerrit integration

This change tests the integration between Gerrit and GitLab.com
using OAuth authentication.
"
```

### 4. Submit for Review
```bash
# Push to Gerrit for review
git push gerrit HEAD:refs/for/main

# You should see output like:
# remote: Processing changes: new: 1, done    
# remote: 
# remote: SUCCESS
# remote: 
# remote:   http://10.11.53.35:8080/c/your-repo/+/1 Test Gerrit integration [NEW]
# remote: 
# To ssh://your-gitlab-username@10.11.53.35:29418/your-username/your-repo
#  * [new branch]      HEAD -> refs/for/main
```

### 5. Review the Change
```bash
# Go to the URL shown in the push output
# Example: http://10.11.53.35:8080/c/your-repo/+/1

# Or browse to: http://10.11.53.35:8080
# Click "Changes" to see your submitted change
```

## 📋 Expected Workflow

### 1. Authentication Flow
```
Browser → Gerrit → GitLab.com → OAuth → Gerrit
```

### 2. Development Flow
```
Local Git → Gerrit Review → Merge → (Optional: Back to GitLab.com)
```

## 🔍 Troubleshooting

### If OAuth fails:
```bash
# Check container logs
sudo docker-compose -f docker-compose-gitlab.yml logs

# If OAuth plugin download fails, container will automatically 
# fall back to development authentication
```

### If SSH authentication fails:
```bash
# Generate SSH key if you don't have one
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"

# Add public key to Gerrit
# Go to http://10.11.53.35:8080
# Settings → SSH Keys → Add Key
# Paste content of ~/.ssh/id_rsa.pub
```

### Test SSH connection:
```bash
# Test SSH connection to Gerrit
ssh -p 29418 your-gitlab-username@10.11.53.35

# Should show:
# Welcome to Gerrit Code Review
```

## 🎯 OAuth Application Settings

Make sure your GitLab.com OAuth application has these settings:

### In GitLab.com (https://gitlab.com/-/profile/applications):
- **Name:** Gerrit Code Review
- **Redirect URI:** `http://10.11.53.35:8080/oauth`
- **Scopes:** `read_user`, `read_api`, `read_repository`
- **Application ID:** `f795a4f270fc5baccd1a938ccac1ac3f41923b1a42c9cc48ed5a5c0634a39683`
- **Secret:** `gloas-d6e4293946290322e85b6624fee134de419f5332e74296b35f7ebc8ed5082891`

## 🚀 Complete Test Scenario

### Full Integration Test:
```bash
# 1. Start Gerrit
sudo docker-compose -f docker-compose-gitlab.yml up -d

# 2. Login via GitLab.com
# Open http://10.11.53.35:8080 → Sign In → GitLab OAuth

# 3. Clone and setup repository
git clone https://gitlab.com/your-username/your-repo.git
cd your-repo
git remote add gerrit ssh://your-gitlab-username@10.11.53.35:29418/your-username/your-repo
curl -Lo .git/hooks/commit-msg http://10.11.53.35:8080/tools/hooks/commit-msg
chmod +x .git/hooks/commit-msg

# 4. Make change and submit
echo "GitLab.com + Gerrit integration test" > gitlab-test.txt
git add gitlab-test.txt
git commit -m "Add GitLab integration test"
git push gerrit HEAD:refs/for/main

# 5. Review and merge in Gerrit UI
# Go to http://10.11.53.35:8080
# Find your change → Review → Vote +2 → Submit

# 6. Verify in GitLab.com (optional)
# Check your GitLab.com repository for the merged change
```

## 📚 Key Points

1. **OAuth Authentication** - Uses your GitLab.com credentials
2. **Repository Integration** - Works with any GitLab.com repository
3. **Code Review Flow** - Standard Gerrit review process
4. **Automatic Fallback** - Falls back to development auth if OAuth fails
5. **SSH Access** - Full SSH support for Git operations

The setup is now ready for testing GitLab.com integration!
