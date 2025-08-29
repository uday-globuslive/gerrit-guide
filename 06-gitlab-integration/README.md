# Chapter 6: Integration with GitLab

## Overview

Integrating Gerrit with GitLab creates a powerful hybrid development workflow. You get GitLab's excellent project management, CI/CD capabilities, and user interface, combined with Gerrit's superior code review process. This chapter will guide you through various integration strategies and their practical implementation.

## Why Integrate Gerrit with GitLab?

### Benefits of Integration

#### From GitLab
- **Rich UI/UX**: Modern, intuitive interface
- **Issue Tracking**: Comprehensive bug and feature tracking
- **CI/CD Pipelines**: Powerful automation and deployment
- **Wiki & Documentation**: Built-in documentation tools
- **Project Management**: Milestones, boards, and planning tools

#### From Gerrit
- **Superior Code Review**: More detailed review workflow
- **Change-based Reviews**: Better than branch-based reviews
- **Granular Permissions**: Fine-grained access control
- **Review History**: Complete audit trail
- **Pre-submit Validation**: Prevent bad code from entering

### Integration Architectures

## Architecture 1: GitLab as Primary with Gerrit Review

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│    Developer    │    │     Gerrit      │    │     GitLab      │
│                 │    │  Code Review    │    │   Repository    │
│  Local Machine  │───►│     Server      │───►│   & CI/CD       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
  Git Operations          Review Process        Final Repository
  (commit, push)         (approve, merge)       (master branch)
```

## Architecture 2: Bidirectional Sync

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│     GitLab      │◄──►│ Sync Service    │◄──►│     Gerrit      │
│   Repository    │    │   (Webhook/     │    │  Code Review    │
│    & CI/CD      │    │    Polling)     │    │     Server      │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
    GitLab Features      Synchronization         Gerrit Reviews
   (Issues, CI/CD)      (Changes, Commits)       (Review Process)
```

## Part 1: Setting Up GitLab

### 1.1 Install GitLab (Local Development)

For this tutorial, we'll set up a local GitLab instance:

#### Using Docker (Recommended)

```powershell
# Create directory for GitLab data
mkdir C:\gitlab
cd C:\gitlab

# Create docker-compose.yml
@"
version: '3.6'
services:
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: always
    hostname: 'localhost'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://localhost:8081'
        gitlab_rails['gitlab_shell_ssh_port'] = 2224
    ports:
      - '8081:8081'
      - '2224:22'
    volumes:
      - './config:/etc/gitlab'
      - './logs:/var/log/gitlab'
      - './data:/var/opt/gitlab'
    shm_size: '256m'
"@ | Out-File -FilePath "docker-compose.yml" -Encoding UTF8

# Start GitLab
docker-compose up -d

# Wait for GitLab to start (this takes several minutes)
Write-Host "Waiting for GitLab to start... This may take 5-10 minutes"
```

#### Access GitLab

1. **Wait for startup**: GitLab takes 5-10 minutes to fully start
2. **Open browser**: Go to http://localhost:8081
3. **Get initial password**:
   ```powershell
   # Get the initial root password
   docker exec -it gitlab grep 'Password:' /etc/gitlab/initial_root_password
   ```
4. **Login**: Use username `root` and the password from step 3

### 1.2 Create a Test Project in GitLab

1. **Create New Project**:
   - Click "New Project"
   - Choose "Create blank project"
   - Project name: `gerrit-gitlab-integration`
   - Initialize with README: ✅

2. **Configure Git locally**:
   ```powershell
   # Clone the GitLab project
   git clone http://localhost:8081/root/gerrit-gitlab-integration.git
   cd gerrit-gitlab-integration
   
   # Configure git
   git config user.name "Your Name"
   git config user.email "your.email@example.com"
   ```

## Part 2: Basic Integration - Mirror Setup

### 2.1 Configure Gerrit Project

First, let's create a corresponding project in Gerrit:

1. **Access Gerrit** (http://localhost:8080)
2. **Create New Project**:
   - Browse → Admin → Projects
   - Click "Create New Project"
   - Project Name: `gerrit-gitlab-integration`
   - ✅ Create initial empty commit
   - Click "Create Project"

### 2.2 Set Up Repository Mirroring

#### Configure Gerrit to Push to GitLab

Add GitLab as a replication target in Gerrit:

```powershell
# Access Gerrit server (if running locally)
cd C:\gerrit\gerrit_site

# Create replication.config
@"
[remote "gitlab"]
    url = http://root:your-gitlab-token@localhost:8081/root/gerrit-gitlab-integration.git
    push = +refs/heads/*:refs/heads/*
    push = +refs/tags/*:refs/tags/*
    projects = gerrit-gitlab-integration
    replicationDelay = 5
"@ | Out-File -FilePath "etc\replication.config" -Encoding UTF8
```

**Note**: Replace `your-gitlab-token` with a GitLab access token:
1. GitLab → Settings → Access Tokens
2. Create token with `api` and `write_repository` scopes

#### Install Replication Plugin

```powershell
# Download replication plugin
wget https://gerrit-ci.gerritforge.com/job/plugin-replication-stable-3.8/lastSuccessfulBuild/artifact/replication.jar

# Move to plugins directory
mv replication.jar plugins/

# Restart Gerrit
./bin/gerrit.sh restart
```

### 2.3 Test Basic Mirroring

```powershell
# Clone Gerrit project
git clone http://localhost:8080/gerrit-gitlab-integration
cd gerrit-gitlab-integration

# Install commit-msg hook
curl -Lo .git/hooks/commit-msg http://localhost:8080/tools/hooks/commit-msg
chmod +x .git/hooks/commit-msg  # On Linux/Mac

# Make a change
echo "# Integration Test" > test-integration.md
git add test-integration.md
git commit -m "Add integration test file

This file tests the Gerrit-GitLab integration setup.

Change-Id: I1111111111111111111111111111111111111111"

# Push to Gerrit
git push origin HEAD:refs/for/master

# Approve and submit the change in Gerrit web interface
# Then check if it appears in GitLab
```

## Part 3: Advanced Integration - Webhook-Based Sync

### 3.1 Create Sync Service

Let's create a simple Python service to handle bidirectional sync:

```python
# sync-service.py
import requests
import json
from flask import Flask, request, jsonify
import hmac
import hashlib
import os

app = Flask(__name__)

# Configuration
GERRIT_URL = "http://localhost:8080"
GERRIT_USER = "admin"
GERRIT_PASSWORD = "your-password"

GITLAB_URL = "http://localhost:8081"
GITLAB_TOKEN = "your-gitlab-token"

class GitLabGerritSync:
    def __init__(self):
        self.gerrit_auth = (GERRIT_USER, GERRIT_PASSWORD)
        self.gitlab_headers = {
            'Authorization': f'Bearer {GITLAB_TOKEN}',
            'Content-Type': 'application/json'
        }
    
    def create_gitlab_mr_from_gerrit_change(self, change_data):
        """Create GitLab Merge Request from Gerrit Change"""
        
        change_id = change_data['id']
        project = change_data['project']
        branch = change_data['branch']
        subject = change_data['subject']
        
        # Create branch in GitLab
        branch_name = f"gerrit-change-{change_id}"
        
        # Get change details from Gerrit
        gerrit_response = requests.get(
            f"{GERRIT_URL}/a/changes/{change_id}/revisions/current/files",
            auth=self.gerrit_auth
        )
        
        if gerrit_response.status_code == 200:
            # Create merge request in GitLab
            mr_data = {
                "source_branch": branch_name,
                "target_branch": branch,
                "title": f"[Gerrit] {subject}",
                "description": f"Automatically created from Gerrit change {change_id}\n\nReview in Gerrit: {GERRIT_URL}/c/{change_id}"
            }
            
            gitlab_response = requests.post(
                f"{GITLAB_URL}/api/v4/projects/1/merge_requests",
                headers=self.gitlab_headers,
                json=mr_data
            )
            
            return gitlab_response.status_code == 201
        
        return False
    
    def sync_gerrit_approval_to_gitlab(self, change_data, approval_data):
        """Sync Gerrit approval to GitLab MR"""
        
        # Find corresponding GitLab MR
        change_id = change_data['id']
        
        # Search for MR with Gerrit change ID in description
        gitlab_response = requests.get(
            f"{GITLAB_URL}/api/v4/projects/1/merge_requests",
            headers=self.gitlab_headers,
            params={'search': f"gerrit-change-{change_id}"}
        )
        
        if gitlab_response.status_code == 200:
            mrs = gitlab_response.json()
            if mrs:
                mr_iid = mrs[0]['iid']
                
                # Add comment about Gerrit approval
                comment_data = {
                    "body": f"Gerrit Review Update:\n{approval_data['message']}\nScore: {approval_data['value']}"
                }
                
                requests.post(
                    f"{GITLAB_URL}/api/v4/projects/1/merge_requests/{mr_iid}/notes",
                    headers=self.gitlab_headers,
                    json=comment_data
                )

sync_service = GitLabGerritSync()

@app.route('/gerrit-webhook', methods=['POST'])
def handle_gerrit_webhook():
    """Handle webhooks from Gerrit"""
    
    data = request.json
    event_type = data.get('type')
    
    if event_type == 'change-created':
        # Create corresponding GitLab MR
        sync_service.create_gitlab_mr_from_gerrit_change(data['change'])
        
    elif event_type == 'comment-added':
        # Sync approval to GitLab
        sync_service.sync_gerrit_approval_to_gitlab(
            data['change'], 
            data['approval']
        )
    
    return jsonify({'status': 'processed'})

@app.route('/gitlab-webhook', methods=['POST'])
def handle_gitlab_webhook():
    """Handle webhooks from GitLab"""
    
    data = request.json
    event_type = request.headers.get('X-Gitlab-Event')
    
    if event_type == 'Merge Request Hook':
        # Handle GitLab MR events
        mr_data = data['object_attributes']
        
        if mr_data['action'] == 'approved':
            # Could sync approval back to Gerrit
            pass
    
    return jsonify({'status': 'processed'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### 3.2 Configure Webhooks

#### Gerrit Webhook Configuration

Add to `gerrit.config`:

```ini
[plugin "webhooks"]
    remote = http://localhost:5000/gerrit-webhook
    event = change-created
    event = comment-added
    event = change-merged
```

#### GitLab Webhook Configuration

1. **Go to GitLab project** → Settings → Webhooks
2. **Add webhook**:
   - URL: `http://localhost:5000/gitlab-webhook`
   - Trigger: Merge request events
   - Enable SSL verification: No (for local development)

### 3.3 Run the Sync Service

```powershell
# Install dependencies
pip install flask requests

# Run the sync service
python sync-service.py
```

## Part 4: CI/CD Integration

### 4.1 GitLab CI Pipeline for Gerrit Changes

Create `.gitlab-ci.yml` in your project:

```yaml
# .gitlab-ci.yml
stages:
  - test
  - verify
  - deploy

variables:
  GIT_DEPTH: 0

# Test stage - runs on all changes
test:
  stage: test
  script:
    - echo "Running tests..."
    - python -m pytest tests/ || echo "No tests found"
    - echo "Linting code..."
    - flake8 . || echo "No Python files found"
  only:
    - merge_requests
    - master

# Verify stage - posts results back to Gerrit
verify_gerrit:
  stage: verify
  script:
    - |
      # Post verification result to Gerrit
      if [ "$CI_JOB_STATUS" == "success" ]; then
        VERIFY_SCORE="+1"
        MESSAGE="Build and tests passed in GitLab CI"
      else
        VERIFY_SCORE="-1"
        MESSAGE="Build or tests failed in GitLab CI"
      fi
      
      # Post to Gerrit (if change ID is available)
      if [ ! -z "$GERRIT_CHANGE_ID" ]; then
        curl -X POST \
          -u "$GERRIT_USER:$GERRIT_PASSWORD" \
          -H "Content-Type: application/json" \
          -d "{\"message\":\"$MESSAGE\",\"labels\":{\"Verified\":$VERIFY_SCORE}}" \
          "$GERRIT_URL/a/changes/$GERRIT_CHANGE_ID/revisions/current/review"
      fi
  only:
    - merge_requests
  when: always

# Deploy stage - only on master
deploy:
  stage: deploy
  script:
    - echo "Deploying to production..."
    - echo "Application deployed successfully"
  only:
    - master
  when: manual
```

### 4.2 Enhanced Integration Script

Create a more sophisticated integration that handles both directions:

```python
# enhanced-sync.py
import asyncio
import aiohttp
import json
from datetime import datetime

class EnhancedGitLabGerritSync:
    def __init__(self):
        self.gerrit_url = "http://localhost:8080"
        self.gitlab_url = "http://localhost:8081"
        self.session = None
    
    async def setup_session(self):
        self.session = aiohttp.ClientSession()
    
    async def sync_change_status(self, change_id, status):
        """Sync change status between systems"""
        
        # Get Gerrit change details
        async with self.session.get(
            f"{self.gerrit_url}/a/changes/{change_id}"
        ) as response:
            if response.status == 200:
                change_data = await response.json()
                
                # Find corresponding GitLab MR
                await self.update_gitlab_mr_status(change_data, status)
    
    async def update_gitlab_mr_status(self, change_data, status):
        """Update GitLab MR based on Gerrit change status"""
        
        project_id = 1  # Your GitLab project ID
        
        # Search for corresponding MR
        search_params = {
            'search': f"gerrit-{change_data['_number']}"
        }
        
        async with self.session.get(
            f"{self.gitlab_url}/api/v4/projects/{project_id}/merge_requests",
            params=search_params,
            headers={'Authorization': f'Bearer {GITLAB_TOKEN}'}
        ) as response:
            
            if response.status == 200:
                mrs = await response.json()
                
                for mr in mrs:
                    if status == 'MERGED':
                        # Auto-merge in GitLab
                        await self.merge_gitlab_mr(project_id, mr['iid'])
                    elif status == 'ABANDONED':
                        # Close in GitLab
                        await self.close_gitlab_mr(project_id, mr['iid'])
    
    async def merge_gitlab_mr(self, project_id, mr_iid):
        """Merge GitLab MR"""
        
        merge_data = {
            'merge_commit_message': 'Merged via Gerrit approval'
        }
        
        async with self.session.put(
            f"{self.gitlab_url}/api/v4/projects/{project_id}/merge_requests/{mr_iid}/merge",
            json=merge_data,
            headers={'Authorization': f'Bearer {GITLAB_TOKEN}'}
        ) as response:
            
            return response.status == 200
    
    async def close_gitlab_mr(self, project_id, mr_iid):
        """Close GitLab MR"""
        
        close_data = {
            'state_event': 'close'
        }
        
        async with self.session.put(
            f"{self.gitlab_url}/api/v4/projects/{project_id}/merge_requests/{mr_iid}",
            json=close_data,
            headers={'Authorization': f'Bearer {GITLAB_TOKEN}'}
        ) as response:
            
            return response.status == 200
    
    async def sync_comments(self, change_id, comment_data):
        """Sync comments between systems"""
        
        # Implementation for comment synchronization
        pass
    
    async def cleanup(self):
        if self.session:
            await self.session.close()

# Usage
async def main():
    sync = EnhancedGitLabGerritSync()
    await sync.setup_session()
    
    try:
        # Example: sync change status
        await sync.sync_change_status("12345", "MERGED")
    finally:
        await sync.cleanup()

if __name__ == "__main__":
    asyncio.run(main())
```

## Part 5: Workflow Examples

### 5.1 Development Workflow with Integration

#### Daily Developer Workflow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GL as GitLab
    participant G as Gerrit
    participant CI as CI/CD
    
    Dev->>GL: 1. Create issue/feature request
    Dev->>Dev: 2. Implement feature locally
    Dev->>G: 3. Push change for review
    G->>GL: 4. Auto-create MR (webhook)
    G->>CI: 5. Trigger CI build
    CI->>G: 6. Post verification result
    Dev->>G: 7. Address review feedback
    G->>GL: 8. Update MR status
    G->>GL: 9. Merge to master (after approval)
    GL->>CI: 10. Trigger deployment pipeline
```

#### Step-by-Step Process

1. **Create Issue in GitLab**:
   ```
   Title: Add user authentication feature
   Description: Implement JWT-based authentication...
   Assignee: developer@company.com
   Labels: enhancement, backend
   ```

2. **Implement Feature Locally**:
   ```powershell
   git checkout master
   git pull origin master
   git checkout -b feature/user-auth
   
   # Implement feature
   # Write tests
   # Update documentation
   ```

3. **Submit for Review in Gerrit**:
   ```powershell
   git add .
   git commit -m "Add JWT-based user authentication

   Implements secure user authentication using JWT tokens.
   Includes comprehensive test coverage and documentation.

   Closes: #123
   Change-Id: I5555555555555555555555555555555555555555"
   
   git push origin HEAD:refs/for/master
   ```

4. **Review Process**:
   - Gerrit creates change for review
   - Webhook creates corresponding GitLab MR
   - CI runs tests and posts results to Gerrit
   - Team reviews code in Gerrit
   - GitLab MR shows status updates

5. **Merge and Deploy**:
   - Gerrit change gets approved
   - Automatic merge to master
   - GitLab CI triggers deployment

### 5.2 Hotfix Workflow

For urgent production fixes:

```powershell
# Create hotfix from production branch
git checkout production
git pull origin production
git checkout -b hotfix/critical-bug

# Implement fix
git add .
git commit -m "Fix critical security vulnerability

Addresses SQL injection vulnerability in user login.
Requires immediate deployment to production.

Priority: Critical
Change-Id: I6666666666666666666666666666666666666666"

# Push to Gerrit with high priority
git push origin HEAD:refs/for/production%topic=critical-hotfix

# Fast-track review process
# Emergency approval and merge
# Automatic deployment via GitLab CI
```

## Part 6: Monitoring and Troubleshooting

### 6.1 Monitoring Integration Health

Create a health check service:

```python
# health-check.py
import requests
import time
import logging

class IntegrationHealthCheck:
    def __init__(self):
        self.gerrit_url = "http://localhost:8080"
        self.gitlab_url = "http://localhost:8081"
        self.sync_service_url = "http://localhost:5000"
    
    def check_gerrit_health(self):
        try:
            response = requests.get(f"{self.gerrit_url}/config/server/info")
            return response.status_code == 200
        except:
            return False
    
    def check_gitlab_health(self):
        try:
            response = requests.get(f"{self.gitlab_url}/api/v4/version")
            return response.status_code == 200
        except:
            return False
    
    def check_sync_service_health(self):
        try:
            response = requests.get(f"{self.sync_service_url}/health")
            return response.status_code == 200
        except:
            return False
    
    def check_integration_sync(self):
        """Test if integration is working by checking recent sync"""
        # Implementation to verify recent sync operations
        return True
    
    def run_health_check(self):
        checks = {
            'gerrit': self.check_gerrit_health(),
            'gitlab': self.check_gitlab_health(),
            'sync_service': self.check_sync_service_health(),
            'integration': self.check_integration_sync()
        }
        
        all_healthy = all(checks.values())
        
        status = {
            'overall_health': 'healthy' if all_healthy else 'unhealthy',
            'timestamp': time.time(),
            'services': checks
        }
        
        return status

# Run health check every 5 minutes
if __name__ == "__main__":
    health_checker = IntegrationHealthCheck()
    
    while True:
        status = health_checker.run_health_check()
        
        if status['overall_health'] == 'unhealthy':
            logging.error(f"Integration health check failed: {status}")
            # Send alerts
        else:
            logging.info("Integration health check passed")
        
        time.sleep(300)  # Wait 5 minutes
```

### 6.2 Common Integration Issues

#### Issue 1: Sync Delays
**Symptoms**: Changes appear in Gerrit but not in GitLab
**Solutions**:
- Check webhook configuration
- Verify sync service is running
- Check network connectivity
- Review sync service logs

#### Issue 2: Authentication Failures
**Symptoms**: 401/403 errors in sync service
**Solutions**:
- Verify API tokens are valid
- Check token permissions
- Ensure tokens haven't expired
- Verify user accounts exist

#### Issue 3: Duplicate Changes
**Symptoms**: Multiple MRs for same Gerrit change
**Solutions**:
- Implement idempotency checks
- Add change tracking in sync service
- Use unique identifiers for correlation

#### Issue 4: CI/CD Pipeline Failures
**Symptoms**: Builds fail during integration
**Solutions**:
- Check CI configuration
- Verify environment variables
- Test webhook endpoints
- Review build logs

### 6.3 Integration Best Practices

#### Security Best Practices
- **Use service accounts** for integration
- **Rotate API tokens** regularly
- **Implement webhook validation**
- **Use HTTPS** in production
- **Limit token permissions** to minimum required

#### Performance Best Practices
- **Implement rate limiting** in sync service
- **Use asynchronous processing** for webhooks
- **Cache frequently accessed data**
- **Monitor resource usage**
- **Implement retry logic** with exponential backoff

#### Operational Best Practices
- **Monitor integration health** continuously
- **Set up alerting** for failures
- **Maintain detailed logs**
- **Document integration architecture**
- **Test disaster recovery** procedures

## Chapter Summary

You've learned how to integrate Gerrit with GitLab:

- **Multiple integration architectures** and their trade-offs
- **Setting up GitLab** for integration
- **Repository mirroring** and synchronization
- **Webhook-based sync services** for real-time integration
- **CI/CD integration** with both systems
- **Complete development workflows** using both platforms
- **Monitoring and troubleshooting** integration issues
- **Best practices** for secure and reliable integration

This integration gives you the best of both worlds: GitLab's excellent project management and CI/CD with Gerrit's superior code review capabilities.

## What's Next?

In the next chapter, we'll explore integrating Gerrit with GitHub, which follows similar patterns but with GitHub-specific features and considerations.

---

**Ready for GitHub integration?** Continue to [Chapter 7: Integration with GitHub](../07-github-integration/README.md)

---

*Continue to [Chapter 7: Integration with GitHub](../07-github-integration/README.md)*
