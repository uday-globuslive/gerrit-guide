# Practical Gerrit Examples

## Real-World Scenarios with Step-by-Step Solutions

### Scenario 1: Your First Code Review Submission

**Context:** You're a new developer who needs to fix a typo in documentation.

**Step-by-Step:**

1. **Setup your environment:**
   ```bash
   # Clone the repository
   git clone https://gerrit.example.com/myproject
   cd myproject
   
   # Install commit-msg hook
   curl -Lo .git/hooks/commit-msg https://gerrit.example.com/tools/hooks/commit-msg
   chmod +x .git/hooks/commit-msg
   ```

2. **Make your change:**
   ```bash
   # Edit the file
   notepad README.md  # Fix typo: "recieve" → "receive"
   ```

3. **Commit and submit:**
   ```bash
   git add README.md
   git commit -m "Fix typo in README.md
   
   Changed 'recieve' to 'receive' in installation instructions.
   "
   
   git push origin HEAD:refs/for/main
   ```

4. **What happens next:**
   - Gerrit creates a change (e.g., Change 12345)
   - You get a URL to the change in Gerrit UI
   - Reviewers can now see and review your change

---

### Scenario 2: Handling Review Feedback

**Context:** A reviewer found issues with your code and left comments.

**Review Comments:**
```
Line 23: Consider using Optional<String> instead of null checks
Line 45: This method is too long, consider breaking it down
Line 67: Add JavaDoc comment for this public method
```

**Step-by-Step Response:**

1. **Address the feedback:**
   ```bash
   # Make the requested changes
   # Edit UserService.java:
   # - Line 23: Refactor to use Optional<String>
   # - Line 45: Split method into smaller methods
   # - Line 67: Add JavaDoc comment
   ```

2. **Update your change:**
   ```bash
   git add UserService.java
   git commit --amend  # This updates the existing commit
   git push origin HEAD:refs/for/main
   ```

3. **Respond to reviewer:**
   In Gerrit UI, add a comment:
   ```
   "Thanks for the review! I've addressed all the points:
   - Refactored to use Optional<String> for null safety
   - Split the large method into validateUser() and processUser()
   - Added comprehensive JavaDoc documentation
   
   Please take another look when you have time."
   ```

---

### Scenario 3: Testing Someone Else's Change

**Context:** Your colleague submitted a change that affects your area of expertise.

**Change Details:**
- Change number: 23456
- Title: "Add user authentication middleware"
- Affects: login functionality you maintain

**Step-by-Step Review:**

1. **Download the change:**
   ```bash
   # Get the change (found in Gerrit UI)
   git fetch origin refs/changes/56/23456/2 && git checkout FETCH_HEAD
   ```

2. **Test the change:**
   ```bash
   # Run the tests
   ./gradlew test
   
   # Start the application
   ./gradlew run
   
   # Manually test login functionality
   # - Try valid credentials
   # - Try invalid credentials  
   # - Test edge cases
   ```

3. **Leave meaningful feedback:**
   In Gerrit UI:
   ```
   "I tested this change thoroughly:
   
   ✅ All existing tests pass
   ✅ Manual testing with valid/invalid credentials works
   ✅ Code follows our security patterns
   
   One minor suggestion:
   Line 89: Consider adding a rate limiting mechanism to prevent brute force attacks.
   
   Otherwise looks good! +1"
   ```

4. **Return to your branch:**
   ```bash
   git checkout main
   ```

---

### Scenario 4: Resolving Merge Conflicts

**Context:** While your change was under review, someone else merged changes that conflict with yours.

**Error Message:**
```
Cannot merge due to conflicts with the following files:
- src/main/java/com/example/UserController.java
```

**Step-by-Step Resolution:**

1. **Fetch latest changes:**
   ```bash
   git fetch origin
   ```

2. **Rebase your change:**
   ```bash
   git rebase origin/main
   ```

3. **Resolve conflicts:**
   ```bash
   # Git will show conflict markers like:
   # <<<<<<< HEAD
   # their code
   # =======
   # your code
   # >>>>>>> your-commit
   
   # Edit the conflicted file
   notepad src/main/java/com/example/UserController.java
   
   # Remove conflict markers and merge the code appropriately
   ```

4. **Complete the rebase:**
   ```bash
   git add src/main/java/com/example/UserController.java
   git rebase --continue
   ```

5. **Update your change in Gerrit:**
   ```bash
   git push origin HEAD:refs/for/main
   ```

---

### Scenario 5: Working with Dependent Changes

**Context:** You need to make two changes where the second depends on the first.

**Changes:**
1. Add new utility method
2. Use the utility method in business logic

**Step-by-Step:**

1. **Submit first change:**
   ```bash
   # Make first change
   git add Utils.java
   git commit -m "Add validation utility method
   
   Add isValidEmail() method to Utils class for email validation.
   This will be used by user registration and profile update features.
   "
   
   git push origin HEAD:refs/for/main
   # This creates Change 34567
   ```

2. **Create second change (depends on first):**
   ```bash
   # Don't reset! Keep building on the same branch
   # Make second change
   git add UserService.java
   git commit -m "Use Utils.isValidEmail() in user registration
   
   Replace custom email validation with centralized utility method.
   This ensures consistent validation across the application.
   
   Depends-On: I1234567890abcdef (Change-Id of first change)
   "
   
   git push origin HEAD:refs/for/main
   # This creates Change 34568 that depends on 34567
   ```

3. **What happens:**
   - First change gets reviewed and merged
   - Second change automatically rebases and becomes ready for review
   - Gerrit shows the dependency relationship in the UI

---

### Scenario 6: Abandoning a Change

**Context:** You realized your approach was wrong and want to start over.

**Step-by-Step:**

1. **Abandon in Gerrit UI:**
   - Go to your change in Gerrit
   - Click "Abandon"
   - Add reason: "Switching to different approach based on team discussion"

2. **Clean up locally:**
   ```bash
   # Reset to clean state
   git reset --hard origin/main
   
   # Start fresh
   git checkout main
   git pull origin main
   ```

3. **Start new approach:**
   ```bash
   # Implement new solution
   # git add, commit, push as normal
   ```

---

### Scenario 7: Working with Multiple Reviewers

**Context:** Your change needs approval from both the code owner and security team.

**Gerrit Configuration:**
- Needs +2 from code owner
- Needs +1 from security team  
- No -2 votes allowed

**Step-by-Step:**

1. **Submit change with clear description:**
   ```bash
   git commit -m "Add OAuth2 integration for third-party login
   
   This change adds OAuth2 support for Google and GitHub login.
   
   Security considerations:
   - All tokens are encrypted at rest
   - Refresh tokens have 30-day expiration
   - Rate limiting applied to prevent abuse
   
   Testing:
   - Unit tests for all new methods
   - Integration tests for full OAuth flow
   - Security review requested
   
   Reviewers: @code-owner, @security-team
   "
   ```

2. **Track review progress:**
   - Code owner gives +2 (approved)
   - Security team gives +1 (looks good)
   - Change becomes mergeable

3. **Handle additional feedback:**
   If security team requests changes:
   ```bash
   # Make requested security improvements
   git add .
   git commit --amend
   git push origin HEAD:refs/for/main
   ```

---

## Quick Troubleshooting Guide

### Problem: "Missing Change-Id"
**Solution:**
```bash
# Install hook if not already installed
curl -Lo .git/hooks/commit-msg https://gerrit.example.com/tools/hooks/commit-msg
chmod +x .git/hooks/commit-msg

# Amend commit to generate Change-Id
git commit --amend
```

### Problem: "Cannot push to refs/for/main"
**Possible causes:**
1. **No push permissions:** Contact admin
2. **Wrong branch name:** Check if branch exists
3. **Malformed commit:** Check commit message format

### Problem: "Change already exists"
**Solution:**
```bash
# You're trying to push the same commit again
# Either abandon the existing change or amend your commit
git commit --amend
git push origin HEAD:refs/for/main
```

### Problem: "Merge conflicts"
**Solution:**
```bash
git fetch origin
git rebase origin/main
# Resolve conflicts
git add .
git rebase --continue
git push origin HEAD:refs/for/main
```

---

## Best Practices Summary

### Commit Messages
```
Good:
"Fix null pointer exception in user login

Added null check for user object before accessing properties.
This prevents crashes when invalid credentials are provided.

Bug: #12345
Test: Added unit test for null user scenario
"

Bad:
"fix bug"
```

### Review Comments
```
Good:
"Consider using a StringBuilder here for better performance when 
concatenating multiple strings in a loop (lines 45-52)."

Bad:
"This is wrong."
```

### Change Size
```
Good: 
- One logical change per commit
- Under 400 lines of code
- Single responsibility

Bad:
- Multiple unrelated changes
- Huge refactoring + new features
- Mixed bug fixes and features
```
