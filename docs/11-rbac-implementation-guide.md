# 11 - Role-Based Access Control (RBAC) Implementation Guide

This document provides a comprehensive overview of the RBAC implementation for Salad.Inc infrastructure, combining LDAP directory structure with Keycloak SSO integration.

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [LDAP Directory Structure](#2-ldap-directory-structure)
3. [Keycloak Configuration](#3-keycloak-configuration)
4. [Service Integration Matrix](#4-service-integration-matrix)
5. [SSH Access Control](#5-ssh-access-control)
6. [GitLab Team Isolation](#6-gitlab-team-isolation)
7. [Implementation Checklist](#7-implementation-checklist)

## 1. Architecture Overview

### 1.1. Identity & Access Management Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    Identity Architecture                    │
└─────────────────────────────────────────────────────────────┘

┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│   OpenLDAP   │────────▶│   Keycloak   │────────▶│   Services   │
│ (ldap.salad) │         │ (sso.salad)  │         │ (GitLab, etc)│
└──────────────┘         └──────────────┘         └──────────────┘
      │                         │                         │
      │                         │                         │
   Users &                  Federation              SSO/OIDC/SAML
   Groups                   & Roles                 Authentication
```

**Key Principles:**

1. **Single Source of Truth**: OpenLDAP stores all users, groups, and organizational structure
2. **Federation**: Keycloak federates with LDAP for user authentication
3. **SSO Gateway**: Keycloak provides modern SSO protocols (OIDC/SAML) to applications
4. **Least Privilege**: Users only get access to services they need based on their role
5. **Team Isolation**: Team members can only access their team's resources

### 1.2. Components

| Component | Purpose | Location |
|-----------|---------|----------|
| **OpenLDAP** | User directory, groups, roles | `ldap.salad.com` |
| **LAM** | Web UI for LDAP management | `http://lam.salad.com` |
| **Keycloak** | SSO/Identity Provider | `https://keycloak.salad.com` |
| **SSSD** | LDAP authentication on VMs | Installed on each VM |

## 2. LDAP Directory Structure

### 2.1. Complete Hierarchy

```
dc=salad,dc=com
│
├── ou=people                           # All users
│   ├── ou=team-alpha                  # Team Alpha members
│   │   ├── uid=alice.dev
│   │   └── uid=bob.senior
│   ├── ou=team-beta                   # Team Beta members
│   │   └── uid=charlie.dev
│   └── ou=team-infra                  # Infrastructure team
│       ├── uid=diana.arch
│       ├── uid=frank.sre
│       └── uid=grace.sec
│
├── ou=groups                           # Functional groups
│   ├── ou=service-access              # Service access control
│   │   ├── cn=gitlab-users
│   │   ├── cn=prometheus-users
│   │   ├── cn=grafana-users
│   │   ├── cn=portainer-users
│   │   ├── cn=artifactory-users
│   │   ├── cn=lam-admins
│   │   ├── cn=traefik-users
│   │   └── cn=pgadmin-users
│   │
│   ├── ou=ssh-access                  # SSH access control
│   │   ├── cn=ssh-developers          # Non-root SSH
│   │   ├── cn=ssh-admins              # Root SSH
│   │   └── cn=ssh-readonly            # Read-only SSH
│   │
│   └── ou=gitlab-access               # GitLab team projects
│       ├── cn=gitlab-team-alpha
│       ├── cn=gitlab-team-beta
│       └── cn=gitlab-team-infra
│
└── ou=roles                            # Company roles
    ├── cn=developers
    ├── cn=senior-developers
    ├── cn=architects
    ├── cn=managers
    ├── cn=sre-engineers
    └── cn=security-engineers
```

### 2.2. User Attributes

Each user entry contains:

```ldif
dn: uid=alice.dev,ou=team-alpha,ou=people,dc=salad,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: alice.dev                          # Username
cn: Alice Developer                     # Full name
givenName: Alice                        # First name
sn: Developer                           # Last name
mail: alice.dev@salad.com            # Email
uidNumber: 10001                        # UNIX UID
gidNumber: 10001                        # UNIX GID
homeDirectory: /home/alice.dev          # Home directory
loginShell: /bin/bash                   # Shell
userPassword: {SSHA}...                 # Hashed password
```

### 2.3. Group Membership

Groups use `groupOfNames` objectClass:

```ldif
dn: cn=gitlab-users,ou=service-access,ou=groups,dc=salad,dc=com
objectClass: groupOfNames
cn: gitlab-users
description: Users with GitLab access
member: uid=alice.dev,ou=team-alpha,ou=people,dc=salad,dc=com
member: uid=bob.senior,ou=team-alpha,ou=people,dc=salad,dc=com
member: uid=charlie.dev,ou=team-beta,ou=people,dc=salad,dc=com
```

## 3. Keycloak Configuration

### 3.1. Realm Structure

- **master**: Admin realm (Keycloak management only)
- **salad**: Production realm for Salad.Inc

### 3.2. LDAP Federation

Keycloak syncs users and groups from LDAP:

| Setting | Value |
|---------|-------|
| **Provider** | ldap-salad |
| **Edit Mode** | READ_ONLY |
| **Connection URL** | ldap://ldap.salad.com:389 |
| **Users DN** | ou=people,dc=salad,dc=com |
| **Bind DN** | cn=admin,dc=salad,dc=com |
| **Search Scope** | Subtree |

### 3.3. Group Mappers

Four LDAP group mappers sync different group OUs:

1. **service-access-mapper**: `ou=service-access,ou=groups,dc=salad,dc=com`
2. **ssh-access-mapper**: `ou=ssh-access,ou=groups,dc=salad,dc=com`
3. **gitlab-access-mapper**: `ou=gitlab-access,ou=groups,dc=salad,dc=com`
4. **roles-mapper**: `ou=roles,dc=salad,dc=com`

### 3.4. Keycloak Roles

Roles define permissions in Keycloak:

| Role | Purpose |
|------|---------|
| `developer` | Base developer role |
| `senior-developer` | Senior developer role |
| `architect` | Architect role |
| `manager` | Manager role |
| `sre-engineer` | SRE engineer role |
| `security-engineer` | Security engineer role |
| `gitlab-user` | Can access GitLab |
| `prometheus-user` | Can access Prometheus |
| `grafana-user` | Can access Grafana (view) |
| `grafana-admin` | Grafana administrator |
| `portainer-user` | Can access Portainer (view) |
| `portainer-admin` | Portainer administrator |
| `artifactory-user` | Can access Artifactory |
| `artifactory-admin` | Artifactory administrator |
| `lam-admin` | LAM administrator |
| `traefik-user` | Can access Traefik dashboard |
| `pgadmin-user` | Can access pgAdmin |
| `ssh-developer` | Non-root SSH access |
| `ssh-admin` | Root SSH access |

### 3.5. LDAP Group to Keycloak Role Mapping

| LDAP Group | Keycloak Roles Assigned |
|------------|------------------------|
| `cn=developers` | `developer`, `gitlab-user`, `grafana-user`, `artifactory-user`, `ssh-developer` |
| `cn=senior-developers` | `senior-developer`, `gitlab-user`, `prometheus-user`, `grafana-user`, `artifactory-user`, `ssh-developer` |
| `cn=architects` | `architect`, `gitlab-user`, `prometheus-user`, `grafana-user`, `grafana-admin`, `portainer-user`, `artifactory-admin`, `traefik-user`, `ssh-developer` |
| `cn=managers` | `manager`, `gitlab-user`, `prometheus-user`, `grafana-user` |
| `cn=sre-engineers` | `sre-engineer`, `gitlab-user`, `prometheus-user`, `grafana-admin`, `portainer-admin`, `artifactory-admin`, `traefik-user`, `pgadmin-user`, `ssh-developer`, `ssh-admin` |
| `cn=security-engineers` | `security-engineer`, `gitlab-user`, `prometheus-user`, `grafana-admin`, `portainer-user`, `artifactory-user`, `lam-admin`, `traefik-user`, `pgadmin-user`, `ssh-developer`, `ssh-admin` |

### 3.6. Service Clients

Each service has a client in Keycloak:

| Client ID | Service | Redirect URI |
|-----------|---------|--------------|
| `gitlab` | GitLab | `https://gitlab.salad.com/users/auth/openid_connect/callback` |
| `grafana` | Grafana | `https://grafana.salad.com/login/generic_oauth` |
| `portainer` | Portainer | `https://portainer.salad.com/*` |
| `nexus` | Nexus | `https://nexus.salad.com/ui/login` |
| `prometheus` | Prometheus (via OAuth2 Proxy) | `https://prometheus.salad.com/oauth2/callback` |
| `traefik` | Traefik (via OAuth2 Proxy) | `https://traefik.salad.com/oauth2/callback` |
| `lam` | LAM | `https://lam.salad.com/*` |

## 4. Service Integration Matrix

### 4.1. Access Control by Role

| Service | Developer | Senior Dev | Architect | Manager | SRE | Security |
|---------|-----------|------------|-----------|---------|-----|----------|
| **GitLab** | Team only | Team only | All repos | View all | Infra repos | View all |
| **Prometheus** | ✗ | View | View | View | Admin | Admin |
| **Grafana** | View | View | Edit | View | Admin | Admin |
| **Portainer** | ✗ | ✗ | View | ✗ | Admin | View |
| **Artifactory** | Pull | Push/Pull | Admin | ✗ | Admin | View |
| **LAM** | ✗ | ✗ | ✗ | ✗ | ✗ | Admin |
| **Traefik** | ✗ | ✗ | View | ✗ | Admin | View |
| **pgAdmin** | ✗ | ✗ | ✗ | ✗ | Admin | View |
| **SSH (Non-Root)** | Team VMs | Team VMs | All VMs | ✗ | All VMs | All VMs |
| **SSH (Root)** | ✗ | ✗ | ✗ | ✗ | All VMs | Critical VMs |

### 4.2. Example User Permissions

**Alice (Developer, Team Alpha)**
- ✅ Access GitLab (Team Alpha projects only)
- ✅ View Grafana dashboards
- ✅ Pull from Artifactory
- ✅ SSH to Team Alpha VMs (non-root)
- ❌ No Prometheus access
- ❌ No Portainer access
- ❌ No root SSH

**Frank (SRE Engineer, Team Infra)**
- ✅ Access GitLab (Infrastructure projects)
- ✅ Full Prometheus admin
- ✅ Full Grafana admin
- ✅ Full Portainer admin
- ✅ Full Artifactory admin
- ✅ Traefik dashboard admin
- ✅ pgAdmin access
- ✅ SSH to all VMs (non-root)
- ✅ Root SSH to all VMs

## 5. SSH Access Control

### 5.1. SSSD Configuration

Each VM runs SSSD to authenticate against LDAP:

```ini
[sssd]
services = nss, pam, ssh
domains = salad.com

[domain/salad.com]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://ldap.salad.com
ldap_search_base = dc=salad,dc=com
ldap_user_search_base = ou=people,dc=salad,dc=com
ldap_group_search_base = ou=groups,dc=salad,dc=com
```

### 5.2. SSH Access Groups

| Group | Access Level | Members |
|-------|--------------|---------|
| `ssh-admins` | Root SSH to all VMs | SRE engineers, Security engineers |
| `ssh-developers` | Non-root SSH to team VMs | Developers, Senior developers, Architects |
| `ssh-readonly` | Read-only SSH | Managers (if needed) |

### 5.3. Sudo Configuration

`/etc/sudoers.d/ldap-groups`:

```
# Full sudo for admins
%ssh-admins ALL=(ALL:ALL) ALL

# Limited sudo for developers
%ssh-developers ALL=(ALL) NOPASSWD: /bin/systemctl restart *
```

### 5.4. VM-Specific Access Control

You can further restrict access per VM using `/etc/security/access.conf`:

**Example: Only Team Alpha can SSH to a specific VM**

```
+:alice.dev bob.senior:ALL
+:ssh-admins:ALL
-:ALL:ALL
```

## 6. GitLab Team Isolation

### 6.1. GitLab Group Structure

GitLab groups mirror LDAP teams:

```
GitLab Root
├── team-alpha/
│   ├── project-a
│   └── project-b
├── team-beta/
│   ├── project-c
│   └── project-d
└── team-infra/
    ├── infrastructure
    └── ci-cd-pipelines
```

### 6.2. Group Membership Sync

Map LDAP groups to GitLab groups:

| LDAP Group | GitLab Group | Permission Level |
|------------|--------------|------------------|
| `cn=gitlab-team-alpha` | `team-alpha` | Developer/Maintainer |
| `cn=gitlab-team-beta` | `team-beta` | Developer/Maintainer |
| `cn=gitlab-team-infra` | `team-infra` | Developer/Maintainer |

### 6.3. Project Visibility Settings

- **Private**: Only group members can see and access
- **Internal**: All authenticated users can see (not recommended for team isolation)
- **Public**: Anyone can see (not recommended)

**Recommendation**: Set all team projects to **Private**.

### 6.4. Protected Branches

For each project:

1. Go to **Settings** → **Repository** → **Protected Branches**
2. Protect `main` and `develop` branches
3. Set **Allowed to merge**: Maintainers only
4. Set **Allowed to push**: No one (use merge requests)

### 6.5. GitLab CE vs EE Considerations

| Feature | GitLab CE | GitLab EE |
|---------|-----------|-----------|
| **LDAP Group Sync** | Manual | Automatic |
| **Group-level SAML** | ✗ | ✓ |
| **Advanced RBAC** | Basic | Advanced |

**For GitLab CE**, you'll need to:
- Manually add users to groups
- Use GitLab API to automate group membership
- Rely on project visibility for isolation

## 7. Implementation Checklist

### 7.1. LDAP Setup (ldap.salad.com)

- [ ] Install OpenLDAP and LAM
- [ ] Configure base DN: `dc=salad,dc=com`
- [ ] Create organizational units:
  - [ ] `ou=people` with team sub-OUs
  - [ ] `ou=groups` with service-access, ssh-access, gitlab-access sub-OUs
  - [ ] `ou=roles`
- [ ] Create service access groups (gitlab-users, prometheus-users, etc.)
- [ ] Create SSH access groups (ssh-developers, ssh-admins)
- [ ] Create GitLab team groups (gitlab-team-alpha, gitlab-team-beta, gitlab-team-infra)
- [ ] Create role groups (developers, senior-developers, architects, managers, sre-engineers, security-engineers)
- [ ] Create sample users in team OUs
- [ ] Assign users to appropriate groups
- [ ] Test LDAP queries with `ldapsearch`

### 7.2. Keycloak Setup (sso.salad.com)

- [ ] Install and configure Keycloak
- [ ] Create `salad` realm
- [ ] Configure realm settings (email, login options)
- [ ] Add LDAP user federation
- [ ] Configure LDAP group mappers (4 mappers)
- [ ] Synchronize users and groups
- [ ] Create Keycloak roles
- [ ] Map LDAP groups to Keycloak roles
- [ ] Create service clients:
  - [ ] GitLab client with OIDC
  - [ ] Grafana client with OAuth
  - [ ] Portainer client with OAuth
  - [ ] Artifactory client with OIDC
  - [ ] Prometheus client (for OAuth2 Proxy)
  - [ ] Traefik client (for OAuth2 Proxy)
  - [ ] LAM client (optional)
- [ ] Configure client scopes and mappers (groups, roles)
- [ ] Test user login to Keycloak

### 7.3. Service Integration

#### GitLab
- [ ] Configure GitLab OmniAuth for Keycloak OIDC
- [ ] Add client secret to GitLab config
- [ ] Reconfigure GitLab
- [ ] Create GitLab groups (team-alpha, team-beta, team-infra)
- [ ] Set up LDAP group sync (or manual assignment for CE)
- [ ] Create sample projects in each group
- [ ] Set project visibility to Private
- [ ] Configure protected branches
- [ ] Test login with SSO
- [ ] Verify team isolation

#### Grafana
- [ ] Configure generic OAuth in grafana.ini
- [ ] Add client secret
- [ ] Configure role mapping
- [ ] Restart Grafana
- [ ] Test login with SSO
- [ ] Verify role-based permissions

#### Portainer
- [ ] Configure OAuth in Portainer UI
- [ ] Add Keycloak endpoints
- [ ] Test login with SSO
- [ ] Verify role-based permissions

#### Artifactory
- [ ] Configure OIDC in Artifactory
- [ ] Add Keycloak endpoints
- [ ] Test login with SSO
- [ ] Configure repository permissions

#### Prometheus & Traefik (via OAuth2 Proxy)
- [ ] Deploy OAuth2 Proxy container
- [ ] Configure OIDC settings
- [ ] Update Nginx to proxy through OAuth2 Proxy
- [ ] Test login with SSO

### 7.4. SSH Access Control

For each VM:
- [ ] Install SSSD and ldap-utils
- [ ] Configure `/etc/sssd/sssd.conf`
- [ ] Set correct permissions (600)
- [ ] Enable and start SSSD
- [ ] Configure PAM for SSH (`/etc/pam.d/sshd`)
- [ ] Configure access control (`/etc/security/access.conf`)
- [ ] Update SSH config (`/etc/ssh/sshd_config`)
- [ ] Configure sudo (`/etc/sudoers.d/ldap-groups`)
- [ ] Restart SSH service
- [ ] Test SSH login as developer
- [ ] Test SSH login as SRE
- [ ] Test sudo access
- [ ] Verify group-based restrictions

### 7.5. Testing & Validation

- [ ] Test user authentication against LDAP
- [ ] Test user login to Keycloak
- [ ] Test SSO login to GitLab
- [ ] Test SSO login to Grafana
- [ ] Test SSO login to Portainer
- [ ] Test SSO login to Artifactory
- [ ] Test SSH access as different users
- [ ] Test sudo permissions
- [ ] Verify team isolation in GitLab
- [ ] Verify service access restrictions
- [ ] Test password reset flow
- [ ] Test group membership changes propagation

### 7.6. Documentation

- [ ] Document LDAP structure
- [ ] Document Keycloak configuration
- [ ] Document service integration steps
- [ ] Document SSH access procedures
- [ ] Create user onboarding guide
- [ ] Create troubleshooting guide
- [ ] Document backup procedures for LDAP
- [ ] Document disaster recovery procedures

## 8. Troubleshooting

### 8.1. LDAP Issues

**Problem**: Cannot connect to LDAP
```bash
# Test LDAP connectivity
ldapsearch -x -H ldap://ldap.salad.com -b "dc=salad,dc=com"

# Check LDAP service
ssh ldap.salad.com
systemctl status slapd
```

**Problem**: User not found in LDAP
```bash
# Search for specific user
ldapsearch -x -H ldap://ldap.salad.com -b "dc=salad,dc=com" "(uid=alice.dev)"

# Check user's DN
ldapsearch -x -H ldap://ldap.salad.com -b "ou=people,dc=salad,dc=com" "(uid=alice.dev)" dn
```

### 8.2. Keycloak Issues

**Problem**: Users not syncing from LDAP
- Check LDAP connection in Keycloak
- Click "Synchronize all users"
- Check Keycloak logs: `/opt/keycloak/data/log/`

**Problem**: Groups not syncing
- Verify LDAP group mappers are configured
- Click "Sync LDAP Groups to Keycloak" for each mapper
- Check group DN paths are correct

### 8.3. SSH Issues

**Problem**: Cannot SSH with LDAP user
```bash
# Check SSSD status
systemctl status sssd

# Test user lookup
getent passwd alice.dev

# Test group lookup
getent group ssh-developers

# Check SSSD logs
tail -f /var/log/sssd/sssd.log
```

**Problem**: User can SSH but no sudo
```bash
# Check group membership
id alice.dev

# Verify sudoers file
sudo visudo -c
cat /etc/sudoers.d/ldap-groups
```

### 8.4. Service Integration Issues

**Problem**: SSO login fails
- Check client configuration in Keycloak
- Verify redirect URIs match
- Check service logs for OAuth errors
- Verify client secret is correct

## 9. Security Best Practices

1. **LDAP Security**:
   - Use LDAPS (LDAP over SSL) in production
   - Restrict LDAP admin password access
   - Regular backup of LDAP database
   - Enable audit logging

2. **Keycloak Security**:
   - Use strong admin password
   - Enable 2FA for admin accounts
   - Regular security updates
   - Monitor failed login attempts
   - Configure session timeouts

3. **SSH Security**:
   - Disable password authentication (use keys only)
   - Use fail2ban for brute force protection
   - Regular audit of SSH access logs
   - Rotate SSH keys periodically

4. **Service Security**:
   - Use HTTPS for all services
   - Regular security updates
   - Monitor access logs
   - Implement rate limiting
   - Regular security audits

## 10. Maintenance

### 10.1. Adding a New User

1. Create user in LDAP (via LAM or LDIF)
2. Assign to appropriate team OU
3. Add to role group (developers, sre-engineers, etc.)
4. Add to service access groups
5. Add to SSH access group
6. Sync users in Keycloak
7. Verify user can login to services

### 10.2. Removing a User

1. Remove from all LDAP groups
2. Disable user account in LDAP
3. Sync users in Keycloak
4. Revoke SSH keys
5. Audit access logs

### 10.3. Changing User Role

1. Remove from old role group in LDAP
2. Add to new role group
3. Update service access groups accordingly
4. Sync groups in Keycloak
5. Verify new permissions

### 10.4. Adding a New Service

1. Create service access group in LDAP
2. Assign users to group
3. Create client in Keycloak
4. Configure service for OIDC/SAML
5. Test SSO login
6. Update documentation

## 11. References

- **LDAP Setup**: `docs/07-ldap-setup.md`
- **Keycloak Setup**: `docs/10-sso-keycloak-setup.md`
- **OpenLDAP Documentation**: https://www.openldap.org/doc/
- **Keycloak Documentation**: https://www.keycloak.org/documentation
- **SSSD Documentation**: https://sssd.io/docs/
- **GitLab LDAP Integration**: https://docs.gitlab.com/ee/administration/auth/ldap/
- **Grafana OAuth**: https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/generic-oauth/
