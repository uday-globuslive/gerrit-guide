# Gerrit Command Reference

## 🚀 Essential Commands (The 80/20 Rule)

These 5 commands handle 80% of your daily Gerrit work:

### 1. Submit New Change
```bash
git push origin HEAD:refs/for/main
```
**Use when:** You want to submit code for review

### 2. Update Existing Change
```bash
git commit --amend
git push origin HEAD:refs/for/main
```
**Use when:** Responding to review feedback

### 3. Download Change for Review
```bash
git fetch origin refs/changes/XX/YYYY/Z && git checkout FETCH_HEAD
```
**Use when:** You want to test someone else's change

### 4. Rebase Before Submit
```bash
git rebase origin/main
git push origin HEAD:refs/for/main
```
**Use when:** Your change conflicts with newer changes

### 5. Check Status
```bash
git log --oneline -5
```
**Use when:** You want to see recent commits and Change-Ids

---

## 📋 Complete Command Reference

### Repository Setup

```bash
# Clone repository
git clone https://gerrit.example.com/myproject
cd myproject

# Install commit-msg hook
curl -Lo .git/hooks/commit-msg https://gerrit.example.com/tools/hooks/commit-msg
chmod +x .git/hooks/commit-msg

# Configure Git (if needed)
git config user.name "Your Name"
git config user.email "your.email@example.com"
```

### Working with Changes

#### Submit Changes
```bash
# Submit to main branch
git push origin HEAD:refs/for/main

# Submit to specific branch
git push origin HEAD:refs/for/develop

# Submit as draft (private review)
git push origin HEAD:refs/drafts/main

# Submit with specific reviewers
git push origin HEAD:refs/for/main%r=reviewer1@example.com,r=reviewer2@example.com

# Submit with topic
git push origin HEAD:refs/for/main%topic=feature-login
```

#### Update Changes
```bash
# Update existing change (amend commit)
git commit --amend
git push origin HEAD:refs/for/main

# Update with new commit message
git commit --amend -m "New commit message"
git push origin HEAD:refs/for/main

# Update specific files only
git add specific-file.txt
git commit --amend --no-edit
git push origin HEAD:refs/for/main
```

#### Download Changes
```bash
# Download specific change (patch set 1)
git fetch origin refs/changes/23/12345/1 && git checkout FETCH_HEAD

# Download latest patch set
git fetch origin refs/changes/23/12345/3 && git checkout FETCH_HEAD

# Download and create branch
git fetch origin refs/changes/23/12345/1
git checkout -b review-12345 FETCH_HEAD

# Download multiple changes
git fetch origin refs/changes/23/12345/1 refs/changes/24/12346/1
```

### Branch Management

```bash
# Switch back to main branch
git checkout main

# Get latest changes
git pull origin main

# Create new branch for development
git checkout -b feature-branch

# List all branches
git branch -a

# Delete local branch
git branch -d feature-branch
```

### Conflict Resolution

```bash
# Fetch latest changes
git fetch origin

# Rebase your change
git rebase origin/main

# During conflict resolution:
# Edit conflicted files
git add conflicted-file.txt
git rebase --continue

# Abort rebase if needed
git rebase --abort

# Force push after rebase
git push origin HEAD:refs/for/main
```

### Change Information

```bash
# View commit history
git log --oneline -10

# View Change-Id in commits
git log --format="%h %s %b" -5

# Show specific commit details
git show HEAD

# View current branch
git branch

# Check git status
git status

# See what changed
git diff HEAD~1

# Compare with remote
git diff origin/main
```

### Advanced Commands

#### Cherry-pick Changes
```bash
# Cherry-pick specific change
git fetch origin refs/changes/23/12345/1
git cherry-pick FETCH_HEAD

# Cherry-pick with new Change-Id
git cherry-pick -x FETCH_HEAD
```

#### Work with Multiple Changes
```bash
# Submit multiple dependent changes
git push origin HEAD:refs/for/main

# Each commit becomes a separate change
# Gerrit automatically detects dependencies
```

#### Reset and Clean
```bash
# Reset to remote state
git reset --hard origin/main

# Clean untracked files
git clean -fd

# Reset specific file
git checkout HEAD -- filename.txt

# Unstage changes
git reset HEAD filename.txt
```

### Gerrit-specific Git Commands

#### Change-Id Management
```bash
# Generate Change-Id for existing commit
git commit --amend

# View Change-Id
git log --format="%h %s %b" -1

# Remove Change-Id (for new change)
git commit --amend
# Delete Change-Id line manually
```

#### Topic Management
```bash
# Submit with topic
git push origin HEAD:refs/for/main%topic=my-feature

# Submit multiple changes with same topic
git push origin HEAD:refs/for/main%topic=login-improvement
```

#### Review Labels
```bash
# Submit with Work-In-Progress label
git push origin HEAD:refs/for/main%wip

# Submit with Ready label
git push origin HEAD:refs/for/main%ready

# Submit with private flag
git push origin HEAD:refs/for/main%private
```

---

## 🔧 Gerrit REST API Commands

### Using curl

```bash
# Get change information
curl -u username:password https://gerrit.example.com/a/changes/12345

# Get change details
curl -u username:password https://gerrit.example.com/a/changes/12345/detail

# Submit change
curl -u username:password -X POST https://gerrit.example.com/a/changes/12345/submit

# Abandon change
curl -u username:password -X POST https://gerrit.example.com/a/changes/12345/abandon

# Add reviewer
curl -u username:password -X POST \
  -H "Content-Type: application/json" \
  -d '{"reviewer":"reviewer@example.com"}' \
  https://gerrit.example.com/a/changes/12345/reviewers
```

### Using git-review (Alternative Tool)

```bash
# Install git-review
pip install git-review

# Configure git-review
git review -s

# Submit change
git review

# Download change
git review -d 12345

# Compare patch sets
git review -c 12345,1,2
```

---

## 🐛 Troubleshooting Commands

### Common Issues

#### Missing Change-Id
```bash
# Install hook
curl -Lo .git/hooks/commit-msg https://gerrit.example.com/tools/hooks/commit-msg
chmod +x .git/hooks/commit-msg

# Fix existing commit
git commit --amend
```

#### Push Rejected
```bash
# Check permissions
git ls-remote origin

# Verify branch exists
git branch -r

# Check commit message format
git log --format="%h %s %b" -1
```

#### Authentication Issues
```bash
# Check credentials
git config --list | grep user

# Update credentials
git config user.name "Your Name"
git config user.email "your.email@example.com"

# Test connection
git ls-remote origin
```

#### Merge Conflicts
```bash
# Check conflict status
git status

# View conflicted files
git diff --name-only --diff-filter=U

# Resolve conflicts
git mergetool

# Continue after resolution
git rebase --continue
```

### Diagnostic Commands

```bash
# Check Git configuration
git config --list

# Check remote configuration
git remote -v

# Check branch tracking
git branch -vv

# Check commit-msg hook
ls -la .git/hooks/commit-msg

# Test hook
.git/hooks/commit-msg test-message.txt

# View Git log with graph
git log --oneline --graph -10

# Check reflog
git reflog
```

---

## 📊 Useful Aliases

Add these to your `.gitconfig` file:

```bash
[alias]
    # Gerrit-specific aliases
    gerrit-push = push origin HEAD:refs/for/main
    gerrit-draft = push origin HEAD:refs/drafts/main
    gerrit-rebase = !git fetch origin && git rebase origin/main
    
    # Common operations
    st = status
    co = checkout
    br = branch
    ci = commit
    amend = commit --amend
    
    # Log viewing
    lg = log --oneline --graph -10
    changes = log --format="%h %s %b" -5
    
    # Diff operations
    diff-staged = diff --cached
    diff-last = diff HEAD~1
    
    # Reset operations
    reset-hard = reset --hard origin/main
    clean-all = clean -fd
```

To use these aliases:
```bash
git gerrit-push
git gerrit-rebase
git lg
git changes
```

---

## 🎯 Command Cheat Sheet

### Daily Workflow
```bash
# 1. Get latest changes
git checkout main && git pull origin main

# 2. Make your changes
# ... edit files ...

# 3. Stage and commit
git add . && git commit -m "Your message"

# 4. Submit for review
git push origin HEAD:refs/for/main

# 5. Update after feedback
git commit --amend && git push origin HEAD:refs/for/main
```

### Review Workflow
```bash
# 1. Download change
git fetch origin refs/changes/XX/YYYY/Z && git checkout FETCH_HEAD

# 2. Test the change
# ... run tests, manual testing ...

# 3. Return to main
git checkout main

# 4. Leave feedback in Gerrit UI
```

### Conflict Resolution
```bash
# 1. Fetch and rebase
git fetch origin && git rebase origin/main

# 2. Resolve conflicts
# ... edit conflicted files ...

# 3. Continue
git add . && git rebase --continue

# 4. Update change
git push origin HEAD:refs/for/main
```

---

## 💡 Pro Tips

### Speed up your workflow:
1. **Use aliases** for common commands
2. **Learn keyboard shortcuts** in Gerrit UI
3. **Set up IDE integration** for Gerrit
4. **Use git-review** for simplified commands
5. **Create templates** for commit messages

### Best practices:
1. **Always rebase** before submitting
2. **Test changes** before submitting
3. **Write clear commit messages**
4. **Keep changes small** and focused
5. **Respond promptly** to review feedback

Remember: These commands are your toolkit. Master the essential ones first, then gradually add more advanced commands as needed!
