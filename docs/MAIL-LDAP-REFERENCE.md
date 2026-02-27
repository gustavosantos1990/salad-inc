# Mail Server LDAP Integration - Quick Reference

## Overview

The mail server (mail.salad.local) is configured to authenticate users against the LDAP directory, enabling centralized user management.

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

All LDAP users have email addresses in the format: `username@salad.local`

| User | Email | Password |
|------|-------|----------|
| alice.dev | alice.dev@salad.local | <user-password> |
| bob.senior | bob.senior@salad.local | <user-password> |
| charlie.dev | charlie.dev@salad.local | <user-password> |
| diana.arch | diana.arch@salad.local | <user-password> |
| eve.manager | eve.manager@salad.local | <user-password> |
| frank.sre | frank.sre@salad.local | <user-password> |
| grace.sec | grace.sec@salad.local | <user-password> |

## Mailbox Locations

- **Virtual mailbox base:** `/var/mail/vhosts/salad.local/`
- **User mailbox:** `/var/mail/vhosts/salad.local/USERNAME/Maildir/`
- **Ownership:** `mail:mail` (UID 8, GID 12)
- **Auto-creation:** Mailboxes are automatically created on first login or when receiving first email

## LDAP Connection

**Note:** All LDAP configurations use the IP address `192.168.123.46` instead of `ldap.salad.local` due to DNS resolution issues with Alpine Linux's OpenLDAP client library.

## Testing Commands

### Test LDAP Connection
```bash
ldapsearch -x -H ldap://ldap.salad.local -b "ou=people,dc=salad,dc=local" "(uid=alice.dev)" mail
```

### Test Postfix LDAP Lookup
```bash
postmap -q alice.dev@salad.local ldap:/etc/postfix/ldap-virtual-mailbox-maps.cf
```

### Test Dovecot Authentication
```bash
doveadm auth test alice.dev <user-password>
```

### Test IMAP Login
```bash
telnet mail.salad.local 143
# Then type:
a1 LOGIN alice.dev <user-password>
a2 LIST "" "*"
a3 LOGOUT
```

### Test SMTP
```bash
telnet mail.salad.local 25
# Then type:
EHLO salad.local
MAIL FROM:<alice.dev@salad.local>
RCPT TO:<bob.senior@salad.local>
DATA
Subject: Test
Test message
.
QUIT
```

## Email Client Configuration

### For Thunderbird/Outlook

**Incoming Mail (IMAP):**
- Server: `mail.salad.local`
- Port: `143` (or `993` for SSL if configured)
- Username: `alice.dev` (just the username, not full email)
- Password: `<user-password>`
- Connection security: None (or SSL/TLS if configured)

**Outgoing Mail (SMTP):**
- Server: `mail.salad.local`
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
ldapsearch -x -H ldap://ldap.salad.local -b "dc=salad,dc=local"
```

### Check Mailbox Permissions
```bash
ls -la /var/mail/vhosts/salad.local/
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
- Test LDAP connection: `ping ldap.salad.local`
- Check LDAP service: `ssh ldap.salad.local "systemctl status slapd"`

### Issue: "User not found"
**Cause:** User doesn't exist in LDAP or wrong search base
**Solution:**
- Verify user exists: `ldapsearch -x -H ldap://ldap.salad.local -b "ou=people,dc=salad,dc=local" "(uid=alice.dev)"`
- Check search_base in LDAP config files

### Issue: "Permission denied" on mailbox
**Cause:** Mailbox directory has wrong ownership/permissions
**Solution:**
```bash
chown -R mail:mail /var/mail/vhosts/salad.local/USERNAME
chmod -R 700 /var/mail/vhosts/salad.local/USERNAME
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
