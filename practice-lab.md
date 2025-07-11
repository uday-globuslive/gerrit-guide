# Gerrit Practice Lab

## 🎯 Hands-On Exercises to Master Gerrit

These exercises progress from basic to advanced. Complete them in order for best results.

---

## Exercise 1: Setup and First Change (Beginner)

### Goal
Learn how to set up Gerrit and submit your first change.

### Prerequisites
- Git installed
- Access to a Gerrit server (or use a local setup)

### Steps

1. **Clone a repository:**
   ```bash
   git clone https://gerrit.example.com/practice-repo
   cd practice-repo
   ```

2. **Install commit-msg hook:**
   ```bash
   curl -Lo .git/hooks/commit-msg https://gerrit.example.com/tools/hooks/commit-msg
   chmod +x .git/hooks/commit-msg
   ```

3. **Make a simple change:**
   ```bash
   # Create or edit a file
   echo "Hello Gerrit!" > hello.txt
   
   # Stage and commit
   git add hello.txt
   git commit -m "Add hello.txt file
   
   This is my first change submitted to Gerrit for review.
   "
   ```

4. **Submit to Gerrit:**
   ```bash
   git push origin HEAD:refs/for/main
   ```

### Success Criteria
- [ ] Change appears in Gerrit UI
- [ ] Change has a Change-Id
- [ ] You can see the change details in browser

### Common Issues
- **Missing Change-Id:** Install the commit-msg hook
- **Push rejected:** Check permissions or branch name
- **Authentication failed:** Verify your credentials

---

## Exercise 2: Handle Review Feedback (Intermediate)

### Goal
Learn how to update a change based on review feedback.

### Setup
Use the change from Exercise 1 or create a new one.

### Steps

1. **Create a change that needs improvement:**
   ```bash
   # Create a file with intentional issues
   cat > calculator.py << 'EOF'
   def add(a, b):
       result = a + b
       return result
   
   def divide(a, b):
       return a / b  # This has a bug!
   EOF
   
   git add calculator.py
   git commit -m "Add calculator functions"
   git push origin HEAD:refs/for/main
   ```

2. **Simulate review feedback:**
   In Gerrit UI, add these comments (or ask someone to):
   - "Line 7: Need to handle division by zero"
   - "Consider adding type hints for better code clarity"
   - "Add docstrings for the functions"

3. **Address the feedback:**
   ```bash
   # Fix the issues
   cat > calculator.py << 'EOF'
   def add(a: int, b: int) -> int:
       """Add two numbers and return the result."""
       result = a + b
       return result
   
   def divide(a: int, b: int) -> float:
       """Divide two numbers and return the result."""
       if b == 0:
           raise ValueError("Cannot divide by zero")
       return a / b
   EOF
   ```

4. **Update the change:**
   ```bash
   git add calculator.py
   git commit --amend
   git push origin HEAD:refs/for/main
   ```

### Success Criteria
- [ ] Change updated in Gerrit (same change number, new patch set)
- [ ] All feedback addressed
- [ ] Change ready for final review

---

## Exercise 3: Review Someone Else's Change (Intermediate)

### Goal
Learn how to properly review and test changes submitted by others.

### Setup
Work with a partner or use a demo change.

### Steps

1. **Find a change to review:**
   - Go to Gerrit UI
   - Find an "Active" change
   - Note the change number (e.g., 12345)

2. **Download the change:**
   ```bash
   # Replace XX/YYYY/Z with actual change reference
   git fetch origin refs/changes/45/12345/1 && git checkout FETCH_HEAD
   ```

3. **Test the change:**
   ```bash
   # If it's code, run tests
   python -m pytest  # or appropriate test command
   
   # If it's a script, run it
   python calculator.py
   
   # Manual testing
   # Try different inputs, edge cases, etc.
   ```

4. **Leave detailed feedback:**
   In Gerrit UI, add comments like:
   ```
   "I tested this change with the following scenarios:
   
   ✅ Normal cases: add(2, 3) returns 5
   ✅ Edge cases: divide(10, 0) raises ValueError as expected
   ✅ All unit tests pass
   
   Code looks good! One suggestion:
   Line 12: Consider using f-strings for better readability
   
   +1 (Looks good to me, but someone else must approve)"
   ```

5. **Return to your branch:**
   ```bash
   git checkout main
   ```

### Success Criteria
- [ ] Successfully downloaded and tested the change
- [ ] Left meaningful, constructive feedback
- [ ] Gave appropriate score (+1, +2, -1, or -2)

---

## Exercise 4: Handle Merge Conflicts (Advanced)

### Goal
Learn how to resolve conflicts when your change conflicts with others.

### Setup
This requires two conflicting changes. Work with a partner or simulate.

### Steps

1. **Create two conflicting changes:**
   
   **Change A (yours):**
   ```bash
   # Edit config.yml
   echo "version: 1.0" > config.yml
   echo "debug: true" >> config.yml
   
   git add config.yml
   git commit -m "Add debug configuration"
   git push origin HEAD:refs/for/main
   ```

   **Change B (someone else's - simulate):**
   ```bash
   # In another terminal or ask partner
   git checkout main
   git pull origin main
   
   echo "version: 1.0" > config.yml
   echo "logging: enabled" >> config.yml
   
   git add config.yml
   git commit -m "Add logging configuration"
   git push origin HEAD:refs/for/main
   
   # This change gets merged first
   ```

2. **Your change now conflicts:**
   ```bash
   # Try to rebase
   git fetch origin
   git rebase origin/main
   
   # Git will show conflicts in config.yml
   ```

3. **Resolve the conflicts:**
   ```bash
   # Edit config.yml to merge both changes
   cat > config.yml << 'EOF'
   version: 1.0
   debug: true
   logging: enabled
   EOF
   
   git add config.yml
   git rebase --continue
   ```

4. **Update your change:**
   ```bash
   git push origin HEAD:refs/for/main
   ```

### Success Criteria
- [ ] Successfully resolved merge conflicts
- [ ] Both features preserved in final version
- [ ] Change ready for review again

---

## Exercise 5: Work with Dependent Changes (Advanced)

### Goal
Learn how to create changes that depend on other changes.

### Steps

1. **Create first change (foundation):**
   ```bash
   # Create a utility module
   cat > utils.py << 'EOF'
   def validate_email(email):
       """Validate email format."""
       import re
       pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
       return re.match(pattern, email) is not None
   EOF
   
   git add utils.py
   git commit -m "Add email validation utility
   
   This utility will be used by user registration and profile features.
   "
   git push origin HEAD:refs/for/main
   ```

2. **Create second change (depends on first):**
   ```bash
   # Don't reset! Build on the same branch
   cat > user_service.py << 'EOF'
   from utils import validate_email
   
   def register_user(username, email):
       """Register a new user."""
       if not validate_email(email):
           raise ValueError("Invalid email format")
       
       # Registration logic here
       print(f"User {username} registered with email {email}")
   EOF
   
   git add user_service.py
   git commit -m "Add user registration service
   
   Uses the email validation utility for input validation.
   "
   git push origin HEAD:refs/for/main
   ```

3. **Observe the dependency:**
   - Check Gerrit UI
   - See how second change depends on first
   - Notice how merging first change affects second

### Success Criteria
- [ ] Two changes created with proper dependency
- [ ] Gerrit shows dependency relationship
- [ ] Changes can be merged in correct order

---

## Exercise 6: Advanced Review Scenarios (Expert)

### Goal
Practice complex review scenarios you'll encounter in real projects.

### Scenario A: Large Refactoring

1. **Create a large change:**
   ```bash
   # Create multiple files to simulate large refactoring
   mkdir -p src/models src/services src/utils
   
   # Create several files (simulate real refactoring)
   echo "class User: pass" > src/models/user.py
   echo "class UserService: pass" > src/services/user_service.py
   echo "def helper(): pass" > src/utils/helpers.py
   
   git add src/
   git commit -m "Refactor user management system
   
   This large refactoring reorganizes the user management code:
   - Moved models to dedicated package
   - Created service layer for business logic
   - Extracted common utilities
   
   Breaking changes:
   - Import paths have changed
   - UserManager class split into UserService and User model
   
   Migration guide:
   - Update imports from 'user' to 'src.models.user'
   - Replace UserManager with UserService
   "
   git push origin HEAD:refs/for/main
   ```

2. **Review considerations:**
   - How do you review such a large change?
   - What questions would you ask?
   - How would you test it?

### Scenario B: Security-Critical Change

1. **Create a security-sensitive change:**
   ```bash
   cat > auth.py << 'EOF'
   import hashlib
   import secrets
   
   def hash_password(password):
       """Hash password with salt."""
       salt = secrets.token_hex(16)
       password_hash = hashlib.pbkdf2_hmac('sha256', 
                                         password.encode(), 
                                         salt.encode(), 
                                         100000)
       return salt + password_hash.hex()
   
   def verify_password(password, stored_hash):
       """Verify password against stored hash."""
       salt = stored_hash[:32]
       stored_password_hash = stored_hash[32:]
       password_hash = hashlib.pbkdf2_hmac('sha256',
                                         password.encode(),
                                         salt.encode(),
                                         100000)
       return password_hash.hex() == stored_password_hash
   EOF
   
   git add auth.py
   git commit -m "Add secure password hashing
   
   Implements PBKDF2 with SHA-256 for password hashing.
   Uses 100,000 iterations and random salt for security.
   
   Security review required.
   "
   git push origin HEAD:refs/for/main
   ```

2. **Security review checklist:**
   - [ ] Uses cryptographically secure algorithms
   - [ ] Proper salt generation
   - [ ] Sufficient iteration count
   - [ ] No hardcoded secrets
   - [ ] Timing attack resistance

### Success Criteria
- [ ] Understand how to approach large changes
- [ ] Know what to look for in security reviews
- [ ] Can provide meaningful feedback on complex changes

---

## Exercise 7: Gerrit Administration Basics (Expert)

### Goal
Learn basic Gerrit administration tasks.

### Prerequisites
- Admin access to Gerrit instance
- OR use docker-compose setup for local Gerrit

### Steps

1. **Set up local Gerrit (optional):**
   ```bash
   # Using docker-compose
   cat > docker-compose.yml << 'EOF'
   version: '3'
   services:
     gerrit:
       image: gerritcodereview/gerrit:latest
       ports:
         - "8080:8080"
         - "29418:29418"
       volumes:
         - gerrit_data:/var/gerrit
         - ./gerrit.config:/var/gerrit/etc/gerrit.config
       
   volumes:
     gerrit_data:
   EOF
   
   docker-compose up -d
   ```

2. **Configure project settings:**
   - Create new project
   - Set up branch permissions
   - Configure review requirements

3. **Manage users and groups:**
   - Add users to groups
   - Set up code owners
   - Configure email notifications

### Success Criteria
- [ ] Successfully set up Gerrit instance
- [ ] Created project with proper permissions
- [ ] Configured review workflow

---

## Final Challenge: Real-World Simulation

### Goal
Simulate a complete development workflow with multiple developers.

### Scenario
You're working on a team project with the following requirements:
- Feature: Add user authentication
- Requirements: Code review, testing, documentation
- Team: 3 developers, 1 reviewer, 1 maintainer

### Your Tasks

1. **Plan the work:**
   - Break down into multiple changes
   - Identify dependencies
   - Assign responsibilities

2. **Implement your part:**
   - Create well-structured commits
   - Write good commit messages
   - Submit for review

3. **Review others' work:**
   - Test changes thoroughly
   - Provide constructive feedback
   - Ensure code quality

4. **Integrate everything:**
   - Resolve any conflicts
   - Ensure all tests pass
   - Merge in correct order

### Success Criteria
- [ ] All features implemented
- [ ] All changes properly reviewed
- [ ] Clean commit history
- [ ] No breaking changes
- [ ] Complete documentation

---

## Self-Assessment Checklist

After completing these exercises, you should be able to:

### Basic Skills
- [ ] Set up Gerrit hooks and configuration
- [ ] Submit changes for review
- [ ] Respond to review feedback
- [ ] Update changes with amendments

### Intermediate Skills
- [ ] Review others' changes effectively
- [ ] Handle merge conflicts
- [ ] Work with dependent changes
- [ ] Use Gerrit UI efficiently

### Advanced Skills
- [ ] Handle complex review scenarios
- [ ] Manage large refactorings
- [ ] Understand security implications
- [ ] Troubleshoot common issues

### Expert Skills
- [ ] Configure Gerrit projects
- [ ] Set up workflows and permissions
- [ ] Train other developers
- [ ] Optimize team processes

---

## Next Steps

Once you've mastered these exercises:

1. **Practice with real projects**
2. **Learn advanced Gerrit features** (plugins, custom labels, etc.)
3. **Study your team's specific workflows**
4. **Contribute to improving team processes**
5. **Consider becoming a Gerrit administrator**

Remember: The key to mastering Gerrit is consistent practice and understanding the underlying Git concepts!
