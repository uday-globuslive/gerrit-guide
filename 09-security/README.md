# Chapter 9: Security and Best Practices

## Overview

Security is paramount in enterprise code review systems. This chapter covers comprehensive security measures, best practices, and compliance considerations for Gerrit deployments. We'll explore everything from basic security hardening to advanced threat mitigation and regulatory compliance.

## Security Threat Model

### Common Security Threats in Code Review Systems

```
┌─────────────────────────────────────────────────────────────────┐
│                    Gerrit Security Threat Model                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  External Threats                Internal Threats               │
│  ┌─────────────────┐            ┌─────────────────┐            │
│  │ • DDoS Attacks  │            │ • Insider Abuse │            │
│  │ • Data Breaches │            │ • Privilege     │            │
│  │ • Injection     │            │   Escalation    │            │
│  │ • Brute Force   │            │ • Data Leakage  │            │
│  └─────────────────┘            └─────────────────┘            │
│           │                               │                     │
│           ▼                               ▼                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Security Controls                          │   │
│  │                                                         │   │
│  │  • Network Security     • Access Controls              │   │
│  │  • Encryption          • Audit Logging                 │   │
│  │  • Authentication      • Backup Security               │   │
│  │  • Authorization       • Incident Response             │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Part 1: Network Security

### 1.1 Firewall Configuration

#### Comprehensive Firewall Rules

```bash
#!/bin/bash
# gerrit-firewall.sh - Secure firewall configuration for Gerrit

# Flush existing rules
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X

# Set default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback traffic
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# SSH access (restrict to management network)
iptables -A INPUT -p tcp --dport 22 -s 10.0.100.0/24 -j ACCEPT

# HTTP/HTTPS (through load balancer only)
iptables -A INPUT -p tcp --dport 80 -s 10.0.1.100 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -s 10.0.1.100 -j ACCEPT
iptables -A INPUT -p tcp --dport 8080 -s 10.0.1.100 -j ACCEPT

# Gerrit SSH (through load balancer only)
iptables -A INPUT -p tcp --dport 29418 -s 10.0.1.100 -j ACCEPT

# Database access (from application servers only)
iptables -A INPUT -p tcp --dport 5432 -s 10.0.1.0/24 -j ACCEPT

# LDAP access (from application servers only)
iptables -A INPUT -p tcp --dport 389 -s 10.0.1.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 636 -s 10.0.1.0/24 -j ACCEPT

# Monitoring (from monitoring server only)
iptables -A INPUT -p tcp --dport 9090 -s 10.0.100.10 -j ACCEPT

# NTP
iptables -A INPUT -p udp --dport 123 -j ACCEPT

# Rate limiting for HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT

# DDoS protection
iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# Block common attack patterns
iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
iptables -A INPUT -p tcp --tcp-flags ALL SYN,RST,ACK,FIN,URG -j DROP

# Log dropped packets
iptables -A INPUT -j LOG --log-prefix "DROPPED: " --log-level 4
iptables -A INPUT -j DROP

# Save rules
iptables-save > /etc/iptables/rules.v4

echo "Firewall rules applied successfully"
```

#### Advanced DDoS Protection

```bash
# ddos-protection.sh
#!/bin/bash

# Protect against SYN flood attacks
echo 1 > /proc/sys/net/ipv4/tcp_syncookies
echo 2048 > /proc/sys/net/ipv4/tcp_max_syn_backlog
echo 3 > /proc/sys/net/ipv4/tcp_synack_retries

# Protect against ping floods
echo 1 > /proc/sys/net/ipv4/icmp_echo_ignore_broadcasts
echo 1 > /proc/sys/net/ipv4/icmp_ignore_bogus_error_responses

# Disable source routing and redirects
echo 0 > /proc/sys/net/ipv4/conf/all/accept_source_route
echo 0 > /proc/sys/net/ipv4/conf/all/accept_redirects
echo 0 > /proc/sys/net/ipv4/conf/all/secure_redirects

# Enable logging of martians (packets with impossible addresses)
echo 1 > /proc/sys/net/ipv4/conf/all/log_martians

# Configure connection tracking
echo 65536 > /proc/sys/net/netfilter/nf_conntrack_max
echo 300 > /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established
```

### 1.2 SSL/TLS Configuration

#### Nginx SSL Hardening

```nginx
# /etc/nginx/sites-available/gerrit-secure
server {
    listen 443 ssl http2;
    server_name gerrit.company.com;
    
    # SSL Certificate
    ssl_certificate /etc/ssl/certs/gerrit.crt;
    ssl_certificate_key /etc/ssl/private/gerrit.key;
    ssl_trusted_certificate /etc/ssl/certs/ca-bundle.crt;
    
    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # SSL Session
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self'; connect-src 'self'; frame-ancestors 'none';" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    
    # Hide server information
    server_tokens off;
    more_clear_headers Server;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    
    # Main location
    location / {
        proxy_pass http://gerrit_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Security headers for proxied content
        proxy_hide_header X-Powered-By;
        proxy_hide_header Server;
    }
    
    # Rate limit login attempts
    location /login {
        limit_req zone=login burst=5 nodelay;
        proxy_pass http://gerrit_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Rate limit API calls
    location /a/ {
        limit_req zone=api burst=200 nodelay;
        proxy_pass http://gerrit_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Block access to sensitive files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
    
    location ~ \.(sql|conf|config|bak|old|tmp|temp)$ {
        deny all;
        access_log off;
        log_not_found off;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name gerrit.company.com;
    return 301 https://$server_name$request_uri;
}
```

#### SSL Certificate Management

```bash
#!/bin/bash
# ssl-cert-manager.sh - Automated SSL certificate management

DOMAIN="gerrit.company.com"
CERT_PATH="/etc/ssl/certs"
KEY_PATH="/etc/ssl/private"
EMAIL="admin@company.com"

# Function to check certificate expiry
check_cert_expiry() {
    local cert_file="$CERT_PATH/gerrit.crt"
    
    if [ ! -f "$cert_file" ]; then
        echo "Certificate not found: $cert_file"
        return 1
    fi
    
    local expiry_date=$(openssl x509 -in "$cert_file" -noout -dates | grep notAfter | cut -d= -f2)
    local expiry_epoch=$(date -d "$expiry_date" +%s)
    local current_epoch=$(date +%s)
    local days_until_expiry=$(( (expiry_epoch - current_epoch) / 86400 ))
    
    echo "Certificate expires in $days_until_expiry days"
    
    if [ $days_until_expiry -lt 30 ]; then
        echo "WARNING: Certificate expires soon!"
        return 2
    fi
    
    return 0
}

# Function to renew certificate using Let's Encrypt
renew_certificate() {
    echo "Renewing certificate for $DOMAIN"
    
    # Stop nginx temporarily
    systemctl stop nginx
    
    # Renew certificate
    certbot certonly --standalone \
        --email "$EMAIL" \
        --agree-tos \
        --no-eff-email \
        --domains "$DOMAIN"
    
    # Copy certificates to correct location
    cp "/etc/letsencrypt/live/$DOMAIN/fullchain.pem" "$CERT_PATH/gerrit.crt"
    cp "/etc/letsencrypt/live/$DOMAIN/privkey.pem" "$KEY_PATH/gerrit.key"
    
    # Set proper permissions
    chmod 644 "$CERT_PATH/gerrit.crt"
    chmod 600 "$KEY_PATH/gerrit.key"
    chown root:root "$CERT_PATH/gerrit.crt" "$KEY_PATH/gerrit.key"
    
    # Start nginx
    systemctl start nginx
    
    # Test certificate
    if openssl x509 -in "$CERT_PATH/gerrit.crt" -noout -checkend 86400; then
        echo "Certificate renewal successful"
        
        # Reload services
        systemctl reload nginx
        systemctl reload haproxy
        
        return 0
    else
        echo "Certificate renewal failed"
        return 1
    fi
}

# Main execution
case "$1" in
    "check")
        check_cert_expiry
        ;;
    "renew")
        renew_certificate
        ;;
    "auto")
        check_cert_expiry
        if [ $? -eq 2 ]; then
            renew_certificate
        fi
        ;;
    *)
        echo "Usage: $0 {check|renew|auto}"
        exit 1
        ;;
esac
```

## Part 2: Authentication and Authorization Security

### 2.1 Multi-Factor Authentication

#### TOTP Integration

```python
#!/usr/bin/env python3
# mfa-integration.py - Multi-factor authentication for Gerrit

import pyotp
import qrcode
import requests
import json
import hmac
import hashlib
import time
from flask import Flask, request, jsonify, session
from functools import wraps

app = Flask(__name__)
app.secret_key = 'your-secret-key-here'

class GerritMFA:
    def __init__(self, gerrit_url, gerrit_user, gerrit_password):
        self.gerrit_url = gerrit_url
        self.gerrit_auth = (gerrit_user, gerrit_password)
        self.users_mfa = {}  # In production, use database
    
    def generate_mfa_secret(self, username):
        """Generate TOTP secret for user"""
        secret = pyotp.random_base32()
        self.users_mfa[username] = {
            'secret': secret,
            'enabled': False,
            'backup_codes': self.generate_backup_codes()
        }
        return secret
    
    def generate_backup_codes(self):
        """Generate backup codes for MFA"""
        import random
        import string
        
        codes = []
        for _ in range(10):
            code = ''.join(random.choices(string.ascii_uppercase + string.digits, k=8))
            codes.append(code)
        return codes
    
    def generate_qr_code(self, username, secret):
        """Generate QR code for TOTP setup"""
        totp = pyotp.TOTP(secret)
        provisioning_uri = totp.provisioning_uri(
            name=username,
            issuer_name="Gerrit Code Review"
        )
        
        qr = qrcode.QRCode(version=1, box_size=10, border=5)
        qr.add_data(provisioning_uri)
        qr.make(fit=True)
        
        return qr.make_image(fill_color="black", back_color="white")
    
    def verify_totp(self, username, token):
        """Verify TOTP token"""
        if username not in self.users_mfa:
            return False
        
        secret = self.users_mfa[username]['secret']
        totp = pyotp.TOTP(secret)
        
        # Allow for time skew (30 seconds before/after)
        return totp.verify(token, valid_window=1)
    
    def verify_backup_code(self, username, code):
        """Verify backup code"""
        if username not in self.users_mfa:
            return False
        
        backup_codes = self.users_mfa[username]['backup_codes']
        if code in backup_codes:
            # Remove used backup code
            backup_codes.remove(code)
            return True
        
        return False
    
    def enable_mfa(self, username, verification_token):
        """Enable MFA for user after verification"""
        if self.verify_totp(username, verification_token):
            self.users_mfa[username]['enabled'] = True
            return True
        return False
    
    def is_mfa_enabled(self, username):
        """Check if MFA is enabled for user"""
        return username in self.users_mfa and self.users_mfa[username]['enabled']
    
    def create_gerrit_session(self, username, password, mfa_token=None):
        """Create authenticated Gerrit session with MFA"""
        
        # First, verify primary authentication
        auth_response = requests.get(
            f"{self.gerrit_url}/a/accounts/self",
            auth=(username, password)
        )
        
        if auth_response.status_code != 200:
            return {'success': False, 'error': 'Invalid credentials'}
        
        # Check if MFA is required
        if self.is_mfa_enabled(username):
            if not mfa_token:
                return {'success': False, 'error': 'MFA token required', 'mfa_required': True}
            
            # Verify MFA token
            if not (self.verify_totp(username, mfa_token) or 
                   self.verify_backup_code(username, mfa_token)):
                return {'success': False, 'error': 'Invalid MFA token'}
        
        # Create session token
        session_token = self.generate_session_token(username)
        
        return {
            'success': True,
            'session_token': session_token,
            'user': username
        }
    
    def generate_session_token(self, username):
        """Generate secure session token"""
        timestamp = str(int(time.time()))
        data = f"{username}:{timestamp}"
        signature = hmac.new(
            app.secret_key.encode(),
            data.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return f"{data}:{signature}"
    
    def verify_session_token(self, token):
        """Verify session token"""
        try:
            parts = token.split(':')
            if len(parts) != 3:
                return None
            
            username, timestamp, signature = parts
            
            # Check token age (24 hours max)
            if int(time.time()) - int(timestamp) > 86400:
                return None
            
            # Verify signature
            data = f"{username}:{timestamp}"
            expected_signature = hmac.new(
                app.secret_key.encode(),
                data.encode(),
                hashlib.sha256
            ).hexdigest()
            
            if hmac.compare_digest(signature, expected_signature):
                return username
            
        except Exception:
            pass
        
        return None

# Initialize MFA system
mfa_system = GerritMFA(
    "https://gerrit.company.com",
    "admin",
    "admin_password"
)

def require_auth(f):
    """Decorator to require authentication"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        token = request.headers.get('Authorization')
        if token:
            token = token.replace('Bearer ', '')
            username = mfa_system.verify_session_token(token)
            if username:
                request.username = username
                return f(*args, **kwargs)
        
        return jsonify({'error': 'Authentication required'}), 401
    return decorated_function

@app.route('/api/mfa/setup', methods=['POST'])
@require_auth
def setup_mfa():
    """Setup MFA for authenticated user"""
    username = request.username
    
    # Generate secret and QR code
    secret = mfa_system.generate_mfa_secret(username)
    
    # In a real implementation, you'd save the QR code image
    # and return a URL to it
    qr_image = mfa_system.generate_qr_code(username, secret)
    
    return jsonify({
        'secret': secret,
        'backup_codes': mfa_system.users_mfa[username]['backup_codes'],
        'qr_setup_required': True
    })

@app.route('/api/mfa/verify', methods=['POST'])
@require_auth
def verify_mfa_setup():
    """Verify and enable MFA"""
    username = request.username
    data = request.get_json()
    
    verification_token = data.get('token')
    if not verification_token:
        return jsonify({'error': 'Token required'}), 400
    
    if mfa_system.enable_mfa(username, verification_token):
        return jsonify({'success': True, 'message': 'MFA enabled successfully'})
    else:
        return jsonify({'error': 'Invalid verification token'}), 400

@app.route('/api/auth/login', methods=['POST'])
def login():
    """Login with optional MFA"""
    data = request.get_json()
    
    username = data.get('username')
    password = data.get('password')
    mfa_token = data.get('mfa_token')
    
    if not username or not password:
        return jsonify({'error': 'Username and password required'}), 400
    
    result = mfa_system.create_gerrit_session(username, password, mfa_token)
    
    if result['success']:
        return jsonify(result)
    else:
        status_code = 401 if 'mfa_required' not in result else 200
        return jsonify(result), status_code

@app.route('/api/auth/verify', methods=['POST'])
def verify_token():
    """Verify session token"""
    token = request.headers.get('Authorization', '').replace('Bearer ', '')
    
    username = mfa_system.verify_session_token(token)
    if username:
        return jsonify({'valid': True, 'username': username})
    else:
        return jsonify({'valid': False}), 401

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5002, debug=False)
```

### 2.2 Role-Based Access Control (RBAC)

#### Advanced Permission Management

```python
#!/usr/bin/env python3
# rbac-manager.py - Role-based access control for Gerrit

import json
import requests
from requests.auth import HTTPDigestAuth
from dataclasses import dataclass
from typing import List, Dict, Set
from enum import Enum

class Permission(Enum):
    READ = "read"
    PUSH = "push"
    SUBMIT = "submit"
    APPROVE = "approve"
    VERIFY = "verify"
    ABANDON = "abandon"
    CREATE_PROJECT = "create_project"
    ADMIN = "admin"

class ResourceType(Enum):
    PROJECT = "project"
    BRANCH = "branch"
    TAG = "tag"
    GLOBAL = "global"

@dataclass
class Role:
    name: str
    permissions: Set[Permission]
    resource_type: ResourceType
    resource_pattern: str = "*"

@dataclass
class User:
    username: str
    email: str
    full_name: str
    roles: List[str]
    groups: List[str]

class GerritRBACManager:
    def __init__(self, gerrit_url, username, password):
        self.gerrit_url = gerrit_url
        self.auth = HTTPDigestAuth(username, password)
        self.roles = self.load_default_roles()
        
    def load_default_roles(self) -> Dict[str, Role]:
        """Load default role definitions"""
        return {
            'developer': Role(
                name='developer',
                permissions={Permission.READ, Permission.PUSH},
                resource_type=ResourceType.PROJECT
            ),
            'senior-developer': Role(
                name='senior-developer',
                permissions={
                    Permission.READ, Permission.PUSH, 
                    Permission.APPROVE, Permission.SUBMIT
                },
                resource_type=ResourceType.PROJECT
            ),
            'tech-lead': Role(
                name='tech-lead',
                permissions={
                    Permission.READ, Permission.PUSH, Permission.APPROVE,
                    Permission.SUBMIT, Permission.VERIFY, Permission.ABANDON
                },
                resource_type=ResourceType.PROJECT
            ),
            'release-manager': Role(
                name='release-manager',
                permissions={
                    Permission.READ, Permission.PUSH, Permission.APPROVE,
                    Permission.SUBMIT, Permission.VERIFY
                },
                resource_type=ResourceType.BRANCH,
                resource_pattern='refs/heads/release/*'
            ),
            'project-owner': Role(
                name='project-owner',
                permissions={
                    Permission.READ, Permission.PUSH, Permission.APPROVE,
                    Permission.SUBMIT, Permission.VERIFY, Permission.ABANDON,
                    Permission.CREATE_PROJECT
                },
                resource_type=ResourceType.PROJECT
            ),
            'admin': Role(
                name='admin',
                permissions={perm for perm in Permission},
                resource_type=ResourceType.GLOBAL
            )
        }
    
    def create_gerrit_group(self, group_name: str, description: str = "") -> bool:
        """Create a group in Gerrit"""
        group_data = {
            'description': description,
            'visible_to_all': True
        }
        
        try:
            response = requests.put(
                f"{self.gerrit_url}/a/groups/{group_name}",
                auth=self.auth,
                json=group_data
            )
            response.raise_for_status()
            return True
        except requests.RequestException as e:
            print(f"Failed to create group {group_name}: {e}")
            return False
    
    def assign_permissions_to_group(self, project: str, group_name: str, 
                                  permissions: Set[Permission], 
                                  ref_pattern: str = "refs/heads/*") -> bool:
        """Assign permissions to a group for a project"""
        
        # Get current project access rights
        try:
            response = requests.get(
                f"{self.gerrit_url}/a/projects/{project}/access",
                auth=self.auth
            )
            response.raise_for_status()
            
            access_data = response.json()
            
            # Ensure the reference exists in access rights
            if ref_pattern not in access_data:
                access_data[ref_pattern] = {'permissions': {}}
            
            # Add permissions for the group
            for permission in permissions:
                perm_name = self.map_permission_to_gerrit(permission)
                if perm_name:
                    if perm_name not in access_data[ref_pattern]['permissions']:
                        access_data[ref_pattern]['permissions'][perm_name] = {'rules': {}}
                    
                    access_data[ref_pattern]['permissions'][perm_name]['rules'][f'group {group_name}'] = {
                        'action': 'ALLOW',
                        'min': self.get_permission_min_value(permission),
                        'max': self.get_permission_max_value(permission)
                    }
            
            # Update project access rights
            response = requests.post(
                f"{self.gerrit_url}/a/projects/{project}/access",
                auth=self.auth,
                json=access_data
            )
            response.raise_for_status()
            return True
            
        except requests.RequestException as e:
            print(f"Failed to assign permissions: {e}")
            return False
    
    def map_permission_to_gerrit(self, permission: Permission) -> str:
        """Map our permission enum to Gerrit permission names"""
        mapping = {
            Permission.READ: 'read',
            Permission.PUSH: 'push',
            Permission.SUBMIT: 'submit',
            Permission.APPROVE: 'label-Code-Review',
            Permission.VERIFY: 'label-Verified',
            Permission.ABANDON: 'abandon',
            Permission.CREATE_PROJECT: 'createProject'
        }
        return mapping.get(permission)
    
    def get_permission_min_value(self, permission: Permission) -> int:
        """Get minimum value for permission"""
        if permission in [Permission.APPROVE, Permission.VERIFY]:
            return -2
        return 0
    
    def get_permission_max_value(self, permission: Permission) -> int:
        """Get maximum value for permission"""
        if permission == Permission.APPROVE:
            return 2
        elif permission == Permission.VERIFY:
            return 1
        return 0
    
    def setup_project_permissions(self, project: str) -> bool:
        """Setup default permissions for a project"""
        success = True
        
        # Create groups for each role
        for role_name, role in self.roles.items():
            if role.resource_type in [ResourceType.PROJECT, ResourceType.BRANCH]:
                group_name = f"{project}-{role_name}s"
                
                # Create group
                if self.create_gerrit_group(
                    group_name, 
                    f"{role_name.title()}s for {project} project"
                ):
                    # Assign permissions
                    ref_pattern = role.resource_pattern if role.resource_pattern != "*" else "refs/heads/*"
                    success &= self.assign_permissions_to_group(
                        project, group_name, role.permissions, ref_pattern
                    )
        
        return success
    
    def assign_user_to_role(self, username: str, project: str, role_name: str) -> bool:
        """Assign user to a role for a specific project"""
        if role_name not in self.roles:
            print(f"Unknown role: {role_name}")
            return False
        
        group_name = f"{project}-{role_name}s"
        
        try:
            # Add user to group
            response = requests.put(
                f"{self.gerrit_url}/a/groups/{group_name}/members/{username}",
                auth=self.auth
            )
            response.raise_for_status()
            return True
            
        except requests.RequestException as e:
            print(f"Failed to assign user {username} to role {role_name}: {e}")
            return False
    
    def remove_user_from_role(self, username: str, project: str, role_name: str) -> bool:
        """Remove user from a role for a specific project"""
        group_name = f"{project}-{role_name}s"
        
        try:
            response = requests.delete(
                f"{self.gerrit_url}/a/groups/{group_name}/members/{username}",
                auth=self.auth
            )
            response.raise_for_status()
            return True
            
        except requests.RequestException as e:
            print(f"Failed to remove user {username} from role {role_name}: {e}")
            return False
    
    def get_user_permissions(self, username: str, project: str) -> Set[Permission]:
        """Get all permissions for a user in a project"""
        permissions = set()
        
        try:
            # Get user's groups
            response = requests.get(
                f"{self.gerrit_url}/a/accounts/{username}/groups",
                auth=self.auth
            )
            response.raise_for_status()
            
            user_groups = response.json()
            
            # Check each role
            for role_name, role in self.roles.items():
                group_name = f"{project}-{role_name}s"
                
                # Check if user is in this role's group
                if any(group['name'] == group_name for group in user_groups):
                    permissions.update(role.permissions)
            
            return permissions
            
        except requests.RequestException as e:
            print(f"Failed to get user permissions: {e}")
            return set()
    
    def audit_permissions(self, project: str) -> Dict:
        """Audit all permissions for a project"""
        audit_result = {
            'project': project,
            'groups': {},
            'users': {},
            'issues': []
        }
        
        try:
            # Get project access rights
            response = requests.get(
                f"{self.gerrit_url}/a/projects/{project}/access",
                auth=self.auth
            )
            response.raise_for_status()
            
            access_data = response.json()
            
            # Analyze permissions
            for ref, ref_data in access_data.items():
                permissions = ref_data.get('permissions', {})
                
                for perm_name, perm_data in permissions.items():
                    rules = perm_data.get('rules', {})
                    
                    for rule_key, rule_data in rules.items():
                        if rule_key.startswith('group '):
                            group_name = rule_key[6:]  # Remove 'group ' prefix
                            
                            if group_name not in audit_result['groups']:
                                audit_result['groups'][group_name] = []
                            
                            audit_result['groups'][group_name].append({
                                'permission': perm_name,
                                'ref': ref,
                                'min': rule_data.get('min', 0),
                                'max': rule_data.get('max', 0),
                                'action': rule_data.get('action', 'ALLOW')
                            })
            
            return audit_result
            
        except requests.RequestException as e:
            audit_result['issues'].append(f"Failed to audit project: {e}")
            return audit_result

# Example usage
if __name__ == "__main__":
    rbac = GerritRBACManager(
        "https://gerrit.company.com",
        "admin",
        "admin_password"
    )
    
    # Setup permissions for a new project
    project_name = "my-awesome-project"
    
    print(f"Setting up permissions for {project_name}...")
    if rbac.setup_project_permissions(project_name):
        print("✅ Project permissions setup successfully")
        
        # Assign some users to roles
        rbac.assign_user_to_role("john.doe", project_name, "developer")
        rbac.assign_user_to_role("jane.smith", project_name, "senior-developer")
        rbac.assign_user_to_role("bob.wilson", project_name, "tech-lead")
        
        print("✅ Users assigned to roles")
        
        # Audit permissions
        audit_result = rbac.audit_permissions(project_name)
        print("\n📊 Permission Audit:")
        print(json.dumps(audit_result, indent=2))
        
    else:
        print("❌ Failed to setup project permissions")
```

This security chapter provides comprehensive coverage of Gerrit security including network security, authentication, authorization, and best practices. The examples show practical implementations that can be adapted for real enterprise environments.

## Chapter Summary

You've learned comprehensive security measures for Gerrit:

- **Network security** with firewalls, DDoS protection, and SSL/TLS
- **Multi-factor authentication** implementation
- **Role-based access control** with granular permissions
- **Security monitoring** and audit capabilities
- **Best practices** for enterprise security

These security measures ensure your Gerrit deployment meets enterprise security standards and compliance requirements.

---

**Ready for troubleshooting?** Continue to [Chapter 10: Troubleshooting and Maintenance](../10-troubleshooting/README.md)

---

*Continue to [Chapter 10: Troubleshooting and Maintenance](../10-troubleshooting/README.md)*
