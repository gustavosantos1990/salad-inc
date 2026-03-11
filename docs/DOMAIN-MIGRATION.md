# 15. Domain Migration Guide: `salad.local` → `salad.com`

This guide covers **every change required** to migrate the infrastructure domain from `salad.local` to `salad.com`. Changes are organized per VM, from OS-level settings to service-specific configurations.

> [!CAUTION]
> **LDAP base DN change** (`dc=salad,dc=local` → `dc=salad,dc=com`) requires a full LDAP export/reimport. This affects every service that connects to LDAP. Plan for downtime.

> [!IMPORTANT]
> **Order of operations matters.** Follow this sequence:
> 1. DNS VM (so new names resolve)
> 2. LDAP VM (reimport directory with new base DN)
> 3. Proxy VM (new SSL cert + nginx configs)
> 4. All other VMs (service configs)
> 5. Host machine (systemd-resolved)

---

## Summary of Impact

| Change Type | Old Value | New Value | Affected VMs |
|-------------|-----------|-----------|-------------|
| Hostnames | `*.salad.local` | `*.salad.com` | All 10 VMs |
| DNS domain | `salad.local` | `salad.com` | DNS, Host |
| SSL wildcard cert | `*.salad.local` | `*.salad.com` | Proxy + all VMs trusting the cert |
| LDAP base DN | `dc=salad,dc=local` | `dc=salad,dc=com` | LDAP, Keycloak, Mail, Nexus, SSSD VMs |
| Email domain | `@salad.local` | `@salad.com` | Mail, LDAP, GitLab, Keycloak |
| Postfix virtual domain | `salad.local` | `salad.com` | Mail |
| Mailbox paths | `/var/mail/vhosts/salad.local/` | `/var/mail/vhosts/salad.com/` | Mail |
| SSL cert file names | `salad.local.crt/.key` | `salad.com.crt/.key` | Proxy, GitLab, others |

---

## 1. DNS VM (`dns.salad.local` → `dns.salad.com`)

**VM:** 192.168.123.10 · Alpine Linux · dnsmasq

### OS Level

| File | Change |
|------|--------|
| `/etc/hostname` | `dns` (no change needed if short hostname) |
| `/etc/hosts` | Remove/update any `salad.local` references |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### dnsmasq (`/etc/dnsmasq.conf`)

```diff
-local=/salad.local/
-domain=salad.local
+local=/salad.com/
+domain=salad.com

 # Service DNS (→ Nginx)
-address=/dns.salad.local/192.168.123.10
-address=/proxy.salad.local/192.168.123.30
-address=/gitlab.salad.local/192.168.123.30
-address=/nexus.salad.local/192.168.123.30
-address=/lam.salad.local/192.168.123.30
-address=/grafana.salad.local/192.168.123.30
-address=/keycloak.salad.local/192.168.123.30
-address=/portainer.salad.local/192.168.123.30
+address=/dns.salad.com/192.168.123.10
+address=/proxy.salad.com/192.168.123.30
+address=/gitlab.salad.com/192.168.123.30
+address=/nexus.salad.com/192.168.123.30
+address=/lam.salad.com/192.168.123.30
+address=/grafana.salad.com/192.168.123.30
+address=/keycloak.salad.com/192.168.123.30
+address=/portainer.salad.com/192.168.123.30

 # VM DNS (→ actual IPs)
-address=/repository.salad.local/192.168.123.20
-address=/database.salad.local/192.168.123.21
-address=/containerization.salad.local/192.168.123.22
-address=/artifact-repository.salad.local/192.168.123.23
-address=/monitor.salad.local/192.168.123.31
-address=/sso.salad.local/192.168.123.41
-address=/mail.salad.local/192.168.123.45
-address=/ldap.salad.local/192.168.123.46
+address=/repository.salad.com/192.168.123.20
+address=/database.salad.com/192.168.123.21
+address=/containerization.salad.com/192.168.123.22
+address=/artifact-repository.salad.com/192.168.123.23
+address=/monitor.salad.com/192.168.123.31
+address=/sso.salad.com/192.168.123.41
+address=/mail.salad.com/192.168.123.45
+address=/ldap.salad.com/192.168.123.46

 # App wildcard
-address=/.app.salad.local/192.168.123.30
+address=/.app.salad.com/192.168.123.30

 # Docker registries
-address=/registry-push.salad.local/192.168.123.30
-address=/registry.salad.local/192.168.123.30
+address=/registry-push.salad.com/192.168.123.30
+address=/registry.salad.com/192.168.123.30
```

### Restart

```bash
rc-service dnsmasq restart
```

### Ref Docs
- [02-dns-config.md](file:///home/gustavo/workspace/salad.inc/docs/02-dns-config.md)

---

## 2. LDAP VM (`ldap.salad.local` → `ldap.salad.com`)

**VM:** 192.168.123.46 · Debian 12 · OpenLDAP + LAM

### OS Level

| File | Change |
|------|--------|
| `/etc/hosts` | Remove `127.0.1.1 ldap.salad.local` line |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### OpenLDAP — Full Reimport Required

The base DN changes from `dc=salad,dc=local` to `dc=salad,dc=com`. This requires modifying the directory export and re-initializing the database.

1. **Export** current directory:
   ```bash
   slapcat -l /tmp/ldap-backup.ldif
   ```

2. **Find & replace** both the base DN and email attributes in the LDIF file:
   ```bash
   sed -i 's/dc=salad,dc=local/dc=salad,dc=com/g' /tmp/ldap-backup.ldif
   sed -i 's/@salad.local/@salad.com/g' /tmp/ldap-backup.ldif
   ```

3. **Reconfigure** slapd:
   ```bash
   dpkg-reconfigure slapd
   ```
   Provide the following answers when prompted:
   - Omit OpenLDAP server configuration? **No**
   - DNS domain name: **salad.com**
   - Organization name: **salad**
   - Database backend to use: **MDB**
   - Do you want the database to be removed when slapd is purged? **Yes**
   - Move old database? **Yes**
   - Allow LDAPv2 protocol? **No**

4. **Stop the LDAP service** to clear default entries and safely reimport:
   ```bash
   systemctl stop slapd
   ```

5. **Remove the auto-generated database files** (so `slapadd` doesn't conflict on `dc=salad,dc=com` existing):
   ```bash
   rm -f /var/lib/ldap/data.mdb /var/lib/ldap/lock.mdb
   ```

6. **Reimport** all data from your full backup:
   ```bash
   slapadd -l /tmp/ldap-backup.ldif
   ```

7. **Fix file permissions** and restart the service:
   ```bash
   chown -R openldap:openldap /var/lib/ldap
   systemctl start slapd
   ```

8. **Verify** the migration was successful by checking that user email addresses updated correctly:
   ```bash
   ldapsearch -x -b "ou=people,dc=salad,dc=com" -H ldap://localhost "(objectClass=inetOrgPerson)" uid mail
   ```

### Email Attributes in LDAP

All user `mail` attributes change:

| Old | New |
|-----|-----|
| `alice.dev@salad.local` | `alice.dev@salad.com` |
| `bob.senior@salad.local` | `bob.senior@salad.com` |
| `charlie.dev@salad.local` | `charlie.dev@salad.com` |
| `diana.arch@salad.local` | `diana.arch@salad.com` |
| `eve.manager@salad.local` | `eve.manager@salad.com` |
| `frank.sre@salad.local` | `frank.sre@salad.com` |
| `grace.sec@salad.local` | `grace.sec@salad.com` |

> This is handled automatically by the second `sed` replacement command during the export step above.

### LAM (LDAP Account Manager)

Update the LAM server profile to use the new base DN (`dc=salad,dc=com`). Accessible via `https://lam.salad.com/lam`.

### Ref Docs
- [07-ldap-setup.md](file:///home/gustavo/workspace/salad.inc/docs/07-ldap-setup.md)
- [00-rbac-quick-start.md](file:///home/gustavo/workspace/salad.inc/docs/00-rbac-quick-start.md)
- [11-rbac-implementation-guide.md](file:///home/gustavo/workspace/salad.inc/docs/11-rbac-implementation-guide.md)

---

## 3. Proxy VM (`proxy.salad.local` → `proxy.salad.com`)

**VM:** 192.168.123.30 · Alpine Linux · Nginx

### OS Level

| File | Change |
|------|--------|
| `/etc/hosts` | Remove `127.0.0.1 proxy.salad.local` line |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### SSL Certificate — Regenerate

```bash
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/salad.com.key \
  -out /etc/nginx/ssl/salad.com.crt \
  -subj "/C=BR/ST=SP/L=SaoPaulo/O=Salad Inc/CN=*.salad.com"
```

```bash
chmod 600 /etc/nginx/ssl/salad.com.key
chmod 644 /etc/nginx/ssl/salad.com.crt
```

> **Optional cleanup:** Remove old `salad.local.crt` and `salad.local.key` files.

### Nginx — All Virtual Hosts

Every `server` block in the Nginx config needs two types of changes:

1. **`server_name`**: `*.salad.local` → `*.salad.com`
2. **SSL cert/key paths**: `salad.local.crt/.key` → `salad.com.crt/.key`
3. **Backend `proxy_pass`/`set $backend`**: `*.salad.local` → `*.salad.com`

**Affected server blocks:**

| server_name | Backend upstream |
|-------------|-----------------|
| `gitlab.salad.com` | `repository.salad.com:80` |
| `nexus.salad.com` | `artifact-repository.salad.com:8082` |
| `grafana.salad.com` | `monitor.salad.com:3000` |
| `keycloak.salad.com` | `sso.salad.com:8080` |
| `lam.salad.com` | `ldap.salad.com` |
| `pgadmin.salad.com` | `containerization.salad.com:5050` |
| `portainer.salad.com` | `containerization.salad.com:9000` |
| `traefik.salad.com` | `containerization.salad.com:8080` |
| `registry-push.salad.com` | `artifact-repository.salad.com:8082` |
| `registry.salad.com` | `artifact-repository.salad.com:8083` |
| `*.app.salad.com` (wildcard) | `containerization.salad.com:80` |

### Quick sed replacement

```bash
sed -i 's/salad\.local/salad.com/g' /etc/nginx/http.d/default.conf
```

### Restart

```bash
rc-service nginx restart
```

### Ref Docs
- [03-proxy-config.md](file:///home/gustavo/workspace/salad.inc/docs/03-proxy-config.md)

---

## 4. GitLab / Repository VM (`repository.salad.local` → `repository.salad.com`)

**VM:** 192.168.123.20 · Debian 12 · GitLab Omnibus

### OS Level

| File | Change |
|------|--------|
| `/etc/hosts` | Remove `127.0.1.1 repository.salad.local` line |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### GitLab Configuration (`/etc/gitlab/gitlab.rb`)

| Setting | Old | New |
|---------|-----|-----|
| `external_url` | `http://repository.salad.local` | `http://repository.salad.com` |
| `gitlab_email_from` | `gitlab@salad.local` | `gitlab@salad.com` |
| `gitlab_email_reply_to` | `noreply@salad.local` | `noreply@salad.com` |
| `smtp_address` | `mail.salad.local` | `mail.salad.com` |
| `smtp_domain` | `salad.local` | `salad.com` |

### GitLab SSO/OIDC (`/etc/gitlab/gitlab.rb`)

| Setting | Old | New |
|---------|-----|-----|
| `issuer` | `https://keycloak.salad.local/realms/salad` | `https://keycloak.salad.com/realms/salad` |
| `redirect_uri` | `https://gitlab.salad.local/users/auth/openid_connect/callback` | `https://gitlab.salad.com/users/auth/openid_connect/callback` |

### Trusted SSL Certificate

Replace `/etc/gitlab/trusted-certs/salad.local.crt` with the new certificate:

```bash
# Copy new cert from proxy VM
scp root@proxy.salad.com:/etc/nginx/ssl/salad.com.crt /etc/gitlab/trusted-certs/salad.com.crt
chmod 644 /etc/gitlab/trusted-certs/salad.com.crt
rm /etc/gitlab/trusted-certs/salad.local.crt
```

### Reconfigure

```bash
gitlab-ctl reconfigure
```

### GitLab Admin UI

Update in **Admin Area → Settings → General → Sign-in restrictions**:
- **Email notification email**: `gitlab@salad.com`
- **Custom hostname**: `gitlab.salad.com`

### Ref Docs
- [05-gitlab-setup.md](file:///home/gustavo/workspace/salad.inc/docs/05-gitlab-setup.md)
- [GITLAB-WELCOME-EMAIL.md](file:///home/gustavo/workspace/salad.inc/docs/GITLAB-WELCOME-EMAIL.md)

---

## 5. Mail VM (`mail.salad.local` → `mail.salad.com`)

**VM:** 192.168.123.45 · Alpine Linux · Postfix + Dovecot

### OS Level

| File | Change |
|------|--------|
| `/etc/hosts` | Update any `salad.local` references |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### SSL Certificate — Regenerate / Copy

Copy the new certificates from the proxy VM to the mail VM:

```bash
scp root@proxy.salad.com:/etc/nginx/ssl/salad.com.crt /etc/ssl/certs/salad.com.crt
scp root@proxy.salad.com:/etc/nginx/ssl/salad.com.key /etc/ssl/private/salad.com.key

chmod 644 /etc/ssl/certs/salad.com.crt
chmod 600 /etc/ssl/private/salad.com.key
```

### Postfix (`/etc/postfix/main.cf`)

```diff
-myhostname = mail.salad.local
-mydomain = salad.local
+myhostname = mail.salad.com
+mydomain = salad.com

-smtpd_tls_cert_file=/etc/ssl/certs/salad.local.crt
-smtpd_tls_key_file=/etc/ssl/private/salad.local.key
+smtpd_tls_cert_file=/etc/ssl/certs/salad.com.crt
+smtpd_tls_key_file=/etc/ssl/private/salad.com.key
```

> `virtual_mailbox_domains` also references the domain — update accordingly.

### Postfix LDAP Maps

Update **all three** LDAP map files:
- `/etc/postfix/ldap-virtual-mailbox-maps.cf`
- `/etc/postfix/ldap-virtual-alias-maps.cf`
- `/etc/postfix/ldap-virtual-domains.cf`

Changes in each file:

```diff
-bind_dn = cn=admin,dc=salad,dc=local
-search_base = ou=people,dc=salad,dc=local
+bind_dn = cn=admin,dc=salad,dc=com
+search_base = ou=people,dc=salad,dc=com
```

### Dovecot

| File | Setting | Old | New |
|------|---------|-----|-----|
| `/etc/dovecot/dovecot.conf` | `mail_location` | `maildir:/var/mail/vhosts/salad.local/%n/Maildir` | `maildir:/var/mail/vhosts/salad.com/%n/Maildir` |
| `/etc/dovecot/dovecot.conf` | `mail_home` | `/var/mail/vhosts/salad.local/%n` | `/var/mail/vhosts/salad.com/%n` |
| `/etc/dovecot/dovecot-ldap.conf.ext` | `dn` | `cn=admin,dc=salad,dc=local` | `cn=admin,dc=salad,dc=com` |
| `/etc/dovecot/dovecot-ldap.conf.ext` | `base` | `ou=people,dc=salad,dc=local` | `ou=people,dc=salad,dc=com` |
| `/etc/dovecot/dovecot-ldap.conf.ext` | `user_attrs` | `=home=/var/mail/vhosts/salad.local/%n` | `=home=/var/mail/vhosts/salad.com/%n` |

### Mailbox Directory

```bash
# Move existing mailboxes
mv /var/mail/vhosts/salad.local /var/mail/vhosts/salad.com
chown -R mail:mail /var/mail/vhosts/salad.com
```

### Restart

```bash
rc-service postfix restart
rc-service dovecot restart
```

### Ref Docs
- [06-mail-setup.md](file:///home/gustavo/workspace/salad.inc/docs/06-mail-setup.md)
- [MAIL-CRITICAL-CONFIG.md](file:///home/gustavo/workspace/salad.inc/docs/MAIL-CRITICAL-CONFIG.md)
- [MAIL-LDAP-REFERENCE.md](file:///home/gustavo/workspace/salad.inc/docs/MAIL-LDAP-REFERENCE.md)

---

## 6. Database VM (`database.salad.local` → `database.salad.com`)

**VM:** 192.168.123.21 · Debian 12 · PostgreSQL

### OS Level

| File | Change |
|------|--------|
| `/etc/hosts` | Remove `127.0.1.1 database.salad.local` line |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### PostgreSQL

No domain-specific configuration inside PostgreSQL itself. It listens on IP addresses, not hostnames. **No application-level changes needed.**

### Ref Docs
- [09-database-setup.md](file:///home/gustavo/workspace/salad.inc/docs/09-database-setup.md)

---

## 7. Containerization VM (`containerization.salad.local` → `containerization.salad.com`)

**VM:** 192.168.123.22 · Debian 12 · Docker + Traefik + Portainer

### OS Level

| File | Change |
|------|--------|
| `/etc/hosts` | Remove `127.0.1.1 containerization.salad.local` line |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### Docker Stacks

**pgAdmin stack** (`pgadmin.yml` or similar):

```diff
-      - "database.salad.local:192.168.123.21"
+      - "database.salad.com:192.168.123.21"

-      PGADMIN_DEFAULT_EMAIL: "admin@salad.local"
+      PGADMIN_DEFAULT_EMAIL: "admin@salad.com"
```

**Portainer stack**: Update any Traefik labels referencing `salad.local`:

```diff
-        - "traefik.http.routers....rule=Host(`portainer.salad.local`)"
+        - "traefik.http.routers....rule=Host(`portainer.salad.com`)"
```

### App Deployments (Traefik Labels)

Any deployed app with Traefik routing labels must update:

```diff
-        - "traefik.http.routers.demo.rule=Host(`demo.app.salad.local`)"
+        - "traefik.http.routers.demo.rule=Host(`demo.app.salad.com`)"
```

### SSSD (if configured for SSH LDAP auth)

File: `/etc/sssd/sssd.conf`

```diff
-domains = salad.local
-[domain/salad.local]
-ldap_uri = ldap://ldap.salad.local
-ldap_search_base = dc=salad,dc=local
-ldap_user_search_base = ou=people,dc=salad,dc=local
-ldap_group_search_base = ou=groups,dc=salad,dc=local
+domains = salad.com
+[domain/salad.com]
+ldap_uri = ldap://ldap.salad.com
+ldap_search_base = dc=salad,dc=com
+ldap_user_search_base = ou=people,dc=salad,dc=com
+ldap_group_search_base = ou=groups,dc=salad,dc=com
```

```bash
systemctl restart sssd
```

### Docker Registry Login

Re-login to registries after domain change:

```bash
docker login registry-push.salad.com
docker login registry.salad.com
```

### Ref Docs
- [08-containerization-setup.md](file:///home/gustavo/workspace/salad.inc/docs/08-containerization-setup.md)
- [14-app-deployment-guide.md](file:///home/gustavo/workspace/salad.inc/docs/14-app-deployment-guide.md)

---

## 8. SSO / Keycloak VM (`sso.salad.local` → `sso.salad.com`)

**VM:** 192.168.123.41 · Debian 12 · Keycloak

### OS Level

| File | Change |
|------|--------|
| `/etc/hosts` | Remove `127.0.1.1 sso.salad.local` line |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### Keycloak Config (`/opt/keycloak/conf/keycloak.conf`)

```diff
-db-url=jdbc:postgresql://database.salad.local:5432/keycloak
+db-url=jdbc:postgresql://database.salad.com:5432/keycloak

-hostname=keycloak.salad.local
+hostname=keycloak.salad.com
```

### Keycloak Admin Console (UI Changes)

**Realm Settings → General:**
- **Frontend URL**: `https://keycloak.salad.com/`

**Realm Settings → Email:**
- **From**: `noreply@salad.com`
- **Host**: `mail.salad.com`

**LDAP User Federation:**

| Setting | Old | New |
|---------|-----|-----|
| Connection URL | `ldap://ldap.salad.local:389` | `ldap://ldap.salad.com:389` |
| Users DN | `ou=people,dc=salad,dc=local` | `ou=people,dc=salad,dc=com` |
| Bind DN | `cn=admin,dc=salad,dc=local` | `cn=admin,dc=salad,dc=com` |

**LDAP Group Mappers:**

| Mapper | Old LDAP Groups DN | New LDAP Groups DN |
|--------|--------------------|--------------------|
| service-access-mapper | `ou=service-access,ou=groups,dc=salad,dc=local` | `ou=service-access,ou=groups,dc=salad,dc=com` |
| ssh-access-mapper | `ou=ssh-access,ou=groups,dc=salad,dc=local` | `ou=ssh-access,ou=groups,dc=salad,dc=com` |
| gitlab-access-mapper | `ou=gitlab-access,ou=groups,dc=salad,dc=local` | `ou=gitlab-access,ou=groups,dc=salad,dc=com` |
| roles-mapper | `ou=roles,dc=salad,dc=local` | `ou=roles,dc=salad,dc=com` |

**Client Configurations** (all clients need URL updates):

| Client | Setting | Old | New |
|--------|---------|-----|-----|
| `gitlab` | Root URL | `https://gitlab.salad.local` | `https://gitlab.salad.com` |
| `gitlab` | Valid redirect URI | `https://gitlab.salad.local/users/auth/openid_connect/callback` | `https://gitlab.salad.com/users/auth/openid_connect/callback` |
| `gitlab` | Web origins | `https://gitlab.salad.local` | `https://gitlab.salad.com` |
| `grafana` | Valid redirect URI | `https://grafana.salad.local/login/generic_oauth` | `https://grafana.salad.com/login/generic_oauth` |
| `grafana` | Web origins | `https://grafana.salad.local` | `https://grafana.salad.com` |
| `portainer` | Valid redirect URI | `https://portainer.salad.local/*` | `https://portainer.salad.com/*` |
| `portainer` | Web origins | `https://portainer.salad.local` | `https://portainer.salad.com` |
| `nexus` | Valid redirect URI | `https://nexus.salad.local/ui/login` | `https://nexus.salad.com/ui/login` |
| `nexus` | Web origins | `https://nexus.salad.local` | `https://nexus.salad.com` |
| `prometheus` | Valid redirect URI | `https://prometheus.salad.local/oauth2/callback` | `https://prometheus.salad.com/oauth2/callback` |
| `traefik` | Valid redirect URI | `https://traefik.salad.local/oauth2/callback` | `https://traefik.salad.com/oauth2/callback` |
| `lam` | Valid redirect URI | `https://lam.salad.local/*` | `https://lam.salad.com/*` |

### Restart

```bash
systemctl restart keycloak
```

### Ref Docs
- [10-sso-keycloak-setup.md](file:///home/gustavo/workspace/salad.inc/docs/10-sso-keycloak-setup.md)

---

## 9. Monitoring VM (`monitor.salad.local` → `monitor.salad.com`)

**VM:** 192.168.123.31 · Debian 12 · Prometheus + Grafana + Loki

### OS Level

| File | Change |
|------|--------|
| `/etc/hosts` | Remove `127.0.1.1 monitor.salad.local` line |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### Prometheus (`/etc/prometheus/prometheus.yml`)

Update **all** scrape target hostnames:

```diff
 - job_name: 'dns'
-      - targets: ["dns.salad.local:9100"]
+      - targets: ["dns.salad.com:9100"]

 - job_name: 'repository'
-      - targets: ["repository.salad.local:9100"]
+      - targets: ["repository.salad.com:9100"]

 - job_name: 'database'
-      - targets: ["database.salad.local:9100"]
+      - targets: ["database.salad.com:9100"]

 - job_name: 'containerization'
-      - targets: ["containerization.salad.local:9100"]
+      - targets: ["containerization.salad.com:9100"]

 - job_name: 'nexus'
-      - targets: ["artifact-repository.salad.local:9100"]
+      - targets: ["artifact-repository.salad.com:9100"]

 - job_name: 'proxy'
-      - targets: ["proxy.salad.local:9100"]
+      - targets: ["proxy.salad.com:9100"]

 - job_name: 'sso'
-      - targets: ["sso.salad.local:9100"]
+      - targets: ["sso.salad.com:9100"]

 - job_name: 'mail'
-      - targets: ["mail.salad.local:9100"]
+      - targets: ["mail.salad.com:9100"]

 - job_name: 'ldap'
-      - targets: ["ldap.salad.local:9100"]
+      - targets: ["ldap.salad.com:9100"]

 - job_name: 'nexus_exporter'
-      - targets: ["artifact-repository.salad.local:9256"]
+      - targets: ["artifact-repository.salad.com:9256"]
```

### Grafana (`/etc/grafana/grafana.ini`)

```diff
-domain = grafana.salad.local
-root_url = https://grafana.salad.local/
+domain = grafana.salad.com
+root_url = https://grafana.salad.com/

-host = database.salad.local:5432
+host = database.salad.com:5432
```

### Grafana SSO/OIDC

```diff
-auth_url = https://keycloak.salad.local/realms/salad/protocol/openid-connect/auth
-token_url = https://keycloak.salad.local/realms/salad/protocol/openid-connect/token
-api_url = https://keycloak.salad.local/realms/salad/protocol/openid-connect/userinfo
+auth_url = https://keycloak.salad.com/realms/salad/protocol/openid-connect/auth
+token_url = https://keycloak.salad.com/realms/salad/protocol/openid-connect/token
+api_url = https://keycloak.salad.com/realms/salad/protocol/openid-connect/userinfo
```

### Promtail (Alloy) Log Agent

All VMs running the log agent have hostname labels:

```diff
-          host: ${HOSTNAME}.salad.local
+          host: ${HOSTNAME}.salad.com
```

### Restart

```bash
systemctl restart prometheus grafana-server
```

### Ref Docs
- [12-monitoring-setup.md](file:///home/gustavo/workspace/salad.inc/docs/12-monitoring-setup.md)

---

## 10. Nexus / Artifact Repository VM (`artifact-repository.salad.local` → `artifact-repository.salad.com`)

**VM:** 192.168.123.23 · Debian 12 · Sonatype Nexus CE

### OS Level

| File | Change |
|------|--------|
| `/etc/hosts` | Remove `127.0.1.1 artifact-repository.salad.local` line |
| `/etc/resolv.conf` | `search salad.local` → `search salad.com` |

### Nexus Properties (`/opt/nexus-data/etc/nexus.properties`)

```diff
-nexus.datastore.nexus.jdbcUrl=jdbc:postgresql://database.salad.local:5432/nexus
+nexus.datastore.nexus.jdbcUrl=jdbc:postgresql://database.salad.com:5432/nexus
```

### Nexus Admin UI — LDAP Configuration

| Setting | Old | New |
|---------|-----|-----|
| Hostname | `ldap.salad.local` | `ldap.salad.com` |
| Search base | `dc=salad,dc=local` | `dc=salad,dc=com` |
| System username | `cn=admin,dc=salad,dc=local` | `cn=admin,dc=salad,dc=com` |

### Nexus Admin UI — CI User

| Setting | Old | New |
|---------|-----|-----|
| Email | `gitlab-ci@salad.local` | `gitlab-ci@salad.com` |

### Restart

```bash
systemctl restart nexus
```

### Ref Docs
- [13-nexus-setup.md](file:///home/gustavo/workspace/salad.inc/docs/13-nexus-setup.md)

---

## 11. Host Machine (Laptop)

### QEMU/Libvirt Network — `salad-net` (`virbr1`)

**File:** [`infrastructure/libvirt/salad-net.xml`](file:///home/gustavo/workspace/salad.inc/infrastructure/libvirt/salad-net.xml)

> [!NOTE]
> ✅ This file has **already been updated** to `salad.com`.

The network XML definition requires:
- `<domain name='salad.com'/>` (was `salad.local`)
- All `<hostname>` entries updated (e.g., `dns.salad.com`, `repository.salad.com`, etc.)

To apply changes to the running network:

```bash
sudo virsh net-destroy salad-net
sudo virsh net-undefine salad-net
sudo virsh net-define infrastructure/libvirt/salad-net.xml
sudo virsh net-start salad-net
sudo virsh net-autostart salad-net
```

### systemd-resolved Split DNS

File: `/etc/systemd/resolved.conf.d/salad.conf` (or equivalent)

```diff
-Domains=~salad.local
+Domains=~salad.com
```

Or via CLI:

```bash
sudo resolvectl dns virbr1 192.168.123.10
sudo resolvectl domain virbr1 ~salad.com
```

### Verify

```bash
ping -c 1 repository.salad.com
resolvectl status virbr1
```

### Ref Docs
- [01-qemu-network.md](file:///home/gustavo/workspace/salad.inc/docs/01-qemu-network.md)
- [04-local-dns-config.md](file:///home/gustavo/workspace/salad.inc/docs/04-local-dns-config.md)

---

## 12. SSH Helper Tool

If using the `ssh-helper` TUI, update the TOML config to reference `.salad.com` hostnames.

---

## 13. Post-Migration Checklist

- [ ] DNS resolves all `*.salad.com` hostnames correctly
- [ ] SSL certificate valid for `*.salad.com`
- [ ] New SSL cert distributed to VMs that trust it (GitLab)
- [ ] LDAP directory accessible with new base DN `dc=salad,dc=com`
- [ ] Keycloak LDAP federation syncs users successfully
- [ ] Keycloak clients have updated redirect URIs
- [ ] GitLab SSO login works via `https://gitlab.salad.com`
- [ ] Grafana SSO login works via `https://grafana.salad.com`
- [ ] Portainer SSO login works via `https://portainer.salad.com`
- [ ] Mail delivery works (`user@salad.com`)
- [ ] Dovecot IMAP login works
- [ ] Prometheus scrapes all targets successfully
- [ ] Nexus LDAP authentication works
- [ ] Docker registry push/pull works (`registry-push.salad.com` / `registry.salad.com`)
- [ ] App deployments use `*.app.salad.com` routing
- [ ] All project documentation updated (find & replace `salad.local` → `salad.com` in `/docs`)

---

## 14. Bulk Documentation Update

After completing the infrastructure migration, update all project docs:

```bash
cd /home/gustavo/workspace/salad.inc
grep -rl 'salad\.local' docs/ README.md | xargs sed -i 's/salad\.local/salad.com/g'
grep -rl 'dc=salad,dc=local' docs/ README.md | xargs sed -i 's/dc=salad,dc=local/dc=salad,dc=com/g'
```

> [!WARNING]
> Review the diff carefully before committing — some documentation may reference `salad.local` in a historical/explanatory context that shouldn't be changed.
