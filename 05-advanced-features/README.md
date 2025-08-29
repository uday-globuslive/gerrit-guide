# Chapter 5: Advanced Gerrit Features

## Overview

Now that you've mastered the basics, let's explore Gerrit's powerful advanced features. These tools will make you significantly more productive and help you handle complex development scenarios with confidence.

## Part 1: Advanced Change Management

### 1.1 Working with Dependencies

Sometimes changes depend on other changes. Gerrit handles this elegantly.

#### Creating Dependent Changes

Let's create a series of dependent changes:

```powershell
# Start from master
git checkout master
git pull origin master

# Create first change
echo "# Project Documentation" > README.md
git add README.md
git commit -m "Add project README

Initial documentation for the project with basic
information about the codebase structure.

Change-Id: I1111111111111111111111111111111111111111"

git push origin HEAD:refs/for/master

# Create second change that depends on the first
echo "## Installation" >> README.md
echo "Run: python hello.py" >> README.md
git add README.md
git commit -m "Add installation instructions to README

Provides clear instructions for users on how to
run the Hello World program.

Depends on basic README structure from previous change.

Change-Id: I2222222222222222222222222222222222222222"

git push origin HEAD:refs/for/master
```

#### Dependency Chain Visualization

In Gerrit's web interface, you'll see:

```
Change #3: Add installation instructions ← Depends on Change #2
    ↑
Change #2: Add project README ← Base change
    ↑
master branch
```

#### Benefits of Dependencies
- **Logical grouping** of related changes
- **Easier review** of complex features
- **Controlled merging** order
- **Clear change history**

### 1.2 Cherry-Picking Changes

Cherry-picking allows you to apply changes to different branches.

#### Using Gerrit's Cherry-Pick Feature

1. **Go to your change** in the web interface
2. **Click "Cherry Pick"** button
3. **Select target branch**
4. **Gerrit creates a new change** on the target branch

#### Manual Cherry-Pick

```powershell
# Create a release branch
git checkout master
git checkout -b release-1.0
git push origin release-1.0

# Cherry-pick a specific change
git fetch origin refs/changes/01/1/2  # Fetch patch set 2 of change 1
git cherry-pick FETCH_HEAD

# Push as new change for review
git push origin HEAD:refs/for/release-1.0
```

### 1.3 Rebasing Changes

Rebasing keeps your changes up-to-date with the latest code.

#### When to Rebase
- Target branch has moved forward
- Merge conflicts need resolution
- Want to maintain clean history

#### Rebasing in Gerrit Web Interface

1. **Go to your change**
2. **Click "Rebase"** button
3. **Gerrit automatically rebases** if no conflicts
4. **New patch set created** with updated base

#### Manual Rebase

```powershell
# Update your local master
git checkout master
git pull origin master

# Rebase your change
git checkout your-feature-branch
git rebase master

# Force push the updated change
git push origin HEAD:refs/for/master
```

## Part 2: Advanced Search and Queries

### 2.1 Powerful Search Queries

Gerrit's search is incredibly powerful. Here are essential search patterns:

#### Basic Searches
```
# Find your changes
owner:self

# Find changes you need to review
reviewer:self -owner:self

# Find changes in specific project
project:test-project

# Find changes by status
status:open
status:merged
status:abandoned

# Find changes by branch
branch:master
branch:release-1.0
```

#### Advanced Searches
```
# Changes modified in last week
owner:self age:1w

# Changes with specific file
file:^.*\.py$

# Changes with specific commit message content
message:"bug fix"

# Changes with specific scores
label:Code-Review=+2
label:Verified=+1

# Combine criteria
project:test-project AND status:open AND owner:john.doe
```

#### Search by Time and Size
```
# Recent changes
after:2024-01-01
before:2024-12-31

# Large changes
added:>100
deleted:>50

# Small changes  
delta:<10
```

### 2.2 Saved Searches and Dashboards

#### Creating Custom Dashboards

1. **Create a search query**
2. **Save it** with a meaningful name
3. **Add to dashboard** for quick access

Example dashboard sections:
- **My Open Changes**: `owner:self status:open`
- **Needs My Review**: `reviewer:self -owner:self status:open`
- **Recently Merged**: `owner:self status:merged age:1w`
- **Hot Changes**: `project:critical-app status:open`

#### Dashboard Configuration File

```ini
[dashboard]
  title = My Development Dashboard
  description = Personal dashboard for daily development work

[section "My Changes"]
  query = owner:self status:open

[section "Needs Review"]
  query = reviewer:self -owner:self status:open

[section "Recently Closed"]
  query = owner:self status:merged OR status:abandoned age:2w
```

## Part 3: Inline Comments and Discussions

### 3.1 Adding Inline Comments

Inline comments are crucial for effective code review.

#### How to Add Inline Comments

1. **Open a change** in the web interface
2. **Click on a file** to view the diff
3. **Click on line numbers** to add comments
4. **Type your comment** and click "Save"
5. **Publish all comments** by clicking "Reply"

#### Types of Comments

**Suggestion Comments:**
```
Consider using a more descriptive variable name here.
Instead of 'data', how about 'user_credentials'?
```

**Question Comments:**
```
Why are we using a while loop here instead of a for loop?
Is there a specific performance reason?
```

**Praise Comments:**
```
Nice error handling! This makes the code much more robust.
```

**Critical Issues:**
```
⚠️ SECURITY ISSUE: This could lead to SQL injection.
Please use parameterized queries instead.
```

### 3.2 Resolving Comments

#### Comment States
- **Unresolved**: Needs attention from the author
- **Resolved**: Author has addressed the feedback
- **Done**: Reviewer confirms the issue is fixed

#### Resolution Workflow

1. **Author responds** to comment with code changes
2. **Author marks as "Done"** if implemented
3. **Reviewer verifies** and marks as "Resolved"

#### Best Practices for Comments

**Be Specific:**
```
❌ "This is wrong"
✅ "Line 15: The variable 'user_id' should be validated before use"
```

**Be Constructive:**
```
❌ "This code is terrible"
✅ "Consider refactoring this function for better readability. 
    Maybe split it into smaller, single-purpose functions?"
```

**Explain Why:**
```
✅ "Please add input validation here. Without it, invalid data 
    could crash the application in production."
```

## Part 4: Working with Multiple Commits

### 4.1 Multi-Commit Changes

Sometimes a logical change requires multiple commits.

#### Creating Multi-Commit Changes

```powershell
# Method 1: Push multiple commits
git commit -m "Add user model

Defines the basic User class with essential attributes.

Change-Id: I3333333333333333333333333333333333333333"

git commit -m "Add user validation

Implements validation logic for user data integrity.

Change-Id: I4444444444444444444444444444444444444444"

# Push both commits as separate changes
git push origin HEAD~1:refs/for/master  # First commit
git push origin HEAD:refs/for/master    # Second commit
```

#### Method 2: Topic Branches

Topics group related changes together:

```powershell
# Push with topic
git push origin HEAD:refs/for/master%topic=user-authentication

# All changes in this topic will be grouped together
```

### 4.2 Squashing and Splitting Changes

#### When to Squash
- Multiple small fixes for the same issue
- Clean up commit history
- Combine related changes

#### Squashing Commits

```powershell
# Interactive rebase to squash
git rebase -i HEAD~3

# In the editor, change 'pick' to 'squash' for commits to combine
pick 1234567 Add user model
squash 2345678 Fix user model bug
squash 3456789 Add user model tests

# Update commit message and push
git push origin HEAD:refs/for/master
```

## Part 5: Labels and Custom Workflows

### 5.1 Understanding Labels

Labels are Gerrit's way of tracking different types of approvals.

#### Default Labels
- **Code-Review**: Quality of the code itself
- **Verified**: Whether the change works (usually automated)

#### Custom Labels Examples
- **Security-Review**: Security team approval
- **Performance-Review**: Performance impact assessment
- **Documentation-Review**: Documentation quality
- **Design-Review**: Architecture and design approval

### 5.2 Custom Label Configuration

To add custom labels, modify your project configuration:

```ini
[label "Security-Review"]
    function = MaxWithBlock
    value = -1 Security concerns
    value =  0 No score
    value = +1 Security approved
    copyMinScore = true
    copyMaxScore = true
```

### 5.3 Submit Requirements

Define what approvals are needed before changes can be submitted:

```ini
[submit-requirement "Code-Review"]
    description = Code review by peers
    submittableIf = label:Code-Review=MAX AND -label:Code-Review=MIN

[submit-requirement "Security-Review"]
    description = Security team approval for sensitive changes
    applicableIf = file:^.*security.*$ OR message:security
    submittableIf = label:Security-Review=MAX
    overrideIf = is:owner AND label:Code-Review=MAX
```

## Part 6: Automation and Integration

### 6.1 Automated Verification

#### Setting Up CI Integration

Configure your CI system to automatically verify changes:

```yaml
# Example Jenkins pipeline
pipeline {
    agent any
    
    stages {
        stage('Test') {
            steps {
                script {
                    // Run tests
                    sh 'python -m pytest tests/'
                }
            }
        }
    }
    
    post {
        success {
            script {
                // Post +1 Verified to Gerrit
                sh '''
                ssh -p 29418 gerrit-server gerrit review \
                  --verified +1 \
                  --message "Build successful" \
                  ${GERRIT_CHANGE_NUMBER},${GERRIT_PATCHSET_NUMBER}
                '''
            }
        }
        failure {
            script {
                // Post -1 Verified to Gerrit
                sh '''
                ssh -p 29418 gerrit-server gerrit review \
                  --verified -1 \
                  --message "Build failed" \
                  ${GERRIT_CHANGE_NUMBER},${GERRIT_PATCHSET_NUMBER}
                '''
            }
        }
    }
}
```

### 6.2 Webhooks and Events

Gerrit can send webhooks for various events:

#### Common Webhook Events
- **change-created**: New change uploaded
- **change-merged**: Change successfully merged
- **patchset-created**: New patch set uploaded
- **comment-added**: Review comment added

#### Webhook Configuration

```ini
[plugin "webhooks"]
    remote = http://your-server.com/gerrit-webhook
    event = change-created
    event = change-merged
    event = comment-added
```

## Part 7: Working with Large Changes

### 7.1 Strategies for Large Changes

Large changes (>500 lines) need special handling:

#### Break Down Strategy
```
Large Feature
├── Change 1: Core infrastructure
├── Change 2: Basic functionality  
├── Change 3: Advanced features
└── Change 4: Integration tests
```

#### Review Guidelines for Large Changes
- **Review in chunks**: Focus on one aspect at a time
- **Use drafts**: Mark as WIP until ready
- **Provide context**: Explain the big picture
- **Include tests**: Ensure comprehensive testing

### 7.2 Using Draft Changes

Draft changes let you work in progress without bothering reviewers:

```powershell
# Push as draft
git push origin HEAD:refs/drafts/master

# Or convert existing change to draft in web interface
```

#### When to Use Drafts
- **Work in progress**: Not ready for review
- **Experimental changes**: Testing ideas
- **Large refactoring**: Want feedback before completion

## Part 8: Advanced Branch Management

### 8.1 Branch Permissions

Configure detailed permissions for different branches:

```ini
[access "refs/heads/master"]
    push = group Administrators
    submit = group Project Owners
    read = group Registered Users

[access "refs/heads/release/*"]
    push = group Release Managers
    create = group Release Managers
    submit = group Release Managers

[access "refs/heads/feature/*"]
    push = group Developers
    submit = group Developers
    delete = group Developers
```

### 8.2 Auto-Submit

Configure automatic submission when requirements are met:

```ini
[plugin "automerger"]
    enable = true
    delayTime = 5m  # Wait 5 minutes before auto-submit
```

## Part 9: Performance and Optimization

### 9.1 Optimizing Large Repositories

For large repositories, consider:

#### Shallow Clones
```powershell
# Clone with limited history
git clone --depth 1 http://localhost:8080/large-project
```

#### Partial Clones
```powershell
# Clone without blob objects
git clone --filter=blob:none http://localhost:8080/large-project
```

### 9.2 Gerrit Performance Tips

- **Use topics** for related changes
- **Batch small changes** when possible
- **Regular maintenance** of the Gerrit database
- **Monitor disk space** and clean old data

## Chapter Summary

You've learned advanced Gerrit features:

- **Change dependencies** and cherry-picking
- **Powerful search queries** and dashboards
- **Inline comments** and discussion workflows
- **Multi-commit changes** and topics
- **Custom labels** and submit requirements
- **Automation** and CI integration
- **Large change management** strategies
- **Advanced branch permissions**

These advanced features will make you highly productive with Gerrit in any enterprise environment.

## What's Next?

In the next chapter, we'll explore integrating Gerrit with GitLab, opening up powerful hybrid workflows for your development team.

---

**Ready for GitLab integration?** Continue to [Chapter 6: Integration with GitLab](../06-gitlab-integration/README.md)

---

*Continue to [Chapter 6: Integration with GitLab](../06-gitlab-integration/README.md)*
