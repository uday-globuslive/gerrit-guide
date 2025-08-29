# Chapter 12: Azure Active Directory SSO Integration

## Overview

This chapter provides comprehensive guidance for integrating Gerrit with Azure Active Directory (Azure AD) for Single Sign-On (SSO). You'll learn to configure OAuth 2.0, SAML 2.0, and OpenID Connect authentication methods with Azure AD, enabling seamless user authentication and group management.

## Table of Contents

1. [Azure AD Prerequisites](#azure-ad-prerequisites)
2. [OAuth 2.0 Integration](#oauth-20-integration)
3. [SAML 2.0 Integration](#saml-20-integration)
4. [OpenID Connect Integration](#openid-connect-integration)
5. [Group Synchronization](#group-synchronization)
6. [Advanced Configuration](#advanced-configuration)
7. [Troubleshooting](#troubleshooting)
8. [Best Practices](#best-practices)

## Azure AD Prerequisites

### 1.1 Azure AD Setup Requirements

Before configuring Gerrit SSO, ensure you have:

- **Azure AD tenant** with appropriate permissions
- **Application registration** privileges in Azure AD
- **Gerrit instance** with admin access
- **SSL/TLS certificate** for secure communication
- **Domain verification** (recommended for production)

### 1.2 Azure Portal Configuration

#### Step 1: Register Gerrit Application

1. **Navigate to Azure Portal**
   ```
   https://portal.azure.com
   ```

2. **Go to Azure Active Directory → App registrations → New registration**

3. **Configure Application Registration**
   ```
   Name: Gerrit Code Review
   Supported account types: Accounts in this organizational directory only
   Redirect URI: https://gerrit.company.com/oauth
   ```

#### Step 2: Configure Authentication

```json
{
  "redirectUris": [
    "https://gerrit.company.com/oauth",
    "https://gerrit.company.com/login/oauth2/code/azure"
  ],
  "logoutUrl": "https://gerrit.company.com/logout",
  "implicitGrantSettings": {
    "enableAccessTokenIssuance": false,
    "enableIdTokenIssuance": true
  }
}
```

#### Step 3: API Permissions Setup

**Required Microsoft Graph Permissions:**

```json
{
  "requiredResourceAccess": [
    {
      "resourceAppId": "00000003-0000-0000-c000-000000000000",
      "resourceAccess": [
        {
          "id": "e1fe6dd8-ba31-4d61-89e7-88639da4683d",
          "type": "Scope"
        },
        {
          "id": "64a6cdd6-aab1-4aaf-94b8-3cc8405e90d0",
          "type": "Scope"
        },
        {
          "id": "7427e0e9-2fba-42fe-b0c0-848c9e6a8182",
          "type": "Scope"
        }
      ]
    }
  ]
}
```

**Permission Details:**
- `User.Read` - Read user profile
- `email` - Read user email address
- `offline_access` - Maintain access to data
- `GroupMember.Read.All` - Read group memberships (optional)

## OAuth 2.0 Integration

### 2.1 OAuth Configuration

OAuth 2.0 is the recommended approach for Azure AD integration with Gerrit.

#### Gerrit Configuration

```ini
# /opt/gerrit/review_site/etc/gerrit.config

[gerrit]
    canonicalWebUrl = https://gerrit.company.com

[auth]
    type = OAUTH
    gitBasicAuthPolicy = OAUTH
    
[oauth]
    # Azure AD OAuth endpoints
    authorizeUrl = https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/authorize
    tokenUrl = https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token
    userInfoUrl = https://graph.microsoft.com/v1.0/me
    
    # Application credentials
    clientId = {application-id}
    clientSecret = {client-secret}
    
    # Scopes
    scope = openid profile email
    
    # User mapping
    userNameAttributeName = userPrincipalName
    displayNameAttributeName = displayName
    emailAttributeName = mail
    
    # Optional: Group mapping
    groupsAttributeName = memberOf
    
[httpd]
    filterClass = com.googlesource.gerrit.plugins.oauth.OAuthFilter
```

#### Secure Configuration

Store sensitive information in `secure.config`:

```ini
# /opt/gerrit/review_site/etc/secure.config

[oauth]
    clientSecret = your-azure-ad-client-secret
```

### 2.2 Advanced OAuth Configuration Script

```python
#!/usr/bin/env python3
# azure-oauth-setup.py

import json
import subprocess
import requests
import configparser
from pathlib import Path

class AzureOAuthConfigurator:
    def __init__(self, gerrit_site, tenant_id, client_id, client_secret):
        self.gerrit_site = Path(gerrit_site)
        self.tenant_id = tenant_id
        self.client_id = client_id
        self.client_secret = client_secret
        self.config_file = self.gerrit_site / 'etc' / 'gerrit.config'
        self.secure_file = self.gerrit_site / 'etc' / 'secure.config'
    
    def validate_azure_config(self):
        """Validate Azure AD configuration"""
        print("Validating Azure AD configuration...")
        
        # Test OAuth endpoints
        auth_url = f"https://login.microsoftonline.com/{self.tenant_id}/oauth2/v2.0/authorize"
        token_url = f"https://login.microsoftonline.com/{self.tenant_id}/oauth2/v2.0/token"
        
        try:
            # Check if endpoints are accessible
            response = requests.get(f"https://login.microsoftonline.com/{self.tenant_id}/.well-known/openid_configuration")
            if response.status_code == 200:
                config = response.json()
                print("✅ Azure AD tenant configuration valid")
                print(f"Issuer: {config.get('issuer')}")
                print(f"Authorization endpoint: {config.get('authorization_endpoint')}")
                print(f"Token endpoint: {config.get('token_endpoint')}")
                return True
            else:
                print(f"❌ Failed to validate tenant: {response.status_code}")
                return False
        except Exception as e:
            print(f"❌ Error validating Azure AD: {e}")
            return False
    
    def configure_gerrit_oauth(self):
        """Configure Gerrit for OAuth authentication"""
        print("Configuring Gerrit OAuth settings...")
        
        config = configparser.ConfigParser()
        config.read(self.config_file)
        
        # Remove existing auth configuration
        if 'auth' in config:
            config.remove_section('auth')
        if 'oauth' in config:
            config.remove_section('oauth')
        
        # Add OAuth configuration
        config.add_section('auth')
        config.set('auth', 'type', 'OAUTH')
        config.set('auth', 'gitBasicAuthPolicy', 'OAUTH')
        config.set('auth', 'cookieSecure', 'true')
        config.set('auth', 'enableRunAs', 'false')
        
        config.add_section('oauth')
        config.set('oauth', 'authorizeUrl', 
                  f'https://login.microsoftonline.com/{self.tenant_id}/oauth2/v2.0/authorize')
        config.set('oauth', 'tokenUrl', 
                  f'https://login.microsoftonline.com/{self.tenant_id}/oauth2/v2.0/token')
        config.set('oauth', 'userInfoUrl', 'https://graph.microsoft.com/v1.0/me')
        config.set('oauth', 'clientId', self.client_id)
        config.set('oauth', 'scope', 'openid profile email User.Read')
        config.set('oauth', 'userNameAttributeName', 'userPrincipalName')
        config.set('oauth', 'displayNameAttributeName', 'displayName')
        config.set('oauth', 'emailAttributeName', 'mail')
        config.set('oauth', 'fixLegacyUserId', 'true')
        
        # Write configuration
        with open(self.config_file, 'w') as f:
            config.write(f)
        
        print("✅ Gerrit OAuth configuration updated")
    
    def configure_secure_settings(self):
        """Configure secure OAuth settings"""
        print("Configuring secure OAuth settings...")
        
        secure_config = configparser.ConfigParser()
        if self.secure_file.exists():
            secure_config.read(self.secure_file)
        
        if 'oauth' not in secure_config:
            secure_config.add_section('oauth')
        
        secure_config.set('oauth', 'clientSecret', self.client_secret)
        
        # Write secure configuration
        with open(self.secure_file, 'w') as f:
            secure_config.write(f)
        
        # Set proper permissions
        self.secure_file.chmod(0o600)
        print("✅ Secure OAuth configuration updated")
    
    def setup_group_sync(self, enable_groups=True):
        """Setup Azure AD group synchronization"""
        if not enable_groups:
            return
        
        print("Configuring Azure AD group synchronization...")
        
        config = configparser.ConfigParser()
        config.read(self.config_file)
        
        # Add group sync configuration
        config.set('oauth', 'groupsAttributeName', 'memberOf')
        config.set('oauth', 'groupsUrl', 'https://graph.microsoft.com/v1.0/me/memberOf')
        
        # Write configuration
        with open(self.config_file, 'w') as f:
            config.write(f)
        
        print("✅ Group synchronization configured")
    
    def create_startup_script(self):
        """Create startup validation script"""
        script_content = f'''#!/bin/bash
# azure-oauth-startup.sh - Validate OAuth configuration on startup

GERRIT_SITE="{self.gerrit_site}"
TENANT_ID="{self.tenant_id}"

echo "Validating Azure AD OAuth configuration..."

# Check configuration files
if [ ! -f "$GERRIT_SITE/etc/gerrit.config" ]; then
    echo "❌ Gerrit configuration not found"
    exit 1
fi

if [ ! -f "$GERRIT_SITE/etc/secure.config" ]; then
    echo "❌ Secure configuration not found"
    exit 1
fi

# Validate OAuth endpoints
DISCOVERY_URL="https://login.microsoftonline.com/$TENANT_ID/.well-known/openid_configuration"
if curl -s "$DISCOVERY_URL" > /dev/null; then
    echo "✅ Azure AD tenant accessible"
else
    echo "❌ Azure AD tenant not accessible"
    exit 1
fi

# Check network connectivity to Microsoft Graph
if curl -s "https://graph.microsoft.com/v1.0/\$metadata" > /dev/null; then
    echo "✅ Microsoft Graph accessible"
else
    echo "❌ Microsoft Graph not accessible"
    exit 1
fi

echo "OAuth configuration validation completed"
'''
        
        script_path = self.gerrit_site / 'bin' / 'azure-oauth-startup.sh'
        with open(script_path, 'w') as f:
            f.write(script_content)
        script_path.chmod(0o755)
        
        print(f"✅ Startup script created: {script_path}")
    
    def test_oauth_flow(self):
        """Test OAuth authentication flow"""
        print("Testing OAuth authentication flow...")
        
        # This would typically require browser automation
        # For now, we'll validate the endpoints
        
        discovery_url = f"https://login.microsoftonline.com/{self.tenant_id}/.well-known/openid_configuration"
        try:
            response = requests.get(discovery_url)
            if response.status_code == 200:
                config = response.json()
                print("✅ OAuth discovery endpoint accessible")
                
                # Validate required endpoints
                required_endpoints = [
                    'authorization_endpoint',
                    'token_endpoint',
                    'userinfo_endpoint'
                ]
                
                for endpoint in required_endpoints:
                    if endpoint in config:
                        print(f"✅ {endpoint}: {config[endpoint]}")
                    else:
                        print(f"❌ Missing {endpoint}")
                
                return True
            else:
                print(f"❌ Discovery endpoint returned: {response.status_code}")
                return False
        except Exception as e:
            print(f"❌ Error testing OAuth flow: {e}")
            return False
    
    def deploy_configuration(self):
        """Deploy complete OAuth configuration"""
        print("Deploying Azure AD OAuth configuration...")
        
        steps = [
            ("Validating Azure configuration", self.validate_azure_config),
            ("Configuring Gerrit OAuth", self.configure_gerrit_oauth),
            ("Setting up secure configuration", self.configure_secure_settings),
            ("Setting up group sync", lambda: self.setup_group_sync(True)),
            ("Creating startup script", self.create_startup_script),
            ("Testing OAuth flow", self.test_oauth_flow)
        ]
        
        for step_name, step_func in steps:
            print(f"\n--- {step_name} ---")
            if not step_func():
                print(f"❌ Failed: {step_name}")
                return False
        
        print("\n🎉 Azure AD OAuth configuration completed successfully!")
        print("\nNext steps:")
        print("1. Restart Gerrit service")
        print("2. Test login via Azure AD")
        print("3. Configure group mappings if needed")
        print("4. Update firewall rules for external authentication")
        
        return True

# Usage example
if __name__ == "__main__":
    import sys
    
    if len(sys.argv) != 5:
        print("Usage: python azure-oauth-setup.py <gerrit_site> <tenant_id> <client_id> <client_secret>")
        sys.exit(1)
    
    configurator = AzureOAuthConfigurator(
        gerrit_site=sys.argv[1],
        tenant_id=sys.argv[2],
        client_id=sys.argv[3],
        client_secret=sys.argv[4]
    )
    
    success = configurator.deploy_configuration()
    sys.exit(0 if success else 1)
```

## SAML 2.0 Integration

### 3.1 SAML Configuration

SAML 2.0 provides enterprise-grade SSO with advanced security features.

#### Azure AD SAML Setup

1. **Create Enterprise Application**
   ```
   Azure Portal → Enterprise Applications → New Application → Create your own application
   ```

2. **Configure SAML Settings**
   ```
   Basic SAML Configuration:
   - Identifier (Entity ID): https://gerrit.company.com/saml/metadata
   - Reply URL: https://gerrit.company.com/saml/acs
   - Sign on URL: https://gerrit.company.com/
   ```

3. **User Attributes & Claims**
   ```json
   {
     "claims": [
       {
         "name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
         "source": "user.userprincipalname"
       },
       {
         "name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
         "source": "user.mail"
       },
       {
         "name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname",
         "source": "user.givenname"
       },
       {
         "name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname",
         "source": "user.surname"
       },
       {
         "name": "http://schemas.microsoft.com/ws/2008/06/identity/claims/groups",
         "source": "user.groups"
       }
     ]
   }
   ```

#### Gerrit SAML Configuration

```ini
# /opt/gerrit/review_site/etc/gerrit.config

[auth]
    type = HTTP_LDAP
    httpHeader = X-Forwarded-User
    httpDisplaynameHeader = X-Forwarded-DisplayName
    httpEmailHeader = X-Forwarded-Email
    
[saml]
    serviceProviderEntityId = https://gerrit.company.com/saml/metadata
    identityProviderMetadataUrl = https://login.microsoftonline.com/{tenant-id}/federationmetadata/2007-06/federationmetadata.xml
    
    # Attribute mapping
    usernameAttributeName = http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name
    displayNameAttributeName = http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname
    emailAddressAttributeName = http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress
    
    # Security settings
    wantAssertionsSigned = true
    signRequests = true
    
[httpd]
    filterClass = com.googlesource.gerrit.plugins.saml.SamlWebFilter
```

### 3.2 SAML Plugin Installation

```bash
#!/bin/bash
# install-saml-plugin.sh

GERRIT_SITE="/opt/gerrit/review_site"
PLUGIN_VERSION="3.8.0"

echo "Installing SAML plugin for Gerrit..."

# Download SAML plugin
wget -O saml.jar "https://gerrit-ci.gerritforge.com/job/plugin-saml-stable-3.8/lastSuccessfulBuild/artifact/bazel-bin/plugins/saml/saml.jar"

# Install plugin
cp saml.jar "$GERRIT_SITE/plugins/"

# Create keystore for SAML signing
keytool -genkeypair -alias saml-signing \
    -keyalg RSA -keysize 2048 \
    -keystore "$GERRIT_SITE/etc/saml-keystore.jks" \
    -dname "CN=gerrit.company.com,OU=IT,O=Company,L=City,ST=State,C=US" \
    -keypass changeit -storepass changeit

echo "SAML plugin installed successfully"
```

## OpenID Connect Integration

### 4.1 OpenID Connect Setup

OpenID Connect provides modern authentication with JWT tokens.

#### Azure AD OIDC Configuration

```ini
# /opt/gerrit/review_site/etc/gerrit.config

[auth]
    type = OAUTH
    gitBasicAuthPolicy = OAUTH
    
[oauth]
    # OpenID Connect endpoints
    authorizeUrl = https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/authorize
    tokenUrl = https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token
    userInfoUrl = https://graph.microsoft.com/oidc/userinfo
    
    # OpenID Connect specific
    issuer = https://login.microsoftonline.com/{tenant-id}/v2.0
    jwksUri = https://login.microsoftonline.com/{tenant-id}/discovery/v2.0/keys
    
    # Client configuration
    clientId = {application-id}
    scope = openid profile email
    
    # Claims mapping
    userNameAttributeName = preferred_username
    displayNameAttributeName = name
    emailAttributeName = email
    
    # Security
    responseType = code
    useIdToken = true
    validateIdToken = true
```

### 4.2 JWT Token Validation Script

```python
#!/usr/bin/env python3
# jwt-validator.py

import jwt
import requests
import json
from datetime import datetime
import sys

class AzureJWTValidator:
    def __init__(self, tenant_id, client_id):
        self.tenant_id = tenant_id
        self.client_id = client_id
        self.issuer = f"https://login.microsoftonline.com/{tenant_id}/v2.0"
        self.jwks_uri = f"https://login.microsoftonline.com/{tenant_id}/discovery/v2.0/keys"
        self.jwks = None
    
    def get_jwks(self):
        """Fetch JSON Web Key Set from Azure AD"""
        try:
            response = requests.get(self.jwks_uri)
            response.raise_for_status()
            self.jwks = response.json()
            return True
        except Exception as e:
            print(f"Failed to fetch JWKS: {e}")
            return False
    
    def validate_token(self, token):
        """Validate JWT token from Azure AD"""
        try:
            # Decode header to get key ID
            header = jwt.get_unverified_header(token)
            kid = header.get('kid')
            
            if not kid:
                print("No key ID in token header")
                return False
            
            # Find matching key in JWKS
            signing_key = None
            for key in self.jwks['keys']:
                if key['kid'] == kid:
                    signing_key = jwt.algorithms.RSAAlgorithm.from_jwk(json.dumps(key))
                    break
            
            if not signing_key:
                print(f"No matching key found for kid: {kid}")
                return False
            
            # Validate token
            payload = jwt.decode(
                token,
                signing_key,
                algorithms=['RS256'],
                audience=self.client_id,
                issuer=self.issuer
            )
            
            print("✅ JWT token validation successful")
            print(f"Subject: {payload.get('sub')}")
            print(f"Name: {payload.get('name')}")
            print(f"Email: {payload.get('email')}")
            print(f"Expires: {datetime.fromtimestamp(payload.get('exp'))}")
            
            return True
            
        except jwt.ExpiredSignatureError:
            print("❌ Token has expired")
            return False
        except jwt.InvalidTokenError as e:
            print(f"❌ Invalid token: {e}")
            return False
        except Exception as e:
            print(f"❌ Token validation error: {e}")
            return False
    
    def test_oidc_discovery(self):
        """Test OpenID Connect discovery endpoint"""
        discovery_url = f"https://login.microsoftonline.com/{self.tenant_id}/v2.0/.well-known/openid_configuration"
        
        try:
            response = requests.get(discovery_url)
            response.raise_for_status()
            
            config = response.json()
            print("✅ OIDC Discovery successful")
            print(f"Issuer: {config.get('issuer')}")
            print(f"Authorization endpoint: {config.get('authorization_endpoint')}")
            print(f"Token endpoint: {config.get('token_endpoint')}")
            print(f"UserInfo endpoint: {config.get('userinfo_endpoint')}")
            print(f"JWKS URI: {config.get('jwks_uri')}")
            
            return True
            
        except Exception as e:
            print(f"❌ OIDC Discovery failed: {e}")
            return False

# Usage
if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: python jwt-validator.py <tenant_id> <client_id>")
        sys.exit(1)
    
    validator = AzureJWTValidator(sys.argv[1], sys.argv[2])
    
    # Test discovery
    if validator.test_oidc_discovery():
        # Fetch JWKS for token validation
        validator.get_jwks()
```

## Group Synchronization

### 5.1 Azure AD Group Mapping

Configure automatic group synchronization between Azure AD and Gerrit.

#### Azure AD Group Configuration

```python
#!/usr/bin/env python3
# azure-group-sync.py

import requests
import json
import configparser
from pathlib import Path
import logging

class AzureGroupSync:
    def __init__(self, tenant_id, client_id, client_secret, gerrit_site):
        self.tenant_id = tenant_id
        self.client_id = client_id
        self.client_secret = client_secret
        self.gerrit_site = Path(gerrit_site)
        self.access_token = None
        
        # Setup logging
        logging.basicConfig(level=logging.INFO)
        self.logger = logging.getLogger(__name__)
    
    def get_access_token(self):
        """Get access token for Microsoft Graph API"""
        token_url = f"https://login.microsoftonline.com/{self.tenant_id}/oauth2/v2.0/token"
        
        data = {
            'grant_type': 'client_credentials',
            'client_id': self.client_id,
            'client_secret': self.client_secret,
            'scope': 'https://graph.microsoft.com/.default'
        }
        
        try:
            response = requests.post(token_url, data=data)
            response.raise_for_status()
            
            token_data = response.json()
            self.access_token = token_data['access_token']
            self.logger.info("✅ Access token obtained successfully")
            return True
            
        except Exception as e:
            self.logger.error(f"❌ Failed to get access token: {e}")
            return False
    
    def get_azure_groups(self):
        """Fetch all groups from Azure AD"""
        if not self.access_token:
            if not self.get_access_token():
                return []
        
        headers = {
            'Authorization': f'Bearer {self.access_token}',
            'Content-Type': 'application/json'
        }
        
        groups = []
        url = "https://graph.microsoft.com/v1.0/groups"
        
        while url:
            try:
                response = requests.get(url, headers=headers)
                response.raise_for_status()
                
                data = response.json()
                groups.extend(data.get('value', []))
                url = data.get('@odata.nextLink')
                
            except Exception as e:
                self.logger.error(f"Error fetching groups: {e}")
                break
        
        self.logger.info(f"✅ Retrieved {len(groups)} groups from Azure AD")
        return groups
    
    def get_group_members(self, group_id):
        """Get members of a specific Azure AD group"""
        if not self.access_token:
            return []
        
        headers = {
            'Authorization': f'Bearer {self.access_token}',
            'Content-Type': 'application/json'
        }
        
        members = []
        url = f"https://graph.microsoft.com/v1.0/groups/{group_id}/members"
        
        while url:
            try:
                response = requests.get(url, headers=headers)
                response.raise_for_status()
                
                data = response.json()
                members.extend(data.get('value', []))
                url = data.get('@odata.nextLink')
                
            except Exception as e:
                self.logger.error(f"Error fetching group members: {e}")
                break
        
        return members
    
    def create_gerrit_groups(self, azure_groups, group_mapping=None):
        """Create corresponding groups in Gerrit"""
        group_mapping = group_mapping or {}
        
        gerrit_groups = []
        
        for azure_group in azure_groups:
            group_name = azure_group['displayName']
            group_id = azure_group['id']
            
            # Apply group mapping if provided
            gerrit_group_name = group_mapping.get(group_name, group_name)
            
            # Sanitize group name for Gerrit
            sanitized_name = self.sanitize_group_name(gerrit_group_name)
            
            gerrit_group = {
                'name': sanitized_name,
                'azure_id': group_id,
                'azure_name': group_name,
                'description': azure_group.get('description', f'Synced from Azure AD: {group_name}')
            }
            
            gerrit_groups.append(gerrit_group)
        
        self.logger.info(f"✅ Prepared {len(gerrit_groups)} groups for Gerrit")
        return gerrit_groups
    
    def sanitize_group_name(self, name):
        """Sanitize group name for Gerrit compatibility"""
        # Replace spaces and special characters
        sanitized = name.replace(' ', '-').replace('_', '-')
        # Remove special characters except hyphens
        sanitized = ''.join(c for c in sanitized if c.isalnum() or c == '-')
        # Convert to lowercase
        return sanitized.lower()
    
    def generate_group_mapping_config(self, gerrit_groups):
        """Generate group mapping configuration for Gerrit"""
        config_content = """# Azure AD Group Mapping Configuration
# This file maps Azure AD groups to Gerrit groups

[groups]
# Format: azure-group-name = gerrit-group-name

"""
        
        for group in gerrit_groups:
            config_content += f'"{group["azure_name"]}" = {group["name"]}\n'
        
        config_file = self.gerrit_site / 'etc' / 'azure-group-mapping.conf'
        with open(config_file, 'w') as f:
            f.write(config_content)
        
        self.logger.info(f"✅ Group mapping configuration saved to {config_file}")
    
    def update_gerrit_config(self):
        """Update Gerrit configuration for Azure AD group sync"""
        config_file = self.gerrit_site / 'etc' / 'gerrit.config'
        
        config = configparser.ConfigParser()
        config.read(config_file)
        
        # Add Azure AD group sync configuration
        if 'oauth' not in config:
            config.add_section('oauth')
        
        config.set('oauth', 'groupsAttributeName', 'groups')
        config.set('oauth', 'groupsUrl', 'https://graph.microsoft.com/v1.0/me/memberOf')
        config.set('oauth', 'groupsScope', 'GroupMember.Read.All')
        
        # Add plugin configuration
        if 'plugin "azure-group-sync"' not in config:
            config.add_section('plugin "azure-group-sync"')
        
        config.set('plugin "azure-group-sync"', 'enabled', 'true')
        config.set('plugin "azure-group-sync"', 'syncInterval', '3600')  # 1 hour
        config.set('plugin "azure-group-sync"', 'mappingFile', 'etc/azure-group-mapping.conf')
        
        # Write configuration
        with open(config_file, 'w') as f:
            config.write(f)
        
        self.logger.info("✅ Gerrit configuration updated for Azure AD group sync")
    
    def create_sync_script(self):
        """Create automated group synchronization script"""
        script_content = f'''#!/bin/bash
# azure-group-sync.sh - Automated Azure AD group synchronization

GERRIT_SITE="{self.gerrit_site}"
LOG_FILE="$GERRIT_SITE/logs/azure-group-sync.log"

log() {{
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}}

log "Starting Azure AD group synchronization"

# Run Python sync script
python3 "$GERRIT_SITE/bin/azure-group-sync.py" \\
    "{self.tenant_id}" \\
    "{self.client_id}" \\
    "${{AZURE_CLIENT_SECRET:-{self.client_secret}}}" \\
    "$GERRIT_SITE"

if [ $? -eq 0 ]; then
    log "Group synchronization completed successfully"
    
    # Reload Gerrit groups cache
    ssh -p 29418 admin@localhost gerrit flush-caches --cache groups
    log "Gerrit group cache flushed"
else
    log "Group synchronization failed"
    exit 1
fi
'''
        
        script_path = self.gerrit_site / 'bin' / 'azure-group-sync.sh'
        with open(script_path, 'w') as f:
            f.write(script_content)
        script_path.chmod(0o755)
        
        self.logger.info(f"✅ Sync script created: {script_path}")
    
    def setup_cron_job(self):
        """Setup cron job for regular group synchronization"""
        cron_entry = f"""# Azure AD Group Synchronization
0 */6 * * * {self.gerrit_site}/bin/azure-group-sync.sh
"""
        
        cron_file = self.gerrit_site / 'etc' / 'azure-group-sync.cron'
        with open(cron_file, 'w') as f:
            f.write(cron_entry)
        
        print(f"Add this to crontab for automated synchronization:")
        print(cron_entry)
        self.logger.info("✅ Cron configuration prepared")
    
    def run_full_sync(self):
        """Perform complete group synchronization setup"""
        self.logger.info("Starting complete Azure AD group synchronization setup")
        
        # Get Azure AD groups
        azure_groups = self.get_azure_groups()
        if not azure_groups:
            self.logger.error("No groups found in Azure AD")
            return False
        
        # Create Gerrit group mappings
        gerrit_groups = self.create_gerrit_groups(azure_groups)
        
        # Generate configuration files
        self.generate_group_mapping_config(gerrit_groups)
        self.update_gerrit_config()
        self.create_sync_script()
        self.setup_cron_job()
        
        self.logger.info("🎉 Azure AD group synchronization setup completed!")
        return True

# Usage
if __name__ == "__main__":
    import sys
    
    if len(sys.argv) != 5:
        print("Usage: python azure-group-sync.py <tenant_id> <client_id> <client_secret> <gerrit_site>")
        sys.exit(1)
    
    sync = AzureGroupSync(
        tenant_id=sys.argv[1],
        client_id=sys.argv[2],
        client_secret=sys.argv[3],
        gerrit_site=sys.argv[4]
    )
    
    success = sync.run_full_sync()
    sys.exit(0 if success else 1)
```

## Advanced Configuration

### 6.1 Multi-Factor Authentication (MFA)

Configure MFA requirements for Azure AD authentication.

#### Conditional Access Policy

```json
{
  "displayName": "Gerrit Code Review - Require MFA",
  "state": "enabled",
  "conditions": {
    "applications": {
      "includeApplications": ["{gerrit-app-id}"]
    },
    "users": {
      "includeUsers": ["All"]
    },
    "locations": {
      "includeLocations": ["All"]
    }
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": [
      "mfa",
      "compliantDevice"
    ]
  }
}
```

#### Gerrit MFA Configuration

```ini
# /opt/gerrit/review_site/etc/gerrit.config

[auth]
    type = OAUTH
    requireMultiFactor = true
    
[oauth]
    # MFA-specific claims
    mfaClaimName = amr
    mfaRequiredValue = mfa
    
    # Additional security
    maxSessionAge = 28800  # 8 hours
    enableSessionValidation = true
```

### 6.2 Custom Claims Processing

```python
#!/usr/bin/env python3
# azure-claims-processor.py

import json
import base64
from datetime import datetime, timezone

class AzureClaimsProcessor:
    def __init__(self):
        self.claim_mappings = {
            'preferred_username': 'username',
            'name': 'displayName',
            'email': 'emailAddress',
            'given_name': 'firstName',
            'family_name': 'lastName',
            'groups': 'groups',
            'roles': 'roles'
        }
    
    def process_id_token(self, id_token):
        """Process Azure AD ID token claims"""
        try:
            # Decode token (validation should be done separately)
            parts = id_token.split('.')
            payload = parts[1]
            
            # Add padding if needed
            payload += '=' * (4 - len(payload) % 4)
            
            # Decode base64
            claims = json.loads(base64.b64decode(payload))
            
            return self.process_claims(claims)
            
        except Exception as e:
            print(f"Error processing ID token: {e}")
            return {}
    
    def process_claims(self, claims):
        """Process and map Azure AD claims"""
        processed_claims = {}
        
        # Standard claim mapping
        for azure_claim, gerrit_claim in self.claim_mappings.items():
            if azure_claim in claims:
                processed_claims[gerrit_claim] = claims[azure_claim]
        
        # Custom processing
        processed_claims.update(self.process_custom_claims(claims))
        
        return processed_claims
    
    def process_custom_claims(self, claims):
        """Process custom Azure AD claims"""
        custom_claims = {}
        
        # Process authentication methods
        if 'amr' in claims:
            custom_claims['authMethods'] = claims['amr']
            custom_claims['hasMFA'] = 'mfa' in claims['amr']
        
        # Process tenant information
        if 'tid' in claims:
            custom_claims['tenantId'] = claims['tid']
        
        # Process application roles
        if 'roles' in claims:
            custom_claims['applicationRoles'] = claims['roles']
        
        # Process group memberships
        if 'groups' in claims:
            custom_claims['groups'] = self.process_groups(claims['groups'])
        
        # Process device information
        if 'deviceid' in claims:
            custom_claims['deviceId'] = claims['deviceid']
            custom_claims['isCompliantDevice'] = claims.get('is_compliant', False)
        
        return custom_claims
    
    def process_groups(self, groups):
        """Process Azure AD group claims"""
        processed_groups = []
        
        for group in groups:
            if isinstance(group, str):
                # Group ID format
                processed_groups.append({
                    'id': group,
                    'type': 'azure_ad_group'
                })
            elif isinstance(group, dict):
                # Extended group information
                processed_groups.append({
                    'id': group.get('id'),
                    'name': group.get('displayName'),
                    'type': 'azure_ad_group'
                })
        
        return processed_groups
    
    def validate_claims(self, claims):
        """Validate processed claims"""
        validation_results = {
            'valid': True,
            'errors': [],
            'warnings': []
        }
        
        # Required claims validation
        required_claims = ['username', 'emailAddress']
        for claim in required_claims:
            if claim not in claims or not claims[claim]:
                validation_results['valid'] = False
                validation_results['errors'].append(f"Missing required claim: {claim}")
        
        # Email format validation
        if 'emailAddress' in claims:
            email = claims['emailAddress']
            if '@' not in email:
                validation_results['valid'] = False
                validation_results['errors'].append(f"Invalid email format: {email}")
        
        # Username format validation
        if 'username' in claims:
            username = claims['username']
            if not username.replace('@', '').replace('.', '').replace('-', '').replace('_', '').isalnum():
                validation_results['warnings'].append(f"Username contains special characters: {username}")
        
        return validation_results

# Example usage
if __name__ == "__main__":
    processor = AzureClaimsProcessor()
    
    # Example claims from Azure AD
    sample_claims = {
        'preferred_username': 'user@company.com',
        'name': 'John Doe',
        'email': 'john.doe@company.com',
        'given_name': 'John',
        'family_name': 'Doe',
        'groups': ['group1-id', 'group2-id'],
        'amr': ['pwd', 'mfa'],
        'tid': 'tenant-id'
    }
    
    processed = processor.process_claims(sample_claims)
    validation = processor.validate_claims(processed)
    
    print("Processed Claims:")
    print(json.dumps(processed, indent=2))
    
    print("\nValidation Results:")
    print(json.dumps(validation, indent=2))
```

## Troubleshooting

### 7.1 Common Issues and Solutions

#### Issue: Authentication Redirect Loop

**Symptoms:**
- Users get stuck in endless redirect loop
- Browser shows multiple redirects to Azure AD

**Solution:**
```bash
#!/bin/bash
# fix-redirect-loop.sh

# Check redirect URI configuration
echo "Checking redirect URI configuration..."

GERRIT_CONFIG="/opt/gerrit/review_site/etc/gerrit.config"
CANONICAL_URL=$(grep "canonicalWebUrl" "$GERRIT_CONFIG" | cut -d'=' -f2 | tr -d ' ')

echo "Canonical URL: $CANONICAL_URL"

# Verify Azure AD redirect URI matches
echo "Ensure Azure AD redirect URI is: ${CANONICAL_URL}/oauth"

# Check for HTTPS configuration issues
if [[ "$CANONICAL_URL" == https://* ]]; then
    echo "✅ Using HTTPS"
else
    echo "⚠️ Warning: Not using HTTPS"
fi

# Test OAuth endpoints
TENANT_ID=$(grep "tokenUrl" "$GERRIT_CONFIG" | sed 's/.*\/\([^\/]*\)\/oauth2.*/\1/')
if [ -n "$TENANT_ID" ]; then
    echo "Testing OAuth endpoints for tenant: $TENANT_ID"
    curl -s "https://login.microsoftonline.com/$TENANT_ID/.well-known/openid_configuration" > /dev/null
    if [ $? -eq 0 ]; then
        echo "✅ OAuth endpoints accessible"
    else
        echo "❌ OAuth endpoints not accessible"
    fi
fi
```

#### Issue: Group Synchronization Failures

**Diagnostic Script:**
```python
#!/usr/bin/env python3
# diagnose-group-sync.py

import requests
import json
import sys

def diagnose_group_sync_issues(tenant_id, client_id, client_secret):
    """Diagnose Azure AD group synchronization issues"""
    
    print("Diagnosing Azure AD group synchronization issues...")
    
    # Test 1: Access token acquisition
    print("\n1. Testing access token acquisition...")
    token_url = f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token"
    
    data = {
        'grant_type': 'client_credentials',
        'client_id': client_id,
        'client_secret': client_secret,
        'scope': 'https://graph.microsoft.com/.default'
    }
    
    try:
        response = requests.post(token_url, data=data)
        response.raise_for_status()
        token_data = response.json()
        access_token = token_data['access_token']
        print("✅ Access token acquired successfully")
    except Exception as e:
        print(f"❌ Failed to acquire access token: {e}")
        return False
    
    # Test 2: Groups API access
    print("\n2. Testing Groups API access...")
    headers = {'Authorization': f'Bearer {access_token}'}
    
    try:
        response = requests.get("https://graph.microsoft.com/v1.0/groups?$top=1", headers=headers)
        response.raise_for_status()
        print("✅ Groups API accessible")
    except Exception as e:
        print(f"❌ Groups API access failed: {e}")
        return False
    
    # Test 3: User groups access
    print("\n3. Testing user groups access...")
    try:
        response = requests.get("https://graph.microsoft.com/v1.0/me/memberOf", headers=headers)
        if response.status_code == 200:
            print("✅ User groups API accessible")
        else:
            print(f"⚠️ User groups API returned: {response.status_code}")
    except Exception as e:
        print(f"❌ User groups API failed: {e}")
    
    # Test 4: Application permissions
    print("\n4. Checking application permissions...")
    try:
        response = requests.get(f"https://graph.microsoft.com/v1.0/applications/{client_id}", headers=headers)
        if response.status_code == 200:
            app_data = response.json()
            required_perms = app_data.get('requiredResourceAccess', [])
            print(f"✅ Application permissions: {len(required_perms)} resource access configured")
        else:
            print(f"⚠️ Application permissions check returned: {response.status_code}")
    except Exception as e:
        print(f"❌ Application permissions check failed: {e}")
    
    return True

if __name__ == "__main__":
    if len(sys.argv) != 4:
        print("Usage: python diagnose-group-sync.py <tenant_id> <client_id> <client_secret>")
        sys.exit(1)
    
    success = diagnose_group_sync_issues(sys.argv[1], sys.argv[2], sys.argv[3])
    sys.exit(0 if success else 1)
```

#### Issue: Token Validation Failures

**Token Debugger:**
```python
#!/usr/bin/env python3
# token-debugger.py

import jwt
import json
import base64
from datetime import datetime

def debug_jwt_token(token):
    """Debug JWT token issues"""
    
    print("JWT Token Debugger")
    print("==================")
    
    try:
        # Decode header
        header = jwt.get_unverified_header(token)
        print(f"Header: {json.dumps(header, indent=2)}")
        
        # Decode payload (without verification)
        payload = jwt.decode(token, options={"verify_signature": False})
        print(f"\nPayload: {json.dumps(payload, indent=2)}")
        
        # Check expiration
        if 'exp' in payload:
            exp_time = datetime.fromtimestamp(payload['exp'])
            now = datetime.now()
            if exp_time < now:
                print(f"\n❌ Token expired at: {exp_time}")
            else:
                print(f"\n✅ Token valid until: {exp_time}")
        
        # Check issuer
        if 'iss' in payload:
            print(f"Issuer: {payload['iss']}")
        
        # Check audience
        if 'aud' in payload:
            print(f"Audience: {payload['aud']}")
        
        # Check subject
        if 'sub' in payload:
            print(f"Subject: {payload['sub']}")
        
        return True
        
    except Exception as e:
        print(f"❌ Error decoding token: {e}")
        return False

if __name__ == "__main__":
    import sys
    
    if len(sys.argv) != 2:
        print("Usage: python token-debugger.py <jwt_token>")
        sys.exit(1)
    
    debug_jwt_token(sys.argv[1])
```

## Best Practices

### 8.1 Security Best Practices

1. **Use HTTPS Always**
   ```ini
   [gerrit]
       canonicalWebUrl = https://gerrit.company.com
   
   [httpd]
       listenUrl = https://*:8443/
   ```

2. **Implement Proper Certificate Management**
   ```bash
   # Use proper SSL certificates
   openssl req -x509 -newkey rsa:4096 -keyout gerrit.key -out gerrit.crt -days 365
   ```

3. **Configure Session Security**
   ```ini
   [auth]
       cookieSecure = true
       cookieHttpOnly = true
       maxSessionAge = 28800  # 8 hours
   ```

4. **Enable Audit Logging**
   ```ini
   [log]
       jsonLogging = true
       
   [audit]
       enabled = true
       logPath = logs/audit.log
   ```

### 8.2 Performance Optimization

1. **Cache Configuration**
   ```ini
   [cache "oauth_tokens"]
       memoryLimit = 1024
       maxAge = 1h
       
   [cache "azure_groups"]
       memoryLimit = 512
       maxAge = 6h
   ```

2. **Connection Pooling**
   ```ini
   [httpd]
       maxThreads = 100
       maxQueued = 50
   ```

### 8.3 Monitoring and Alerting

```python
#!/usr/bin/env python3
# azure-sso-monitor.py

import requests
import time
import json
from datetime import datetime
import smtplib
from email.mime.text import MimeText

class AzureSSOMonitor:
    def __init__(self, config):
        self.config = config
        self.alerts = []
    
    def check_oauth_endpoints(self):
        """Monitor OAuth endpoint availability"""
        tenant_id = self.config['tenant_id']
        endpoints = [
            f"https://login.microsoftonline.com/{tenant_id}/.well-known/openid_configuration",
            f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/authorize",
            f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token"
        ]
        
        for endpoint in endpoints:
            try:
                response = requests.get(endpoint, timeout=10)
                if response.status_code != 200:
                    self.alerts.append(f"OAuth endpoint unhealthy: {endpoint} - {response.status_code}")
            except Exception as e:
                self.alerts.append(f"OAuth endpoint unreachable: {endpoint} - {e}")
    
    def check_graph_api(self):
        """Monitor Microsoft Graph API availability"""
        try:
            response = requests.get("https://graph.microsoft.com/v1.0/$metadata", timeout=10)
            if response.status_code != 200:
                self.alerts.append(f"Microsoft Graph API unhealthy: {response.status_code}")
        except Exception as e:
            self.alerts.append(f"Microsoft Graph API unreachable: {e}")
    
    def check_gerrit_auth(self):
        """Check Gerrit authentication status"""
        gerrit_url = self.config['gerrit_url']
        try:
            response = requests.get(f"{gerrit_url}/config/server/info", timeout=10)
            if response.status_code != 200:
                self.alerts.append(f"Gerrit authentication check failed: {response.status_code}")
        except Exception as e:
            self.alerts.append(f"Gerrit unreachable: {e}")
    
    def send_alerts(self):
        """Send alert notifications"""
        if not self.alerts:
            return
        
        smtp_config = self.config.get('smtp', {})
        if not smtp_config:
            print("No SMTP configuration - alerts not sent")
            return
        
        alert_message = "\n".join(self.alerts)
        
        msg = MimeText(f"""
Azure AD SSO Monitor Alert
========================

Timestamp: {datetime.now()}

Issues detected:
{alert_message}

Please investigate and resolve these issues.
""")
        
        msg['Subject'] = 'Azure AD SSO Monitor Alert'
        msg['From'] = smtp_config['from_email']
        msg['To'] = smtp_config['to_email']
        
        try:
            with smtplib.SMTP(smtp_config['server'], smtp_config['port']) as server:
                if smtp_config.get('use_tls'):
                    server.starttls()
                if smtp_config.get('username'):
                    server.login(smtp_config['username'], smtp_config['password'])
                server.send_message(msg)
            print("Alert sent successfully")
        except Exception as e:
            print(f"Failed to send alert: {e}")
    
    def run_monitoring_cycle(self):
        """Run complete monitoring cycle"""
        print(f"Starting monitoring cycle at {datetime.now()}")
        
        self.alerts = []
        
        self.check_oauth_endpoints()
        self.check_graph_api()
        self.check_gerrit_auth()
        
        if self.alerts:
            print(f"Found {len(self.alerts)} issues")
            self.send_alerts()
        else:
            print("All systems healthy")

# Configuration example
config = {
    'tenant_id': 'your-tenant-id',
    'gerrit_url': 'https://gerrit.company.com',
    'smtp': {
        'server': 'smtp.company.com',
        'port': 587,
        'use_tls': True,
        'username': 'monitor@company.com',
        'password': 'password',
        'from_email': 'monitor@company.com',
        'to_email': 'admin@company.com'
    }
}

if __name__ == "__main__":
    monitor = AzureSSOMonitor(config)
    
    # Run continuous monitoring
    while True:
        monitor.run_monitoring_cycle()
        time.sleep(300)  # Check every 5 minutes
```

## Chapter Summary

You've learned comprehensive Azure AD SSO integration:

- **OAuth 2.0 Configuration** - Modern authentication with Azure AD
- **SAML 2.0 Setup** - Enterprise-grade SSO with advanced security
- **OpenID Connect** - Standards-based authentication with JWT tokens
- **Group Synchronization** - Automated Azure AD group mapping
- **Advanced Security** - MFA, conditional access, and audit logging
- **Troubleshooting** - Comprehensive diagnostic and debugging tools
- **Best Practices** - Security, performance, and monitoring guidelines

Azure AD integration provides enterprise-grade authentication with seamless user experience and centralized management.

---

**Ready for the complete enterprise setup?** Continue practicing with your Azure AD environment.

---

*Continue to apply these configurations in your production environment.*
