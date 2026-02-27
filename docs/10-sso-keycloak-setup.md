# 10 - Provisioning SSO Server (Keycloak)

We will provision the **SSO** VM (`sso.salad.local`). This is a **Debian 12** instance running **Keycloak**, which provides Identity and Access Management (IAM) for our infrastructure (GitLab, Portainer, Grafana, etc.).

## 1. VM Directory Structure

Create the directory for the VM:

```bash
mkdir -p infrastructure/vms/sso
```

## 2. Create Virtual Disk

We will create a single disk for the OS and Keycloak installation (since data resides in the DB).

```bash
qemu-img create -f qcow2 infrastructure/vms/sso/sso-os.qcow2 10G
```

## 3. Install Debian (ISO)

```bash
sudo virt-install \
  --name sso \
  --ram 2048 \
  --vcpus 2 \
  --network network=salad-net,mac=52:54:00:00:00:41 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/sso/sso-os.qcow2,size=10 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/debian-12.12.0-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics spice \
  --noautoconsole
```

### 3.1. Connect to Console

```bash
virt-viewer --connect qemu:///system --wait sso
```

### Installation Pointers
1.  **Hostname:** `sso.salad.local`
2.  **Domain:** `salad.local`
3.  **Partitioning:** Standard / (Root).
4.  **Software Selection:**
    *   [X] SSH server
    *   [X] Standard system utilities
    *   [ ] (Uncheck everything else)

## 4. Post-Install Configuration

Login as `root`.

### 1. System Prep
```bash
apt-get update && apt-get install -y curl openssh-server net-tools dnsutils vim git openjdk-17-jre-headless
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

**Fix /etc/hosts Conflict**:
Edit `/etc/hosts`:
```bash
vi /etc/hosts
```
Remove/comment: `127.0.1.1 sso.salad.local sso`

### 2. Configure SSH Access
```bash
mkdir -p /root/.ssh && \
chmod 700 /root/.ssh && \
echo "<your-ssh-public-key>" > /root/.ssh/authorized_keys && \
chmod 600 /root/.ssh/authorized_keys
```

## 5. Install Keycloak

We will install Keycloak from the official release.

### 1. Download and Extract
```bash
cd /opt
wget https://github.com/keycloak/keycloak/releases/download/23.0.7/keycloak-23.0.7.tar.gz
tar -xvzf keycloak-23.0.7.tar.gz
mv keycloak-23.0.7 keycloak
rm keycloak-23.0.7.tar.gz
```

### 2. Create Keycloak User
```bash
groupadd keycloak
useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
chown -R keycloak:keycloak /opt/keycloak
chmod o+x /opt/keycloak/bin/
```

### 3. Configure Database

We assume the `keycloak` database and user were created in the **Database** VM step.

Edit configuration file `conf/keycloak.conf`:
```bash
vi /opt/keycloak/conf/keycloak.conf
```

Add/Uncomment the following:

```properties
# Database
db=postgres
db-username=keycloak
db-password=keycloak_password
db-url=jdbc:postgresql://database.salad.local:5432/keycloak

# Hostname
hostname=keycloak.salad.local
hostname-strict=false
# We use a proxy (Nginx) which terminates SSL
proxy=edge

# HTTP
http-enabled=true
http-port=8080
http-host=0.0.0.0
```

### 4. Build Keycloak
Run the build to optimize for Postgres:
```bash
cd /opt/keycloak/bin
./kc.sh build
```

### 5. Create Systemd Service

Create `/etc/systemd/system/keycloak.service`:

```ini
[Unit]
Description=Keycloak
After=network.target

[Service]
Type=simple
User=keycloak
Group=keycloak
WorkingDirectory=/opt/keycloak
ExecStart=/opt/keycloak/bin/kc.sh start --optimized
Restart=always

[Install]
WantedBy=multi-user.target
```

Reload and Start:
```bash
systemctl daemon-reload
systemctl enable --now keycloak
```

### 6. Create Admin User (Environment Variable Method)

Since we are running as a service, we can't easily interact with the console. We can set temporary environment variables in the service or run a one-off command.

**One-off command method (Stop service first):**
```bash
systemctl stop keycloak
export KEYCLOAK_ADMIN=admin
export KEYCLOAK_ADMIN_PASSWORD=admin
/opt/keycloak/bin/kc.sh start
# Check logs to see if user created, then Ctrl+C
```

**Service method (Better):**
Edit the service file temporarily:
```bash
vi /etc/systemd/system/keycloak.service
```
Add to `[Service]` section:
```ini
Environment=KEYCLOAK_ADMIN=admin
Environment=KEYCLOAK_ADMIN_PASSWORD=admin
```
Reload and restart:
```bash
systemctl daemon-reload
systemctl restart keycloak
```
**Verify connection**, then **REMOVE** those lines and restart again to secure it.

## 6. Verification

From your host machine:

1.  Open `https://keycloak.salad.local/`.
2.  Click "Administration Console".
3.  Login with `admin` / `admin`.

## 7. Install Node Exporter

```bash
apt-get install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
```

## 8. Configure LDAP Integration

We will connect Keycloak to our OpenLDAP server (`ldap.salad.local`) to verify users against the central directory and synchronize groups/roles.

### 8.1. Prerequisite (LDAP Structure)

Ensure your LDAP directory has the complete structure as defined in `docs/07-ldap-setup.md`:
- `ou=people,dc=salad,dc=local` with team sub-OUs
- `ou=groups,dc=salad,dc=local` with service-access, ssh-access, and gitlab-access sub-OUs
- `ou=roles,dc=salad,dc=local` with role groups
- Sample users created and assigned to appropriate groups

### 8.2. Add LDAP User Federation

1.  Log in to the **Keycloak Admin Console** (`https://keycloak.salad.local/admin`).
2.  Select the **master** realm (or create a new realm called **salad** - recommended).
3.  Navigate to **User Federation**.
4.  Click **Add provider** and select **ldap**.
5.  Configure the settings:

    | Setting | Value |
    | :--- | :--- |
    | **Console Display Name** | `ldap-salad` |
    | **Enabled** | `ON` |
    | **Edit Mode** | `READ_ONLY` (LDAP is source of truth) |
    | **Sync Registrations** | `OFF` |
    | **Vendor** | `Other` (OpenLDAP) |
    | **Username LDAP Attribute** | `uid` |
    | **RDN LDAP Attribute** | `uid` |
    | **UUID LDAP Attribute** | `entryUUID` |
    | **User Object Classes** | `inetOrgPerson, organizationalPerson` |
    | **Connection URL** | `ldap://ldap.salad.local:389` |
    | **Users DN** | `ou=people,dc=salad,dc=local` |
    | **Custom User LDAP Filter** | (leave empty or use `(objectClass=inetOrgPerson)`) |
    | **Search Scope** | `Subtree` (to include team sub-OUs) |
    | **Bind Type** | `simple` |
    | **Bind DN** | `cn=admin,dc=salad,dc=local` |
    | **Bind Credential** | *(Your LDAP Admin Password)* |

6.  Click **Test connection** (Ensure it succeeds).
7.  Click **Test authentication** (using the Admin DN and Password).
8.  Click **Save**.

### 8.3. Synchronize Users

1.  After saving, scroll down to the **Synchronization Settings** section.
2.  Set **Periodic Full Sync** to `ON` and set interval (e.g., `86400` seconds = 24 hours).
3.  Set **Periodic Changed Users Sync** to `ON` and set interval (e.g., `3600` seconds = 1 hour).
4.  Click **Save**.
5.  Click **"Synchronize all users"** button to perform initial sync.
6.  Go to **Users** menu → Click **"View all users"** to verify users are imported.

### 8.4. Configure LDAP Group Mapping

We need to map LDAP groups to Keycloak groups/roles.

#### 8.4.1. Map Service Access Groups

1.  In the **ldap-salad** provider settings, click the **Mappers** tab.
2.  Click **Create** to add a new mapper.
3.  Configure for service access groups:

    | Setting | Value |
    | :--- | :--- |
    | **Name** | `service-access-mapper` |
    | **Mapper Type** | `group-ldap-mapper` |
    | **LDAP Groups DN** | `ou=service-access,ou=groups,dc=salad,dc=local` |
    | **Group Name LDAP Attribute** | `cn` |
    | **Group Object Classes** | `groupOfNames` |
    | **Membership LDAP Attribute** | `member` |
    | **Membership Attribute Type** | `DN` |
    | **Membership User LDAP Attribute** | `uid` |
    | **Mode** | `READ_ONLY` |
    | **User Groups Retrieve Strategy** | `LOAD_GROUPS_BY_MEMBER_ATTRIBUTE` |
    | **Member-Of LDAP Attribute** | `memberOf` |
    | **Mapped Group Attributes** | (leave empty) |
    | **Drop non-existing groups during sync** | `OFF` |

4.  Click **Save**.
5.  Repeat for other group OUs:
    - **ssh-access-mapper**: `ou=ssh-access,ou=groups,dc=salad,dc=local`
    - **gitlab-access-mapper**: `ou=gitlab-access,ou=groups,dc=salad,dc=local`
    - **roles-mapper**: `ou=roles,dc=salad,dc=local`

6.  After creating all mappers, click **"Sync LDAP Groups to Keycloak"** for each mapper.

### 8.5. Verification

1.  Go to **Groups** in the left menu.
2.  You should see all LDAP groups imported (gitlab-users, prometheus-users, developers, etc.).
3.  Click on a group (e.g., `gitlab-users`) → **Members** tab to see assigned users.
4.  Go to **Users** → Select a user (e.g., `alice.dev`) → **Groups** tab to see their group memberships.

## 9. Create Keycloak Realm for Salad.Inc

Instead of using the **master** realm (which is for admin purposes), create a dedicated realm for your organization.

### 9.1. Create Realm

1.  In the Keycloak Admin Console, hover over **Master** (top left dropdown).
2.  Click **Create Realm**.
3.  **Realm name**: `salad`
4.  **Enabled**: `ON`
5.  Click **Create**.

### 9.2. Configure Realm Settings

1.  Go to **Realm Settings** → **General** tab:
    - **Display name**: `Salad Inc`
    - **HTML Display name**: `<b>Salad</b> Inc`
    - **Frontend URL**: `https://keycloak.salad.local/`

2.  Go to **Login** tab:
    - **User registration**: `OFF` (users come from LDAP)
    - **Forgot password**: `ON`
    - **Remember me**: `ON`
    - **Email as username**: `OFF`
    - **Login with email**: `ON`

3.  Go to **Email** tab (configure for password reset):
    - **From**: `noreply@salad.local`
    - **Host**: `mail.salad.local`
    - **Port**: `25`
    - **Enable SSL**: `OFF`
    - **Enable StartTLS**: `OFF`

4.  Click **Save**.

### 9.3. Re-configure LDAP Federation for Salad Realm

Switch to the **salad** realm and repeat steps from section 8.2-8.4 to add LDAP federation to this realm.

## 10. Create Keycloak Roles

Roles in Keycloak define what users can do. We'll create roles that match our LDAP groups.

### 10.1. Create Realm Roles

1.  In the **salad** realm, go to **Realm roles**.
2.  Click **Create role** and add the following roles:

    | Role Name | Description |
    |-----------|-------------|
    | `developer` | Developer role |
    | `senior-developer` | Senior Developer role |
    | `architect` | Architect role |
    | `manager` | Manager role |
    | `sre-engineer` | SRE Engineer role |
    | `security-engineer` | Security Engineer role |
    | `gitlab-user` | Can access GitLab |
    | `prometheus-user` | Can access Prometheus |
    | `grafana-user` | Can access Grafana |
    | `grafana-admin` | Grafana administrator |
    | `portainer-user` | Can access Portainer |
    | `portainer-admin` | Portainer administrator |
    | `artifactory-user` | Can access Artifactory |
    | `artifactory-admin` | Artifactory administrator |
    | `lam-admin` | LAM administrator |
    | `traefik-user` | Can access Traefik dashboard |
    | `pgadmin-user` | Can access pgAdmin |
    | `ssh-developer` | Non-root SSH access |
    | `ssh-admin` | Root SSH access |

### 10.2. Map LDAP Groups to Keycloak Roles

We need to automatically assign roles based on LDAP group membership.

1.  Go to **Groups** in the left menu.
2.  Click on a group (e.g., `developers`).
3.  Go to **Role mapping** tab.
4.  Click **Assign role**.
5.  Select the corresponding role (e.g., `developer`).
6.  Click **Assign**.

Repeat for all groups according to this mapping:

| LDAP Group | Keycloak Roles |
|------------|----------------|
| `cn=developers` | `developer`, `gitlab-user`, `grafana-user`, `artifactory-user`, `ssh-developer` |
| `cn=senior-developers` | `senior-developer`, `gitlab-user`, `prometheus-user`, `grafana-user`, `artifactory-user`, `ssh-developer` |
| `cn=architects` | `architect`, `gitlab-user`, `prometheus-user`, `grafana-user`, `grafana-admin`, `portainer-user`, `artifactory-admin`, `traefik-user`, `ssh-developer` |
| `cn=managers` | `manager`, `gitlab-user`, `prometheus-user`, `grafana-user` |
| `cn=sre-engineers` | `sre-engineer`, `gitlab-user`, `prometheus-user`, `grafana-admin`, `portainer-admin`, `artifactory-admin`, `traefik-user`, `pgadmin-user`, `ssh-developer`, `ssh-admin` |
| `cn=security-engineers` | `security-engineer`, `gitlab-user`, `prometheus-user`, `grafana-admin`, `portainer-user`, `artifactory-user`, `lam-admin`, `traefik-user`, `pgadmin-user`, `ssh-developer`, `ssh-admin` |
| `cn=gitlab-users` | `gitlab-user` |
| `cn=prometheus-users` | `prometheus-user` |
| `cn=grafana-users` | `grafana-user` |
| `cn=portainer-users` | `portainer-user` |
| `cn=artifactory-users` | `artifactory-user` |
| `cn=lam-admins` | `lam-admin` |
| `cn=traefik-users` | `traefik-user` |
| `cn=pgadmin-users` | `pgadmin-user` |
| `cn=ssh-developers` | `ssh-developer` |
| `cn=ssh-admins` | `ssh-admin` |

## 11. Configure Service Clients

Each service that integrates with Keycloak needs a **client** configuration.

### 11.1. GitLab Client

1.  Go to **Clients** → Click **Create client**.
2.  **Client type**: `OpenID Connect`
3.  **Client ID**: `gitlab`
4.  Click **Next**.
5.  **Client authentication**: `ON`
6.  **Authorization**: `OFF`
7.  **Authentication flow**: Enable `Standard flow` and `Direct access grants`
8.  Click **Next**.
9.  **Root URL**: `https://gitlab.salad.local`
10. **Valid redirect URIs**: `https://gitlab.salad.local/users/auth/openid_connect/callback`
11. **Web origins**: `https://gitlab.salad.local`
12. Click **Save**.

#### Configure Client Scopes and Mappers

1.  Go to the **gitlab** client → **Client scopes** tab.
2.  Click on **gitlab-dedicated** scope.
3.  Click **Add mapper** → **By configuration** → **Group Membership**.
4.  Configure:
    - **Name**: `groups`
    - **Token Claim Name**: `groups`
    - **Full group path**: `OFF`
    - **Add to ID token**: `ON`
    - **Add to access token**: `ON`
    - **Add to userinfo**: `ON`
5.  Click **Save**.

6.  Add another mapper for roles:
    - Click **Add mapper** → **By configuration** → **User Realm Role**.
    - **Name**: `roles`
    - **Token Claim Name**: `roles`
    - **Add to ID token**: `ON`
    - **Add to access token**: `ON`
    - **Add to userinfo**: `ON`
7.  Click **Save**.

#### Get Client Secret

1.  Go to **Credentials** tab.
2.  Copy the **Client secret** (you'll need this for GitLab configuration).

### 11.2. Grafana Client

1.  Create client with **Client ID**: `grafana`
2.  **Client authentication**: `ON`
3.  **Valid redirect URIs**: `https://grafana.salad.local/login/generic_oauth`
4.  **Web origins**: `https://grafana.salad.local`
5.  Add mappers for `groups` and `roles` (same as GitLab).

### 11.3. Portainer Client

1.  Create client with **Client ID**: `portainer`
2.  **Client authentication**: `ON`
3.  **Valid redirect URIs**: `https://portainer.salad.local/*`
4.  **Web origins**: `https://portainer.salad.local`
5.  Add mappers for `groups` and `roles`.

### 11.4. Artifactory Client

1.  Create client with **Client ID**: `artifactory`
2.  **Client authentication**: `ON`
3.  **Valid redirect URIs**: `https://artifactory.salad.local/ui/login`
4.  **Web origins**: `https://artifactory.salad.local`
5.  Add mappers for `groups` and `roles`.

### 11.5. Prometheus Client (Optional)

Prometheus doesn't natively support OAuth, but you can use **OAuth2 Proxy** as a middleware.

1.  Create client with **Client ID**: `prometheus`
2.  **Client authentication**: `ON`
3.  **Valid redirect URIs**: `https://prometheus.salad.local/oauth2/callback`
4.  **Web origins**: `https://prometheus.salad.local`

### 11.6. Traefik Dashboard Client (Optional)

Similar to Prometheus, use OAuth2 Proxy.

1.  Create client with **Client ID**: `traefik`
2.  **Client authentication**: `ON`
3.  **Valid redirect URIs**: `https://traefik.salad.local/oauth2/callback`
4.  **Web origins**: `https://traefik.salad.local`

### 11.7. LAM Client (Optional)

LAM can use LDAP directly, but if you want SSO:

1.  Create client with **Client ID**: `lam`
2.  **Client authentication**: `ON`
3.  **Valid redirect URIs**: `https://lam.salad.local/*`
4.  **Web origins**: `https://lam.salad.local`

## 12. GitLab Integration with Keycloak

### 12.1. Configure GitLab

SSH into the GitLab VM and edit `/etc/gitlab/gitlab.rb`:

```ruby
gitlab_rails['omniauth_enabled'] = true
gitlab_rails['omniauth_allow_single_sign_on'] = ['openid_connect']
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_auto_link_ldap_user'] = false
gitlab_rails['omniauth_auto_link_saml_user'] = false
gitlab_rails['omniauth_auto_link_user'] = ['openid_connect']

gitlab_rails['omniauth_providers'] = [
  {
    name: 'openid_connect',
    label: 'Salad SSO',
    args: {
      name: 'openid_connect',
      scope: ['openid', 'profile', 'email'],
      response_type: 'code',
      issuer: 'https://keycloak.salad.local/realms/salad',
      client_auth_method: 'query',
      discovery: true,
      uid_field: 'preferred_username',
      client_options: {
        identifier: 'gitlab',
        secret: 'YOUR_CLIENT_SECRET_FROM_KEYCLOAK',
        redirect_uri: 'https://gitlab.salad.local/users/auth/openid_connect/callback'
      }
    }
  }
]
```

Reconfigure GitLab:

```bash
gitlab-ctl reconfigure
```

### 12.2. Configure GitLab Groups Based on LDAP

GitLab supports **group sync** with LDAP. You can map LDAP groups to GitLab groups.

1.  In GitLab, create groups: `team-alpha`, `team-beta`, `team-infra`.
2.  For each group, go to **Settings** → **LDAP Synchronization**.
3.  Add LDAP group DN (e.g., `cn=gitlab-team-alpha,ou=gitlab-access,ou=groups,dc=salad,dc=local`).
4.  Set permissions (Developer, Maintainer, Owner).

**Note**: GitLab CE has limited LDAP group sync. GitLab EE (Enterprise Edition) has full support. For CE, you may need to manually assign users to groups or use API automation.

### 12.3. GitLab Project Visibility

To ensure Team A doesn't see Team B's projects:

1.  Create projects under the respective group (e.g., `team-alpha/project1`).
2.  Set project visibility to **Private**.
3.  Only group members will have access.
4.  Use **Protected Branches** to restrict push access.

## 13. SSH Access Control with LDAP

To control SSH access based on LDAP groups, we'll use **SSSD** (System Security Services Daemon) and **PAM** (Pluggable Authentication Modules).

### 13.1. Install SSSD on VMs

On each VM where you want LDAP-based SSH authentication:

```bash
apt-get install -y sssd sssd-ldap ldap-utils
```

### 13.2. Configure SSSD

Create `/etc/sssd/sssd.conf`:

```ini
[sssd]
services = nss, pam, ssh
config_file_version = 2
domains = salad.local

[domain/salad.local]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://ldap.salad.local
ldap_search_base = dc=salad,dc=local
ldap_default_bind_dn = cn=admin,dc=salad,dc=local
ldap_default_authtok = YOUR_LDAP_ADMIN_PASSWORD
ldap_tls_reqcert = never

# User and group settings
ldap_user_search_base = ou=people,dc=salad,dc=local
ldap_group_search_base = ou=groups,dc=salad,dc=local
ldap_user_object_class = inetOrgPerson
ldap_group_object_class = groupOfNames
ldap_user_name = uid
ldap_group_member = member

# Enable enumeration (for getent)
enumerate = true

# Access control
access_provider = ldap
ldap_access_filter = (objectClass=inetOrgPerson)
```

Set permissions:

```bash
chmod 600 /etc/sssd/sssd.conf
chown root:root /etc/sssd/sssd.conf
```

Enable and start SSSD:

```bash
systemctl enable --now sssd
```

### 13.3. Configure PAM for SSH

Edit `/etc/pam.d/sshd` and ensure it includes:

```
account required pam_sss.so
auth required pam_sss.so
password required pam_sss.so
session required pam_sss.so
```

### 13.4. Configure SSH Access by Group

Edit `/etc/security/access.conf`:

```
# Allow ssh-admins group (root access)
+:ssh-admins:ALL

# Allow ssh-developers group (non-root access)
+:ssh-developers:ALL

# Deny all others
-:ALL:ALL
```

Edit `/etc/ssh/sshd_config`:

```
AllowGroups ssh-admins ssh-developers
```

Restart SSH:

```bash
systemctl restart sshd
```

### 13.5. Configure Sudo Access for Root

Edit `/etc/sudoers.d/ldap-groups`:

```
# SRE engineers and security engineers have full sudo
%ssh-admins ALL=(ALL:ALL) ALL

# Developers have limited sudo (example: restart services)
%ssh-developers ALL=(ALL) NOPASSWD: /bin/systemctl restart *
```

### 13.6. Verification

From your host machine:

```bash
# Test SSH as a developer (should work if in ssh-developers group)
ssh alice.dev@containerization.salad.local

# Test SSH as SRE (should work with sudo)
ssh frank.sre@containerization.salad.local
sudo -i  # Should work without password
```

## 14. Additional Service Integrations

### 14.1. Grafana OAuth Configuration

Edit Grafana configuration (`/etc/grafana/grafana.ini`):

```ini
[auth.generic_oauth]
enabled = true
name = Salad SSO
allow_sign_up = true
client_id = grafana
client_secret = YOUR_CLIENT_SECRET_FROM_KEYCLOAK
scopes = openid profile email
auth_url = https://keycloak.salad.local/realms/salad/protocol/openid-connect/auth
token_url = https://keycloak.salad.local/realms/salad/protocol/openid-connect/token
api_url = https://keycloak.salad.local/realms/salad/protocol/openid-connect/userinfo
role_attribute_path = contains(roles[*], 'grafana-admin') && 'Admin' || contains(roles[*], 'grafana-user') && 'Viewer'
```

Restart Grafana:

```bash
systemctl restart grafana-server
```

### 14.2. Portainer OAuth Configuration

In Portainer UI:

1.  Go to **Settings** → **Authentication**.
2.  Select **OAuth**.
3.  **Provider**: Custom
4.  **Client ID**: `portainer`
5.  **Client Secret**: YOUR_CLIENT_SECRET_FROM_KEYCLOAK
6.  **Authorization URL**: `https://keycloak.salad.local/realms/salad/protocol/openid-connect/auth`
7.  **Access Token URL**: `https://keycloak.salad.local/realms/salad/protocol/openid-connect/token`
8.  **Resource URL**: `https://keycloak.salad.local/realms/salad/protocol/openid-connect/userinfo`
9.  **Redirect URL**: `https://portainer.salad.local`
10. **User Identifier**: `preferred_username`
11. **Scopes**: `openid profile email`

### 14.3. Artifactory OIDC Configuration

In Artifactory UI:

1.  Go to **Administration** → **Security** → **Settings**.
2.  Enable **OAuth SSO**.
3.  **Provider Type**: Custom
4.  **Client ID**: `artifactory`
5.  **Client Secret**: YOUR_CLIENT_SECRET_FROM_KEYCLOAK
6.  **Authorization URL**: `https://keycloak.salad.local/realms/salad/protocol/openid-connect/auth`
7.  **Token URL**: `https://keycloak.salad.local/realms/salad/protocol/openid-connect/token`
8.  **User Info URL**: `https://keycloak.salad.local/realms/salad/protocol/openid-connect/userinfo`

### 14.4. OAuth2 Proxy for Prometheus/Traefik

For services without native OAuth support, deploy **OAuth2 Proxy**:

```bash
docker run -d \
  --name oauth2-proxy-prometheus \
  -p 4180:4180 \
  quay.io/oauth2-proxy/oauth2-proxy:latest \
  --provider=oidc \
  --client-id=prometheus \
  --client-secret=YOUR_CLIENT_SECRET \
  --redirect-url=https://prometheus.salad.local/oauth2/callback \
  --oidc-issuer-url=https://keycloak.salad.local/realms/salad \
  --cookie-secret=RANDOM_32_CHAR_STRING \
  --email-domain=salad.local \
  --upstream=http://localhost:9090 \
  --http-address=0.0.0.0:4180
```

Configure Nginx to proxy through OAuth2 Proxy.
