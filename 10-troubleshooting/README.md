# Chapter 10: Troubleshooting and Maintenance

## Overview

This chapter provides comprehensive troubleshooting guides, maintenance procedures, and operational best practices for keeping your Gerrit installation running smoothly. From common issues to complex performance problems, we'll cover systematic approaches to diagnosis and resolution.

## Troubleshooting Methodology

### The GERRIT Troubleshooting Framework

```
G - Gather Information
E - Examine Logs
R - Reproduce the Issue
R - Review Configuration
I - Isolate Components
T - Test Solutions
```

### Diagnostic Information Collection

#### System Information Script

```bash
#!/bin/bash
# gerrit-diagnostic.sh - Comprehensive system diagnostics

GERRIT_SITE="/opt/gerrit/review_site"
OUTPUT_DIR="/tmp/gerrit-diagnostics-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$OUTPUT_DIR"

echo "Gerrit Diagnostic Information Collection"
echo "======================================="
echo "Output directory: $OUTPUT_DIR"

# System Information
echo "=== System Information ===" > "$OUTPUT_DIR/system-info.txt"
{
    echo "Date: $(date)"
    echo "Hostname: $(hostname)"
    echo "Uptime: $(uptime)"
    echo "Kernel: $(uname -a)"
    echo "Distribution: $(lsb_release -a 2>/dev/null || cat /etc/os-release)"
    echo ""
    echo "=== CPU Information ==="
    lscpu
    echo ""
    echo "=== Memory Information ==="
    free -h
    echo ""
    echo "=== Disk Usage ==="
    df -h
    echo ""
    echo "=== Network Interfaces ==="
    ip addr show
    echo ""
    echo "=== Network Connections ==="
    ss -tulpn | grep -E "(8080|29418|5432)"
} >> "$OUTPUT_DIR/system-info.txt"

# Gerrit Process Information
echo "=== Gerrit Process Information ===" > "$OUTPUT_DIR/gerrit-process.txt"
{
    GERRIT_PID=$(pgrep -f "GerritCodeReview")
    if [ -n "$GERRIT_PID" ]; then
        echo "Gerrit PID: $GERRIT_PID"
        echo ""
        echo "=== Process Details ==="
        ps -fp "$GERRIT_PID"
        echo ""
        echo "=== Memory Usage ==="
        cat "/proc/$GERRIT_PID/status" | grep -E "(VmSize|VmRSS|VmPeak)"
        echo ""
        echo "=== Open Files ==="
        lsof -p "$GERRIT_PID" | head -20
        echo ""
        echo "=== JVM Information ==="
        jstat -gc "$GERRIT_PID"
        echo ""
        jstat -gccapacity "$GERRIT_PID"
    else
        echo "Gerrit process not found"
    fi
} >> "$OUTPUT_DIR/gerrit-process.txt"

# Configuration Files
echo "=== Configuration Files ===" > "$OUTPUT_DIR/configuration.txt"
{
    echo "=== gerrit.config ==="
    if [ -f "$GERRIT_SITE/etc/gerrit.config" ]; then
        cat "$GERRIT_SITE/etc/gerrit.config"
    else
        echo "gerrit.config not found"
    fi
    echo ""
    echo "=== secure.config (passwords redacted) ==="
    if [ -f "$GERRIT_SITE/etc/secure.config" ]; then
        sed 's/password = .*/password = [REDACTED]/' "$GERRIT_SITE/etc/secure.config"
    else
        echo "secure.config not found"
    fi
} >> "$OUTPUT_DIR/configuration.txt"

# Recent Logs
echo "=== Recent Logs ===" > "$OUTPUT_DIR/recent-logs.txt"
{
    echo "=== Error Log (last 100 lines) ==="
    if [ -f "$GERRIT_SITE/logs/error_log" ]; then
        tail -100 "$GERRIT_SITE/logs/error_log"
    else
        echo "error_log not found"
    fi
    echo ""
    echo "=== Gerrit Log (last 100 lines) ==="
    if [ -f "$GERRIT_SITE/logs/gerrit.log" ]; then
        tail -100 "$GERRIT_SITE/logs/gerrit.log"
    else
        echo "gerrit.log not found"
    fi
} >> "$OUTPUT_DIR/recent-logs.txt"

# Database Connection Test
echo "=== Database Connectivity ===" > "$OUTPUT_DIR/database-test.txt"
{
    DB_HOST=$(grep "hostname" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
    DB_PORT=$(grep "port" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
    
    if [ -n "$DB_HOST" ] && [ -n "$DB_PORT" ]; then
        echo "Testing connection to $DB_HOST:$DB_PORT"
        if nc -z "$DB_HOST" "$DB_PORT"; then
            echo "✅ Database connection successful"
        else
            echo "❌ Database connection failed"
        fi
    else
        echo "Database configuration not found or using H2"
    fi
} >> "$OUTPUT_DIR/database-test.txt"

# Permissions Check
echo "=== Permissions Check ===" > "$OUTPUT_DIR/permissions.txt"
{
    echo "=== Gerrit Site Directory ==="
    ls -la "$GERRIT_SITE"
    echo ""
    echo "=== Git Repositories ==="
    ls -la "$GERRIT_SITE/git" | head -10
    echo ""
    echo "=== Cache Directory ==="
    ls -la "$GERRIT_SITE/cache" | head -10
} >> "$OUTPUT_DIR/permissions.txt"

# Create archive
tar -czf "$OUTPUT_DIR.tar.gz" -C "/tmp" "$(basename "$OUTPUT_DIR")"
echo ""
echo "Diagnostic information collected: $OUTPUT_DIR.tar.gz"
echo "Please attach this file when reporting issues"
```

## Part 1: Common Issues and Solutions

### 1.1 Startup and Connection Issues

#### Issue: Gerrit Won't Start

**Symptoms:**
- Gerrit service fails to start
- No response on web interface
- Error messages in logs

**Diagnostic Steps:**

```bash
# Check if Gerrit is running
ps aux | grep -i gerrit

# Check ports
netstat -tulpn | grep -E "(8080|29418)"

# Check recent logs
tail -f /opt/gerrit/review_site/logs/error_log

# Check Java version
java -version

# Check disk space
df -h /opt/gerrit
```

**Common Solutions:**

```bash
# 1. Port already in use
# Find what's using the port
lsof -i :8080
# Kill the process or change Gerrit's port

# 2. Insufficient permissions
# Fix ownership
chown -R gerrit:gerrit /opt/gerrit/review_site
chmod -R 755 /opt/gerrit/review_site

# 3. Java heap space issues
# Edit gerrit.config
[container]
    javaOptions = -Xmx4g

# 4. Database connection issues
# Test database connectivity
pg_isready -h db_host -p 5432

# 5. Corrupted index
# Rebuild search index
java -jar gerrit.war reindex -d /opt/gerrit/review_site
```

#### Issue: Database Connection Failures

**Symptoms:**
- "Cannot connect to database" errors
- Slow query responses
- Connection timeout errors

**Advanced Database Troubleshooting Script:**

```python
#!/usr/bin/env python3
# db-troubleshoot.py

import psycopg2
import time
import sys
from datetime import datetime

class DatabaseTroubleshooter:
    def __init__(self, host, port, database, username, password):
        self.host = host
        self.port = port
        self.database = database
        self.username = username
        self.password = password
        self.conn = None
    
    def test_connection(self):
        """Test basic database connectivity"""
        try:
            self.conn = psycopg2.connect(
                host=self.host,
                port=self.port,
                database=self.database,
                user=self.username,
                password=self.password,
                connect_timeout=10
            )
            print("✅ Database connection successful")
            return True
        except psycopg2.Error as e:
            print(f"❌ Database connection failed: {e}")
            return False
    
    def check_database_stats(self):
        """Check database statistics"""
        if not self.conn:
            return
        
        try:
            cursor = self.conn.cursor()
            
            # Check connection count
            cursor.execute("""
                SELECT count(*) as active_connections,
                       max(now() - query_start) as longest_running_query
                FROM pg_stat_activity 
                WHERE state = 'active' AND datname = %s;
            """, (self.database,))
            
            result = cursor.fetchone()
            print(f"Active connections: {result[0]}")
            print(f"Longest running query: {result[1]}")
            
            # Check database size
            cursor.execute("""
                SELECT pg_size_pretty(pg_database_size(%s)) as db_size;
            """, (self.database,))
            
            result = cursor.fetchone()
            print(f"Database size: {result[0]}")
            
            # Check slow queries
            cursor.execute("""
                SELECT query, calls, total_time, mean_time
                FROM pg_stat_statements
                WHERE mean_time > 1000
                ORDER BY mean_time DESC
                LIMIT 5;
            """)
            
            slow_queries = cursor.fetchall()
            if slow_queries:
                print("\nSlow queries (>1 second):")
                for query in slow_queries:
                    print(f"  - {query[0][:50]}... ({query[3]:.2f}ms avg)")
            
        except psycopg2.Error as e:
            print(f"Error checking database stats: {e}")
    
    def check_locks(self):
        """Check for database locks"""
        if not self.conn:
            return
        
        try:
            cursor = self.conn.cursor()
            cursor.execute("""
                SELECT blocked_locks.pid AS blocked_pid,
                       blocked_activity.usename AS blocked_user,
                       blocking_locks.pid AS blocking_pid,
                       blocking_activity.usename AS blocking_user,
                       blocked_activity.query AS blocked_statement
                FROM pg_catalog.pg_locks blocked_locks
                JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
                JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
                    AND blocking_locks.DATABASE IS NOT DISTINCT FROM blocked_locks.DATABASE
                    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
                    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
                    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
                    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
                    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
                    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
                    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
                    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
                    AND blocking_locks.pid != blocked_locks.pid
                JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
                WHERE NOT blocked_locks.GRANTED;
            """)
            
            locks = cursor.fetchall()
            if locks:
                print("\n⚠️ Database locks detected:")
                for lock in locks:
                    print(f"  - PID {lock[0]} blocked by PID {lock[2]}")
                    print(f"    Query: {lock[4][:100]}...")
            else:
                print("✅ No blocking locks found")
                
        except psycopg2.Error as e:
            print(f"Error checking locks: {e}")
    
    def run_maintenance(self):
        """Run database maintenance"""
        if not self.conn:
            return
        
        print("\n🔧 Running database maintenance...")
        
        try:
            cursor = self.conn.cursor()
            
            # Update statistics
            print("Updating table statistics...")
            cursor.execute("ANALYZE;")
            
            # Vacuum analyze
            print("Running vacuum analyze...")
            old_isolation = self.conn.isolation_level
            self.conn.set_isolation_level(psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT)
            cursor.execute("VACUUM ANALYZE;")
            self.conn.set_isolation_level(old_isolation)
            
            print("✅ Maintenance completed")
            
        except psycopg2.Error as e:
            print(f"Error during maintenance: {e}")
    
    def close(self):
        if self.conn:
            self.conn.close()

# Usage
if __name__ == "__main__":
    troubleshooter = DatabaseTroubleshooter(
        host="localhost",
        port=5432,
        database="reviewdb",
        username="gerrit",
        password="your_password"
    )
    
    if troubleshooter.test_connection():
        troubleshooter.check_database_stats()
        troubleshooter.check_locks()
        troubleshooter.run_maintenance()
    
    troubleshooter.close()
```

### 1.2 Performance Issues

#### Issue: Slow Web Interface

**Symptoms:**
- Long page load times
- Timeouts in browser
- High CPU usage

**Performance Analysis Script:**

```bash
#!/bin/bash
# performance-analysis.sh

GERRIT_PID=$(pgrep -f "GerritCodeReview")
ANALYSIS_TIME=60  # seconds

if [ -z "$GERRIT_PID" ]; then
    echo "Gerrit process not found"
    exit 1
fi

echo "Analyzing Gerrit performance for $ANALYSIS_TIME seconds..."
echo "PID: $GERRIT_PID"

# CPU usage monitoring
{
    echo "=== CPU Usage Over Time ==="
    for i in $(seq 1 12); do
        CPU=$(ps -p "$GERRIT_PID" -o %cpu --no-headers)
        echo "$(date '+%H:%M:%S'): CPU: ${CPU}%"
        sleep 5
    done
} &

# Memory usage monitoring
{
    echo "=== Memory Usage Over Time ==="
    for i in $(seq 1 12); do
        MEM=$(ps -p "$GERRIT_PID" -o %mem --no-headers)
        RSS=$(ps -p "$GERRIT_PID" -o rss --no-headers)
        echo "$(date '+%H:%M:%S'): MEM: ${MEM}% RSS: ${RSS}KB"
        sleep 5
    done
} &

# JVM monitoring
{
    echo "=== JVM GC Analysis ==="
    jstat -gc "$GERRIT_PID" 5s 12 | while read line; do
        echo "$(date '+%H:%M:%S'): $line"
    done
} &

# Thread analysis
{
    echo "=== Thread Count ==="
    for i in $(seq 1 12); do
        THREADS=$(ls /proc/"$GERRIT_PID"/task | wc -l)
        echo "$(date '+%H:%M:%S'): Threads: $THREADS"
        sleep 5
    done
} &

# Wait for monitoring to complete
wait

echo "Performance analysis completed"

# Generate recommendations
echo ""
echo "=== Performance Recommendations ==="

# Check heap usage
HEAP_INFO=$(jstat -gc "$GERRIT_PID" | tail -1)
HEAP_USED=$(echo "$HEAP_INFO" | awk '{print ($3+$4+$6+$8)/1024}')
HEAP_TOTAL=$(jstat -gccapacity "$GERRIT_PID" | tail -1 | awk '{print ($4+$5+$7+$8)/1024}')
HEAP_PERCENT=$(echo "scale=2; $HEAP_USED/$HEAP_TOTAL*100" | bc)

if (( $(echo "$HEAP_PERCENT > 80" | bc -l) )); then
    echo "⚠️  High heap usage (${HEAP_PERCENT}%) - consider increasing heap size"
fi

# Check GC frequency
GC_COUNT=$(echo "$HEAP_INFO" | awk '{print $12+$14}')
if [ "$GC_COUNT" -gt 1000 ]; then
    echo "⚠️  High GC count ($GC_COUNT) - may indicate memory pressure"
fi

# Check thread count
THREAD_COUNT=$(ls /proc/"$GERRIT_PID"/task | wc -l)
if [ "$THREAD_COUNT" -gt 500 ]; then
    echo "⚠️  High thread count ($THREAD_COUNT) - check for thread leaks"
fi
```

#### Issue: Slow Git Operations

**Git Performance Troubleshooting:**

```bash
#!/bin/bash
# git-performance-check.sh

GERRIT_SITE="/opt/gerrit/review_site"
GIT_DIR="$GERRIT_SITE/git"

echo "Git Performance Analysis"
echo "======================="

# Check repository sizes
echo "=== Large Repositories ==="
find "$GIT_DIR" -name "*.git" -type d | while read repo; do
    size=$(du -sh "$repo" | cut -f1)
    echo "$size $repo"
done | sort -hr | head -10

# Check for repositories needing maintenance
echo ""
echo "=== Repositories Needing Maintenance ==="
find "$GIT_DIR" -name "*.git" -type d | while read repo; do
    if [ -d "$repo/objects" ]; then
        loose_objects=$(find "$repo/objects" -type f | wc -l)
        pack_files=$(find "$repo/objects/pack" -name "*.pack" 2>/dev/null | wc -l)
        
        if [ "$loose_objects" -gt 1000 ] || [ "$pack_files" -gt 50 ]; then
            echo "⚠️  $repo: $loose_objects loose objects, $pack_files pack files"
        fi
    fi
done

# Git maintenance script
echo ""
echo "=== Running Git Maintenance ==="

# Function to maintain a repository
maintain_repo() {
    local repo=$1
    echo "Maintaining $repo..."
    
    cd "$repo" || return
    
    # Garbage collection
    git gc --aggressive --prune=now
    
    # Repack for better compression
    git repack -ad
    
    # Update server info
    git update-server-info
    
    echo "✅ Completed: $repo"
}

# Export function for parallel execution
export -f maintain_repo

# Find all repositories and maintain them
find "$GIT_DIR" -name "*.git" -type d | head -5 | xargs -I {} -P 2 bash -c 'maintain_repo "$@"' _ {}

echo ""
echo "Git maintenance completed"
```

### 1.3 Authentication and Authorization Issues

#### Issue: LDAP Authentication Failures

**LDAP Troubleshooting Script:**

```python
#!/usr/bin/env python3
# ldap-troubleshoot.py

import ldap
import ssl
import sys
from datetime import datetime

class LDAPTroubleshooter:
    def __init__(self, server_uri, bind_dn, bind_password, base_dn):
        self.server_uri = server_uri
        self.bind_dn = bind_dn
        self.bind_password = bind_password
        self.base_dn = base_dn
        self.conn = None
    
    def test_connection(self):
        """Test LDAP connection"""
        try:
            print(f"Testing connection to {self.server_uri}")
            
            # Initialize connection
            self.conn = ldap.initialize(self.server_uri)
            
            # Set LDAP options
            self.conn.set_option(ldap.OPT_REFERRALS, 0)
            self.conn.set_option(ldap.OPT_PROTOCOL_VERSION, 3)
            self.conn.set_option(ldap.OPT_X_TLS_REQUIRE_CERT, ldap.OPT_X_TLS_NEVER)
            
            # Test simple bind
            self.conn.simple_bind_s(self.bind_dn, self.bind_password)
            print("✅ LDAP connection successful")
            return True
            
        except ldap.INVALID_CREDENTIALS:
            print("❌ Invalid credentials")
            return False
        except ldap.SERVER_DOWN:
            print("❌ LDAP server is down or unreachable")
            return False
        except ldap.LDAPError as e:
            print(f"❌ LDAP error: {e}")
            return False
    
    def test_user_search(self, username):
        """Test user search"""
        if not self.conn:
            return False
        
        try:
            search_filter = f"(uid={username})"
            print(f"Searching for user: {search_filter}")
            
            result = self.conn.search_s(
                self.base_dn,
                ldap.SCOPE_SUBTREE,
                search_filter,
                ['cn', 'mail', 'uid']
            )
            
            if result:
                print("✅ User found:")
                for dn, attrs in result:
                    print(f"  DN: {dn}")
                    for attr, values in attrs.items():
                        print(f"  {attr}: {[v.decode('utf-8') for v in values]}")
                return True
            else:
                print("❌ User not found")
                return False
                
        except ldap.LDAPError as e:
            print(f"❌ Search error: {e}")
            return False
    
    def test_group_search(self, group_name):
        """Test group search"""
        if not self.conn:
            return False
        
        try:
            search_filter = f"(cn={group_name})"
            print(f"Searching for group: {search_filter}")
            
            result = self.conn.search_s(
                self.base_dn,
                ldap.SCOPE_SUBTREE,
                search_filter,
                ['cn', 'member', 'description']
            )
            
            if result:
                print("✅ Group found:")
                for dn, attrs in result:
                    print(f"  DN: {dn}")
                    for attr, values in attrs.items():
                        print(f"  {attr}: {[v.decode('utf-8') for v in values[:3]]}...")
                return True
            else:
                print("❌ Group not found")
                return False
                
        except ldap.LDAPError as e:
            print(f"❌ Group search error: {e}")
            return False
    
    def test_user_authentication(self, username, password):
        """Test user authentication"""
        try:
            # Find user DN first
            search_filter = f"(uid={username})"
            result = self.conn.search_s(
                self.base_dn,
                ldap.SCOPE_SUBTREE,
                search_filter,
                ['dn']
            )
            
            if not result:
                print(f"❌ User {username} not found")
                return False
            
            user_dn = result[0][0]
            print(f"Found user DN: {user_dn}")
            
            # Test authentication
            test_conn = ldap.initialize(self.server_uri)
            test_conn.set_option(ldap.OPT_REFERRALS, 0)
            test_conn.set_option(ldap.OPT_PROTOCOL_VERSION, 3)
            test_conn.simple_bind_s(user_dn, password)
            
            print("✅ User authentication successful")
            test_conn.unbind()
            return True
            
        except ldap.INVALID_CREDENTIALS:
            print("❌ Invalid user credentials")
            return False
        except ldap.LDAPError as e:
            print(f"❌ Authentication error: {e}")
            return False
    
    def diagnose_ldap_config(self):
        """Diagnose LDAP configuration issues"""
        print("\n=== LDAP Configuration Diagnosis ===")
        
        # Check SSL/TLS
        if self.server_uri.startswith('ldaps://'):
            print("✅ Using LDAPS (SSL)")
        elif self.server_uri.startswith('ldap://'):
            print("⚠️  Using plain LDAP (not encrypted)")
        
        # Check server response time
        try:
            start_time = datetime.now()
            self.conn.search_s(self.base_dn, ldap.SCOPE_BASE, '(objectClass=*)', [''])
            response_time = (datetime.now() - start_time).total_seconds()
            print(f"Server response time: {response_time:.2f} seconds")
            
            if response_time > 5:
                print("⚠️  Slow LDAP response time")
            
        except ldap.LDAPError as e:
            print(f"❌ Server response test failed: {e}")
    
    def close(self):
        if self.conn:
            self.conn.unbind()

# Usage
if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python ldap-troubleshoot.py <username> [password]")
        sys.exit(1)
    
    username = sys.argv[1]
    password = sys.argv[2] if len(sys.argv) > 2 else None
    
    troubleshooter = LDAPTroubleshooter(
        server_uri="ldap://ldap.company.com:389",
        bind_dn="cn=gerrit,ou=service-accounts,dc=company,dc=com",
        bind_password="bind_password",
        base_dn="dc=company,dc=com"
    )
    
    if troubleshooter.test_connection():
        troubleshooter.test_user_search(username)
        
        if password:
            troubleshooter.test_user_authentication(username, password)
        
        troubleshooter.diagnose_ldap_config()
    
    troubleshooter.close()
```

## Part 2: Maintenance Procedures

### 2.1 Regular Maintenance Tasks

#### Daily Maintenance Script

```bash
#!/bin/bash
# daily-maintenance.sh

GERRIT_SITE="/opt/gerrit/review_site"
LOG_FILE="/var/log/gerrit/maintenance.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log() {
    echo "[$DATE] $1" | tee -a "$LOG_FILE"
}

log "Starting daily maintenance"

# Check disk space
DISK_USAGE=$(df "$GERRIT_SITE" | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt 80 ]; then
    log "WARNING: Disk usage is ${DISK_USAGE}%"
    # Clean up old logs
    find "$GERRIT_SITE/logs" -name "*.log.*" -mtime +7 -delete
    log "Cleaned up old log files"
fi

# Check Gerrit process
if ! pgrep -f "GerritCodeReview" > /dev/null; then
    log "ERROR: Gerrit process not running"
    # Attempt to restart
    systemctl start gerrit
    sleep 10
    if pgrep -f "GerritCodeReview" > /dev/null; then
        log "Gerrit restarted successfully"
    else
        log "ERROR: Failed to restart Gerrit"
    fi
fi

# Check cache sizes
CACHE_SIZE=$(du -sh "$GERRIT_SITE/cache" | cut -f1)
log "Current cache size: $CACHE_SIZE"

# Check error log for issues
ERROR_COUNT=$(grep -c "ERROR" "$GERRIT_SITE/logs/error_log" 2>/dev/null || echo "0")
if [ "$ERROR_COUNT" -gt 10 ]; then
    log "WARNING: $ERROR_COUNT errors in error log"
fi

# Database connection test
if command -v psql >/dev/null; then
    DB_HOST=$(grep "hostname" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
    if [ -n "$DB_HOST" ]; then
        if pg_isready -h "$DB_HOST" >/dev/null 2>&1; then
            log "Database connection OK"
        else
            log "WARNING: Database connection failed"
        fi
    fi
fi

log "Daily maintenance completed"
```

#### Weekly Maintenance Script

```bash
#!/bin/bash
# weekly-maintenance.sh

GERRIT_SITE="/opt/gerrit/review_site"
LOG_FILE="/var/log/gerrit/weekly-maintenance.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log() {
    echo "[$DATE] $1" | tee -a "$LOG_FILE"
}

log "Starting weekly maintenance"

# Stop Gerrit for maintenance
log "Stopping Gerrit for maintenance"
systemctl stop gerrit

# Backup before maintenance
BACKUP_DIR="/backup/gerrit/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

log "Creating backup"
tar -czf "$BACKUP_DIR/gerrit-site.tar.gz" -C "$(dirname "$GERRIT_SITE")" "$(basename "$GERRIT_SITE")"

# Cache cleanup
log "Cleaning caches"
rm -rf "$GERRIT_SITE/cache"/*
mkdir -p "$GERRIT_SITE/cache"

# Git repository maintenance
log "Running git maintenance on repositories"
find "$GERRIT_SITE/git" -name "*.git" -type d | head -10 | while read repo; do
    log "Maintaining $repo"
    cd "$repo" || continue
    git gc --auto
    git repack -ad
done

# Database maintenance (if PostgreSQL)
DB_HOST=$(grep "hostname" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
if [ -n "$DB_HOST" ]; then
    log "Running database maintenance"
    psql -h "$DB_HOST" -U gerrit -d reviewdb -c "VACUUM ANALYZE;" 2>/dev/null || log "Database maintenance failed"
fi

# Reindex search
log "Reindexing search"
java -jar "$GERRIT_SITE/bin/gerrit.war" reindex -d "$GERRIT_SITE" --verbose

# Start Gerrit
log "Starting Gerrit"
systemctl start gerrit

# Wait for startup
sleep 30

# Verify startup
if pgrep -f "GerritCodeReview" > /dev/null; then
    log "Gerrit started successfully"
else
    log "ERROR: Gerrit failed to start after maintenance"
fi

log "Weekly maintenance completed"
```

### 2.2 Backup and Recovery

#### Comprehensive Backup Script

```bash
#!/bin/bash
# gerrit-backup.sh

GERRIT_SITE="/opt/gerrit/review_site"
BACKUP_ROOT="/backup/gerrit"
RETENTION_DAYS=30
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$BACKUP_ROOT/$DATE"

# Database configuration
DB_HOST=$(grep "hostname" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
DB_PORT=$(grep "port" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
DB_NAME=$(grep "database" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
DB_USER=$(grep "username" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

create_backup_dir() {
    mkdir -p "$BACKUP_DIR"
    if [ $? -ne 0 ]; then
        log "ERROR: Failed to create backup directory"
        exit 1
    fi
}

backup_database() {
    if [ -n "$DB_HOST" ]; then
        log "Backing up database"
        pg_dump -h "$DB_HOST" -p "${DB_PORT:-5432}" -U "$DB_USER" -d "$DB_NAME" \
                --clean --create --format=custom \
                > "$BACKUP_DIR/database.dump"
        
        if [ $? -eq 0 ]; then
            log "Database backup completed"
        else
            log "ERROR: Database backup failed"
            return 1
        fi
    else
        log "Skipping database backup (H2 or not configured)"
    fi
}

backup_git_repositories() {
    log "Backing up Git repositories"
    tar -czf "$BACKUP_DIR/git-repositories.tar.gz" -C "$GERRIT_SITE" git/
    
    if [ $? -eq 0 ]; then
        log "Git repositories backup completed"
    else
        log "ERROR: Git repositories backup failed"
        return 1
    fi
}

backup_configuration() {
    log "Backing up configuration"
    tar -czf "$BACKUP_DIR/configuration.tar.gz" -C "$GERRIT_SITE" etc/
    
    if [ $? -eq 0 ]; then
        log "Configuration backup completed"
    else
        log "ERROR: Configuration backup failed"
        return 1
    fi
}

backup_index() {
    log "Backing up search index"
    if [ -d "$GERRIT_SITE/index" ]; then
        tar -czf "$BACKUP_DIR/index.tar.gz" -C "$GERRIT_SITE" index/
        log "Search index backup completed"
    else
        log "No search index found"
    fi
}

backup_plugins() {
    log "Backing up plugins"
    if [ -d "$GERRIT_SITE/plugins" ]; then
        tar -czf "$BACKUP_DIR/plugins.tar.gz" -C "$GERRIT_SITE" plugins/
        log "Plugins backup completed"
    else
        log "No plugins directory found"
    fi
}

create_metadata() {
    log "Creating backup metadata"
    cat > "$BACKUP_DIR/metadata.txt" << EOF
Backup Date: $(date)
Gerrit Site: $GERRIT_SITE
Hostname: $(hostname)
Gerrit Version: $(java -jar "$GERRIT_SITE/bin/gerrit.war" version 2>/dev/null || echo "Unknown")
Database Host: ${DB_HOST:-"Not configured"}
Database Name: ${DB_NAME:-"Not configured"}
Backup Size: $(du -sh "$BACKUP_DIR" | cut -f1)
EOF
}

cleanup_old_backups() {
    log "Cleaning up old backups (older than $RETENTION_DAYS days)"
    find "$BACKUP_ROOT" -type d -name "20*" -mtime +$RETENTION_DAYS -exec rm -rf {} + 2>/dev/null
    log "Cleanup completed"
}

verify_backup() {
    log "Verifying backup integrity"
    
    # Check if all expected files exist
    local errors=0
    
    if [ -n "$DB_HOST" ] && [ ! -f "$BACKUP_DIR/database.dump" ]; then
        log "ERROR: Database backup missing"
        errors=$((errors + 1))
    fi
    
    if [ ! -f "$BACKUP_DIR/git-repositories.tar.gz" ]; then
        log "ERROR: Git repositories backup missing"
        errors=$((errors + 1))
    fi
    
    if [ ! -f "$BACKUP_DIR/configuration.tar.gz" ]; then
        log "ERROR: Configuration backup missing"
        errors=$((errors + 1))
    fi
    
    # Test archive integrity
    for archive in "$BACKUP_DIR"/*.tar.gz; do
        if [ -f "$archive" ]; then
            if ! tar -tzf "$archive" >/dev/null 2>&1; then
                log "ERROR: Archive $archive is corrupted"
                errors=$((errors + 1))
            fi
        fi
    done
    
    if [ $errors -eq 0 ]; then
        log "Backup verification successful"
        return 0
    else
        log "Backup verification failed with $errors errors"
        return 1
    fi
}

# Main backup process
log "Starting Gerrit backup"

create_backup_dir

# Create hot backup (no service interruption)
backup_database
backup_git_repositories
backup_configuration
backup_index
backup_plugins
create_metadata

# Verify backup
if verify_backup; then
    log "Backup completed successfully: $BACKUP_DIR"
    cleanup_old_backups
else
    log "Backup completed with errors: $BACKUP_DIR"
    exit 1
fi

# Calculate total backup size
TOTAL_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
log "Total backup size: $TOTAL_SIZE"
```

#### Recovery Script

```bash
#!/bin/bash
# gerrit-restore.sh

usage() {
    echo "Usage: $0 <backup_directory> [--database-only|--config-only|--git-only]"
    echo "Example: $0 /backup/gerrit/20241201_120000"
    exit 1
}

if [ $# -lt 1 ]; then
    usage
fi

BACKUP_DIR="$1"
RESTORE_TYPE="full"

case "$2" in
    --database-only) RESTORE_TYPE="database" ;;
    --config-only) RESTORE_TYPE="config" ;;
    --git-only) RESTORE_TYPE="git" ;;
esac

GERRIT_SITE="/opt/gerrit/review_site"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

validate_backup() {
    if [ ! -d "$BACKUP_DIR" ]; then
        log "ERROR: Backup directory does not exist: $BACKUP_DIR"
        exit 1
    fi
    
    if [ ! -f "$BACKUP_DIR/metadata.txt" ]; then
        log "ERROR: Backup metadata not found"
        exit 1
    fi
    
    log "Backup validation successful"
}

stop_gerrit() {
    log "Stopping Gerrit service"
    systemctl stop gerrit
    
    # Wait for process to stop
    for i in {1..30}; do
        if ! pgrep -f "GerritCodeReview" >/dev/null; then
            break
        fi
        sleep 1
    done
    
    if pgrep -f "GerritCodeReview" >/dev/null; then
        log "ERROR: Gerrit process still running"
        exit 1
    fi
}

restore_database() {
    if [ -f "$BACKUP_DIR/database.dump" ]; then
        log "Restoring database"
        
        # Get database configuration
        DB_HOST=$(grep "hostname" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
        DB_PORT=$(grep "port" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
        DB_NAME=$(grep "database" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
        DB_USER=$(grep "username" "$GERRIT_SITE/etc/gerrit.config" | cut -d'=' -f2 | tr -d ' ')
        
        # Restore database
        pg_restore -h "$DB_HOST" -p "${DB_PORT:-5432}" -U "$DB_USER" \
                  --clean --create -d postgres "$BACKUP_DIR/database.dump"
        
        if [ $? -eq 0 ]; then
            log "Database restore completed"
        else
            log "ERROR: Database restore failed"
            exit 1
        fi
    else
        log "No database backup found"
    fi
}

restore_configuration() {
    if [ -f "$BACKUP_DIR/configuration.tar.gz" ]; then
        log "Restoring configuration"
        
        # Backup current config
        if [ -d "$GERRIT_SITE/etc" ]; then
            mv "$GERRIT_SITE/etc" "$GERRIT_SITE/etc.backup.$(date +%Y%m%d_%H%M%S)"
        fi
        
        # Restore configuration
        tar -xzf "$BACKUP_DIR/configuration.tar.gz" -C "$GERRIT_SITE/"
        
        if [ $? -eq 0 ]; then
            log "Configuration restore completed"
        else
            log "ERROR: Configuration restore failed"
            exit 1
        fi
    else
        log "No configuration backup found"
    fi
}

restore_git_repositories() {
    if [ -f "$BACKUP_DIR/git-repositories.tar.gz" ]; then
        log "Restoring Git repositories"
        
        # Backup current repositories
        if [ -d "$GERRIT_SITE/git" ]; then
            mv "$GERRIT_SITE/git" "$GERRIT_SITE/git.backup.$(date +%Y%m%d_%H%M%S)"
        fi
        
        # Restore repositories
        tar -xzf "$BACKUP_DIR/git-repositories.tar.gz" -C "$GERRIT_SITE/"
        
        if [ $? -eq 0 ]; then
            log "Git repositories restore completed"
        else
            log "ERROR: Git repositories restore failed"
            exit 1
        fi
    else
        log "No Git repositories backup found"
    fi
}

restore_index() {
    if [ -f "$BACKUP_DIR/index.tar.gz" ]; then
        log "Restoring search index"
        
        # Remove current index
        rm -rf "$GERRIT_SITE/index"
        
        # Restore index
        tar -xzf "$BACKUP_DIR/index.tar.gz" -C "$GERRIT_SITE/"
        
        log "Search index restore completed"
    else
        log "No search index backup found - will rebuild on startup"
    fi
}

restore_plugins() {
    if [ -f "$BACKUP_DIR/plugins.tar.gz" ]; then
        log "Restoring plugins"
        
        # Remove current plugins
        rm -rf "$GERRIT_SITE/plugins"
        
        # Restore plugins
        tar -xzf "$BACKUP_DIR/plugins.tar.gz" -C "$GERRIT_SITE/"
        
        log "Plugins restore completed"
    else
        log "No plugins backup found"
    fi
}

fix_permissions() {
    log "Fixing file permissions"
    chown -R gerrit:gerrit "$GERRIT_SITE"
    chmod -R 755 "$GERRIT_SITE"
    chmod 600 "$GERRIT_SITE/etc/secure.config" 2>/dev/null || true
}

start_gerrit() {
    log "Starting Gerrit service"
    systemctl start gerrit
    
    # Wait for startup
    for i in {1..60}; do
        if pgrep -f "GerritCodeReview" >/dev/null; then
            log "Gerrit started successfully"
            return 0
        fi
        sleep 1
    done
    
    log "ERROR: Gerrit failed to start"
    exit 1
}

# Main restore process
log "Starting Gerrit restore from: $BACKUP_DIR"
log "Restore type: $RESTORE_TYPE"

validate_backup

# Show backup metadata
if [ -f "$BACKUP_DIR/metadata.txt" ]; then
    log "Backup metadata:"
    cat "$BACKUP_DIR/metadata.txt"
fi

# Confirm restore
read -p "Continue with restore? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
    log "Restore cancelled"
    exit 0
fi

stop_gerrit

case "$RESTORE_TYPE" in
    "full")
        restore_database
        restore_configuration
        restore_git_repositories
        restore_index
        restore_plugins
        ;;
    "database")
        restore_database
        ;;
    "config")
        restore_configuration
        ;;
    "git")
        restore_git_repositories
        ;;
esac

fix_permissions
start_gerrit

log "Restore completed successfully"
```

This troubleshooting and maintenance chapter provides comprehensive tools and procedures for keeping Gerrit running smoothly in production environments. The scripts and procedures covered here address the most common issues and provide systematic approaches to problem resolution.

## Chapter Summary

You've learned comprehensive troubleshooting and maintenance:

- **Systematic diagnostic approaches** with automated tools
- **Common issue resolution** for startup, performance, and authentication
- **Performance analysis** and optimization techniques
- **Regular maintenance procedures** for daily and weekly tasks
- **Backup and recovery strategies** with automated scripts
- **Monitoring and alerting** for proactive issue detection

These tools and procedures ensure reliable Gerrit operations in enterprise environments.

---

**Ready for practical exercises?** Continue to [Chapter 11: Practical Exercises](../11-exercises/README.md)

---

*Continue to [Chapter 11: Practical Exercises](../11-exercises/README.md)*
