# Mail Server Critical Configuration Checklist

This document lists all the critical configurations required for the mail server to work properly with LDAP authentication.

## ✅ Package Installation

```bash
apk add postfix dovecot dovecot-lmtpd prometheus-node-exporter
apk add openldap-clients postfix-ldap dovecot-ldap
```

**Critical:** `dovecot-lmtpd` is required for LMTP mail delivery.

---

## ✅ Postfix Configuration (`/etc/postfix/main.cf`)

### 1. Basic Settings

```text
myhostname = mail.salad.com
mydomain = salad.com
myorigin = $mydomain
inet_interfaces = all
inet_protocols = ipv4
mydestination = localhost
mynetworks = 127.0.0.0/8 192.168.123.0/24
```

**Critical:** `mydestination = localhost` (NOT including `salad.com`)
- `salad.com` must ONLY be in `virtual_mailbox_domains`
- Including it in both causes "unknown user" errors

### 2. Virtual Mailbox Settings

```text
virtual_mailbox_domains = ldap:/etc/postfix/ldap-virtual-domains.cf
virtual_mailbox_maps = ldap:/etc/postfix/ldap-virtual-mailbox-maps.cf
virtual_alias_maps = ldap:/etc/postfix/ldap-virtual-alias-maps.cf
virtual_mailbox_base = /var/mail/vhosts
virtual_minimum_uid = 100
virtual_uid_maps = static:8
virtual_gid_maps = static:12
virtual_transport = lmtp:unix:private/dovecot-lmtp
```

### 3. Alias Database

```text
alias_maps = lmdb:/etc/postfix/aliases
alias_database = lmdb:/etc/postfix/aliases
```

**Note:** Use `lmdb` not `btree` (Alpine doesn't support btree by default)

---

## ✅ Dovecot Configuration

### 1. Enable LMTP Protocol (`/etc/dovecot/dovecot.conf`)

```text
protocols = imap lmtp
```

**Critical:** Without this, LMTP service won't start!

### 2. Authentication (`/etc/dovecot/conf.d/10-auth.conf`)

```text
# Allow plaintext authentication (lab environment only!)
disable_plaintext_auth = no

# Strip domain from username
auth_username_format = %Ln

# Disable all other auth methods
#!include auth-system.conf.ext
#!include auth-passwdfile.conf.ext

# Enable LDAP auth
!include auth-ldap.conf.ext
```

**Critical:**
- `disable_plaintext_auth = no` - Required for non-SSL connections
- `auth_username_format = %Ln` - Strips `@salad.com` from usernames
- Comment out ALL other auth includes

### 3. LDAP Configuration (`/etc/dovecot/dovecot-ldap.conf.ext`)

```text
# LDAP connection settings
debug_level = 1
hosts = 192.168.123.46:389
dn = cn=admin,dc=salad,dc=com
dnpass = YOUR_LDAP_ADMIN_PASSWORD
ldap_version = 3
base = ou=people,dc=salad,dc=com
scope = subtree

# User authentication - use password comparison
auth_bind = no

# User lookup
user_filter = (&(objectClass=inetOrgPerson)(uid=%n))
pass_filter = (&(objectClass=inetOrgPerson)(uid=%n))

# User attributes
user_attrs = \
  =home=/var/mail/vhosts/salad.com/%n, \
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
```

**Critical:**
- Use IP address `192.168.123.46` not `ldap.salad.com` (DNS resolution issue)
- `auth_bind = no` - Alpine's Dovecot doesn't support `%{ldap:dn}` variable
- `userPassword=password` in `pass_attrs` for password comparison

### 4. LDAP Auth Driver (`/etc/dovecot/conf.d/auth-ldap.conf.ext`)

```text
passdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
}

userdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
}
```

### 5. LMTP Service (`/etc/dovecot/conf.d/10-master.conf`)

```text
service lmtp {
  unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}
```

### 6. Mail Location (`/etc/dovecot/conf.d/10-mail.conf`)

```text
mail_location = maildir:/var/mail/vhosts/salad.com/%n/Maildir
mail_uid = mail
mail_gid = mail
first_valid_uid = 8
first_valid_gid = 12
mail_home = /var/mail/vhosts/salad.com/%n
```

### 7. Mailbox Auto-Creation (`/etc/dovecot/conf.d/15-mailboxes.conf`)

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

---

## ✅ Directory Setup

```bash
mkdir -p /var/mail/vhosts/salad.com
chown -R mail:mail /var/mail/vhosts
chmod -R 770 /var/mail/vhosts
```

---

## ✅ Verification Commands

### Check Services Running

```bash
netstat -tulnp | grep -E '(25|143)'
```

Should show:
- Port 25 (Postfix SMTP)
- Port 143 (Dovecot IMAP)

### Check LMTP Socket

```bash
ls -la /var/spool/postfix/private/dovecot-lmtp
```

Should exist with permissions `srw------- postfix postfix`

### Test LDAP Authentication

```bash
doveadm auth test bob.senior <user-password>
```

Should return: `passdb: bob.senior auth succeeded`

### Test LDAP Lookup

```bash
ldapsearch -x -H ldap://192.168.123.46 -b "ou=people,dc=salad,dc=com" "(uid=alice.dev)" mail
```

### Test Postfix LDAP Maps

```bash
postmap -q alice.dev@salad.com ldap:/etc/postfix/ldap-virtual-mailbox-maps.cf
```

Should return: `alice.dev@salad.com/Maildir/`

### Send Test Email

```bash
echo "Test message" | mail -s "Test" -r charlie.dev@salad.com alice.dev@salad.com
```

### Check Logs

```bash
tail -f /var/log/messages | grep -E "(postfix|dovecot)"
```

### Check Mailbox Created

```bash
ls -la /var/mail/vhosts/salad.com/alice.dev/Maildir/new/
```

---

## 🚨 Common Issues and Fixes

### Issue: "unknown user" in Postfix logs

**Cause:** `salad.com` is in both `mydestination` and `virtual_mailbox_domains`

**Fix:** Set `mydestination = localhost` in `/etc/postfix/main.cf`

### Issue: "No such file or directory" for LMTP socket

**Cause:** `dovecot-lmtpd` package not installed or LMTP protocol not enabled

**Fix:**
```bash
apk add dovecot-lmtpd
# Add 'lmtp' to protocols in /etc/dovecot/dovecot.conf
service dovecot restart
```

### Issue: "Failed to expand auth_bind_userdn=%{ldap:dn}"

**Cause:** Alpine's Dovecot doesn't support `%{ldap:dn}` variable

**Fix:** Use `auth_bind = no` with password comparison instead

### Issue: "Plaintext authentication not allowed"

**Cause:** `disable_plaintext_auth = yes` (default)

**Fix:** Set `disable_plaintext_auth = no` in `/etc/dovecot/conf.d/10-auth.conf`

### Issue: "passwd-file(user): unknown user"

**Cause:** Dovecot using passwd-file auth instead of LDAP

**Fix:** Comment out `!include auth-passwdfile.conf.ext` and ensure `!include auth-ldap.conf.ext` is enabled

### Issue: "unsupported dictionary type: btree"

**Cause:** Alpine's Postfix doesn't support btree

**Fix:** Use `lmdb` instead of `btree` for TLS cache databases

---

## 📋 Quick Setup Checklist

- [ ] Install all required packages (including `dovecot-lmtpd`)
- [ ] Set `mydestination = localhost` in Postfix
- [ ] Configure virtual mailbox domains via LDAP
- [ ] Use IP address `192.168.123.46` for LDAP connections
- [ ] Set `auth_bind = no` in Dovecot LDAP config
- [ ] Add `lmtp` to protocols in `dovecot.conf`
- [ ] Set `disable_plaintext_auth = no` in Dovecot auth config
- [ ] Add `auth_username_format = %Ln` in Dovecot auth config
- [ ] Comment out all non-LDAP auth includes
- [ ] Create `/var/mail/vhosts/salad.com` directory
- [ ] Verify LMTP socket exists
- [ ] Test LDAP authentication with `doveadm auth test`
- [ ] Send test email and verify delivery

---

## 🎯 Success Criteria

✅ `doveadm auth test bob.senior <user-password>` succeeds  
✅ LMTP socket exists at `/var/spool/postfix/private/dovecot-lmtp`  
✅ Test email delivers to `/var/mail/vhosts/salad.com/USERNAME/Maildir/new/`  
✅ Thunderbird can connect via IMAP (port 143) and send via SMTP (port 25)  
✅ Mailboxes auto-create on first login or email receipt  
✅ No "unknown user" errors in Postfix logs  
✅ No "passwd-file" references in Dovecot logs (should use LDAP)
