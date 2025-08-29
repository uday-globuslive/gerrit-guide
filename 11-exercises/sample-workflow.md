# Sample Use Case: Complete Review Workflow with Jenkins

This comprehensive example demonstrates how Gerrit works with Jenkins CI/CD in a real-world enterprise environment.

## Scenario: Adding a New Feature to a Web Application

**Team:** Sarah (Developer), Mike (Senior Developer), Lisa (DevOps Engineer)  
**Project:** E-commerce website  
**Task:** Add a shopping cart feature

## Step-by-Step Workflow

### 1. **Developer Creates Change (Sarah)**

```bash
# Sarah clones the repository
git clone https://gerrit.company.com/ecommerce-app
cd ecommerce-app

# Creates a new feature branch
git checkout -b feature/shopping-cart

# Makes code changes
echo "Shopping cart implementation" > src/cart.js
echo "Cart tests" > tests/cart.test.js

# Commits the changes
git add .
git commit -m "Add shopping cart functionality

- Implement cart.js with add/remove items
- Add comprehensive unit tests
- Update API documentation

Bug: JIRA-1234
Change-Id: I1234567890abcdef1234567890abcdef12345678"

# Pushes for review (not to master!)
git push origin HEAD:refs/for/master
```

### 2. **Gerrit Receives the Change**

When Sarah pushes, Gerrit automatically:
- Creates a new change request (e.g., Change #12345)
- Assigns a Change-Id for tracking
- Triggers Jenkins build pipeline
- Notifies reviewers via email

**Change Details in Gerrit:**
```
Change 12345: Add shopping cart functionality
Status: Work in Progress
Branch: master
Owner: Sarah Johnson <sarah@company.com>
Reviewers: Mike Thompson, Lisa Chen
Files: 2 files changed (+127, -3)
```

### 3. **Jenkins CI Pipeline Triggered**

Jenkins automatically runs when the change is uploaded:

```yaml
# Jenkinsfile pipeline
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                // Jenkins checks out the specific change
                sh 'git fetch origin $GERRIT_REFSPEC'
                sh 'git checkout FETCH_HEAD'
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm install'
                sh 'npm run build'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm run test'
                sh 'npm run test:integration'
                sh 'npm run lint'
            }
        }
        
        stage('Security Scan') {
            steps {
                sh 'npm audit'
                sh 'sonar-scanner'
            }
        }
        
        stage('Performance Test') {
            steps {
                sh 'npm run test:performance'
            }
        }
    }
    
    post {
        always {
            // Report results back to Gerrit
            publishTestResults testResultsPattern: 'test-results.xml'
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'coverage',
                reportFiles: 'index.html',
                reportName: 'Coverage Report'
            ])
        }
        
        success {
            // Vote +1 for Verified if all tests pass
            sh '''
                ssh -p 29418 jenkins@gerrit.company.com \
                gerrit review --verified +1 \
                --message "Build successful: All tests passed" \
                $GERRIT_CHANGE_NUMBER,$GERRIT_PATCHSET_NUMBER
            '''
        }
        
        failure {
            // Vote -1 for Verified if tests fail
            sh '''
                ssh -p 29418 jenkins@gerrit.company.com \
                gerrit review --verified -1 \
                --message "Build failed: Check console output for details" \
                $GERRIT_CHANGE_NUMBER,$GERRIT_PATCHSET_NUMBER
            '''
        }
    }
}
```

### 4. **Automated Verification Results**

After 5 minutes, Jenkins completes and votes:

**✅ Verified +1 (Jenkins)**
- ✅ Build: SUCCESS (2m 30s)
- ✅ Unit Tests: 47/47 passed
- ✅ Integration Tests: 12/12 passed
- ✅ Code Coverage: 94% (requirement: >90%)
- ✅ Linting: No issues
- ✅ Security Scan: No vulnerabilities
- ✅ Performance: API response < 200ms

### 5. **Human Code Review (Mike)**

Mike receives email notification and reviews the code:

```javascript
// Sarah's cart.js implementation
class ShoppingCart {
    constructor() {
        this.items = [];
    }
    
    addItem(product, quantity = 1) {
        // Mike spots a potential issue
        const existingItem = this.items.find(item => item.id === product.id);
        if (existingItem) {
            existingItem.quantity += quantity;
        } else {
            this.items.push({ ...product, quantity });
        }
    }
}
```

**Mike's Review Comments:**
```
File: src/cart.js, Line 8

💬 Mike Thompson: "Good implementation! However, we should validate 
that quantity is a positive number. What happens if someone passes 
a negative quantity?"

Suggestion:
if (quantity <= 0) {
    throw new Error('Quantity must be positive');
}

Also, consider using a Map for better performance with large carts.

Code-Review: 0 (needs revision)
```

### 6. **Sarah Addresses Feedback**

Sarah sees Mike's feedback and updates her code:

```bash
# Sarah makes the requested changes
vim src/cart.js  # Adds validation and improves performance

git add .
git commit --amend  # Amends the original commit
git push origin HEAD:refs/for/master  # Uploads new patchset
```

**Updated Implementation:**
```javascript
class ShoppingCart {
    constructor() {
        this.items = new Map();  // Better performance
    }
    
    addItem(product, quantity = 1) {
        if (quantity <= 0) {  // Added validation
            throw new Error('Quantity must be positive');
        }
        
        if (this.items.has(product.id)) {
            const item = this.items.get(product.id);
            item.quantity += quantity;
        } else {
            this.items.set(product.id, { ...product, quantity });
        }
    }
}
```

### 7. **Jenkins Re-runs Automatically**

New patchset triggers another Jenkins build:
- ✅ All tests still pass
- ✅ New validation tests added
- ✅ Performance improved

**Jenkins votes: Verified +1**

### 8. **Final Review and Approval**

Mike reviews the updated code:

```
💬 Mike Thompson: "Perfect! The validation is exactly what we needed, 
and the Map implementation is much more efficient. Great work!"

Code-Review: +2 ✅
```

Lisa (DevOps) also approves:
```
💬 Lisa Chen: "Deployment looks good. No infrastructure concerns."

Code-Review: +1 ✅
```

### 9. **Change Gets Submitted**

With required approvals:
- ✅ Verified +1 (Jenkins)
- ✅ Code-Review +2 (Mike)
- ✅ Code-Review +1 (Lisa)

Sarah clicks "Submit" and the change is merged to master.

### 10. **Post-Merge Actions**

Gerrit automatically:
- Merges the change to master branch
- Triggers production deployment pipeline
- Sends notification to team
- Updates JIRA ticket status
- Creates release notes entry

## Review Process Rating System

Gerrit uses a sophisticated rating system for reviews:

### **Verified Label (Automated)**
- **+1**: All automated tests pass, build successful
- **0**: No verification or tests pending
- **-1**: Tests failed or build broken

### **Code-Review Label (Human)**
- **+2**: "Looks good to me, approved" (Can submit)
- **+1**: "Looks good to me, but someone else must approve"
- **0**: "No score" (neutral, or work in progress)
- **-1**: "I would prefer this is not submitted as is"
- **-2**: "This shall not be submitted" (blocks submission)

### **Custom Labels (Configurable)**
- **QA-Review**: Quality assurance approval
- **Security-Review**: Security team approval
- **Performance-Review**: Performance impact assessment
- **Documentation**: Documentation updates required

## Advanced Workflow Features

### **Branch-Based Reviews**
```bash
# Different rules for different branches
master branch:    Requires +2 Code-Review + Verified +1
release branch:   Requires +2 Code-Review + QA-Review +1
hotfix branch:    Requires +1 Code-Review (expedited)
```

### **Conditional Submission**
```yaml
# Submit requirements (gerrit.config)
[submit-requirement "Code-Review"]
    submittableIf = label:Code-Review=MAX AND -label:Code-Review=MIN
    description = At least one maximum vote and no minimum votes

[submit-requirement "Verified"]
    submittableIf = label:Verified=+1
    description = Build and tests must pass
```

### **Automated Code Reviewers**
```bash
# Bot reviewers for specific checks
SonarQube Bot:     Code quality metrics
Security Bot:      Vulnerability scanning
Dependency Bot:    License and dependency checks
Style Bot:         Code formatting compliance
```

## Real-World Benefits

This workflow provides:

1. **Quality Gates**: Nothing reaches production without review
2. **Fast Feedback**: Automated tests catch issues in minutes
3. **Knowledge Sharing**: Senior developers mentor through reviews
4. **Audit Trail**: Complete history of all changes and decisions
5. **Risk Mitigation**: Multiple eyes on every change
6. **Continuous Learning**: Team members learn from each other's code

## Metrics and Reporting

Teams can track:
- **Review Time**: Average time from upload to submission
- **Rejection Rate**: Percentage of changes requiring rework
- **Test Coverage**: Code coverage trends over time
- **Review Participation**: Who's reviewing and how often
- **Hotfix Frequency**: Emergency changes bypassing normal process

## Key Takeaways

- **Automation reduces manual work** while maintaining quality
- **Multiple approval layers** ensure thorough review
- **Fast feedback loops** help developers learn quickly
- **Clear voting system** makes decisions transparent
- **Integration with CI/CD** provides comprehensive quality gates

This workflow demonstrates how Gerrit enables high-quality software development at scale while maintaining development velocity.
