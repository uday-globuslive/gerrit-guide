# Chapter 4: Your First Code Review

## Overview

It's time to get hands-on! In this chapter, we'll walk through creating your very first code change, submitting it for review, and going through the complete review process. By the end, you'll have practical experience with the entire Gerrit workflow.

## Prerequisites

Before starting, ensure you have:
- [x] Gerrit installed and running (Chapter 2)
- [x] Understanding of Gerrit concepts (Chapter 3)
- [x] A test project created in Gerrit
- [x] Git configured on your local machine

## Part 1: Setting Up Your Local Environment

### 1.1 Clone the Test Project

First, let's clone the test project we created in Chapter 2:

```powershell
# Navigate to your workspace
cd C:\

# Clone the project
git clone http://localhost:8080/test-project
cd test-project

# Configure Git user (if not already done)
git config user.name "Your Name"
git config user.email "your.email@example.com"
```

### 1.2 Install the Commit-msg Hook

This is **crucial** for Gerrit to work properly:

**Option 1: Using SCP (if SSH is configured)**
```powershell
scp -p -P 29418 admin@localhost:hooks/commit-msg .git/hooks/
```

**Option 2: Manual Download (recommended for beginners)**
1. Open browser and go to: `http://localhost:8080/tools/hooks/commit-msg`
2. Save the file to your project's `.git/hooks/` directory
3. On Windows, you might need to rename it to remove `.txt` extension

**Option 3: Using Gerrit Web Interface**
1. Go to your project page in Gerrit web interface
2. Click "Clone" button
3. Copy the "Clone with commit-msg hook" command

### 1.3 Verify Your Setup

```powershell
# Check Git configuration
git config --list | Select-String "user"

# Verify the hook is installed
ls .git/hooks/commit-msg

# Check Gerrit remote
git remote -v
```

Expected output:
```
origin  http://localhost:8080/test-project (fetch)
origin  http://localhost:8080/test-project (push)
```

## Part 2: Creating Your First Change

### 2.1 Create a Simple File

Let's create a simple "Hello World" program:

```powershell
# Create a new file
New-Item -Name "hello.py" -ItemType File

# Add content to the file (using your preferred editor)
@"
#!/usr/bin/env python3
"""
Simple Hello World program for Gerrit tutorial
"""

def main():
    print("Hello, Gerrit World!")
    print("This is my first code review!")

if __name__ == "__main__":
    main()
"@ | Out-File -FilePath "hello.py" -Encoding UTF8
```

### 2.2 Test Your Code

```powershell
# Test the program
python hello.py
```

Expected output:
```
Hello, Gerrit World!
This is my first code review!
```

### 2.3 Add and Commit Your Changes

```powershell
# Add the file to Git
git add hello.py

# Commit with a proper message
git commit -m "Add Hello World Python program

This program demonstrates basic Python functionality
and serves as a first example for code review.

The program simply prints greeting messages to
demonstrate the Gerrit review workflow."
```

**Important**: Notice how the commit-msg hook automatically added a Change-Id to your commit message!

### 2.4 Verify the Change-Id

```powershell
# View the commit message
git log --oneline -1
git show --no-patch --format=fuller
```

You should see something like:
```
commit 1234567890abcdef...
Author: Your Name <your.email@example.com>
Commit: Your Name <your.email@example.com>

    Add Hello World Python program
    
    This program demonstrates basic Python functionality
    and serves as a first example for code review.
    
    The program simply prints greeting messages to
    demonstrate the Gerrit review workflow.
    
    Change-Id: I7a8b9c0d1e2f3g4h5i6j7k8l9m0n1o2p3q4r5s6t
```

## Part 3: Pushing to Gerrit for Review

### 3.1 Push to Gerrit

This is where Gerrit differs from regular Git:

```powershell
# Push to Gerrit for review (NOT directly to master)
git push origin HEAD:refs/for/master
```

**Important**: Notice the `refs/for/master` instead of just `master`. This tells Gerrit to create a change for review rather than pushing directly.

Expected output:
```
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 8 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 456 bytes | 456.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
remote: Processing changes: new: 1, done
remote:
remote: SUCCESS
remote:
remote:   http://localhost:8080/c/test-project/+/1 Add Hello World Python program
remote:
To http://localhost:8080/test-project
 * [new branch]      HEAD -> refs/for/master
```

### 3.2 Understanding the Output

Let's break down what happened:

- **`new: 1`**: One new change was created
- **`SUCCESS`**: The push was successful
- **`http://localhost:8080/c/test-project/+/1`**: Direct link to your change
- **Change number**: Your change was assigned number `1`

## Part 4: Exploring Your Change in the Web Interface

### 4.1 Access Your Change

1. **Open the provided URL** or go to http://localhost:8080
2. **Find your change** in the dashboard under "Outgoing"
3. **Click on the change** to view details

### 4.2 Understanding the Change Page

Your change page shows:

```
┌─────────────────────────────────────────────────────┐
│ Change 1: Add Hello World Python program           │
├─────────────────────────────────────────────────────┤
│ Status: New          Project: test-project         │
│ Branch: master       Owner: Your Name              │
├─────────────────────────────────────────────────────┤
│ [Full commit message displayed]                     │
├─────────────────────────────────────────────────────┤
│ Files (1):                                          │
│ A hello.py (+12)                                   │
├─────────────────────────────────────────────────────┤
│ [No reviewers assigned yet]                         │
└─────────────────────────────────────────────────────┘
```

### 4.3 Examining the File Changes

1. **Click on `hello.py`** to see the diff
2. **Review the changes**: Green lines show additions
3. **Try different view modes**: Unified diff, Side-by-side

## Part 5: The Review Process

### 5.1 Self-Review (Best Practice)

Before asking others to review, always review your own change:

1. **Read through your code** as if you're reviewing someone else's work
2. **Check for**:
   - Typos in comments
   - Code style consistency
   - Missing edge cases
   - Unclear variable names

### 5.2 Adding Reviewers

Since we're in development mode, let's simulate having multiple users:

1. **Add yourself as a reviewer**:
   - In the change page, find "Reviewers" section
   - Click "ADD REVIEWER"
   - Type your username
   - Click "ADD"

2. **Add a comment**:
   - Click "REPLY" button
   - Add a comment: "This looks good to me! The code is clean and well-documented."
   - Give it a score: +1 (Code-Review)
   - Click "SEND"

### 5.3 Simulating Team Review

In a real team environment, here's what would happen:

#### Reviewer 1 (Senior Developer) - Positive Review
```
Comment: "Good first submission! I like the clear documentation.
Just a few minor suggestions:

1. Consider adding a docstring to the main() function
2. The file could use a license header
3. Maybe add some error handling?

Overall: +1, but would like to see the docstring added."

Score: +1 (Code-Review)
```

#### Reviewer 2 (Team Lead) - Conditional Approval
```
Comment: "Looks good overall. Please address the docstring comment
from Jane, then this is ready to go.

Score: +2 (Code-Review) - conditional on addressing feedback"
```

## Part 6: Addressing Review Feedback

### 6.1 Making Changes Based on Feedback

Let's address the feedback about adding a docstring:

```powershell
# Edit the file
@"
#!/usr/bin/env python3
"""
Simple Hello World program for Gerrit tutorial

This module demonstrates basic Python functionality
and serves as a first example for code review workflow.
"""

def main():
    """
    Main function that prints greeting messages.
    
    This function demonstrates basic output operations
    and serves as the entry point for the program.
    """
    print("Hello, Gerrit World!")
    print("This is my first code review!")

if __name__ == "__main__":
    main()
"@ | Out-File -FilePath "hello.py" -Encoding UTF8
```

### 6.2 Creating a New Patch Set

```powershell
# Add the modified file
git add hello.py

# Commit with the SAME Change-Id
git commit --amend

# Your editor will open with the previous commit message
# Keep the Change-Id the same, but you can update the description:
```

Updated commit message:
```
Add Hello World Python program

This program demonstrates basic Python functionality
and serves as a first example for code review.

Updated to include proper docstrings for better
documentation as requested in code review.

The program simply prints greeting messages to
demonstrate the Gerrit review workflow.

Change-Id: I7a8b9c0d1e2f3g4h5i6j7k8l9m0n1o2p3q4r5s6t
```

### 6.3 Push the Updated Change

```powershell
# Push the updated change
git push origin HEAD:refs/for/master
```

Expected output:
```
remote: Processing changes: updated: 1, done
remote:
remote: SUCCESS
remote:
remote:   http://localhost:8080/c/test-project/+/1 Add Hello World Python program [UPDATED]
remote:
```

Notice it says **"updated: 1"** instead of **"new: 1"** - this created Patch Set 2!

### 6.4 Review the New Patch Set

1. **Refresh the change page** in your browser
2. **Notice "Patch Set 2"** is now selected
3. **View the differences** between patch sets
4. **Add a comment** about what you changed:

```
"Updated to address review feedback:
- Added comprehensive docstring to main() function
- Added module-level documentation
- Improved code documentation overall

Ready for final review!"
```

## Part 7: Final Approval and Submission

### 7.1 Final Review Scores

In a real scenario, reviewers would now give final approval:

```
Senior Developer: +2 (Code-Review)
"Perfect! All feedback addressed. Ready to submit."

CI System: +1 (Verified)
"Build successful, all tests pass."
```

### 7.2 Submitting the Change

Once you have the required approvals:

1. **Go to your change page**
2. **Click the "SUBMIT" button**
3. **Confirm submission**

The change will be merged into the master branch!

### 7.3 Verify the Merge

```powershell
# Pull the latest changes
git pull origin master

# Check the history
git log --oneline

# Your change should now be in master!
```

## Part 8: Understanding What Just Happened

### 8.1 The Complete Flow

Let's trace what happened:

1. **Created code** locally
2. **Committed** with Change-Id
3. **Pushed** to `refs/for/master` (created Change)
4. **Reviewed** through web interface
5. **Updated** based on feedback (new Patch Set)
6. **Approved** by reviewers
7. **Submitted** (merged to master)

### 8.2 Key Learnings

- **Change-Id** allowed updating the same change
- **Patch Sets** tracked different versions
- **Review scores** controlled approval workflow
- **Submission** merged to the target branch

## Common Issues and Solutions

### Issue 1: Missing Change-Id
**Error**: `missing Change-Id in message footer`
**Solution**:
```powershell
# Install the commit-msg hook properly
# Then amend your commit:
git commit --amend
# The hook will add the Change-Id
```

### Issue 2: Push Rejected
**Error**: `[remote rejected] HEAD -> refs/for/master (no new changes)`
**Solution**: You're trying to push the same change. Either:
- Make code modifications first, or
- Check if you already pushed this change

### Issue 3: Can't Submit
**Symptoms**: Submit button is disabled
**Causes**:
- Insufficient approvals (+2 Code-Review needed)
- Failed verification (-1 Verified)
- Merge conflicts

### Issue 4: Wrong Target Branch
**Error**: Pushed to wrong branch
**Solution**:
```powershell
# Abandon the wrong change in web interface
# Then push to correct branch:
git push origin HEAD:refs/for/correct-branch-name
```

## Best Practices You've Learned

### 1. Commit Messages
- **Clear, descriptive titles**
- **Detailed descriptions** explaining why, not just what
- **Proper formatting** (short title, blank line, detailed description)

### 2. Code Quality
- **Self-review** before pushing
- **Address feedback** promptly and thoroughly
- **Test your changes** before submitting

### 3. Review Process
- **Be responsive** to reviewer comments
- **Ask questions** if feedback is unclear
- **Update descriptions** when making significant changes

### 4. Gerrit-Specific
- **Always use** `refs/for/branch` for pushes
- **Keep the same Change-Id** when updating
- **Use amend commits** for updates, not new commits

## Quick Reference: Essential Commands

```powershell
# Clone project
git clone http://localhost:8080/project-name

# Create change
git add .
git commit -m "Description"
git push origin HEAD:refs/for/master

# Update change (after feedback)
git add .
git commit --amend  # Keep same Change-Id!
git push origin HEAD:refs/for/master
```

## What's Next?

Congratulations! You've successfully completed your first code review cycle in Gerrit. You now understand:

- How to create and push changes
- How the review process works
- How to address feedback with patch sets
- How changes get approved and submitted

In the next chapter, we'll explore advanced Gerrit features that will make you even more productive.

---

**Ready for advanced features?** Continue to [Chapter 5: Advanced Gerrit Features](../05-advanced-features/README.md)

## Chapter Summary

You've learned the complete Gerrit workflow:
- **Creating changes** with proper commit messages
- **Pushing to Gerrit** using `refs/for/branch`
- **Reviewing changes** in the web interface
- **Updating changes** with new patch sets
- **Getting approval** and submitting changes
- **Troubleshooting** common issues

This hands-on experience forms the foundation for all your future Gerrit work!

---

*Continue to [Chapter 5: Advanced Gerrit Features](../05-advanced-features/README.md)*
