# Gerrit Workflow Visualizations

## 1. Change Lifecycle Flowchart

```
┌─────────────┐
│   Write     │
│    Code     │
└─────┬───────┘
      │
      ▼
┌─────────────┐
│   Commit    │
│   Changes   │
└─────┬───────┘
      │
      ▼
┌─────────────┐
│    Push     │
│refs/for/main│
└─────┬───────┘
      │
      ▼
┌─────────────┐    ┌─────────────┐
│   ACTIVE    │───▶│   REVIEW    │
│   Change    │    │   Process   │
└─────┬───────┘    └─────┬───────┘
      │                  │
      ▼                  ▼
┌─────────────┐    ┌─────────────┐
│  ABANDONED  │    │   MERGED    │
│             │    │             │
└─────────────┘    └─────────────┘
```

## 2. Review Process Flow

```
Developer              Reviewer              Maintainer
    │                      │                      │
    ├─ Submit Change ─────▶│                      │
    │                      ├─ Review Code ──────▶│
    │                      │                      │
    │◀─ Request Changes ───┤                      │
    │                      │                      │
    ├─ Update Change ─────▶│                      │
    │                      ├─ Approve (+2) ─────▶│
    │                      │                      ├─ Merge
    │                      │                      │
    │◀─ Change Merged ─────┴──────────────────────┤
    │                                             │
```

## 3. Git vs Gerrit Comparison

```
Traditional Git Flow:
──────────────────────
main     A───B───C───D───E
              \         /
feature        F───G───H

Gerrit Flow:
───────────
main     A───B───C───D───E
              │   │   │
              │   │   └─ Change 3 (reviewed)
              │   └───── Change 2 (reviewed)  
              └───────── Change 1 (reviewed)

Each letter represents a reviewed, approved change
```

## 4. Change States Visualization

```
                 ┌─────────────┐
                 │    DRAFT    │
                 │ (Optional)  │
                 └─────┬───────┘
                       │
                       ▼
                 ┌─────────────┐
                 │   ACTIVE    │
                 │ (Under      │
                 │  Review)    │
                 └─────┬───────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  ABANDONED  │ │   MERGED    │ │   UPDATED   │
│             │ │             │ │   (New PS)  │
└─────────────┘ └─────────────┘ └─────┬───────┘
                                      │
                                      ▼
                                ┌─────────────┐
                                │   ACTIVE    │
                                │  (Again)    │
                                └─────────────┘
```

## 5. Merge Strategies Visualization

### Fast Forward Merge
```
Before:
main    A───B───C
             \
              D (your change)

After:
main    A───B───C───D
```

### Merge Commit
```
Before:
main    A───B───C───X
             \
              D (your change)

After:
main    A───B───C───X───M
             \         /
              D───────/
```

### Cherry Pick (Rebase)
```
Before:
main    A───B───C───X
             \
              D (your change)

After:
main    A───B───C───X───D'
                        (rebased)
```

## 6. Review Scoring System

```
┌─────────────────────────────────────────────────────┐
│                 Review Scores                       │
├─────────────────────────────────────────────────────┤
│  +2  │ ✅ Looks good to me, approved                │
│  +1  │ ⚠️  Looks good to me, but someone else       │
│      │    must approve                              │
│   0  │ ⚪ No score                                  │
│  -1  │ ❌ I would prefer this is not merged         │
│  -2  │ 🚫 This shall not be merged                  │
└─────────────────────────────────────────────────────┘

Merge Requirements:
• At least one +2 score
• No -2 scores
• No unresolved comments (usually)
```

## 7. Command Flow Diagram

```
Local Repository          Gerrit Server
       │                       │
       │ git push origin        │
       │ HEAD:refs/for/main     │
       ├──────────────────────▶│
       │                       │
       │                    ┌──┴──┐
       │                    │Change│
       │                    │Created│
       │                    └──┬──┘
       │                       │
       │                    ┌──▼──┐
       │                    │Review│
       │                    │Process│
       │                    └──┬──┘
       │                       │
       │                    ┌──▼──┐
       │                    │Merge│
       │                    │Ready│
       │                    └──┬──┘
       │                       │
       │ git pull origin main  │
       │◀──────────────────────┤
       │                       │
```

## 8. Common Workflow Patterns

### Pattern 1: Simple Change
```
1. git checkout main
2. git pull origin main
3. [make changes]
4. git add .
5. git commit -m "Fix bug in login"
6. git push origin HEAD:refs/for/main
7. [wait for review]
8. [change gets merged]
```

### Pattern 2: Change with Updates
```
1. git checkout main
2. git pull origin main  
3. [make changes]
4. git add .
5. git commit -m "Add new feature"
6. git push origin HEAD:refs/for/main
7. [receive review feedback]
8. [make requested changes]
9. git add .
10. git commit --amend
11. git push origin HEAD:refs/for/main
12. [change gets approved and merged]
```

### Pattern 3: Reviewing Someone's Change
```
1. [find change in Gerrit UI]
2. git fetch origin refs/changes/XX/YYYY/Z
3. git checkout FETCH_HEAD
4. [test the change]
5. [leave comments in Gerrit UI]
6. [give score: +1, +2, -1, or -2]
7. git checkout main
```
