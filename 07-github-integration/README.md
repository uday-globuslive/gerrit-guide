# Chapter 7: Integration with GitHub

## Overview

GitHub integration with Gerrit brings together GitHub's massive ecosystem, excellent collaboration tools, and Actions CI/CD with Gerrit's superior code review workflow. This chapter covers various integration strategies, from simple mirroring to sophisticated bidirectional sync with GitHub's APIs.

## Why Integrate Gerrit with GitHub?

### Benefits of Integration

#### From GitHub
- **Largest Developer Community**: Access to millions of developers
- **Rich Ecosystem**: Extensive marketplace and integrations
- **GitHub Actions**: Powerful CI/CD platform
- **Project Management**: Issues, Projects, Discussions
- **Social Features**: Following, starring, forking
- **Security Features**: Dependabot, security advisories

#### From Gerrit
- **Superior Code Review**: More granular review process
- **Change Tracking**: Better history and auditability
- **Pre-merge Validation**: Prevent issues before merging
- **Fine-grained Permissions**: Enterprise-grade access control
- **Workflow Customization**: Tailored review processes

### Integration Scenarios

## Scenario 1: GitHub as Backup/Mirror

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Developer     │    │     Gerrit      │    │     GitHub      │
│                 │    │  (Primary)      │    │    (Mirror)     │
│ Local Machine   │───►│ Code Review     │───►│   Repository    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
   Git Operations         Review Process           Public Access
  (commit, push)         (approve, merge)         (clone, issues)
```

## Scenario 2: GitHub as Primary with Gerrit Review Gate

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│     GitHub      │    │     Gerrit      │    │     GitHub      │
│   (Primary)     │───►│  Review Gate    │───►│ Protected Branch│
│  Pull Requests  │    │                 │    │    (master)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
   GitHub Features        Gerrit Review          Final Repository
  (Issues, Actions)      (detailed review)      (production code)
```

## Part 1: Setting Up GitHub Integration

### 1.1 Create GitHub Repository

First, let's set up a GitHub repository for integration:

1. **Go to GitHub.com** and create a new repository
2. **Repository Details**:
   - Name: `gerrit-github-integration`
   - Description: `Testing Gerrit integration with GitHub`
   - Public or Private (your choice)
   - Initialize with README: ✅

3. **Clone locally**:
   ```powershell
   git clone https://github.com/yourusername/gerrit-github-integration.git
   cd gerrit-github-integration
   ```

### 1.2 Set Up GitHub Personal Access Token

1. **Go to GitHub** → Settings → Developer settings → Personal access tokens
2. **Generate new token** with these scopes:
   - `repo` (Full control of private repositories)
   - `workflow` (Update GitHub Action workflows)
   - `write:packages` (if using GitHub Packages)

3. **Save the token** securely - you'll need it for integration

### 1.3 Configure Gerrit Project

Create corresponding project in Gerrit:

1. **Access Gerrit** (http://localhost:8080)
2. **Create New Project**:
   - Admin → Projects → Create New Project
   - Project Name: `gerrit-github-integration`
   - ✅ Create initial empty commit

## Part 2: Simple Mirroring Setup

### 2.1 Configure Gerrit to Mirror to GitHub

#### Install Replication Plugin

```powershell
# If not already installed
cd C:\gerrit\gerrit_site
wget https://gerrit-ci.gerritforge.com/job/plugin-replication-stable-3.8/lastSuccessfulBuild/artifact/replication.jar -O plugins/replication.jar

# Restart Gerrit
./bin/gerrit.sh restart
```

#### Configure Replication to GitHub

Edit `etc/replication.config`:

```ini
[remote "github"]
    url = https://yourusername:${GITHUB_TOKEN}@github.com/yourusername/gerrit-github-integration.git
    push = +refs/heads/*:refs/heads/*
    push = +refs/tags/*:refs/tags/*
    projects = gerrit-github-integration
    replicationDelay = 30
    timeout = 30
```

#### Set Up Secure Configuration

Edit `etc/secure.config`:

```ini
[remote "github"]
    password = your-github-personal-access-token
```

### 2.2 Test Basic Mirroring

```powershell
# Clone Gerrit project
git clone http://localhost:8080/gerrit-github-integration
cd gerrit-github-integration

# Install commit-msg hook
curl -Lo .git/hooks/commit-msg http://localhost:8080/tools/hooks/commit-msg

# Create test change
echo "# GitHub Integration Test" > github-test.md
git add github-test.md
git commit -m "Add GitHub integration test file

This file tests the Gerrit-GitHub integration setup
and verifies that changes flow from Gerrit to GitHub.

Change-Id: I7777777777777777777777777777777777777777"

# Push to Gerrit
git push origin HEAD:refs/for/master

# Approve and submit in Gerrit web interface
# Check if it appears in GitHub repository
```

## Part 3: Advanced GitHub Integration

### 3.1 GitHub App for Enhanced Integration

Create a GitHub App for more robust integration:

#### Create GitHub App

1. **Go to GitHub** → Settings → Developer settings → GitHub Apps
2. **New GitHub App**:
   - Name: `Gerrit Integration App`
   - Homepage URL: `http://localhost:8080`
   - Webhook URL: `http://your-server.com/github-webhook`
   - Permissions:
     - Repository permissions:
       - Contents: Read & write
       - Issues: Read & write
       - Pull requests: Read & write
       - Metadata: Read
     - Organization permissions: None (unless needed)

3. **Generate private key** and save it securely

#### GitHub App Integration Service

```python
# github-integration.py
import jwt
import time
import requests
import json
from datetime import datetime, timedelta
from flask import Flask, request, jsonify

app = Flask(__name__)

class GitHubGerritIntegration:
    def __init__(self, app_id, private_key_path, installation_id):
        self.app_id = app_id
        self.private_key_path = private_key_path
        self.installation_id = installation_id
        self.gerrit_url = "http://localhost:8080"
        self.gerrit_auth = ("admin", "your-password")
    
    def get_github_token(self):
        """Generate GitHub App installation token"""
        
        # Read private key
        with open(self.private_key_path, 'r') as key_file:
            private_key = key_file.read()
        
        # Create JWT token
        payload = {
            'iat': int(time.time()),
            'exp': int(time.time()) + 600,  # 10 minutes
            'iss': self.app_id
        }
        
        token = jwt.encode(payload, private_key, algorithm='RS256')
        
        # Get installation token
        headers = {
            'Authorization': f'Bearer {token}',
            'Accept': 'application/vnd.github.v3+json'
        }
        
        response = requests.post(
            f'https://api.github.com/app/installations/{self.installation_id}/access_tokens',
            headers=headers
        )
        
        if response.status_code == 201:
            return response.json()['token']
        else:
            raise Exception(f"Failed to get GitHub token: {response.text}")
    
    def create_github_pr_from_gerrit_change(self, change_data):
        """Create GitHub PR from Gerrit change"""
        
        token = self.get_github_token()
        headers = {
            'Authorization': f'token {token}',
            'Accept': 'application/vnd.github.v3+json'
        }
        
        # Create branch in GitHub
        change_id = change_data['_number']
        branch_name = f"gerrit-change-{change_id}"
        
        # Get change details from Gerrit
        gerrit_response = requests.get(
            f"{self.gerrit_url}/a/changes/{change_id}",
            auth=self.gerrit_auth
        )
        
        if gerrit_response.status_code == 200:
            change_details = gerrit_response.json()
            
            # Create PR
            pr_data = {
                "title": f"[Gerrit] {change_details['subject']}",
                "head": branch_name,
                "base": change_details['branch'],
                "body": f"""
## Gerrit Change Information

**Change ID**: {change_id}
**Review URL**: {self.gerrit_url}/c/{change_id}

### Description
{change_details.get('message', 'No description')}

---
*This PR was automatically created from a Gerrit change.*
*Please review and approve in Gerrit, not GitHub.*
                """,
                "draft": True  # Create as draft since review happens in Gerrit
            }
            
            response = requests.post(
                'https://api.github.com/repos/yourusername/gerrit-github-integration/pulls',
                headers=headers,
                json=pr_data
            )
            
            return response.status_code == 201
        
        return False
    
    def sync_gerrit_status_to_github(self, change_data, status):
        """Sync Gerrit change status to GitHub PR"""
        
        token = self.get_github_token()
        headers = {
            'Authorization': f'token {token}',
            'Accept': 'application/vnd.github.v3+json'
        }
        
        change_id = change_data['_number']
        
        # Find corresponding GitHub PR
        response = requests.get(
            'https://api.github.com/repos/yourusername/gerrit-github-integration/pulls',
            headers=headers,
            params={'head': f'gerrit-change-{change_id}'}
        )
        
        if response.status_code == 200:
            prs = response.json()
            
            for pr in prs:
                pr_number = pr['number']
                
                if status == 'MERGED':
                    # Merge the PR
                    merge_data = {
                        "commit_title": f"Merge Gerrit change {change_id}",
                        "commit_message": f"Merged via Gerrit approval",
                        "merge_method": "merge"
                    }
                    
                    requests.put(
                        f'https://api.github.com/repos/yourusername/gerrit-github-integration/pulls/{pr_number}/merge',
                        headers=headers,
                        json=merge_data
                    )
                
                elif status == 'ABANDONED':
                    # Close the PR
                    close_data = {"state": "closed"}
                    
                    requests.patch(
                        f'https://api.github.com/repos/yourusername/gerrit-github-integration/pulls/{pr_number}',
                        headers=headers,
                        json=close_data
                    )
    
    def post_gerrit_comment_to_github(self, change_data, comment_data):
        """Post Gerrit review comments to GitHub PR"""
        
        token = self.get_github_token()
        headers = {
            'Authorization': f'token {token}',
            'Accept': 'application/vnd.github.v3+json'
        }
        
        change_id = change_data['_number']
        
        # Find corresponding GitHub PR
        response = requests.get(
            'https://api.github.com/repos/yourusername/gerrit-github-integration/pulls',
            headers=headers,
            params={'head': f'gerrit-change-{change_id}'}
        )
        
        if response.status_code == 200:
            prs = response.json()
            
            for pr in prs:
                pr_number = pr['number']
                
                # Post comment
                comment_body = f"""
**Gerrit Review Update**

**Reviewer**: {comment_data.get('author', {}).get('name', 'Unknown')}
**Score**: {comment_data.get('approvals', [{}])[0].get('value', 'N/A') if comment_data.get('approvals') else 'N/A'}

**Comment**:
{comment_data.get('message', 'No comment')}

---
*Review and approve in [Gerrit]({self.gerrit_url}/c/{change_id})*
                """
                
                comment_data = {"body": comment_body}
                
                requests.post(
                    f'https://api.github.com/repos/yourusername/gerrit-github-integration/issues/{pr_number}/comments',
                    headers=headers,
                    json=comment_data
                )

# Initialize integration service
github_integration = GitHubGerritIntegration(
    app_id=12345,  # Your GitHub App ID
    private_key_path='/path/to/private-key.pem',
    installation_id=67890  # Your installation ID
)

@app.route('/gerrit-webhook', methods=['POST'])
def handle_gerrit_webhook():
    """Handle webhooks from Gerrit"""
    
    data = request.json
    event_type = data.get('type')
    
    if event_type == 'change-created':
        github_integration.create_github_pr_from_gerrit_change(data['change'])
        
    elif event_type == 'change-merged':
        github_integration.sync_gerrit_status_to_github(data['change'], 'MERGED')
        
    elif event_type == 'change-abandoned':
        github_integration.sync_gerrit_status_to_github(data['change'], 'ABANDONED')
        
    elif event_type == 'comment-added':
        github_integration.post_gerrit_comment_to_github(
            data['change'],
            data['comment']
        )
    
    return jsonify({'status': 'processed'})

@app.route('/github-webhook', methods=['POST'])
def handle_github_webhook():
    """Handle webhooks from GitHub"""
    
    event_type = request.headers.get('X-GitHub-Event')
    data = request.json
    
    if event_type == 'pull_request':
        action = data['action']
        pr_data = data['pull_request']
        
        # Handle GitHub PR events
        if action in ['opened', 'synchronize']:
            # Could create corresponding Gerrit change
            pass
        elif action == 'closed':
            # Could update Gerrit change status
            pass
    
    elif event_type == 'issue_comment':
        # Handle comments on PRs
        pass
    
    return jsonify({'status': 'processed'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=True)
```

### 3.2 GitHub Actions Integration

Create GitHub Actions workflow for CI/CD integration:

#### Basic GitHub Actions Workflow

Create `.github/workflows/gerrit-integration.yml`:

```yaml
name: Gerrit Integration CI

on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [master, main]
  workflow_dispatch:
    inputs:
      gerrit_change_id:
        description: 'Gerrit Change ID'
        required: true
        type: string

env:
  GERRIT_URL: http://your-gerrit-server.com:8080
  GERRIT_USER: ${{ secrets.GERRIT_USER }}
  GERRIT_PASSWORD: ${{ secrets.GERRIT_PASSWORD }}

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      with:
        fetch-depth: 0
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install pytest flake8 black
        if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
    
    - name: Lint with flake8
      run: |
        # Stop the build if there are Python syntax errors or undefined names
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        # Exit-zero treats all errors as warnings
        flake8 . --count --exit-zero --max-complexity=10 --max-line-length=127 --statistics
    
    - name: Check code formatting
      run: |
        black --check .
    
    - name: Run tests
      run: |
        pytest tests/ -v --junitxml=test-results.xml
    
    - name: Post results to Gerrit
      if: always() && github.event.inputs.gerrit_change_id
      run: |
        # Determine result
        if [ "${{ job.status }}" == "success" ]; then
          VERIFY_SCORE="+1"
          MESSAGE="✅ GitHub Actions build passed: All tests and linting successful"
        else
          VERIFY_SCORE="-1"
          MESSAGE="❌ GitHub Actions build failed: Check the workflow for details"
        fi
        
        # Post to Gerrit
        curl -X POST \
          -u "$GERRIT_USER:$GERRIT_PASSWORD" \
          -H "Content-Type: application/json" \
          -d "{\"message\":\"$MESSAGE\",\"labels\":{\"Verified\":$VERIFY_SCORE}}" \
          "$GERRIT_URL/a/changes/${{ github.event.inputs.gerrit_change_id }}/revisions/current/review"
    
    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: test-results
        path: test-results.xml

  security-scan:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Run security scan
      uses: github/super-linter@v4
      env:
        DEFAULT_BRANCH: master
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        VALIDATE_PYTHON: true
        VALIDATE_DOCKERFILE: true
        VALIDATE_YAML: true
    
    - name: Post security results to Gerrit
      if: always() && github.event.inputs.gerrit_change_id
      run: |
        if [ "${{ job.status }}" == "success" ]; then
          SECURITY_SCORE="+1"
          MESSAGE="🔒 Security scan passed: No security issues found"
        else
          SECURITY_SCORE="-1"
          MESSAGE="⚠️ Security scan failed: Security issues detected"
        fi
        
        # Post to Gerrit (assuming custom Security-Review label)
        curl -X POST \
          -u "$GERRIT_USER:$GERRIT_PASSWORD" \
          -H "Content-Type: application/json" \
          -d "{\"message\":\"$MESSAGE\",\"labels\":{\"Security-Review\":$SECURITY_SCORE}}" \
          "$GERRIT_URL/a/changes/${{ github.event.inputs.gerrit_change_id }}/revisions/current/review"

  deploy:
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    if: github.ref == 'refs/heads/master' && github.event_name == 'push'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Deploy to staging
      run: |
        echo "Deploying to staging environment..."
        # Add your deployment commands here
    
    - name: Run integration tests
      run: |
        echo "Running integration tests..."
        # Add integration test commands here
    
    - name: Deploy to production
      if: success()
      run: |
        echo "Deploying to production environment..."
        # Add production deployment commands here
```

#### Advanced Workflow with Gerrit Trigger

Create a more sophisticated workflow that can be triggered by Gerrit changes:

```yaml
name: Advanced Gerrit CI

on:
  repository_dispatch:
    types: [gerrit-change]
  workflow_dispatch:
    inputs:
      change_id:
        description: 'Gerrit Change ID'
        required: true
      patch_set:
        description: 'Patch Set Number'
        required: true
        default: '1'
      project:
        description: 'Gerrit Project'
        required: true
      branch:
        description: 'Target Branch'
        required: true
        default: 'master'

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      change_id: ${{ steps.params.outputs.change_id }}
      patch_set: ${{ steps.params.outputs.patch_set }}
      project: ${{ steps.params.outputs.project }}
      branch: ${{ steps.params.outputs.branch }}
    
    steps:
    - name: Parse parameters
      id: params
      run: |
        if [ "${{ github.event_name }}" == "repository_dispatch" ]; then
          echo "change_id=${{ github.event.client_payload.change_id }}" >> $GITHUB_OUTPUT
          echo "patch_set=${{ github.event.client_payload.patch_set }}" >> $GITHUB_OUTPUT
          echo "project=${{ github.event.client_payload.project }}" >> $GITHUB_OUTPUT
          echo "branch=${{ github.event.client_payload.branch }}" >> $GITHUB_OUTPUT
        else
          echo "change_id=${{ github.event.inputs.change_id }}" >> $GITHUB_OUTPUT
          echo "patch_set=${{ github.event.inputs.patch_set }}" >> $GITHUB_OUTPUT
          echo "project=${{ github.event.inputs.project }}" >> $GITHUB_OUTPUT
          echo "branch=${{ github.event.inputs.branch }}" >> $GITHUB_OUTPUT
        fi

  fetch-change:
    runs-on: ubuntu-latest
    needs: setup
    
    steps:
    - name: Fetch Gerrit change
      run: |
        # Initialize git repo
        git init
        git remote add origin ${{ env.GERRIT_URL }}/${{ needs.setup.outputs.project }}
        
        # Fetch the specific change
        git fetch origin refs/changes/${{ needs.setup.outputs.change_id }}/${{ needs.setup.outputs.patch_set }}
        git checkout FETCH_HEAD
        
        # Create a temporary branch
        git checkout -b test-change-${{ needs.setup.outputs.change_id }}
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}
        restore-keys: |
          ${{ runner.os }}-pip-
    
    - name: Set up environment
      run: |
        python -m pip install --upgrade pip
        if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
        pip install pytest coverage flake8
    
    - name: Run comprehensive tests
      run: |
        # Unit tests with coverage
        coverage run -m pytest tests/unit/ -v
        
        # Integration tests
        pytest tests/integration/ -v
        
        # Generate coverage report
        coverage report --format=markdown > coverage-report.md
        coverage xml
    
    - name: Post test results to Gerrit
      if: always()
      run: |
        # Prepare test summary
        if [ -f coverage-report.md ]; then
          COVERAGE_SUMMARY=$(cat coverage-report.md)
        else
          COVERAGE_SUMMARY="Coverage report not available"
        fi
        
        if [ "${{ job.status }}" == "success" ]; then
          VERIFY_SCORE="+1"
          MESSAGE="✅ **GitHub Actions Build Successful**\n\n**Test Results:**\n- All unit tests passed\n- All integration tests passed\n\n**Coverage Report:**\n\`\`\`\n$COVERAGE_SUMMARY\n\`\`\`\n\n**Build Details:** ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
        else
          VERIFY_SCORE="-1"
          MESSAGE="❌ **GitHub Actions Build Failed**\n\nPlease check the [build logs](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}) for details."
        fi
        
        # Post to Gerrit
        curl -X POST \
          -u "$GERRIT_USER:$GERRIT_PASSWORD" \
          -H "Content-Type: application/json" \
          -d "{\"message\":\"$MESSAGE\",\"labels\":{\"Verified\":$VERIFY_SCORE}}" \
          "$GERRIT_URL/a/changes/${{ needs.setup.outputs.change_id }}/revisions/${{ needs.setup.outputs.patch_set }}/review"
```

## Part 4: Complete Workflow Examples

### 4.1 Development Workflow with GitHub

#### Standard Development Process

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant G as Gerrit
    participant CI as GitHub Actions
    
    Dev->>GH: 1. Create issue
    Dev->>Dev: 2. Implement locally
    Dev->>G: 3. Push to Gerrit for review
    G->>GH: 4. Create draft PR (webhook)
    G->>CI: 5. Trigger GitHub Actions
    CI->>G: 6. Post verification results
    Dev->>G: 7. Address review feedback
    G->>GH: 8. Update PR status
    G->>GH: 9. Merge after approval
    GH->>CI: 10. Deploy via Actions
```

#### Implementing the Workflow

1. **Create Issue in GitHub**:
   ```markdown
   **Title**: Implement user profile feature
   
   **Description**:
   Add user profile management functionality including:
   - Profile editing
   - Avatar upload
   - Privacy settings
   
   **Acceptance Criteria**:
   - [ ] Users can edit their profiles
   - [ ] Avatar upload works with size limits
   - [ ] Privacy settings are functional
   - [ ] All changes are covered by tests
   
   **Labels**: enhancement, backend, frontend
   ```

2. **Implement Feature Locally**:
   ```powershell
   # Clone and setup
   git clone http://localhost:8080/gerrit-github-integration
   cd gerrit-github-integration
   git checkout master
   git pull origin master
   
   # Create feature branch (optional for Gerrit)
   git checkout -b feature/user-profile
   
   # Implement feature
   # Add tests
   # Update documentation
   ```

3. **Submit to Gerrit**:
   ```powershell
   git add .
   git commit -m "Implement user profile management feature

   Add comprehensive user profile functionality including:
   - Profile editing with form validation
   - Avatar upload with size and type restrictions
   - Privacy settings with granular control
   - Comprehensive test coverage for all components

   The implementation follows security best practices and
   includes proper input validation and error handling.

   Closes: #42
   Change-Id: I8888888888888888888888888888888888888888"
   
   git push origin HEAD:refs/for/master
   ```

4. **Automated Processing**:
   - Gerrit receives change
   - Webhook triggers GitHub PR creation
   - GitHub Actions runs automatically
   - Results posted back to Gerrit

### 4.2 Hotfix Workflow

For critical production issues:

```powershell
# Emergency hotfix process
git checkout master
git pull origin master
git checkout -b hotfix/security-fix

# Implement critical fix
git add .
git commit -m "HOTFIX: Fix critical authentication bypass

Addresses CVE-2024-XXXX authentication bypass vulnerability
that allows unauthorized access to user accounts.

CRITICAL: Deploy immediately after review.

Security-Impact: High
Affects: All user authentication flows
Testing: Manual verification + automated security tests

Change-Id: I9999999999999999999999999999999999999999"

# Push with high priority topic
git push origin HEAD:refs/for/master%topic=critical-security-fix

# Fast-track review and merge
# GitHub Actions will auto-deploy to production
```

## Part 5: Advanced Features and Customization

### 5.1 Custom GitHub Actions for Gerrit

Create custom actions specifically for Gerrit integration:

#### Gerrit Review Action

Create `.github/actions/gerrit-review/action.yml`:

```yaml
name: 'Post to Gerrit Review'
description: 'Post review results to Gerrit change'
inputs:
  gerrit-url:
    description: 'Gerrit server URL'
    required: true
  change-id:
    description: 'Gerrit change ID'
    required: true
  patch-set:
    description: 'Patch set number'
    required: true
    default: 'current'
  username:
    description: 'Gerrit username'
    required: true
  password:
    description: 'Gerrit password'
    required: true
  verified-score:
    description: 'Verified label score (-1, 0, +1)'
    required: false
    default: '0'
  code-review-score:
    description: 'Code-Review label score (-2, -1, 0, +1, +2)'
    required: false
    default: '0'
  message:
    description: 'Review message'
    required: true

runs:
  using: 'composite'
  steps:
    - name: Post to Gerrit
      shell: bash
      run: |
        # Prepare review data
        REVIEW_DATA=$(cat <<EOF
        {
          "message": "${{ inputs.message }}",
          "labels": {
            "Verified": ${{ inputs.verified-score }},
            "Code-Review": ${{ inputs.code-review-score }}
          }
        }
        EOF
        )
        
        # Post to Gerrit
        curl -X POST \
          -u "${{ inputs.username }}:${{ inputs.password }}" \
          -H "Content-Type: application/json" \
          -d "$REVIEW_DATA" \
          "${{ inputs.gerrit-url }}/a/changes/${{ inputs.change-id }}/revisions/${{ inputs.patch-set }}/review"
```

#### Using the Custom Action

```yaml
name: Use Custom Gerrit Action

on:
  workflow_dispatch:
    inputs:
      change_id:
        required: true

jobs:
  test-and-review:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run tests
      id: tests
      run: |
        # Run your tests
        pytest tests/ && echo "tests_passed=true" >> $GITHUB_OUTPUT || echo "tests_passed=false" >> $GITHUB_OUTPUT
    
    - name: Post results to Gerrit
      uses: ./.github/actions/gerrit-review
      with:
        gerrit-url: ${{ env.GERRIT_URL }}
        change-id: ${{ github.event.inputs.change_id }}
        username: ${{ secrets.GERRIT_USER }}
        password: ${{ secrets.GERRIT_PASSWORD }}
        verified-score: ${{ steps.tests.outputs.tests_passed == 'true' && '1' || '-1' }}
        message: ${{ steps.tests.outputs.tests_passed == 'true' && 'All tests passed in GitHub Actions!' || 'Tests failed in GitHub Actions. Check workflow for details.' }}
```

### 5.2 Sophisticated Sync Service

Create a comprehensive sync service that handles complex scenarios:

```python
# advanced-github-sync.py
import asyncio
import aiohttp
import logging
from dataclasses import dataclass
from typing import Dict, List, Optional
import json

@dataclass
class GitHubPullRequest:
    number: int
    title: str
    body: str
    head_sha: str
    base_branch: str
    state: str

@dataclass
class GerritChange:
    number: int
    change_id: str
    subject: str
    status: str
    current_revision: str
    branch: str

class AdvancedGitHubGerritSync:
    def __init__(self, config):
        self.config = config
        self.session = None
        self.logger = logging.getLogger(__name__)
        
        # In-memory tracking of synced items
        self.gerrit_to_github = {}  # gerrit_change_id -> github_pr_number
        self.github_to_gerrit = {}  # github_pr_number -> gerrit_change_id
    
    async def start(self):
        self.session = aiohttp.ClientSession()
        await self.load_sync_state()
    
    async def stop(self):
        if self.session:
            await self.session.close()
        await self.save_sync_state()
    
    async def load_sync_state(self):
        """Load sync state from persistent storage"""
        try:
            with open('sync_state.json', 'r') as f:
                state = json.load(f)
                self.gerrit_to_github = state.get('gerrit_to_github', {})
                self.github_to_gerrit = state.get('github_to_gerrit', {})
        except FileNotFoundError:
            self.logger.info("No previous sync state found, starting fresh")
    
    async def save_sync_state(self):
        """Save sync state to persistent storage"""
        state = {
            'gerrit_to_github': self.gerrit_to_github,
            'github_to_gerrit': self.github_to_gerrit
        }
        with open('sync_state.json', 'w') as f:
            json.dump(state, f, indent=2)
    
    async def sync_gerrit_change_to_github(self, change: GerritChange):
        """Create or update GitHub PR from Gerrit change"""
        
        change_key = str(change.number)
        
        if change_key in self.gerrit_to_github:
            # Update existing PR
            pr_number = self.gerrit_to_github[change_key]
            await self.update_github_pr(pr_number, change)
        else:
            # Create new PR
            pr_number = await self.create_github_pr(change)
            if pr_number:
                self.gerrit_to_github[change_key] = pr_number
                self.github_to_gerrit[str(pr_number)] = change_key
    
    async def create_github_pr(self, change: GerritChange) -> Optional[int]:
        """Create new GitHub PR"""
        
        try:
            # First, create a branch with the change
            branch_name = f"gerrit-{change.number}"
            
            # Get change content from Gerrit
            change_content = await self.get_gerrit_change_content(change)
            
            if not change_content:
                return None
            
            # Create branch in GitHub
            await self.create_github_branch(branch_name, change_content)
            
            # Create PR
            pr_data = {
                "title": f"[Gerrit] {change.subject}",
                "head": branch_name,
                "base": change.branch,
                "body": self.generate_pr_description(change),
                "draft": True
            }
            
            async with self.session.post(
                f"{self.config['github_api_url']}/repos/{self.config['github_repo']}/pulls",
                headers=self.get_github_headers(),
                json=pr_data
            ) as response:
                if response.status == 201:
                    pr = await response.json()
                    self.logger.info(f"Created GitHub PR #{pr['number']} for Gerrit change {change.number}")
                    return pr['number']
                else:
                    self.logger.error(f"Failed to create GitHub PR: {response.status}")
                    return None
                    
        except Exception as e:
            self.logger.error(f"Error creating GitHub PR: {e}")
            return None
    
    async def update_github_pr(self, pr_number: int, change: GerritChange):
        """Update existing GitHub PR"""
        
        try:
            # Update PR title and description
            update_data = {
                "title": f"[Gerrit] {change.subject}",
                "body": self.generate_pr_description(change)
            }
            
            async with self.session.patch(
                f"{self.config['github_api_url']}/repos/{self.config['github_repo']}/pulls/{pr_number}",
                headers=self.get_github_headers(),
                json=update_data
            ) as response:
                if response.status == 200:
                    self.logger.info(f"Updated GitHub PR #{pr_number}")
                else:
                    self.logger.error(f"Failed to update GitHub PR #{pr_number}: {response.status}")
                    
        except Exception as e:
            self.logger.error(f"Error updating GitHub PR: {e}")
    
    async def handle_gerrit_change_merged(self, change: GerritChange):
        """Handle when Gerrit change is merged"""
        
        change_key = str(change.number)
        
        if change_key in self.gerrit_to_github:
            pr_number = self.gerrit_to_github[change_key]
            
            # Mark PR as ready and merge it
            await self.mark_pr_ready(pr_number)
            await self.merge_github_pr(pr_number)
            
            # Clean up tracking
            del self.gerrit_to_github[change_key]
            del self.github_to_gerrit[str(pr_number)]
    
    async def handle_gerrit_change_abandoned(self, change: GerritChange):
        """Handle when Gerrit change is abandoned"""
        
        change_key = str(change.number)
        
        if change_key in self.gerrit_to_github:
            pr_number = self.gerrit_to_github[change_key]
            
            # Close the PR
            await self.close_github_pr(pr_number)
            
            # Clean up tracking
            del self.gerrit_to_github[change_key]
            del self.github_to_gerrit[str(pr_number)]
    
    def generate_pr_description(self, change: GerritChange) -> str:
        """Generate GitHub PR description from Gerrit change"""
        
        return f"""
## Gerrit Change Information

**Change Number**: {change.number}
**Change ID**: {change.change_id}
**Status**: {change.status}
**Review URL**: {self.config['gerrit_url']}/c/{change.number}

### Description
{change.subject}

---

**⚠️ Important**: This PR is automatically managed by Gerrit integration.
- **Review and approve in Gerrit**, not in GitHub
- This PR will be automatically merged when approved in Gerrit
- Comments and discussions should happen in Gerrit

[👉 **Review this change in Gerrit**]({self.config['gerrit_url']}/c/{change.number})
        """
    
    def get_github_headers(self) -> Dict[str, str]:
        """Get GitHub API headers"""
        return {
            'Authorization': f"token {self.config['github_token']}",
            'Accept': 'application/vnd.github.v3+json'
        }
    
    async def get_gerrit_change_content(self, change: GerritChange):
        """Get change content from Gerrit"""
        # Implementation to fetch change details and file contents
        pass
    
    async def create_github_branch(self, branch_name: str, content):
        """Create branch in GitHub with change content"""
        # Implementation to create branch with the change
        pass
    
    async def mark_pr_ready(self, pr_number: int):
        """Mark GitHub PR as ready for review"""
        pass
    
    async def merge_github_pr(self, pr_number: int):
        """Merge GitHub PR"""
        pass
    
    async def close_github_pr(self, pr_number: int):
        """Close GitHub PR"""
        pass

# Usage
async def main():
    config = {
        'gerrit_url': 'http://localhost:8080',
        'github_api_url': 'https://api.github.com',
        'github_repo': 'yourusername/gerrit-github-integration',
        'github_token': 'your-github-token'
    }
    
    sync_service = AdvancedGitHubGerritSync(config)
    await sync_service.start()
    
    try:
        # Your sync logic here
        pass
    finally:
        await sync_service.stop()

if __name__ == "__main__":
    asyncio.run(main())
```

## Part 6: Monitoring and Troubleshooting

### 6.1 Integration Health Monitoring

```python
# github-health-monitor.py
import requests
import time
import json
from datetime import datetime, timedelta

class GitHubGerritHealthMonitor:
    def __init__(self, config):
        self.config = config
        self.health_data = []
    
    def check_github_api_health(self):
        """Check GitHub API health"""
        try:
            response = requests.get('https://api.github.com/rate_limit', 
                                  headers={'Authorization': f"token {self.config['github_token']}"})
            
            if response.status_code == 200:
                rate_limit = response.json()
                return {
                    'status': 'healthy',
                    'remaining_requests': rate_limit['rate']['remaining'],
                    'reset_time': rate_limit['rate']['reset']
                }
            else:
                return {'status': 'unhealthy', 'error': f"Status code: {response.status_code}"}
                
        except Exception as e:
            return {'status': 'unhealthy', 'error': str(e)}
    
    def check_gerrit_health(self):
        """Check Gerrit server health"""
        try:
            response = requests.get(f"{self.config['gerrit_url']}/config/server/info")
            return {'status': 'healthy' if response.status_code == 200 else 'unhealthy'}
        except Exception as e:
            return {'status': 'unhealthy', 'error': str(e)}
    
    def check_webhook_endpoints(self):
        """Check if webhook endpoints are reachable"""
        endpoints = [
            f"{self.config['sync_service_url']}/gerrit-webhook",
            f"{self.config['sync_service_url']}/github-webhook"
        ]
        
        results = {}
        for endpoint in endpoints:
            try:
                response = requests.get(endpoint, timeout=5)
                results[endpoint] = {'status': 'reachable'}
            except Exception as e:
                results[endpoint] = {'status': 'unreachable', 'error': str(e)}
        
        return results
    
    def check_sync_lag(self):
        """Check if there's significant sync lag"""
        # Check recent Gerrit changes vs GitHub PRs
        # Implementation would compare timestamps
        return {'lag_minutes': 2, 'status': 'acceptable'}
    
    def run_health_check(self):
        """Run comprehensive health check"""
        health_status = {
            'timestamp': datetime.now().isoformat(),
            'github_api': self.check_github_api_health(),
            'gerrit': self.check_gerrit_health(),
            'webhooks': self.check_webhook_endpoints(),
            'sync_lag': self.check_sync_lag()
        }
        
        # Determine overall health
        unhealthy_services = [
            service for service, status in health_status.items() 
            if isinstance(status, dict) and status.get('status') == 'unhealthy'
        ]
        
        health_status['overall_status'] = 'healthy' if not unhealthy_services else 'unhealthy'
        health_status['unhealthy_services'] = unhealthy_services
        
        self.health_data.append(health_status)
        
        # Keep only last 24 hours of data
        cutoff_time = datetime.now() - timedelta(hours=24)
        self.health_data = [
            data for data in self.health_data 
            if datetime.fromisoformat(data['timestamp']) > cutoff_time
        ]
        
        return health_status
    
    def get_health_report(self):
        """Generate health report"""
        if not self.health_data:
            return "No health data available"
        
        recent_data = self.health_data[-10:]  # Last 10 checks
        healthy_count = sum(1 for data in recent_data if data['overall_status'] == 'healthy')
        uptime_percentage = (healthy_count / len(recent_data)) * 100
        
        return {
            'uptime_percentage': uptime_percentage,
            'total_checks': len(recent_data),
            'healthy_checks': healthy_count,
            'last_check': recent_data[-1] if recent_data else None
        }

# Continuous monitoring
if __name__ == "__main__":
    config = {
        'github_token': 'your-token',
        'gerrit_url': 'http://localhost:8080',
        'sync_service_url': 'http://localhost:5001'
    }
    
    monitor = GitHubGerritHealthMonitor(config)
    
    while True:
        health_status = monitor.run_health_check()
        
        if health_status['overall_status'] == 'unhealthy':
            print(f"⚠️ Integration unhealthy: {health_status['unhealthy_services']}")
            # Send alerts here
        else:
            print("✅ Integration healthy")
        
        # Sleep for 5 minutes
        time.sleep(300)
```

### 6.2 Common Issues and Solutions

#### Issue 1: GitHub API Rate Limiting
**Symptoms**: 403 errors from GitHub API
**Solutions**:
- Use GitHub App authentication (higher rate limits)
- Implement request queuing
- Cache responses when possible
- Monitor rate limit headers

#### Issue 2: Webhook Delivery Failures
**Symptoms**: Events not reaching sync service
**Solutions**:
- Check webhook endpoint availability
- Verify webhook configuration
- Implement webhook retry logic
- Use webhook secret validation

#### Issue 3: Sync Inconsistencies
**Symptoms**: Changes in Gerrit but no corresponding GitHub PR
**Solutions**:
- Implement sync reconciliation
- Add comprehensive logging
- Use persistent state tracking
- Monitor sync lag

#### Issue 4: Merge Conflicts
**Symptoms**: GitHub PRs cannot be merged
**Solutions**:
- Implement automatic rebase
- Add conflict detection
- Notify users of conflicts
- Provide merge instructions

## Chapter Summary

You've learned comprehensive GitHub integration with Gerrit:

- **Multiple integration architectures** for different needs
- **GitHub App creation** for robust authentication
- **Sophisticated sync services** with error handling
- **GitHub Actions integration** for CI/CD
- **Complete development workflows** using both platforms
- **Advanced features** like custom actions and monitoring
- **Health monitoring** and troubleshooting strategies
- **Best practices** for production deployments

This integration creates a powerful development environment combining GitHub's ecosystem with Gerrit's review capabilities.

## What's Next?

In the next chapter, we'll explore enterprise-level setup and configuration, covering advanced security, scalability, and multi-team management.

---

**Ready for enterprise setup?** Continue to [Chapter 8: Enterprise Setup and Configuration](../08-enterprise-setup/README.md)

---

*Continue to [Chapter 8: Enterprise Setup and Configuration](../08-enterprise-setup/README.md)*
