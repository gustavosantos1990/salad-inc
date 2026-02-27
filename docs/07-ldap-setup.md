# 06 - Provisioning Directory Server (OpenLDAP + LAM)

We will provision the **Directory** VM (`ldap.salad.local`). This is a lightweight **Debian 12** instance running **OpenLDAP** (for user management) and **LDAP Account Manager** (for a nice Web UI).

## 1. Directory & Disk

Create the directory and disk image:

```bash
mkdir -p infrastructure/vms/ldap
```

Create Disk (5G is plenty for a directory):
```bash
qemu-img create -f qcow2 infrastructure/vms/ldap/ldap.qcow2 5G
```

## 2. Install Debian (ISO)

```bash
sudo virt-install \
  --name ldap \
  --ram 512 \
  --vcpus 1 \
  --network network=salad-net,mac=52:54:00:00:00:46 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/ldap/ldap.qcow2,size=5 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/debian-12.12.0-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics spice \
  --noautoconsole
```

### Installation Pointers
1.  **Hostname:** `ldap.salad.local`
2.  **Domain:** `salad.local`
3.  **Partitioning:** Standard / (Root).
4.  **Software Selection:**
    *   [X] SSH server
    *   [X] Standard system utilities

## 3. Post-Install Configuration

Login as `root` (or `sudo -i`).

### 1. System Prep
```bash
apt-get update && apt-get install -y curl openssh-server net-tools dnsutils vim
```

**Disable Swap**:
```bash
swapoff -a
sed -i '/swap/d' /etc/fstab
```

**Configure DNS**:
```bash
echo "nameserver 192.168.123.10" > /etc/resolv.conf
```

**Enable Serial Console**:
```bash
systemctl enable --now serial-getty@ttyS0.service
```

### 2. Install & Configure OpenLDAP

Install the server and utilities:
```bash
apt-get install -y slapd ldap-utils
```
*During install, it might ask for an admin password. Remember it.*

**Reconfigure Directory Domain**:
By default it might pick up the wrong domain. Run this to enforce `salad.local`:

```bash
dpkg-reconfigure slapd
```

**Answers:**
1.  Omit OpenLDAP server configuration? **No**
2.  DNS domain name: **salad.local**
3.  Organization name: **Salad Inc**
4.  Administrator password: **(Set your password)**
5.  Database backend: **MDB**
6.  Remove database when purged? **No**
7.  Move old database? **Yes**

### 3. Install LDAP Account Manager (LAM)

LAM provides a web interface to manage users easily.

```bash
apt-get install -y ldap-account-manager
```

### 4. Install Node Exporter
```bash
apt-get install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
```

## 4. LDAP Directory Structure Design

### 4.1. Understanding LDAP Components

**LDAP (Lightweight Directory Access Protocol)** organizes data in a hierarchical tree structure. Here are the key components:

- **dc (Domain Component)**: Represents parts of your domain name (e.g., `dc=salad,dc=local`)
- **ou (Organizational Unit)**: Containers to organize entries (e.g., `ou=people`, `ou=groups`)
- **cn (Common Name)**: Identifies an entry (e.g., `cn=admin`, `cn=developers`)
- **uid (User ID)**: Unique identifier for users (e.g., `uid=jdoe`)
- **objectClass**: Defines what type of entry it is (e.g., `inetOrgPerson`, `groupOfNames`)

### 4.2. Salad.Inc LDAP Structure

We'll create a structure that supports:
- **Users** organized by teams
- **Groups** for service access control
- **Roles** for company positions
- **SSH Access Groups** for VM access control

```
dc=salad,dc=local
├── ou=people                    # All users
│   ├── ou=team-alpha           # Team Alpha members
│   ├── ou=team-beta            # Team Beta members
│   └── ou=team-infra           # Infrastructure team
│
├── ou=groups                    # Functional groups
│   ├── ou=service-access       # Service access groups
│   │   ├── cn=gitlab-users
│   │   ├── cn=prometheus-users
│   │   ├── cn=grafana-users
│   │   ├── cn=portainer-users
│   │   ├── cn=artifactory-users
│   │   ├── cn=lam-admins
│   │   ├── cn=traefik-users
│   │   └── cn=pgadmin-users
│   │
│   ├── ou=ssh-access           # SSH access groups
│   │   ├── cn=ssh-developers   # Non-root SSH access
│   │   ├── cn=ssh-admins       # Root SSH access
│   │   └── cn=ssh-readonly     # Read-only SSH access
│   │
│   └── ou=gitlab-access        # GitLab project access
│       ├── cn=gitlab-team-alpha
│       ├── cn=gitlab-team-beta
│       └── cn=gitlab-team-infra
│
└── ou=roles                     # Company roles
    ├── cn=developers
    ├── cn=senior-developers
    ├── cn=architects
    ├── cn=managers
    ├── cn=sre-engineers
    └── cn=security-engineers
```

### 4.3. Access Control Matrix

This matrix defines which roles have access to which services:

| Role | GitLab | Prometheus | Grafana | Portainer | Artifactory | LAM | Traefik | pgAdmin | SSH (Non-Root) | SSH (Root) |
|------|--------|------------|---------|-----------|-------------|-----|---------|---------|----------------|------------|
| **Developer** | ✓ (Team) | ✗ | ✓ (View) | ✗ | ✓ (Pull) | ✗ | ✗ | ✗ | ✓ (Team VMs) | ✗ |
| **Senior Developer** | ✓ (Team) | ✓ (View) | ✓ (View) | ✗ | ✓ (Push/Pull) | ✗ | ✗ | ✗ | ✓ (Team VMs) | ✗ |
| **Architect** | ✓ (All) | ✓ (View) | ✓ (Edit) | ✓ (View) | ✓ (Admin) | ✗ | ✓ (View) | ✗ | ✓ (All VMs) | ✗ |
| **Manager** | ✓ (View All) | ✓ (View) | ✓ (View) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| **SRE Engineer** | ✓ (Infra) | ✓ (Admin) | ✓ (Admin) | ✓ (Admin) | ✓ (Admin) | ✗ | ✓ (Admin) | ✓ (Admin) | ✓ (All VMs) | ✓ (All VMs) |
| **Security Engineer** | ✓ (View All) | ✓ (Admin) | ✓ (Admin) | ✓ (View) | ✓ (View) | ✓ (Admin) | ✓ (View) | ✓ (View) | ✓ (All VMs) | ✓ (Critical VMs) |

### 4.4. Example Users

We'll create sample users representing different roles and teams:

| UID | Name | Team | Role | Email |
|-----|------|------|------|-------|
| `alice.dev` | Alice Developer | Team Alpha | Developer | alice.dev@salad.local |
| `bob.senior` | Bob Senior | Team Alpha | Senior Developer | bob.senior@salad.local |
| `charlie.dev` | Charlie Developer | Team Beta | Developer | charlie.dev@salad.local |
| `diana.arch` | Diana Architect | Team Infra | Architect | diana.arch@salad.local |
| `eve.manager` | Eve Manager | - | Manager | eve.manager@salad.local |
| `frank.sre` | Frank SRE | Team Infra | SRE Engineer | frank.sre@salad.local |
| `grace.sec` | Grace Security | Team Infra | Security Engineer | grace.sec@salad.local |

## 5. Creating the LDAP Structure

### 5.1. Using LAM (Web Interface)

From your **Host Machine**:

1.  **Access the UI:**
    Open `http://ldap.salad.local/lam` in your browser.

2.  **Login to LAM:**
    *   Click **"LAM configuration"** (top right) in the login page.
    *   Default password is `lam`.
    *   **Edit Server Profile:**
        *   Server address: `ldap://localhost:389`
        *   Tree suffix: `dc=salad,dc=local`
        *   Admin user: `cn=admin,dc=salad,dc=local`
    *   **Save**.
    *   Go back to login.
    *   Login with password set in Step 3.2.

### 5.2. Remove Auto-Created OUs (If Present)

During the `dpkg-reconfigure slapd` step, Debian's OpenLDAP package may automatically create some default OUs with **uppercase** names (`ou=People` and `ou=Groups`). Since our documentation uses **lowercase** naming conventions throughout, we need to remove these first.

**Check if they exist:**

In LAM's Tree view, expand `dc=salad,dc=local` and look for `ou=People` and `ou=Groups`.

**If they exist, remove them from the LDAP VM:**

```bash
# Delete the uppercase OUs (if they exist)
ldapdelete -x -D "cn=admin,dc=salad,dc=local" -W "ou=People,dc=salad,dc=local"
ldapdelete -x -D "cn=admin,dc=salad,dc=local" -W "ou=Groups,dc=salad,dc=local"
```

Enter your LDAP admin password when prompted.

**Note:** If these OUs don't exist, you'll get an error message - that's fine, just proceed to the next step.

### 5.3. Create Organizational Units

1. Click **"Tree view"** in the top menu
2. **Right-click** on `dc=salad,dc=local`
3. Click **"Create a child entry"** (or similar option from the context menu)
4. Select **"Generic: Organizational Unit"**
5. Create the following OUs (repeat for each):
   - `ou=people`
   - `ou=groups`
   - `ou=roles`

6. Under `ou=people`, create team OUs:
   - `ou=team-alpha,ou=people,dc=salad,dc=local`
   - `ou=team-beta,ou=people,dc=salad,dc=local`
   - `ou=team-infra,ou=people,dc=salad,dc=local`

7. Under `ou=groups`, create sub-OUs:
   - `ou=service-access,ou=groups,dc=salad,dc=local`
   - `ou=ssh-access,ou=groups,dc=salad,dc=local`
   - `ou=gitlab-access,ou=groups,dc=salad,dc=local`

### 5.4. Create Groups

We need to create two types of groups depending on their purpose:

- **groupOfNames**: For application-level access control (Keycloak, GitLab, web apps)
  - Uses `member` attribute with full DNs
  - Required for service authorization
  
- **POSIX Group**: For Linux system-level access (SSH, file permissions, sudo)
  - Uses `memberUid` attribute with usernames only
  - Requires `gidNumber` (numeric group ID)

#### 5.4.1. Service Access Groups (Use **groupOfNames**)

Navigate to `ou=service-access,ou=groups,dc=salad,dc=local`:

1. Right-click → **"Create a child entry"**
2. Select **"Generic: groupOfNames"**
3. Create each group below:

**Groups to create:**
- `cn=gitlab-users`
- `cn=prometheus-users`
- `cn=grafana-users`
- `cn=portainer-users`
- `cn=artifactory-users`
- `cn=lam-admins`
- `cn=traefik-users`
- `cn=pgadmin-users`


**Note:** When creating a `groupOfNames`, LAM requires at least one member. You can use a placeholder like `cn=placeholder,dc=salad,dc=local` initially, then remove it after adding real users.

#### 5.4.2. SSH Access Groups (Use **POSIX Group**)

Navigate to `ou=ssh-access,ou=groups,dc=salad,dc=local`:

1. Right-click → **"Create a child entry"**
2. Select **"Generic: Posix Group"**
3. Set a unique `gidNumber` for each group (e.g., 5001, 5002, 5003)
4. Create each group below:

**Groups to create:**
- `cn=ssh-developers` (gidNumber: 5001)
- `cn=ssh-admins` (gidNumber: 5002)
- `cn=ssh-readonly` (gidNumber: 5003)

**Note:** POSIX groups use `memberUid` (just the username, e.g., `alice.dev`), not full DNs. You'll add members later.

#### 5.4.3. GitLab Access Groups (Use **groupOfNames**)

Navigate to `ou=gitlab-access,ou=groups,dc=salad,dc=local`:

1. Right-click → **"Create a child entry"**
2. Select **"Generic: groupOfNames"**
3. Create each group below:

**Groups to create:**
- `cn=gitlab-team-alpha`
- `cn=gitlab-team-beta`
- `cn=gitlab-team-infra`

#### 5.4.4. Role Groups (Use **groupOfNames**)

Navigate to `ou=roles,dc=salad,dc=local`:

1. Right-click → **"Create a child entry"**
2. Select **"Generic: groupOfNames"**
3. Create each group below:

**Groups to create:**
- `cn=developers`
- `cn=senior-developers`
- `cn=architects`
- `cn=managers`
- `cn=sre-engineers`
- `cn=security-engineers`

### 5.5. Create Users

For each user, navigate to their team OU and create a **"Generic: User Account"**:

**Example: Alice Developer (Team Alpha)**
- Navigate to `ou=team-alpha,ou=people,dc=salad,dc=local`
- Right-click → **"Create a child entry"**
- Select **"inetOrgPerson"**
- Fill in:
  - **User ID (uid)**: `alice.dev`
  - **First Name (givenName)**: `Alice`
  - **Last Name (sn)**: `Developer`
  - **Common Name (cn)**: `Alice Developer`
  - **Email (mail)**: `alice.dev@salad.local`
  - **Password**: Set a password

**Adding users to groups:**

After creating the user, you need to add them to the appropriate groups. The method differs based on group type:

- **For groupOfNames groups** (service-access, gitlab-access, roles):
  - Navigate to the group (e.g., `cn=developers,ou=roles,dc=salad,dc=local`)
  - Edit the group and add the user's **full DN** to the `member` attribute
  - Example: `uid=alice.dev,ou=team-alpha,ou=people,dc=salad,dc=local`

- **For POSIX groups** (ssh-access):
  - Navigate to the group (e.g., `cn=ssh-developers,ou=ssh-access,ou=groups,dc=salad,dc=local`)
  - Edit the group and add the user's **username only** to the `memberUid` attribute
  - Example: `alice.dev`

**Groups for Alice Developer:**
  - `cn=developers,ou=roles,dc=salad,dc=local` (groupOfNames)
  - `cn=gitlab-users,ou=service-access,ou=groups,dc=salad,dc=local` (groupOfNames)
  - `cn=grafana-users,ou=service-access,ou=groups,dc=salad,dc=local` (groupOfNames)
  - `cn=artifactory-users,ou=service-access,ou=groups,dc=salad,dc=local` (groupOfNames)
  - `cn=ssh-developers,ou=ssh-access,ou=groups,dc=salad,dc=local` (POSIX)
  - `cn=gitlab-team-alpha,ou=gitlab-access,ou=groups,dc=salad,dc=local` (groupOfNames)


Repeat for all users following the Access Control Matrix.

### 5.6. Using LDIF Files (Alternative Method)

You can also create the structure using LDIF files. Create a file `salad-structure.ldif`:

```ldif
# Organizational Units
dn: ou=people,dc=salad,dc=local
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=salad,dc=local
objectClass: organizationalUnit
ou: groups

dn: ou=roles,dc=salad,dc=local
objectClass: organizationalUnit
ou: roles

# Team OUs
dn: ou=team-alpha,ou=people,dc=salad,dc=local
objectClass: organizationalUnit
ou: team-alpha

dn: ou=team-beta,ou=people,dc=salad,dc=local
objectClass: organizationalUnit
ou: team-beta

dn: ou=team-infra,ou=people,dc=salad,dc=local
objectClass: organizationalUnit
ou: team-infra

# Group Sub-OUs
dn: ou=service-access,ou=groups,dc=salad,dc=local
objectClass: organizationalUnit
ou: service-access

dn: ou=ssh-access,ou=groups,dc=salad,dc=local
objectClass: organizationalUnit
ou: ssh-access

dn: ou=gitlab-access,ou=groups,dc=salad,dc=local
objectClass: organizationalUnit
ou: gitlab-access

# Service Access Groups
dn: cn=gitlab-users,ou=service-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: gitlab-users
description: Users with GitLab access
member: cn=placeholder,dc=salad,dc=local

dn: cn=prometheus-users,ou=service-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: prometheus-users
description: Users with Prometheus access
member: cn=placeholder,dc=salad,dc=local

dn: cn=grafana-users,ou=service-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: grafana-users
description: Users with Grafana access
member: cn=placeholder,dc=salad,dc=local

dn: cn=portainer-users,ou=service-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: portainer-users
description: Users with Portainer access
member: cn=placeholder,dc=salad,dc=local

dn: cn=artifactory-users,ou=service-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: artifactory-users
description: Users with Artifactory access
member: cn=placeholder,dc=salad,dc=local

dn: cn=lam-admins,ou=service-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: lam-admins
description: LAM administrators
member: cn=placeholder,dc=salad,dc=local

dn: cn=traefik-users,ou=service-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: traefik-users
description: Users with Traefik dashboard access
member: cn=placeholder,dc=salad,dc=local

dn: cn=pgadmin-users,ou=service-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: pgadmin-users
description: Users with pgAdmin access
member: cn=placeholder,dc=salad,dc=local

# SSH Access Groups
dn: cn=ssh-developers,ou=ssh-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: ssh-developers
description: Non-root SSH access to team VMs
member: cn=placeholder,dc=salad,dc=local

dn: cn=ssh-admins,ou=ssh-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: ssh-admins
description: Root SSH access to all VMs
member: cn=placeholder,dc=salad,dc=local

dn: cn=ssh-readonly,ou=ssh-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: ssh-readonly
description: Read-only SSH access
member: cn=placeholder,dc=salad,dc=local

# GitLab Access Groups
dn: cn=gitlab-team-alpha,ou=gitlab-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: gitlab-team-alpha
description: Team Alpha GitLab projects
member: cn=placeholder,dc=salad,dc=local

dn: cn=gitlab-team-beta,ou=gitlab-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: gitlab-team-beta
description: Team Beta GitLab projects
member: cn=placeholder,dc=salad,dc=local

dn: cn=gitlab-team-infra,ou=gitlab-access,ou=groups,dc=salad,dc=local
objectClass: groupOfNames
cn: gitlab-team-infra
description: Infrastructure team GitLab projects
member: cn=placeholder,dc=salad,dc=local

# Role Groups
dn: cn=developers,ou=roles,dc=salad,dc=local
objectClass: groupOfNames
cn: developers
description: Developer role
member: cn=placeholder,dc=salad,dc=local

dn: cn=senior-developers,ou=roles,dc=salad,dc=local
objectClass: groupOfNames
cn: senior-developers
description: Senior Developer role
member: cn=placeholder,dc=salad,dc=local

dn: cn=architects,ou=roles,dc=salad,dc=local
objectClass: groupOfNames
cn: architects
description: Architect role
member: cn=placeholder,dc=salad,dc=local

dn: cn=managers,ou=roles,dc=salad,dc=local
objectClass: groupOfNames
cn: managers
description: Manager role
member: cn=placeholder,dc=salad,dc=local

dn: cn=sre-engineers,ou=roles,dc=salad,dc=local
objectClass: groupOfNames
cn: sre-engineers
description: SRE Engineer role
member: cn=placeholder,dc=salad,dc=local

dn: cn=security-engineers,ou=roles,dc=salad,dc=local
objectClass: groupOfNames
cn: security-engineers
description: Security Engineer role
member: cn=placeholder,dc=salad,dc=local

# Example User: Alice Developer (Team Alpha)
dn: uid=alice.dev,ou=team-alpha,ou=people,dc=salad,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: alice.dev
cn: Alice Developer
givenName: Alice
sn: Developer
mail: alice.dev@salad.local
uidNumber: 10001
gidNumber: 10001
homeDirectory: /home/alice.dev
loginShell: /bin/bash
userPassword: {SSHA}placeholder

# Example User: Frank SRE (Team Infra)
dn: uid=frank.sre,ou=team-infra,ou=people,dc=salad,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: frank.sre
cn: Frank SRE
givenName: Frank
sn: SRE
mail: frank.sre@salad.local
uidNumber: 10006
gidNumber: 10006
homeDirectory: /home/frank.sre
loginShell: /bin/bash
userPassword: {SSHA}placeholder
```

Import the LDIF file from the LDAP VM:

```bash
ldapadd -x -D "cn=admin,dc=salad,dc=local" -W -f salad-structure.ldif
```

## 6. Verification & Testing

### 6.1. Test LDAP Query (from Host)

```bash
# Search all entries
ldapsearch -x -H ldap://192.168.123.46 -b "dc=salad,dc=local"

# Search for users
ldapsearch -x -H ldap://192.168.123.46 -b "ou=people,dc=salad,dc=local"

# Search for groups
ldapsearch -x -H ldap://192.168.123.46 -b "ou=groups,dc=salad,dc=local"

# Search for a specific user
ldapsearch -x -H ldap://192.168.123.46 -b "dc=salad,dc=local" "(uid=alice.dev)"

# Search for group members
ldapsearch -x -H ldap://192.168.123.46 -b "dc=salad,dc=local" "(cn=gitlab-users)"
```

### 6.2. Test User Authentication

```bash
# Test bind as a user
ldapwhoami -x -H ldap://192.168.123.46 -D "uid=alice.dev,ou=team-alpha,ou=people,dc=salad,dc=local" -W
```
