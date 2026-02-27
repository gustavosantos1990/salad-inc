# 05 - Provisioning Mail Server (Postfix)

We will provision the **Mail** VM (`mail.salad.local`). This is an **Alpine Linux** instance running **Postfix** (SMTP) and **Dovecot** (IMAP).

## 1. Directory & Disk

Create the directory and disk image:

```bash
mkdir -p infrastructure/vms/mail
```

Create Disk (2G is enough for Alpine + mail):
```bash
qemu-img create -f qcow2 infrastructure/vms/mail/mail.qcow2 2G
```

## 2. Install Alpine (ISO)

```bash
sudo virt-install \
  --name mail \
  --ram 256 \
  --vcpus 1 \
  --network network=salad-net,mac=52:54:00:00:00:45 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/mail/mail.qcow2,size=2 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/alpine-virt-3.19.1-x86_64.iso \
  --os-variant alpinelinux3.18 \
  --graphics spice \
  --noautoconsole
```

## 3. Connect to Console

```bash
virt-viewer --connect qemu:///system --wait mail
```

## 4. Install Alpine

Follow the `setup-alpine` prompts:

1. **Hostname:** `mail`
2. **Interface:** `eth0`
3. **IP Address:** `dhcp`
4. **Manual network configuration?** `no`
5. **Timezone:** `America` then `Sao_Paulo`
6. **HTTP/FTP Proxy:** `none`
7. **APK Mirror:** `36`
8. **Create extra user:** `salad-maker`
9. **Enter ssh key:** `<your-ssh-public-key>`
10. **SSH server:** `openssh`
11. **Disk to install:** `vda`
12. **How would you like to use it?** `sys`
13. `reboot`

## 5. Manual Configuration (Post-Install)

Login as `root`.

### 1. Configure SSH Access

Create SSH directory:
```bash
mkdir -p /root/.ssh
```

Set directory permissions:
```bash
chmod 700 /root/.ssh
```

Add public key:
```bash
echo "<your-ssh-public-key>" > /root/.ssh/authorized_keys
```

Set key permissions:
```bash
chmod 600 /root/.ssh/authorized_keys
```

Ensure SSHd is running:
```bash
rc-update add sshd default
```
```bash
service sshd start
```

### 2. Install Packages

Enable Community Repo:
```bash
vi /etc/apk/repositories
```
*Uncomment the community line.*

Update and install:
```bash
apk update
```
```bash
apk add postfix dovecot dovecot-lmtpd prometheus-node-exporter
```

**Note:** `dovecot-lmtpd` is required for local mail delivery via LMTP (Local Mail Transfer Protocol).

### 3. Disable Swap

Turn off swap:
```bash
swapoff -a
```
```bash
sed -i '/swap/d' /etc/fstab
```
```bash
free -h
```

### 4. Configure Postfix (SMTP)

Edit main config:
```bash
vi /etc/postfix/main.cf
```

Add/modify these settings:
```text
myhostname = mail.salad.local
mydomain = salad.local
myorigin = $mydomain
inet_interfaces = all
inet_protocols = ipv4
mydestination = localhost
mynetworks = 127.0.0.0/8 192.168.123.0/24
home_mailbox = Maildir/
```

**Important:** `mydestination = localhost` (NOT including `salad.local`) is crucial. The `salad.local` domain will be handled as a virtual domain via LDAP. Including it in both `mydestination` and `virtual_mailbox_domains` causes conflicts.

Enable and start:
```bash
rc-update add postfix default
```
```bash
service postfix start
```

### 5. Configure Dovecot (IMAP)

Edit mail location:
```bash
vi /etc/dovecot/conf.d/10-mail.conf
```

Change to:
```text
mail_location = maildir:~/Maildir
```

Enable and start:
```bash
rc-update add dovecot default
```
```bash
service dovecot start
```

### 6. Enable and Start Services

Enable Node Exporter:
```bash
rc-update add node-exporter default
```

Start Node Exporter:
```bash
rc-service node-exporter start
```

Verify services are listening:
```bash
netstat -tulnp | grep -E '(25|143|9100)'
```

### 7. Configure LDAP Integration

To enable centralized user management, we'll configure both Postfix and Dovecot to authenticate users against the LDAP directory.

#### 7.1. Install LDAP Client Packages

```bash
apk add openldap-clients postfix-ldap dovecot-ldap
```

**Note:** In the following configurations, we use the LDAP server's IP address (`192.168.123.46`) instead of the hostname (`ldap.salad.local`). This is because Alpine Linux's OpenLDAP client library may have issues resolving hostnames in some configurations, even when system DNS works correctly. Using the IP address ensures reliable LDAP connectivity.

#### 7.2. Configure Postfix for LDAP Virtual Mailboxes

**Create LDAP virtual mailbox map:**

```bash
cat > /etc/postfix/ldap-virtual-mailbox-maps.cf << 'EOF'
server_host = 192.168.123.46
server_port = 389
version = 3
bind = yes
bind_dn = cn=admin,dc=salad,dc=local
bind_pw = YOUR_LDAP_ADMIN_PASSWORD
search_base = ou=people,dc=salad,dc=local
scope = sub
query_filter = (&(objectClass=inetOrgPerson)(mail=%s))
result_attribute = mail
result_format = %s/Maildir/
EOF
```

**Create LDAP virtual alias map:**

```bash
cat > /etc/postfix/ldap-virtual-alias-maps.cf << 'EOF'
server_host = 192.168.123.46
server_port = 389
version = 3
bind = yes
bind_dn = cn=admin,dc=salad,dc=local
bind_pw = YOUR_LDAP_ADMIN_PASSWORD
search_base = ou=people,dc=salad,dc=local
scope = sub
query_filter = (&(objectClass=inetOrgPerson)(mail=%s))
result_attribute = mail
EOF
```

**Create LDAP virtual domains map:**

```bash
cat > /etc/postfix/ldap-virtual-domains.cf << 'EOF'
server_host = 192.168.123.46
server_port = 389
version = 3
bind = yes
bind_dn = cn=admin,dc=salad,dc=local
bind_pw = YOUR_LDAP_ADMIN_PASSWORD
search_base = ou=people,dc=salad,dc=local
scope = sub
query_filter = (&(objectClass=inetOrgPerson)(mail=*@%s))
result_attribute = mail
EOF
```

**Update Postfix main configuration:**

Edit `/etc/postfix/main.cf` and add/modify these settings:

```bash
vi /etc/postfix/main.cf
```

Add these lines:

```text
# Virtual mailbox settings for LDAP
virtual_mailbox_domains = ldap:/etc/postfix/ldap-virtual-domains.cf
virtual_mailbox_maps = ldap:/etc/postfix/ldap-virtual-mailbox-maps.cf
virtual_alias_maps = ldap:/etc/postfix/ldap-virtual-alias-maps.cf
virtual_mailbox_base = /var/mail/vhosts
virtual_minimum_uid = 100
virtual_uid_maps = static:8
virtual_gid_maps = static:12
virtual_transport = lmtp:unix:private/dovecot-lmtp
```

**Create virtual mailbox directory:**

```bash
mkdir -p /var/mail/vhosts/salad.local
chown -R mail:mail /var/mail/vhosts
chmod -R 770 /var/mail/vhosts
```

**Enable LMTP in Postfix master.cf:**

Edit `/etc/postfix/master.cf`:

```bash
vi /etc/postfix/master.cf
```

Ensure this line is present (uncomment if needed):

```text
dovecot   unix  -       n       n       -       -       pipe
  flags=DRhu user=mail:mail argv=/usr/libexec/dovecot/dovecot-lda -f ${sender} -d ${recipient}
```

**Restart Postfix:**

```bash
service postfix restart
```

#### 7.3. Configure Dovecot for LDAP Authentication

**Create Dovecot LDAP configuration:**

```bash
cat > /etc/dovecot/dovecot-ldap.conf.ext << 'EOF'
# LDAP connection settings
debug_level = 1
hosts = 192.168.123.46:389
dn = cn=admin,dc=salad,dc=local
dnpass = YOUR_LDAP_ADMIN_PASSWORD
ldap_version = 3
base = ou=people,dc=salad,dc=local
scope = subtree

# User authentication - use password comparison (not auth_bind)
# Note: auth_bind with %{ldap:dn} is not supported in Alpine's Dovecot
auth_bind = no

# User lookup
user_filter = (&(objectClass=inetOrgPerson)(uid=%n))
pass_filter = (&(objectClass=inetOrgPerson)(uid=%n))

# User attributes
user_attrs = \
  =home=/var/mail/vhosts/salad.local/%n, \
  =uid=mail, \
  =gid=mail

pass_attrs = \
  uid=user, \
  userPassword=password

# Default password scheme
default_pass_scheme = SSHA

# Iteration settings
iterate_attrs = uid=user
iterate_filter = (objectClass=inetOrgPerson)
EOF
```

**Important Notes:**
- `auth_bind = no`: Dovecot retrieves the password hash from LDAP and compares it locally
- `debug_level = 1`: Enables LDAP debugging in logs
- Alpine's Dovecot doesn't support `%{ldap:dn}` variable expansion, so we use password comparison instead of bind authentication

**Set proper permissions:**

```bash
chmod 600 /etc/dovecot/dovecot-ldap.conf.ext
chown root:root /etc/dovecot/dovecot-ldap.conf.ext
```

**Configure Dovecot to use LDAP authentication:**

Edit `/etc/dovecot/conf.d/10-auth.conf`:

```bash
vi /etc/dovecot/conf.d/10-auth.conf
```

Disable system auth and enable LDAP:

```text
# Allow plaintext authentication (for lab environment)
# For production, keep this as 'yes' and use SSL/TLS
disable_plaintext_auth = no

# Enable username format normalization
auth_username_format = %Ln

# Disable all other auth methods
#!include auth-system.conf.ext
#!include auth-passwdfile.conf.ext

# Enable LDAP auth
!include auth-ldap.conf.ext
```

**Important:** 
- `disable_plaintext_auth = no` allows authentication without SSL/TLS (lab only!)
- `auth_username_format = %Ln` strips domain from email addresses (bob.senior@salad.local → bob.senior)
- Comment out ALL other auth includes to ensure only LDAP is used

**Create LDAP auth configuration:**

```bash
cat > /etc/dovecot/conf.d/auth-ldap.conf.ext << 'EOF'
passdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
}

userdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
}
EOF
```

**Update mail location in Dovecot:**

Edit `/etc/dovecot/conf.d/10-mail.conf`:

```bash
vi /etc/dovecot/conf.d/10-mail.conf
```

Change to:

```text
mail_location = maildir:/var/mail/vhosts/salad.local/%n/Maildir
mail_uid = mail
mail_gid = mail
first_valid_uid = 8
first_valid_gid = 12
```

**Enable LMTP protocol:**

Edit `/etc/dovecot/dovecot.conf`:

```bash
vi /etc/dovecot/dovecot.conf
```

Find the `protocols` line and add `lmtp`:

```text
protocols = imap lmtp
```

**Note:** Without `lmtp` in the protocols list, the LMTP service won't start even if configured in `10-master.conf`.

**Enable LMTP service in Dovecot:**

Edit `/etc/dovecot/conf.d/10-master.conf`:

```bash
vi /etc/dovecot/conf.d/10-master.conf
```

Add/uncomment the LMTP service:

```text
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}
```

**Enable automatic mailbox creation:**

Edit `/etc/dovecot/conf.d/15-mailboxes.conf`:

```bash
vi /etc/dovecot/conf.d/15-mailboxes.conf
```

Add at the top of the file:

```text
namespace inbox {
  inbox = yes
  
  mailbox Drafts {
    auto = create
    special_use = \Drafts
  }
  mailbox Sent {
    auto = create
    special_use = \Sent
  }
  mailbox Trash {
    auto = create
    special_use = \Trash
  }
  mailbox Spam {
    auto = create
    special_use = \Junk
  }
}
```

**Enable auto-create for mail location:**

Edit `/etc/dovecot/conf.d/10-mail.conf` and add:

```bash
vi /etc/dovecot/conf.d/10-mail.conf
```

Add this line after `mail_location`:

```text
mail_location = maildir:/var/mail/vhosts/salad.local/%n/Maildir
mail_uid = mail
mail_gid = mail
first_valid_uid = 8
first_valid_gid = 12

# Auto-create mailbox directory structure
mail_home = /var/mail/vhosts/salad.local/%n
mail_plugin_dir = /usr/lib/dovecot/modules
```

**Create base directory with proper permissions:**

```bash
mkdir -p /var/mail/vhosts/salad.local
chown -R mail:mail /var/mail/vhosts
chmod -R 770 /var/mail/vhosts
```

**Restart Dovecot:**

```bash
service dovecot restart
```

#### 7.4. Test LDAP Connection

**Test LDAP user lookup:**

```bash
ldapsearch -x -H ldap://192.168.123.46 -b "ou=people,dc=salad,dc=local" "(uid=alice.dev)" mail
```

You should see alice.dev's email address.

**Test Postfix LDAP maps:**

```bash
postmap -q alice.dev@salad.local ldap:/etc/postfix/ldap-virtual-mailbox-maps.cf
```

Should return: `alice.dev@salad.local/Maildir/`

**Test Dovecot authentication:**

```bash
doveadm auth test alice.dev <user-password>
```

Should return: `passdb: alice.dev auth succeeded`


#### 7.5. Mailbox Auto-Creation

With the configuration above, mailboxes will be **automatically created** when:
- A user first logs in via IMAP/POP3
- A user receives their first email

The mailbox directory structure (`/var/mail/vhosts/salad.local/USERNAME/Maildir/`) will be created automatically with the correct permissions (owned by `mail:mail`).

**No manual mailbox creation is required!** ✅

## 6. Verification

From host machine:
```bash
telnet 192.168.123.45 25
```
Type `EHLO salad.local` then `quit`.

### Test LDAP Authentication

**Test IMAP login with LDAP user:**

```bash
# Install a mail client for testing (on your host or any VM)
telnet mail.salad.local 143
```

Then type:
```
a1 LOGIN alice.dev <user-password>
a2 LIST "" "*"
a3 LOGOUT
```

You should see successful login and mailbox listing.

**Test sending email via SMTP (authenticated):**

```bash
# From any VM with mail utilities
echo "Test email body" | mail -s "Test Subject" -r alice.dev@salad.local bob.senior@salad.local
```

**Check mail logs:**

On the mail VM:
```bash
tail -f /var/log/mail.log
```

You should see successful authentication and delivery messages.
