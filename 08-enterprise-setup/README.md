# Chapter 8: Enterprise Setup and Configuration

## Overview

Moving from a basic Gerrit installation to an enterprise-grade deployment requires careful planning, robust security, scalability considerations, and comprehensive management strategies. This chapter covers everything you need to build a production-ready Gerrit system that can serve large organizations with thousands of developers.

## Enterprise Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Enterprise Gerrit Architecture              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│  │    Load     │    │   Reverse   │    │   CDN/WAF   │        │
│  │  Balancer   │───►│    Proxy    │───►│ (Optional)  │        │
│  │   (HAProxy) │    │   (Nginx)   │    │             │        │
│  └─────────────┘    └─────────────┘    └─────────────┘        │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │               Gerrit Application Servers                │   │
│  │                                                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │   │
│  │  │   Gerrit    │  │   Gerrit    │  │   Gerrit    │    │   │
│  │  │  Server 1   │  │  Server 2   │  │  Server 3   │    │   │
│  │  │             │  │             │  │             │    │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Data Layer                           │   │
│  │                                                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │   │
│  │  │ PostgreSQL  │  │    Git      │  │   LDAP/AD   │    │   │
│  │  │  Database   │  │ Repositories│  │    Auth     │    │   │
│  │  │  (Primary)  │  │             │  │             │    │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘    │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │   │
│  │  │ PostgreSQL  │  │ Elasticsearch│  │   Backup    │    │   │
│  │  │ (Replica)   │  │   Search    │  │   Storage   │    │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                 Monitoring & Logging                    │   │
│  │                                                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │   │
│  │  │ Prometheus  │  │    ELK      │  │   Grafana   │    │   │
│  │  │   Metrics   │  │   Logging   │  │ Dashboards  │    │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘    │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Part 1: Infrastructure Planning

### 1.1 Hardware Requirements

#### Small Enterprise (100-500 developers)
```yaml
Gerrit Server:
  CPU: 8 cores, 2.4GHz+
  RAM: 16GB
  Storage: 500GB SSD (OS) + 2TB SSD (Git repos)
  Network: 1Gbps

Database Server:
  CPU: 4 cores, 2.4GHz+
  RAM: 8GB
  Storage: 200GB SSD
  Network: 1Gbps

Backup Requirements:
  Storage: 5TB (including retention)
  Network: Dedicated backup network
```

#### Medium Enterprise (500-2000 developers)
```yaml
Load Balancer:
  CPU: 4 cores
  RAM: 8GB
  Network: 10Gbps

Gerrit Servers (2-3 instances):
  CPU: 16 cores, 2.4GHz+
  RAM: 32GB
  Storage: 1TB SSD (OS) + 5TB SSD (Git repos)
  Network: 10Gbps

Database Server (Primary):
  CPU: 8 cores, 2.4GHz+
  RAM: 32GB
  Storage: 1TB SSD
  Network: 10Gbps

Database Server (Replica):
  CPU: 8 cores, 2.4GHz+
  RAM: 16GB
  Storage: 1TB SSD
  Network: 10Gbps
```

#### Large Enterprise (2000+ developers)
```yaml
Load Balancer (HA Pair):
  CPU: 8 cores
  RAM: 16GB
  Network: 10Gbps

Gerrit Servers (3-5 instances):
  CPU: 24 cores, 2.4GHz+
  RAM: 64GB
  Storage: 2TB SSD (OS) + 10TB SSD (Git repos)
  Network: 10Gbps

Database Cluster:
  Primary: CPU: 16 cores, RAM: 64GB, Storage: 2TB SSD
  Replicas (2): CPU: 16 cores, RAM: 32GB, Storage: 2TB SSD
  
Search Infrastructure:
  Elasticsearch Cluster (3 nodes):
  CPU: 8 cores, RAM: 32GB, Storage: 1TB SSD
```

### 1.2 Network Architecture

#### Network Segmentation

```
┌─────────────────────────────────────────────────────────────┐
│                      DMZ Network                            │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    │
│  │    WAF      │    │   Reverse   │    │    Load     │    │
│  │  (Layer 7)  │    │    Proxy    │    │  Balancer   │    │
│  └─────────────┘    └─────────────┘    └─────────────┘    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Application Network                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Gerrit    │  │   Gerrit    │  │   Gerrit    │        │
│  │  Server 1   │  │  Server 2   │  │  Server 3   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Database Network                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ PostgreSQL  │  │ PostgreSQL  │  │    LDAP     │        │
│  │  Primary    │  │   Replica   │  │   Server    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Management Network                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Monitoring  │  │   Backup    │  │    Jump     │        │
│  │   Server    │  │   Server    │  │    Host     │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

#### Firewall Rules

```bash
# DMZ to Application Network
Allow TCP 80,443 from WAF to Load Balancer
Allow TCP 8080,29418 from Load Balancer to Gerrit Servers

# Application to Database Network
Allow TCP 5432 from Gerrit Servers to PostgreSQL
Allow TCP 389,636 from Gerrit Servers to LDAP

# Management Network
Allow TCP 22 from Jump Host to all servers
Allow TCP 9090,3000 from Monitoring to all servers
Allow TCP 5432 from Backup Server to Database servers

# Deny all other traffic between networks
```

## Part 2: Database Configuration

### 2.1 PostgreSQL Setup for Enterprise

#### Master Database Configuration

```sql
-- postgresql.conf for enterprise setup
# Connection Settings
max_connections = 200
shared_preload_libraries = 'pg_stat_statements,auto_explain'

# Memory Settings
shared_buffers = 8GB                    # 25% of total RAM
effective_cache_size = 24GB             # 75% of total RAM
work_mem = 64MB                         # Per query
maintenance_work_mem = 2GB

# WAL Settings
wal_level = replica
wal_buffers = 64MB
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min
max_wal_size = 4GB

# Replication Settings
hot_standby = on
max_wal_senders = 10
max_replication_slots = 10

# Performance Settings
random_page_cost = 1.1                 # For SSD storage
effective_io_concurrency = 200         # For SSD storage
default_statistics_target = 100

# Logging
log_destination = 'csvlog'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_min_duration_statement = 1000      # Log slow queries
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
```

#### Database Initialization Script

```sql
-- init-gerrit-db.sql
-- Create Gerrit database and user
CREATE USER gerrit WITH PASSWORD 'secure_password_here';
CREATE DATABASE reviewdb WITH OWNER gerrit ENCODING 'UTF8';

-- Grant necessary permissions
GRANT ALL PRIVILEGES ON DATABASE reviewdb TO gerrit;

-- Connect to the database
\c reviewdb;

-- Create extensions
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Configure for Gerrit
ALTER DATABASE reviewdb SET search_path TO "$user", public;

-- Create indexes for performance
-- (Gerrit will create these, but we can optimize)
```

#### Replication Setup

```bash
# On master server - setup replication
# Add to postgresql.conf
echo "
# Replication settings
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/archive/%f'
" >> /etc/postgresql/13/main/postgresql.conf

# Create replication user
sudo -u postgres psql -c "CREATE USER replicator REPLICATION LOGIN CONNECTION LIMIT 1 ENCRYPTED PASSWORD 'repl_password';"

# Configure pg_hba.conf for replication
echo "host replication replicator replica_server_ip/32 md5" >> /etc/postgresql/13/main/pg_hba.conf

# On replica server - setup streaming replication
sudo -u postgres pg_basebackup -h master_server_ip -D /var/lib/postgresql/13/main -U replicator -P -v -R -W

# Configure recovery.conf (PostgreSQL 12+: recovery.signal)
echo "
standby_mode = 'on'
primary_conninfo = 'host=master_server_ip port=5432 user=replicator password=repl_password'
trigger_file = '/tmp/postgresql.trigger'
" > /var/lib/postgresql/13/main/recovery.conf
```

### 2.2 Database Maintenance and Monitoring

#### Automated Maintenance Script

```bash
#!/bin/bash
# db-maintenance.sh

PGUSER="postgres"
PGDATABASE="reviewdb"
LOG_FILE="/var/log/postgresql/maintenance.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> $LOG_FILE
}

# Vacuum and analyze
log "Starting database maintenance"

# Update statistics
log "Updating table statistics"
psql -U $PGUSER -d $PGDATABASE -c "ANALYZE;" >> $LOG_FILE 2>&1

# Vacuum tables
log "Vacuuming database"
psql -U $PGUSER -d $PGDATABASE -c "VACUUM (VERBOSE, ANALYZE);" >> $LOG_FILE 2>&1

# Reindex if needed (run weekly)
if [ $(date +%u) -eq 7 ]; then
    log "Reindexing database (weekly maintenance)"
    psql -U $PGUSER -d $PGDATABASE -c "REINDEX DATABASE reviewdb;" >> $LOG_FILE 2>&1
fi

# Clean up old WAL files
log "Cleaning up WAL archive"
find /var/lib/postgresql/archive -name "*.backup" -mtime +7 -delete
find /var/lib/postgresql/archive -name "*" -mtime +3 -delete

log "Database maintenance completed"

# Check database size and alert if growing too fast
DB_SIZE=$(psql -U $PGUSER -d $PGDATABASE -t -c "SELECT pg_size_pretty(pg_database_size('reviewdb'));")
log "Current database size: $DB_SIZE"

# Monitor slow queries
psql -U $PGUSER -d $PGDATABASE -c "
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements
WHERE mean_time > 1000
ORDER BY mean_time DESC
LIMIT 10;" >> $LOG_FILE 2>&1
```

#### Database Monitoring Queries

```sql
-- Monitor active connections
SELECT 
    datname,
    count(*) as active_connections,
    max(now() - query_start) as longest_running_query
FROM pg_stat_activity 
WHERE state = 'active'
GROUP BY datname;

-- Monitor database size growth
SELECT 
    datname,
    pg_size_pretty(pg_database_size(datname)) as size,
    pg_size_pretty(pg_database_size(datname) - lag(pg_database_size(datname)) 
                   OVER (ORDER BY current_date)) as daily_growth
FROM pg_database 
WHERE datname = 'reviewdb';

-- Monitor replication lag
SELECT 
    client_addr,
    state,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)) as send_lag,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, flush_lsn)) as flush_lag
FROM pg_stat_replication;

-- Top slow queries
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    (total_time / sum(total_time) OVER ()) * 100 as percentage
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```

## Part 3: High Availability Setup

### 3.1 Load Balancer Configuration

#### HAProxy Configuration

```bash
# /etc/haproxy/haproxy.cfg
global
    daemon
    user haproxy
    group haproxy
    log 127.0.0.1:514 local0
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    
    # SSL Configuration
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor
    option redispatch
    retries 3

# Statistics page
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
    stats admin if TRUE

# HTTP Frontend
frontend gerrit_http
    bind *:80
    
    # Redirect HTTP to HTTPS
    redirect scheme https if !{ ssl_fc }

# HTTPS Frontend
frontend gerrit_https
    bind *:443 ssl crt /etc/ssl/certs/gerrit.pem
    
    # Security headers
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains"
    http-response set-header X-Frame-Options "SAMEORIGIN"
    http-response set-header X-Content-Type-Options "nosniff"
    
    # Route to Gerrit backend
    default_backend gerrit_servers

# SSH Frontend (for Git operations)
frontend gerrit_ssh
    mode tcp
    bind *:29418
    default_backend gerrit_ssh_servers

# Gerrit HTTP Backend
backend gerrit_servers
    balance roundrobin
    option httpchk GET /config/server/healthcheck-status
    
    server gerrit1 10.0.1.10:8080 check inter 5s fall 3 rise 2
    server gerrit2 10.0.1.11:8080 check inter 5s fall 3 rise 2
    server gerrit3 10.0.1.12:8080 check inter 5s fall 3 rise 2

# Gerrit SSH Backend
backend gerrit_ssh_servers
    mode tcp
    balance roundrobin
    option tcp-check
    
    server gerrit1 10.0.1.10:29418 check inter 5s fall 3 rise 2
    server gerrit2 10.0.1.11:29418 check inter 5s fall 3 rise 2
    server gerrit3 10.0.1.12:29418 check inter 5s fall 3 rise 2
```

#### Nginx Reverse Proxy Configuration

```nginx
# /etc/nginx/sites-available/gerrit
upstream gerrit_backend {
    least_conn;
    server 10.0.1.10:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:8080 max_fails=3 fail_timeout=30s;
}

# Rate limiting
limit_req_zone $binary_remote_addr zone=gerrit_limit:10m rate=10r/s;

server {
    listen 80;
    server_name gerrit.company.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name gerrit.company.com;
    
    # SSL Configuration
    ssl_certificate /etc/ssl/certs/gerrit.crt;
    ssl_certificate_key /etc/ssl/private/gerrit.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Logging
    access_log /var/log/nginx/gerrit_access.log;
    error_log /var/log/nginx/gerrit_error.log;
    
    # Rate limiting
    limit_req zone=gerrit_limit burst=20 nodelay;
    
    location / {
        proxy_pass http://gerrit_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        
        # Health check
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}

# SSH proxy configuration (stream module)
stream {
    upstream gerrit_ssh {
        server 10.0.1.10:29418 max_fails=3 fail_timeout=30s;
        server 10.0.1.11:29418 max_fails=3 fail_timeout=30s;
        server 10.0.1.12:29418 max_fails=3 fail_timeout=30s;
    }
    
    server {
        listen 29418;
        proxy_pass gerrit_ssh;
        proxy_timeout 1s;
        proxy_responses 1;
    }
}
```

### 3.2 Gerrit Cluster Configuration

#### Shared Storage Setup

```bash
# Setup NFS for shared Git repositories
# On NFS server
apt install nfs-kernel-server -y

# Create shared directory
mkdir -p /export/gerrit/git
chown -R gerrit:gerrit /export/gerrit
chmod 755 /export/gerrit

# Configure NFS exports
echo "/export/gerrit 10.0.1.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports

# Start NFS service
systemctl enable nfs-kernel-server
systemctl start nfs-kernel-server
exportfs -a

# On Gerrit servers
apt install nfs-common -y

# Mount shared storage
mkdir -p /opt/gerrit/shared
echo "nfs-server:/export/gerrit /opt/gerrit/shared nfs defaults 0 0" >> /etc/fstab
mount -a
```

#### Gerrit Multi-Master Configuration

```ini
# gerrit.config for cluster setup
[gerrit]
    basePath = /opt/gerrit/shared/git
    serverId = $(hostname)
    canonicalWebUrl = https://gerrit.company.com/
    
[database]
    type = postgresql
    hostname = db-primary.company.com
    port = 5432
    database = reviewdb
    username = gerrit
    # Password in secure.config

[auth]
    type = LDAP
    gitBasicAuthPolicy = LDAP

[ldap]
    server = ldap://ldap.company.com
    username = cn=gerrit,ou=services,dc=company,dc=com
    accountBase = ou=users,dc=company,dc=com
    accountPattern = (&(objectClass=person)(uid=${username}))
    accountFullName = displayName
    accountEmailAddress = mail
    groupBase = ou=groups,dc=company,dc=com

[sendemail]
    smtpServer = smtp.company.com
    smtpServerPort = 587
    smtpEncryption = tls
    smtpUser = gerrit@company.com
    # Password in secure.config

[sshd]
    listenAddress = *:29418
    batchThreads = 4

[httpd]
    listenUrl = proxy-https://*:8080/
    filterClass = com.googlesource.gerrit.plugins.ootb.FirstTimeSetup
    
[cache]
    directory = cache
    
[index]
    type = elasticsearch
    server = http://es1.company.com:9200,http://es2.company.com:9200,http://es3.company.com:9200
    
[plugin "events-log"]
    storeUrl = jdbc:postgresql://db-primary.company.com:5432/events

[container]
    javaOptions = -Xmx8g -Xms8g
    javaOptions = -XX:+UseG1GC
    javaOptions = -XX:+UnlockExperimentalVMOptions
    javaOptions = -XX:+UseCGroupMemoryLimitForHeap
    
[gc]
    interval = 1 day
    
[receive]
    enableSignedPush = false
    checkReferencedObjectsAreReachable = false
```

#### Session Management for Clustering

```ini
# Configure session management for multi-node setup
[cache "web_sessions"]
    diskLimit = 1g
    maxAge = 7 days

# Use database for session storage in cluster
[database]
    # ... other settings ...
    poolLimit = 50
    poolMinIdle = 10
    poolMaxIdle = 25
    poolMaxWait = 5000
```

## Part 4: Authentication and Authorization

### 4.1 LDAP/Active Directory Integration

#### Advanced LDAP Configuration

```ini
# Advanced LDAP setup in gerrit.config
[ldap]
    server = ldaps://ldap.company.com:636
    
    # Connection settings
    connectionTimeout = 10s
    readTimeout = 30s
    
    # Authentication
    username = cn=gerrit,ou=service-accounts,dc=company,dc=com
    password = # In secure.config
    
    # User account mapping
    accountBase = ou=users,dc=company,dc=com
    accountScope = subtree
    accountPattern = (&(objectClass=person)(uid=${username})(accountStatus=active))
    accountFullName = cn
    accountEmailAddress = mail
    accountSshUsername = uid
    
    # Group mapping
    groupBase = ou=groups,dc=company,dc=com
    groupScope = subtree
    groupPattern = (&(objectClass=groupOfNames)(member=${dn}))
    groupMemberPattern = (&(objectClass=person)(memberOf=${dn}))
    groupName = cn
    
    # Performance optimization
    connectionsPerHost = 10
    maxConnections = 100
    
    # Referral handling
    referral = follow
```

#### LDAP Group Synchronization Script

```python
#!/usr/bin/env python3
# ldap-sync.py - Synchronize LDAP groups with Gerrit

import ldap
import requests
import json
import sys
from requests.auth import HTTPDigestAuth

class LDAPGerritSync:
    def __init__(self, config):
        self.ldap_server = config['ldap_server']
        self.ldap_user = config['ldap_user']
        self.ldap_password = config['ldap_password']
        self.ldap_base = config['ldap_base']
        
        self.gerrit_url = config['gerrit_url']
        self.gerrit_user = config['gerrit_user']
        self.gerrit_password = config['gerrit_password']
        
    def connect_ldap(self):
        """Connect to LDAP server"""
        try:
            conn = ldap.initialize(self.ldap_server)
            conn.simple_bind_s(self.ldap_user, self.ldap_password)
            return conn
        except ldap.LDAPError as e:
            print(f"LDAP connection failed: {e}")
            sys.exit(1)
    
    def get_ldap_groups(self, conn):
        """Get all groups from LDAP"""
        search_filter = "(&(objectClass=groupOfNames)(cn=gerrit-*))"
        attrs = ['cn', 'member', 'description']
        
        try:
            result = conn.search_s(
                self.ldap_base, 
                ldap.SCOPE_SUBTREE, 
                search_filter, 
                attrs
            )
            
            groups = {}
            for dn, attrs in result:
                group_name = attrs['cn'][0].decode('utf-8')
                # Remove 'gerrit-' prefix
                gerrit_group_name = group_name.replace('gerrit-', '')
                
                members = []
                if 'member' in attrs:
                    for member_dn in attrs['member']:
                        # Extract username from DN
                        member_dn_str = member_dn.decode('utf-8')
                        uid = self.extract_uid_from_dn(member_dn_str)
                        if uid:
                            members.append(uid)
                
                description = ""
                if 'description' in attrs:
                    description = attrs['description'][0].decode('utf-8')
                
                groups[gerrit_group_name] = {
                    'members': members,
                    'description': description
                }
            
            return groups
            
        except ldap.LDAPError as e:
            print(f"LDAP search failed: {e}")
            return {}
    
    def extract_uid_from_dn(self, dn):
        """Extract UID from LDAP DN"""
        parts = dn.split(',')
        for part in parts:
            if part.strip().startswith('uid='):
                return part.strip().split('=')[1]
        return None
    
    def get_gerrit_groups(self):
        """Get all groups from Gerrit"""
        try:
            response = requests.get(
                f"{self.gerrit_url}/a/groups/",
                auth=HTTPDigestAuth(self.gerrit_user, self.gerrit_password)
            )
            response.raise_for_status()
            
            # Remove Gerrit's magic prefix
            content = response.text[4:]  # Remove ")]}'"
            return json.loads(content)
            
        except requests.RequestException as e:
            print(f"Failed to get Gerrit groups: {e}")
            return {}
    
    def create_gerrit_group(self, group_name, description=""):
        """Create group in Gerrit"""
        group_data = {
            'description': description,
            'visible_to_all': True
        }
        
        try:
            response = requests.put(
                f"{self.gerrit_url}/a/groups/{group_name}",
                auth=HTTPDigestAuth(self.gerrit_user, self.gerrit_password),
                json=group_data
            )
            response.raise_for_status()
            print(f"Created group: {group_name}")
            return True
            
        except requests.RequestException as e:
            print(f"Failed to create group {group_name}: {e}")
            return False
    
    def update_group_members(self, group_name, members):
        """Update group members in Gerrit"""
        try:
            # Get current members
            response = requests.get(
                f"{self.gerrit_url}/a/groups/{group_name}/members/",
                auth=HTTPDigestAuth(self.gerrit_user, self.gerrit_password)
            )
            response.raise_for_status()
            
            content = response.text[4:]  # Remove ")]}'"
            current_members = {member['username'] for member in json.loads(content)}
            new_members = set(members)
            
            # Add new members
            members_to_add = new_members - current_members
            for member in members_to_add:
                self.add_group_member(group_name, member)
            
            # Remove old members
            members_to_remove = current_members - new_members
            for member in members_to_remove:
                self.remove_group_member(group_name, member)
                
        except requests.RequestException as e:
            print(f"Failed to update group members for {group_name}: {e}")
    
    def add_group_member(self, group_name, username):
        """Add member to Gerrit group"""
        member_data = {'members': [username]}
        
        try:
            response = requests.post(
                f"{self.gerrit_url}/a/groups/{group_name}/members",
                auth=HTTPDigestAuth(self.gerrit_user, self.gerrit_password),
                json=member_data
            )
            response.raise_for_status()
            print(f"Added {username} to {group_name}")
            
        except requests.RequestException as e:
            print(f"Failed to add {username} to {group_name}: {e}")
    
    def remove_group_member(self, group_name, username):
        """Remove member from Gerrit group"""
        try:
            response = requests.delete(
                f"{self.gerrit_url}/a/groups/{group_name}/members/{username}",
                auth=HTTPDigestAuth(self.gerrit_user, self.gerrit_password)
            )
            response.raise_for_status()
            print(f"Removed {username} from {group_name}")
            
        except requests.RequestException as e:
            print(f"Failed to remove {username} from {group_name}: {e}")
    
    def sync_groups(self):
        """Main synchronization method"""
        print("Starting LDAP to Gerrit group synchronization...")
        
        # Connect to LDAP
        ldap_conn = self.connect_ldap()
        
        try:
            # Get groups from both systems
            ldap_groups = self.get_ldap_groups(ldap_conn)
            gerrit_groups = self.get_gerrit_groups()
            
            print(f"Found {len(ldap_groups)} LDAP groups and {len(gerrit_groups)} Gerrit groups")
            
            # Sync each LDAP group
            for group_name, group_data in ldap_groups.items():
                if group_name not in gerrit_groups:
                    # Create new group
                    if self.create_gerrit_group(group_name, group_data['description']):
                        self.update_group_members(group_name, group_data['members'])
                else:
                    # Update existing group
                    self.update_group_members(group_name, group_data['members'])
            
            print("Synchronization completed successfully")
            
        finally:
            ldap_conn.unbind()

# Configuration
config = {
    'ldap_server': 'ldaps://ldap.company.com:636',
    'ldap_user': 'cn=gerrit,ou=service-accounts,dc=company,dc=com',
    'ldap_password': 'ldap_password',
    'ldap_base': 'dc=company,dc=com',
    
    'gerrit_url': 'https://gerrit.company.com',
    'gerrit_user': 'admin',
    'gerrit_password': 'gerrit_password'
}

if __name__ == "__main__":
    sync = LDAPGerritSync(config)
    sync.sync_groups()
```

### 4.2 Single Sign-On (SSO) Integration

#### SAML Configuration

```ini
# SAML SSO configuration
[auth]
    type = HTTP
    httpHeader = X-Forwarded-User
    httpDisplaynameHeader = X-Forwarded-User-Name
    httpEmailHeader = X-Forwarded-User-Email
    httpExternalIdHeader = X-Forwarded-User-Id

[httpd]
    filterClass = com.googlesource.gerrit.httpd.auth.container.HttpAuthFilter
```

#### OAuth 2.0 Integration

```ini
# OAuth configuration for GitHub/Google/etc
[auth]
    type = OAUTH
    
[oauth]
    # Google OAuth
    serviceProviderUrl = https://accounts.google.com/o/oauth2/auth
    accessTokenUrl = https://accounts.google.com/o/oauth2/token
    userInfoUrl = https://www.googleapis.com/oauth2/v1/userinfo
    clientId = your-client-id.googleusercontent.com
    clientSecret = # In secure.config
    
    # Restrict to company domain
    domain = company.com
```

## Part 5: Performance Optimization

### 5.1 JVM Tuning

#### Production JVM Settings

```bash
# Set in gerrit.config [container] section
[container]
    # Heap settings
    javaOptions = -Xms16g
    javaOptions = -Xmx16g
    javaOptions = -XX:NewRatio=1
    
    # Garbage Collection
    javaOptions = -XX:+UseG1GC
    javaOptions = -XX:MaxGCPauseMillis=200
    javaOptions = -XX:G1HeapRegionSize=16m
    javaOptions = -XX:+G1UseAdaptiveIHOP
    javaOptions = -XX:G1MixedGCCountTarget=8
    
    # GC Logging
    javaOptions = -Xlog:gc*:logs/gc.log:time,level,tags
    javaOptions = -XX:+UseStringDeduplication
    
    # Performance tuning
    javaOptions = -XX:+UseLargePages
    javaOptions = -XX:+AlwaysPreTouch
    javaOptions = -XX:+DisableExplicitGC
    javaOptions = -server
    
    # JIT Compiler
    javaOptions = -XX:+UseCompressedOops
    javaOptions = -XX:+OptimizeStringConcat
    
    # Memory management
    javaOptions = -XX:+UnlockExperimentalVMOptions
    javaOptions = -XX:+UseCGroupMemoryLimitForHeap
```

#### Memory Monitoring Script

```bash
#!/bin/bash
# memory-monitor.sh

GERRIT_PID=$(pgrep -f "GerritCodeReview")
LOG_FILE="/var/log/gerrit/memory-monitor.log"

if [ -z "$GERRIT_PID" ]; then
    echo "$(date): Gerrit process not found" >> $LOG_FILE
    exit 1
fi

# Get memory information
HEAP_USED=$(jstat -gc $GERRIT_PID | tail -1 | awk '{print ($3+$4+$6+$8)/1024}')
HEAP_TOTAL=$(jstat -gccapacity $GERRIT_PID | tail -1 | awk '{print ($4+$5+$7+$8)/1024}')
HEAP_PERCENT=$(echo "scale=2; $HEAP_USED/$HEAP_TOTAL*100" | bc)

# Get GC information
GC_INFO=$(jstat -gc $GERRIT_PID | tail -1)
YOUNG_GC=$(echo $GC_INFO | awk '{print $12}')
OLD_GC=$(echo $GC_INFO | awk '{print $14}')
GC_TIME=$(echo $GC_INFO | awk '{print $15+$16}')

# Log the information
echo "$(date): Heap: ${HEAP_USED}MB/${HEAP_TOTAL}MB (${HEAP_PERCENT}%), YGC: $YOUNG_GC, FGC: $OLD_GC, GC Time: ${GC_TIME}s" >> $LOG_FILE

# Alert if heap usage is too high
if (( $(echo "$HEAP_PERCENT > 85" | bc -l) )); then
    echo "$(date): WARNING: High heap usage detected: ${HEAP_PERCENT}%" >> $LOG_FILE
    # Send alert here
fi

# Alert if too many full GCs
if [ "$OLD_GC" -gt 100 ]; then
    echo "$(date): WARNING: High number of full GCs: $OLD_GC" >> $LOG_FILE
    # Send alert here
fi
```

### 5.2 Cache Configuration

#### Optimized Cache Settings

```ini
# Optimized cache configuration
[cache "accounts"]
    memoryLimit = 1024
    diskLimit = 5g
    maxAge = 1 hour

[cache "accounts_byemail"]
    memoryLimit = 1024
    diskLimit = 2g
    maxAge = 1 hour

[cache "accounts_byname"]
    memoryLimit = 1024
    diskLimit = 2g
    maxAge = 1 hour

[cache "diff"]
    memoryLimit = 128m
    diskLimit = 10g
    maxAge = 30 days

[cache "diff_intraline"]
    memoryLimit = 128m
    diskLimit = 5g
    maxAge = 30 days

[cache "git_tags"]
    memoryLimit = 4096
    maxAge = 30 minutes

[cache "groups"]
    memoryLimit = 1024
    diskLimit = 2g
    maxAge = 1 hour

[cache "groups_byinclude"]
    memoryLimit = 1024
    diskLimit = 2g
    maxAge = 1 hour

[cache "groups_byname"]
    memoryLimit = 1024
    diskLimit = 2g
    maxAge = 1 hour

[cache "groups_byuuid"]
    memoryLimit = 1024
    diskLimit = 2g
    maxAge = 1 hour

[cache "projects"]
    memoryLimit = 2048
    diskLimit = 5g
    maxAge = 15 minutes

[cache "project_list"]
    memoryLimit = 1024
    maxAge = 5 minutes

[cache "sshkeys"]
    memoryLimit = 1024
    diskLimit = 1g
    maxAge = 5 minutes

[cache "web_sessions"]
    memoryLimit = 1024
    diskLimit = 1g
    maxAge = 12 hours
```

#### Cache Monitoring Script

```python
#!/usr/bin/env python3
# cache-monitor.py

import requests
import json
import time
from requests.auth import HTTPDigestAuth

class GerritCacheMonitor:
    def __init__(self, gerrit_url, username, password):
        self.gerrit_url = gerrit_url
        self.auth = HTTPDigestAuth(username, password)
    
    def get_cache_stats(self):
        """Get cache statistics from Gerrit"""
        try:
            response = requests.get(
                f"{self.gerrit_url}/a/config/caches/",
                auth=self.auth
            )
            response.raise_for_status()
            
            # Remove Gerrit's magic prefix
            content = response.text[4:]
            return json.loads(content)
            
        except requests.RequestException as e:
            print(f"Failed to get cache stats: {e}")
            return {}
    
    def analyze_cache_performance(self, cache_stats):
        """Analyze cache performance and identify issues"""
        issues = []
        recommendations = []
        
        for cache_name, stats in cache_stats.items():
            hit_ratio = stats.get('hit_ratio', {})
            if 'percentage' in hit_ratio:
                hit_rate = hit_ratio['percentage']
                
                if hit_rate < 70:
                    issues.append(f"Low hit rate for {cache_name}: {hit_rate}%")
                    
                    if cache_name in ['diff', 'diff_intraline']:
                        recommendations.append(f"Consider increasing disk limit for {cache_name}")
                    elif cache_name in ['accounts', 'groups']:
                        recommendations.append(f"Consider increasing memory limit for {cache_name}")
            
            # Check memory usage
            mem_info = stats.get('mem', {})
            if 'percentage' in mem_info:
                mem_usage = mem_info['percentage']
                if mem_usage > 90:
                    issues.append(f"High memory usage for {cache_name}: {mem_usage}%")
                    recommendations.append(f"Increase memory limit for {cache_name}")
            
            # Check disk usage
            disk_info = stats.get('disk', {})
            if 'percentage' in disk_info:
                disk_usage = disk_info['percentage']
                if disk_usage > 90:
                    issues.append(f"High disk usage for {cache_name}: {disk_usage}%")
                    recommendations.append(f"Increase disk limit for {cache_name}")
        
        return {
            'issues': issues,
            'recommendations': recommendations,
            'timestamp': time.time()
        }
    
    def flush_cache(self, cache_name):
        """Flush a specific cache"""
        try:
            response = requests.post(
                f"{self.gerrit_url}/a/config/caches/{cache_name}/flush",
                auth=self.auth
            )
            response.raise_for_status()
            print(f"Successfully flushed cache: {cache_name}")
            
        except requests.RequestException as e:
            print(f"Failed to flush cache {cache_name}: {e}")
    
    def generate_report(self):
        """Generate cache performance report"""
        cache_stats = self.get_cache_stats()
        analysis = self.analyze_cache_performance(cache_stats)
        
        print("=== Gerrit Cache Performance Report ===")
        print(f"Generated: {time.ctime(analysis['timestamp'])}")
        print()
        
        print("Cache Hit Rates:")
        for cache_name, stats in cache_stats.items():
            hit_ratio = stats.get('hit_ratio', {})
            if 'percentage' in hit_ratio:
                print(f"  {cache_name}: {hit_ratio['percentage']}%")
        print()
        
        if analysis['issues']:
            print("Issues Found:")
            for issue in analysis['issues']:
                print(f"  - {issue}")
            print()
        
        if analysis['recommendations']:
            print("Recommendations:")
            for rec in analysis['recommendations']:
                print(f"  - {rec}")
            print()
        
        return analysis

if __name__ == "__main__":
    monitor = GerritCacheMonitor(
        "https://gerrit.company.com",
        "admin",
        "admin_password"
    )
    
    monitor.generate_report()
```

This enterprise setup chapter provides comprehensive guidance for deploying Gerrit in large-scale environments. The configuration examples, monitoring scripts, and best practices shown here form the foundation for a robust, scalable, and secure Gerrit deployment that can handle thousands of developers and complex organizational requirements.

## Chapter Summary

You've learned enterprise-level Gerrit deployment:

- **Infrastructure planning** with hardware and network architecture
- **Database configuration** with PostgreSQL clustering and replication
- **High availability setup** with load balancers and clustering
- **Authentication integration** with LDAP/AD and SSO
- **Performance optimization** through JVM tuning and caching
- **Monitoring and maintenance** strategies for production systems

This foundation enables you to deploy Gerrit at enterprise scale with confidence.

---

**Ready for security best practices?** Continue to [Chapter 9: Security and Best Practices](../09-security/README.md)

---

*Continue to [Chapter 9: Security and Best Practices](../09-security/README.md)*
