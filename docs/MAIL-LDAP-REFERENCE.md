# Mail Server LDAP Integration - Quick Reference

## Overview

The mail server (mail.salad.com) is configured to authenticate users against the LDAP directory, enabling centralized user management.

## Architecture

```
User Email Client (Thunderbird/Outlook)
    ↓
    ├─→ SMTP (Port 25) → Postfix → LDAP lookup → Send/Receive emails
    └─→ IMAP (Port 143) → Dovecot → LDAP auth → Access mailbox
```

## Key Configuration Files

### Postfix (SMTP)
- `/etc/postfix/main.cf` - Main configuration
- `/etc/postfix/ldap-virtual-mailbox-maps.cf` - LDAP mailbox lookup
- `/etc/postfix/ldap-virtual-alias-maps.cf` - LDAP alias lookup
- `/etc/postfix/ldap-virtual-domains.cf` - LDAP domain lookup

### Dovecot (IMAP)
- `/etc/dovecot/dovecot-ldap.conf.ext` - LDAP connection settings
- `/etc/dovecot/conf.d/10-auth.conf` - Authentication configuration
- `/etc/dovecot/conf.d/auth-ldap.conf.ext` - LDAP auth driver
- `/etc/dovecot/conf.d/10-mail.conf` - Mail location settings

## User Email Addresses

All LDAP users have email addresses in the format: `username@salad.com`

| User | Email | Password |
|------|-------|----------|
| alice.dev | alice.dev@salad.com | <user-password> |
| bob.senior | bob.senior@salad.com | <user-password> |
| charlie.dev | charlie.dev@salad.com | <user-password> |
| diana.arch | diana.arch@salad.com | <user-password> |
| eve.manager | eve.manager@salad.com | <user-password> |
| frank.sre | frank.sre@salad.com | <user-password> |
| grace.sec | grace.sec@salad.com | <user-password> |

## Mailbox Locations

- **Virtual mailbox base:** `/var/mail/vhosts/salad.com/`
- **User mailbox:** `/var/mail/vhosts/salad.com/USERNAME/Maildir/`
- **Ownership:** `mail:mail` (UID 8, GID 12)
- **Auto-creation:** Mailboxes are automatically created on first login or when receiving first email

## LDAP Connection

**Note:** All LDAP configurations use the IP address `192.168.123.46` instead of `ldap.salad.com` due to DNS resolution issues with Alpine Linux's OpenLDAP client library.

## Testing Commands

### Test LDAP Connection
```bash
ldapsearch -x -H ldap://ldap.salad.com -b "ou=people,dc=salad,dc=com" "(uid=alice.dev)" mail
```

### Test Postfix LDAP Lookup
```bash
postmap -q alice.dev@salad.com ldap:/etc/postfix/ldap-virtual-mailbox-maps.cf
```

### Test Dovecot Authentication
```bash
doveadm auth test alice.dev <user-password>
```

### Test IMAP Login
```bash
telnet mail.salad.com 143
# Then type:
a1 LOGIN alice.dev <user-password>
a2 LIST "" "*"
a3 LOGOUT
```

### Test SMTP
```bash
telnet mail.salad.com 25
# Then type:
EHLO salad.com
MAIL FROM:<alice.dev@salad.com>
RCPT TO:<bob.senior@salad.com>
DATA
Subject: Test
Test message
.
QUIT
```

## Email Client Configuration

### For Thunderbird/Outlook

**Incoming Mail (IMAP):**
- Server: `mail.salad.com`
- Port: `143` (or `993` for SSL if configured)
- Username: `alice.dev` (just the username, not full email)
- Password: `<user-password>`
- Connection security: None (or SSL/TLS if configured)

**Outgoing Mail (SMTP):**
- Server: `mail.salad.com`
- Port: `25` (or `587` for submission if configured)
- Username: `alice.dev`
- Password: `<user-password>`
- Connection security: None (or STARTTLS if configured)

## Troubleshooting

### Check Postfix Logs
```bash
tail -f /var/log/mail.log
# or
journalctl -u postfix -f
```

### Check Dovecot Logs
```bash
tail -f /var/log/dovecot.log
# or
journalctl -u dovecot -f
```

### Verify Services Running
```bash
netstat -tulnp | grep -E '(25|143)'
```

### Test LDAP Connectivity from Mail VM
```bash
ldapsearch -x -H ldap://ldap.salad.com -b "dc=salad,dc=com"
```

### Check Mailbox Permissions
```bash
ls -la /var/mail/vhosts/salad.com/
# Should show directories owned by mail:mail
```

### Restart Services
```bash
service postfix restart
service dovecot restart
```

## Common Issues

### Issue: "Authentication failed"
**Cause:** LDAP password incorrect or LDAP server unreachable
**Solution:** 
- Verify LDAP admin password in config files
- Test LDAP connection: `ping ldap.salad.com`
- Check LDAP service: `ssh ldap.salad.com "systemctl status slapd"`

### Issue: "User not found"
**Cause:** User doesn't exist in LDAP or wrong search base
**Solution:**
- Verify user exists: `ldapsearch -x -H ldap://ldap.salad.com -b "ou=people,dc=salad,dc=com" "(uid=alice.dev)"`
- Check search_base in LDAP config files

### Issue: "Permission denied" on mailbox
**Cause:** Mailbox directory has wrong ownership/permissions
**Solution:**
```bash
chown -R mail:mail /var/mail/vhosts/salad.com/USERNAME
chmod -R 700 /var/mail/vhosts/salad.com/USERNAME
```

### Issue: "Relay access denied"
**Cause:** Postfix not accepting mail from the sender
**Solution:** Check `mynetworks` in `/etc/postfix/main.cf` includes the sender's network

## Security Notes

**Current Setup (Lab Environment):**
- ✅ LDAP authentication enabled
- ⚠️ No SSL/TLS encryption (plaintext)
- ⚠️ LDAP admin password in config files
- ⚠️ No SPF/DKIM/DMARC

**For Production:**
- Enable SSL/TLS for SMTP (Port 587 with STARTTLS)
- Enable SSL/TLS for IMAP (Port 993 with IMAPS)
- Use LDAPS (Port 636) instead of plain LDAP
- Implement SPF, DKIM, and DMARC
- Use a dedicated LDAP bind user (not admin)
- Enable rate limiting and spam filtering
