# Chapter 3: Understanding Gerrit Concepts

## Overview

Before diving into using Gerrit, it's crucial to understand its core concepts and terminology. Think of this chapter as learning the "language" of Gerrit - once you understand these concepts, everything else will make much more sense.

## The Gerrit Workflow - Visual Overview

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Developer     │    │     Gerrit      │    │  Git Repository │
│                 │    │                 │    │                 │
│ 1. Write Code   │───►│ 2. Code Review  │───►│ 3. Final Commit │
│ 2. Commit       │    │ 3. Feedback     │    │                 │
│ 3. Push         │    │ 4. Approval     │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Core Concepts Explained

### 1. Change (The Foundation)

A **Change** is the fundamental unit in Gerrit. It represents a set of modifications to the codebase.

#### What is a Change?
- Contains one or more file modifications
- Has a unique Change-Id (generated automatically)
- Can be updated with new versions (patch sets)
- Goes through the review process

#### Change Lifecycle
```
Created → Under Review → Approved → Submitted → Merged
   ↑                        ↓
   └─── Needs Revision ←────┘
```

#### Example Change Structure
```
Change #12345: "Fix user login bug"
├── Description: "Resolves issue where users couldn't log in with special characters"
├── Files Modified:
│   ├── src/auth/login.js (15 lines changed)
│   └── tests/auth_test.js (8 lines added)
├── Author: john.doe@company.com
├── Reviewers: jane.smith@company.com, bob.wilson@company.com
└── Status: "Needs Review"
```

### 2. Patch Set (Versions of a Change)

A **Patch Set** is a specific version of a change. When you update your code based on review feedback, you create a new patch set.

#### How Patch Sets Work
- **Patch Set 1**: Your initial submission
- **Patch Set 2**: After addressing first round of feedback
- **Patch Set 3**: After addressing additional feedback
- And so on...

#### Visual Example
```
Change #12345: "Fix user login bug"
├── Patch Set 1: Initial implementation
│   └── Status: "Needs work" (reviewer found issues)
├── Patch Set 2: Fixed issues from PS1
│   └── Status: "Needs review" (waiting for approval)
└── Patch Set 3: Final version
    └── Status: "Ready to submit" (approved)
```

### 3. Review Process

The **Review Process** is where team members examine your code and provide feedback.

#### Review States
- **Work in Progress (WIP)**: Not ready for review
- **Ready for Review**: Waiting for reviewer assignment
- **Under Review**: Reviewers are examining the code
- **Needs Revision**: Changes required before approval
- **Approved**: Ready to be merged
- **Submitted**: Successfully merged

#### Review Flow Diagram
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Submit    │───►│   Review    │───►│   Approve   │
│   Change    │    │   Process   │    │   & Submit  │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       │                   ▼                   ▼
       │            ┌─────────────┐    ┌─────────────┐
       │            │   Request   │    │   Merged    │
       └────────────│  Changes    │    │  to Master  │
                    └─────────────┘    └─────────────┘
```

### 4. Reviewers and Roles

#### Types of Users
- **Author**: Person who created the change
- **Reviewer**: Person assigned to review the change
- **Owner**: Project owner with special permissions
- **Admin**: System administrator

#### Review Permissions
- **Read**: Can view changes and comments
- **Comment**: Can add comments and suggestions
- **Approve**: Can approve or reject changes
- **Submit**: Can merge approved changes

### 5. Scoring System

Gerrit uses a scoring system to indicate the status of reviews:

#### Code Review Scores
- **+2**: Looks good to me, approved (typically from senior developers)
- **+1**: Looks good to me, but someone else must approve
- **0**: No score (just commenting)
- **-1**: I would prefer this is not merged as is
- **-2**: This shall not be merged (blocks submission)

#### Verified Scores (if enabled)
- **+1**: Verified (tests pass, build successful)
- **0**: No verification
- **-1**: Fails verification (tests fail, build broken)

#### Visual Scoring Example
```
Change #12345: "Fix user login bug"
┌─────────────────┐
│ Code Review     │
├─────────────────┤
│ Jane Smith: +2  │ ✅ Senior dev approval
│ Bob Wilson: +1  │ ✅ Regular approval
│ Alice Brown: 0  │ 💬 Just commented
└─────────────────┘
┌─────────────────┐
│ Verified        │
├─────────────────┤
│ CI System: +1   │ ✅ Tests passed
└─────────────────┘
Status: Ready to Submit! 🎉
```

### 6. Change-Id and Commit Messages

#### The Change-Id
- Unique identifier for tracking changes
- Automatically generated by Gerrit's commit-msg hook
- Allows updating changes with new patch sets

#### Proper Commit Message Format
```
Short description (50 chars or less)

Longer description explaining what and why, not how.
Can span multiple lines and paragraphs.

Bug: #12345
Change-Id: I1234567890abcdef1234567890abcdef12345678
```

#### Example Good Commit Message
```
Fix user authentication with special characters

Users with email addresses containing special characters
(like + or .) were unable to log in due to improper
URL encoding in the authentication service.

This change adds proper URL encoding/decoding to handle
all valid email address formats according to RFC 5322.

Bug: #4567
Tested: Unit tests added for special character scenarios
Change-Id: I7a8b9c0d1e2f3g4h5i6j7k8l9m0n1o2p3q4r5s6t
```

### 7. Projects and Branches

#### Projects
- Container for related repositories
- Has its own access controls and settings
- Can inherit from parent projects

#### Branches
- Different versions or variants of the code
- Common branches: `master`, `develop`, `release-1.0`
- Each branch can have different review requirements

#### Project Hierarchy Example
```
Company Root Project
├── Web Applications
│   ├── Frontend App
│   │   ├── master
│   │   ├── develop
│   │   └── feature-branches
│   └── API Service
│       ├── master
│       └── v2-development
└── Mobile Applications
    ├── iOS App
    └── Android App
```

### 8. Access Controls and Permissions

#### Permission Levels
- **No Access**: Cannot see the project
- **Read**: Can view code and changes
- **Push**: Can create new changes for review
- **Submit**: Can merge approved changes
- **Owner**: Full administrative control

#### Group-Based Permissions
```
Project: Frontend App
├── Administrators Group
│   └── Permissions: All
├── Senior Developers Group
│   └── Permissions: Read, Push, Submit, +2 approval
├── Developers Group
│   └── Permissions: Read, Push, +1 approval
└── QA Team Group
    └── Permissions: Read, Verify (+1/-1)
```

## Gerrit vs. Traditional Git Workflow

### Traditional Git (Direct Push)
```bash
git add .
git commit -m "Fix bug"
git push origin master  # Direct to repository
```

### Gerrit Workflow
```bash
git add .
git commit -m "Fix bug"
git push origin HEAD:refs/for/master  # To Gerrit for review
# ↓ Review process happens here ↓
# ↓ After approval ↓
# Gerrit merges to master
```

## Understanding the Web Interface

### Main Dashboard Sections

1. **Changes**
   - **Outgoing**: Changes you created
   - **Incoming**: Changes you need to review
   - **Recently Closed**: Recently completed changes

2. **Projects**
   - Browse all projects
   - Create new projects
   - Manage project settings

3. **People**
   - View all users
   - Manage groups
   - User settings

4. **Plugins**
   - Available plugins
   - Plugin configuration

### Change Detail Page Layout
```
┌─────────────────────────────────────────────────────┐
│ Change #12345: Fix user login bug                   │
├─────────────────────────────────────────────────────┤
│ Author: John Doe                  Status: Under Review │
│ Project: frontend-app             Branch: master    │
├─────────────────────────────────────────────────────┤
│ Description:                                        │
│ Resolves issue where users couldn't log in...      │
├─────────────────────────────────────────────────────┤
│ Files Changed (2):                                  │
│ M src/auth/login.js     (+15, -3)                 │
│ A tests/auth_test.js    (+8, -0)                  │
├─────────────────────────────────────────────────────┤
│ Reviewers:                                          │
│ Jane Smith (+2) ✅                                  │
│ Bob Wilson (+1) ✅                                  │
├─────────────────────────────────────────────────────┤
│ Comments (3):                                       │
│ [View detailed comments and discussions]            │
└─────────────────────────────────────────────────────┘
```

## Key Terminology Quick Reference

| Term | Definition |
|------|------------|
| **Change** | A set of code modifications under review |
| **Patch Set** | A specific version of a change |
| **Change-Id** | Unique identifier for a change |
| **Review** | Process of examining and commenting on code |
| **Score** | Numerical rating (+2, +1, 0, -1, -2) |
| **Submit** | Merge an approved change |
| **Abandon** | Cancel a change without merging |
| **Cherry-pick** | Apply a change to a different branch |
| **Rebase** | Update a change with latest target branch |

## Mental Model: Gerrit as a Post Office

Think of Gerrit like a post office for code:

1. **You write a letter** (code change)
2. **Put it in an envelope** (commit with Change-Id)
3. **Send it to the post office** (push to Gerrit)
4. **Post office inspects it** (review process)
5. **Gets approval stamps** (reviewer approvals)
6. **Finally delivered** (merged to target branch)

If there are issues:
- **Return to sender** (needs revision)
- **Add more stamps** (more approvals needed)
- **Update address** (rebase or modify change)

## Best Practices for Understanding

### 1. Start Small
- Begin with simple, single-file changes
- Focus on understanding the review process
- Don't worry about complex workflows initially

### 2. Use the Web Interface
- Explore all sections of the Gerrit web UI
- Click through different change states
- Examine how other team members' changes look

### 3. Practice the Terminology
- Use correct Gerrit terms when discussing changes
- Read change descriptions and comments from others
- Ask questions when terminology is unclear

### 4. Observe Before Contributing
- Watch how experienced team members use Gerrit
- Read their commit messages and review comments
- Understand your team's specific conventions

## Common Misconceptions

### ❌ "Gerrit is just like GitHub Pull Requests"
**Reality**: While similar in purpose, Gerrit's change-based model is different from branch-based pull requests.

### ❌ "I need to create a new branch for each change"
**Reality**: Gerrit works with commits directly; branches are optional.

### ❌ "Once I push, I can't modify my change"
**Reality**: You can update changes by pushing new patch sets.

### ❌ "Reviews slow down development"
**Reality**: Reviews catch bugs early and improve overall development speed.

## Quiz: Test Your Understanding

1. What's the difference between a Change and a Patch Set?
2. What does a +2 score mean?
3. What is a Change-Id and why is it important?
4. What happens when a change gets "submitted"?
5. How do you update a change based on review feedback?

### Answers:
1. A Change is the overall code modification; a Patch Set is a specific version of that change
2. +2 means "Looks good to me, approved" - typically the highest approval level
3. Change-Id is a unique identifier that allows updating changes with new patch sets
4. The change gets merged into the target branch
5. Make code modifications and push a new patch set with the same Change-Id

## What's Next?

Now that you understand Gerrit's core concepts, you're ready to create your first code review! In the next chapter, we'll walk through the complete process of making a change, getting it reviewed, and having it merged.

---

**Ready for hands-on practice?** Continue to [Chapter 4: Your First Code Review](../04-first-review/README.md)

## Chapter Summary

You've learned:
- **Changes** are the fundamental units in Gerrit
- **Patch Sets** represent versions of changes
- The **Review Process** and scoring system
- **Roles and permissions** in Gerrit
- **Key terminology** and concepts
- How Gerrit differs from traditional Git workflows

These concepts form the foundation for everything you'll do in Gerrit. Take your time to understand them - they'll make the practical exercises much easier!

---

*Continue to [Chapter 4: Your First Code Review](../04-first-review/README.md)*
