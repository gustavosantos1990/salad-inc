# Quick Start Guide: LDAP & Keycloak RBAC Setup

This guide provides a quick overview to get you started with implementing LDAP and Keycloak for role-based access control in your Salad.Inc infrastructure.

## 📋 What We've Designed

### 1. **LDAP Directory Structure** (`docs/07-ldap-setup.md`)

A hierarchical directory that stores:
- **Users** organized by teams (Team Alpha, Team Beta, Team Infra)
- **Groups** for service access control (GitLab, Prometheus, Grafana, etc.)
- **Roles** for company positions (Developer, Architect, Manager, SRE, Security)
- **SSH Access Groups** for VM access control (developers, admins)

### 2. **Keycloak SSO Configuration** (`docs/10-sso-keycloak-setup.md`)

A complete SSO setup that:
- Federates with LDAP for user authentication
- Provides modern SSO protocols (OIDC/SAML) to all services
- Maps LDAP groups to Keycloak roles automatically
- Configures clients for GitLab, Grafana, Portainer, Artifactory, etc.

### 3. **Comprehensive RBAC Guide** (`docs/11-rbac-implementation-guide.md`)

A complete implementation guide with:
- Access control matrix by role
- Service integration instructions
- SSH access control setup
- GitLab team isolation
- Troubleshooting guide
- Implementation checklist

## 🎯 Key Concepts Explained

### What is LDAP?

**LDAP (Lightweight Directory Access Protocol)** is like a phone book for your organization. It stores:
- **Users**: People in your company (alice.dev, frank.sre, etc.)
- **Groups**: Collections of users (developers, gitlab-users, ssh-admins)
- **Organizational Units (OUs)**: Containers to organize entries (ou=people, ou=groups)

**Why use LDAP?**
- Single source of truth for all users
- Centralized user management
- Services can query LDAP to validate users
- SSH authentication can use LDAP

### What is Keycloak?

**Keycloak** is an Identity Provider (IdP) that provides:
- **Single Sign-On (SSO)**: Login once, access all services
- **Federation**: Connects to LDAP to authenticate users
- **Modern Protocols**: OIDC (OpenID Connect) and SAML for web apps
- **Role Management**: Maps LDAP groups to application roles

**Why use Keycloak?**
- Modern web apps don't speak LDAP directly
- Provides OAuth2/OIDC that apps like GitLab, Grafana understand
- Centralized authentication and authorization
- Better security with token-based authentication

### How They Work Together

```
User → Keycloak (SSO Login) → LDAP (Verify User) → Keycloak (Issue Token) → Service (Access Granted)
```

1. User tries to access GitLab
2. GitLab redirects to Keycloak for login
3. Keycloak checks LDAP for user credentials
4. LDAP confirms user exists and password is correct
5. Keycloak checks user's groups in LDAP
6. Keycloak issues a token with user's roles
7. GitLab accepts the token and grants access based on roles

## 🏗️ Architecture Overview

### Components

| Component | What It Does | Where |
|-----------|--------------|-------|
| **OpenLDAP** | Stores users, groups, roles | `ldap.salad.local` |
| **LAM** | Web UI to manage LDAP | `http://lam.salad.local` |
| **Keycloak** | SSO/Identity Provider | `https://keycloak.salad.local` |
| **SSSD** | LDAP auth on VMs | Each VM |

### LDAP Structure

```
dc=salad,dc=local
├── ou=people                    # All users
│   ├── ou=team-alpha           # Team Alpha members
│   ├── ou=team-beta            # Team Beta members
│   └── ou=team-infra           # Infrastructure team
│
├── ou=groups                    # Functional groups
│   ├── ou=service-access       # Service access (gitlab-users, grafana-users, etc.)
│   ├── ou=ssh-access           # SSH access (ssh-developers, ssh-admins)
│   └── ou=gitlab-access        # GitLab teams (gitlab-team-alpha, etc.)
│
└── ou=roles                     # Company roles
    ├── cn=developers
    ├── cn=architects
    ├── cn=managers
    ├── cn=sre-engineers
    └── cn=security-engineers
```

## 👥 Example Users & Their Access

### Alice (Developer, Team Alpha)
- **Can access**:
  - ✅ GitLab (Team Alpha projects only)
  - ✅ Grafana (view dashboards)
  - ✅ Artifactory (pull artifacts)
  - ✅ SSH to Team Alpha VMs (non-root)
- **Cannot access**:
  - ❌ Prometheus
  - ❌ Portainer
  - ❌ Root SSH
  - ❌ Team Beta projects

### Frank (SRE Engineer, Team Infra)
- **Can access**:
  - ✅ All services with admin privileges
  - ✅ GitLab (Infrastructure projects)
  - ✅ Prometheus (admin)
  - ✅ Grafana (admin)
  - ✅ Portainer (admin)
  - ✅ Artifactory (admin)
  - ✅ SSH to all VMs (root access)

## 🚀 Getting Started

### Step 1: Set Up LDAP (30-60 minutes)

Follow `docs/07-ldap-setup.md`:

1. **Install OpenLDAP and LAM** on the LDAP VM
2. **Create the directory structure** using LAM web interface or LDIF files
3. **Create organizational units**: people, groups, roles
4. **Create groups**: service-access, ssh-access, gitlab-access, role groups
5. **Create sample users**: alice.dev, frank.sre, etc.
6. **Assign users to groups** based on their role and team
7. **Test with ldapsearch** to verify structure

**Quick Test:**
```bash
# From your host machine
ldapsearch -x -H ldap://ldap.salad.local -b "dc=salad,dc=local"
```

### Step 2: Set Up Keycloak (45-90 minutes)

Follow `docs/10-sso-keycloak-setup.md`:

1. **Install Keycloak** on the SSO VM
2. **Create the 'salad' realm** (don't use master realm)
3. **Configure LDAP federation** to connect to OpenLDAP
4. **Set up LDAP group mappers** (4 mappers for different group OUs)
5. **Synchronize users and groups** from LDAP
6. **Create Keycloak roles** (developer, gitlab-user, grafana-admin, etc.)
7. **Map LDAP groups to Keycloak roles**
8. **Create service clients** (GitLab, Grafana, Portainer, etc.)

**Quick Test:**
```bash
# Open Keycloak admin console
https://keycloak.salad.local/admin

# Login and check:
# - Users are synced from LDAP
# - Groups are imported
# - Roles are mapped
```

### Step 3: Integrate Services (2-4 hours)

Follow `docs/11-rbac-implementation-guide.md` section 7.3:

1. **GitLab**: Configure OmniAuth for Keycloak OIDC
2. **Grafana**: Configure generic OAuth
3. **Portainer**: Configure OAuth in UI
4. **Artifactory**: Configure OIDC
5. **Prometheus/Traefik**: Deploy OAuth2 Proxy

**Quick Test:**
```bash
# Try logging into GitLab with SSO
https://gitlab.salad.local
# Click "Salad SSO" button
# Login with alice.dev credentials
```

### Step 4: Configure SSH Access (1-2 hours per VM)

Follow `docs/11-rbac-implementation-guide.md` section 13:

For each VM:
1. **Install SSSD** and ldap-utils
2. **Configure SSSD** to connect to LDAP
3. **Configure PAM** for SSH authentication
4. **Set up access control** by group
5. **Configure sudo** permissions

**Quick Test:**
```bash
# From your host machine
ssh alice.dev@containerization.salad.local
# Should work if alice.dev is in ssh-developers group
```

## 📊 Access Control Summary

### By Role

| Role | GitLab | Prometheus | Grafana | Portainer | Artifactory | SSH |
|------|--------|------------|---------|-----------|-------------|-----|
| **Developer** | Team only | ❌ | View | ❌ | Pull | Non-root (team VMs) |
| **Senior Dev** | Team only | View | View | ❌ | Push/Pull | Non-root (team VMs) |
| **Architect** | All repos | View | Edit | View | Admin | Non-root (all VMs) |
| **Manager** | View all | View | View | ❌ | ❌ | ❌ |
| **SRE** | Infra repos | Admin | Admin | Admin | Admin | Root (all VMs) |
| **Security** | View all | Admin | Admin | View | View | Root (critical VMs) |

### GitLab Team Isolation

- **Team Alpha** members can only see/access Team Alpha projects
- **Team Beta** members can only see/access Team Beta projects
- **Team Infra** members can only see/access Infrastructure projects
- **Architects** and **Managers** can view all projects
- **SRE** and **Security** have admin access to their respective areas

## 🔧 Common Tasks

### Adding a New User

1. **In LAM** (`http://lam.salad.local`):
   - Navigate to the user's team OU (e.g., `ou=team-alpha,ou=people`)
   - Create new user with uid, name, email, password
   - Add to role group (e.g., `cn=developers`)
   - Add to service groups (e.g., `cn=gitlab-users`, `cn=grafana-users`)
   - Add to SSH group (e.g., `cn=ssh-developers`)
   - Add to GitLab team group (e.g., `cn=gitlab-team-alpha`)

2. **In Keycloak**:
   - Go to User Federation → ldap-salad
   - Click "Synchronize all users"
   - Verify user appears in Users list

3. **In GitLab** (if using CE):
   - Manually add user to the team group
   - Or use GitLab API to automate

### Changing User Role

1. **In LAM**:
   - Remove user from old role group
   - Add user to new role group
   - Update service access groups accordingly

2. **In Keycloak**:
   - Sync groups
   - Verify role mappings are updated

### Removing User Access

1. **In LAM**:
   - Remove from all groups
   - Disable user account (or delete)

2. **In Keycloak**:
   - Sync users
   - User will lose access to all services

## 🐛 Troubleshooting

### LDAP Issues

**Problem**: Cannot connect to LDAP
```bash
# Test connectivity
ldapsearch -x -H ldap://ldap.salad.local -b "dc=salad,dc=local"

# Check LDAP service
ssh ldap.salad.local
systemctl status slapd
journalctl -u slapd -f
```

**Problem**: User not found
```bash
# Search for user
ldapsearch -x -H ldap://ldap.salad.local -b "dc=salad,dc=local" "(uid=alice.dev)"
```

### Keycloak Issues

**Problem**: Users not syncing
- Check LDAP connection test in Keycloak
- Click "Synchronize all users" button
- Check Keycloak logs: `/opt/keycloak/data/log/`

**Problem**: Groups not syncing
- Verify LDAP group mapper configuration
- Check LDAP Groups DN is correct
- Click "Sync LDAP Groups to Keycloak"

### SSH Issues

**Problem**: Cannot SSH with LDAP user
```bash
# On the VM
systemctl status sssd
getent passwd alice.dev  # Should show user info
getent group ssh-developers  # Should show group members
tail -f /var/log/sssd/sssd.log
```

**Problem**: SSH works but no sudo
```bash
# Check user's groups
id alice.dev

# Verify sudoers file
cat /etc/sudoers.d/ldap-groups
```

### Service Integration Issues

**Problem**: SSO login fails
- Verify client secret is correct in service config
- Check redirect URIs match in Keycloak
- Check service logs for OAuth errors
- Test with Keycloak's built-in account console

## 📚 Documentation References

- **LDAP Setup**: `docs/07-ldap-setup.md`
- **Keycloak Setup**: `docs/10-sso-keycloak-setup.md`
- **RBAC Implementation Guide**: `docs/11-rbac-implementation-guide.md`

## 🎓 Learning Resources

### LDAP Basics
- **What is DN?** Distinguished Name - full path to an entry (e.g., `uid=alice.dev,ou=team-alpha,ou=people,dc=salad,dc=local`)
- **What is CN?** Common Name - identifies an entry (e.g., `cn=developers`)
- **What is OU?** Organizational Unit - container for organizing entries (e.g., `ou=people`)
- **What is objectClass?** Defines the type of entry (e.g., `inetOrgPerson` for users, `groupOfNames` for groups)

### Keycloak Basics
- **Realm**: A namespace for users, roles, and clients (we use `salad`)
- **Client**: An application that uses Keycloak for authentication (e.g., GitLab, Grafana)
- **Role**: A permission that can be assigned to users (e.g., `developer`, `gitlab-user`)
- **Federation**: Connecting to external user stores like LDAP
- **Mapper**: Transforms LDAP data to Keycloak format (e.g., group mapper)

### OAuth/OIDC Basics
- **OAuth2**: Authorization framework (who can access what)
- **OIDC**: Authentication layer on top of OAuth2 (who you are)
- **Token**: Encrypted data containing user info and permissions
- **Redirect URI**: Where to send user after successful login

## ✅ Implementation Checklist

Use this checklist to track your progress:

### LDAP Setup
- [ ] OpenLDAP installed and running
- [ ] LAM installed and accessible
- [ ] Base DN configured (`dc=salad,dc=local`)
- [ ] OUs created (people, groups, roles)
- [ ] Team OUs created under people
- [ ] Service access groups created
- [ ] SSH access groups created
- [ ] GitLab access groups created
- [ ] Role groups created
- [ ] Sample users created
- [ ] Users assigned to groups
- [ ] LDAP queries tested

### Keycloak Setup
- [ ] Keycloak installed and running
- [ ] Salad realm created
- [ ] LDAP federation configured
- [ ] LDAP connection tested
- [ ] Users synchronized from LDAP
- [ ] Group mappers configured (4 mappers)
- [ ] Groups synchronized from LDAP
- [ ] Keycloak roles created
- [ ] LDAP groups mapped to Keycloak roles
- [ ] Service clients created
- [ ] Client scopes and mappers configured
- [ ] Test user login to Keycloak

### Service Integration
- [ ] GitLab SSO configured
- [ ] Grafana OAuth configured
- [ ] Portainer OAuth configured
- [ ] Artifactory OIDC configured
- [ ] Prometheus OAuth2 Proxy (optional)
- [ ] Traefik OAuth2 Proxy (optional)

### SSH Access Control
- [ ] SSSD installed on VMs
- [ ] SSSD configured for LDAP
- [ ] PAM configured for SSH
- [ ] Access control configured
- [ ] Sudo permissions configured
- [ ] SSH access tested

### Testing & Validation
- [ ] Test user login to all services
- [ ] Verify role-based permissions
- [ ] Test team isolation in GitLab
- [ ] Test SSH access as different users
- [ ] Test sudo permissions
- [ ] Verify access restrictions work

## 🎉 Next Steps

After completing the setup:

1. **Create real users** for your team
2. **Configure GitLab groups** and projects
3. **Set up monitoring** dashboards in Grafana
4. **Configure backup** for LDAP database
5. **Enable audit logging** for security
6. **Document** your specific configurations
7. **Train users** on SSO login process

## 💡 Tips

- **Start small**: Set up LDAP and Keycloak first, then integrate services one by one
- **Test frequently**: After each step, verify it works before moving on
- **Use LAM**: The web interface is much easier than manual LDIF files
- **Keep it simple**: Don't overcomplicate the group structure initially
- **Document changes**: Keep notes of what you configure
- **Backup LDAP**: Regular backups of the LDAP database are critical

## 🆘 Getting Help

If you get stuck:

1. **Check the logs**: Most issues are visible in service logs
2. **Test connectivity**: Use `ldapsearch`, `curl`, etc. to test connections
3. **Review documentation**: The detailed docs have troubleshooting sections
4. **Start over**: Sometimes it's faster to recreate a configuration than debug it

Good luck with your implementation! 🚀
