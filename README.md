# Gerrit Mastery Guide: The Essential 20% for 80% Results

## 🎯 What You'll Master

This guide focuses on the **most impactful Gerrit concepts** that will make you productive immediately:

1. **Installation & Setup** - Get Gerrit running quickly
2. **Core Workflow Understanding** - How Gerrit actually works
3. **Change Lifecycle** - From creation to merge
4. **Review Process** - Giving and receiving feedback
5. **Command Line Essentials** - The 5 commands you'll use daily
6. **Common Patterns** - Real-world scenarios you'll encounter

---

## 🚀 Quick Start: What is Gerrit?

**Gerrit in 30 seconds:**
- A code review tool built on top of Git
- Every change must be reviewed before merging
- Think of it as a "quality gate" for your code

**Key Mental Model:**
```
Developer → Gerrit → Reviewers → Merge
    ↓         ↓         ↓         ↓
  Write     Submit    Review   Integrate
   Code     Change    Code     to Main
```

---

## 📋 Table of Contents

1. [Installation & Environment Setup](#1-installation--environment-setup)
2. [Change-Based Workflow](#2-change-based-workflow)
3. [The Review Process](#3-the-review-process)
4. [Essential Commands](#4-essential-commands)
5. [Change States](#5-change-states)
6. [Merge Strategies](#6-merge-strategies)
7. [Common Scenarios](#7-common-scenarios)
8. [Troubleshooting](#8-troubleshooting)
9. [Practice Exercises](#9-practice-exercises)
10. [Quick Reference Card](#10-quick-reference-card)
11. [Next Steps](#11-next-steps)
12. [Resources](#12-resources)

---

## 🛠️ Installation & Environment Setup

### Option 1: Using Existing Gerrit Server (Recommended for Beginners)

### The Big Picture
Unlike GitHub's branch-based workflow, Gerrit works with **individual changes** (commits).

```
GitHub Workflow:
feature-branch → Pull Request → Merge entire branch

Gerrit Workflow:
single-commit → Change Request → Review → Merge single change
```

### Real-World Example
**Scenario:** You need to fix a bug in a login function.

**Traditional Git:**
```bash
git checkout -b fix-login-bug
# make changes
git commit -m "Fix login validation"
git push origin fix-login-bug
# Create PR on GitHub
```

**Gerrit Way:**
```bash
# Work directly on main branch (or any branch)
git checkout main
# make changes
git commit -m "Fix login validation"
git push origin HEAD:refs/for/main
# Gerrit automatically creates a change for review
```

### Key Insight
- **One commit = One change = One review**
- No branches needed for simple changes
- Each change gets a unique Change-Id

---

## 2. The Review Process

### The Review Lifecycle

```
1. DRAFT (optional) → 2. ACTIVE → 3. REVIEWED → 4. MERGED
                           ↓
                      5. ABANDONED
```

### Review Scores Explained

| Score | Meaning | Can Merge? |
|-------|---------|------------|
| +2 | Looks good to me, approved | ✅ Yes |
| +1 | Looks good to me, but someone else must approve | ❌ No |
| 0 | No score | ❌ No |
| -1 | I would prefer this is not merged | ❌ No |
| -2 | This shall not be merged | ❌ No |

### Real-World Review Example

**Bad Review Comment:**
```
"This code is wrong."
```

**Good Review Comment:**
```
"Consider using a switch statement instead of multiple if-else 
statements for better readability. Also, this might throw a 
NullPointerException if user is null on line 23."
```

---

## 3. Essential Commands

### The 5 Commands You'll Use 90% of the Time

#### 1. Submit a Change for Review
```bash
git push origin HEAD:refs/for/main
```
**When to use:** Every time you want to submit code for review

#### 2. Update an Existing Change
```bash
git commit --amend
git push origin HEAD:refs/for/main
```
**When to use:** After receiving review feedback

#### 3. Download a Change for Review
```bash
git fetch origin refs/changes/12/34/5 && git checkout FETCH_HEAD
```
**When to use:** When you want to test someone else's change

#### 4. Check Status
```bash
git log --oneline -5
```
**When to use:** To see recent commits and their Change-Ids

#### 5. Rebase Before Submit
```bash
git rebase origin/main
git push origin HEAD:refs/for/main
```
**When to use:** When your change conflicts with newer changes

---

## 4. Change States

### Understanding Change States

```
NEW → ACTIVE → MERGED
  ↓      ↓        ↑
DRAFT   ↓        ↑
       ABANDONED
```

### What Each State Means

- **NEW/DRAFT:** Change exists but not ready for review
- **ACTIVE:** Change is ready and under review
- **MERGED:** Change has been integrated into the target branch
- **ABANDONED:** Change has been discarded

### Real-World State Transitions

**Scenario:** You submit a change with a typo in the commit message.

1. **Submit change** → State: ACTIVE
2. **Realize typo** → Amend commit message
3. **Push updated change** → State: Still ACTIVE (same change, new version)
4. **Get +2 review** → State: Ready to merge
5. **Merge change** → State: MERGED

---

## 5. Merge Strategies

### The Three Ways Changes Get Merged

#### 1. Fast Forward (Preferred)
```
Before: A---B---C (main)
After:  A---B---C---D (main, your change)
```
**When it happens:** Your change is based on the latest main branch

#### 2. Merge Commit
```
Before: A---B---C (main)
        \
         D (your change)
After:  A---B---C---E (main)
         \         /
          D-------/
```
**When it happens:** Your change conflicts with newer changes

#### 3. Cherry Pick (Common)
```
Before: A---B---C---X (main, new changes)
        \
         D (your change, based on B)
After:  A---B---C---X---D' (main, your change rebased)
```
**When it happens:** Gerrit rebases your change automatically

---

## Common Scenarios

### Scenario 1: "My Change Won't Merge"

**Problem:** You see "Cannot merge due to conflicts"

**Solution:**
```bash
# 1. Fetch latest changes
git fetch origin

# 2. Rebase your change
git rebase origin/main

# 3. Resolve conflicts (if any)
git add .
git rebase --continue

# 4. Push updated change
git push origin HEAD:refs/for/main
```

### Scenario 2: "I Need to Update My Change"

**Problem:** Reviewer asked for changes

**Solution:**
```bash
# 1. Make the requested changes
# edit files...

# 2. Amend the commit (don't create new commit!)
git add .
git commit --amend

# 3. Push updated change
git push origin HEAD:refs/for/main
```

### Scenario 3: "I Want to Review Someone's Change"

**Problem:** How do I test someone else's code?

**Solution:**
```bash
# 1. Find the change number (e.g., 12345) in Gerrit UI
# 2. Download the change
git fetch origin refs/changes/45/12345/3 && git checkout FETCH_HEAD

# 3. Test the change
# run tests, try the feature...

# 4. Go back to your branch
git checkout main
```

---

## Troubleshooting

### Problem: "Missing Change-Id"

**Error message:** `missing Change-Id in commit message footer`

**Solution:**
```bash
# Install the commit-msg hook
curl -Lo .git/hooks/commit-msg http://your-gerrit-server/tools/hooks/commit-msg
chmod +x .git/hooks/commit-msg

# Amend your commit to add Change-Id
git commit --amend
```

### Problem: "Push Rejected"

**Error message:** `[remote rejected] HEAD -> refs/for/main`

**Common causes & solutions:**

1. **No permissions:** Contact your Gerrit admin
2. **Branch doesn't exist:** Check if you're pushing to the right branch
3. **Commit message issues:** Ensure proper format and Change-Id

---

## Practice Exercises

### Exercise 1: Submit Your First Change

1. Clone a repository
2. Make a small change (add a comment)
3. Commit with a good message
4. Push to Gerrit: `git push origin HEAD:refs/for/main`
5. Check the Gerrit UI for your change

### Exercise 2: Handle Review Feedback

1. Find a change that needs updates
2. Make the requested changes
3. Amend the commit: `git commit --amend`
4. Push the update: `git push origin HEAD:refs/for/main`
5. Verify the change updated in Gerrit UI

### Exercise 3: Review Someone's Change

1. Find an active change in Gerrit
2. Download it: `git fetch origin refs/changes/XX/YYYY/Z && git checkout FETCH_HEAD`
3. Test the change
4. Leave meaningful feedback in Gerrit UI
5. Give a score (+1, +2, -1, or -2)

---

## Quick Reference Card

### Essential Commands Cheat Sheet

```bash
# Submit new change
git push origin HEAD:refs/for/main

# Update existing change
git commit --amend
git push origin HEAD:refs/for/main

# Download change for testing
git fetch origin refs/changes/XX/YYYY/Z && git checkout FETCH_HEAD

# Rebase before submit
git rebase origin/main
git push origin HEAD:refs/for/main

# Check your commits
git log --oneline -5
```

### Review Scores Quick Guide

- **+2:** Approve (can merge)
- **+1:** Looks good (needs another +2)
- **-1:** Needs work (blocks merge)
- **-2:** Never merge (strong block)

---

## Next Steps

Once you've mastered these concepts, explore:

1. **Advanced workflows** (multi-commit changes, dependencies)
2. **Gerrit plugins** (CI integration, automated testing)
3. **Branch permissions** and access controls
4. **Custom labels** and review requirements

**Remember:** Master these 5 core concepts first, and you'll be productive with Gerrit immediately!

---

## Resources

- [Official Gerrit Documentation](https://gerrit-review.googlesource.com/Documentation/)
- [Gerrit User Guide](https://gerrit-review.googlesource.com/Documentation/intro-user.html)
- [Git Hooks for Gerrit](https://gerrit-review.googlesource.com/Documentation/cmd-hook-commit-msg.html)
