# 13 - Provisioning Artifact Repository (Sonatype Nexus Repository CE)

We will provision the **Artifact Repository** VM (`artifact-repository.salad.com`). This is a **Debian 12** instance running **Sonatype Nexus Repository Community Edition (CE)**, which serves as:

- **Maven Repository** — stores, proxies, and groups Java build artifacts
- **Docker Registry** — stores and distributes Docker images built by GitLab CI

> **Why Nexus CE over Artifactory OSS?**
> Artifactory OSS does not support Docker repositories (Enterprise/Pro only).
> Nexus CE is genuinely free (Eclipse Public License), supports both Maven and Docker natively, and allows connecting to an external PostgreSQL database.

> ⚠️ **CE Limitation — Docker Group Deployment:** In the Community Edition, push and pull use separate port-based URLs (no single unified URL). Push goes to the hosted repo port; pull goes to the group repo port. This is a minor configuration detail, not a blocker.

---

## 1. VM Directory Structure

```bash
mkdir -p infrastructure/vms/artifact-repository
```

## 2. Create Virtual Disk

```bash
qemu-img create -f qcow2 infrastructure/vms/artifact-repository/artifact-repository-os.qcow2 20G
```

## 3. Install Debian (ISO)

```bash
sudo virt-install \
  --name artifact-repository \
  --ram 3072 \
  --vcpus 2 \
  --network network=salad-net,mac=52:54:00:00:00:23 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/artifact-repository/artifact-repository-os.qcow2,size=20 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/debian-12.12.0-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics spice \
  --noautoconsole
```

### 3.1. Connect to Console

```bash
virt-viewer --connect qemu:///system --wait artifact-repository
```

### Installation Pointers

1. **Hostname:** `artifact-repository.salad.com`
2. **Domain:** `salad.com`
3. **Partitioning:** Standard / (Root).
4. **Software Selection:**
   - [X] SSH server
   - [X] Standard system utilities
   - [ ] (Uncheck everything else)

---

## 4. Post-Install Configuration

Login as `root`.

### 4.1. System Prep

```bash
apt-get update && apt-get install -y curl openssh-server net-tools dnsutils vim wget gnupg apt-transport-https
```

**Disable Swap:**
```bash
swapoff -a && sed -i '/swap/d' /etc/fstab
```

**Configure DNS:**
```bash
echo "nameserver 192.168.123.10" > /etc/resolv.conf
```

**Fix /etc/hosts Conflict:**
```bash
vi /etc/hosts
```
Remove/comment: `127.0.1.1 artifact-repository.salad.com artifact-repository`

**Enable Serial Console:**
```bash
systemctl enable --now serial-getty@ttyS0.service
```

### 4.2. Configure SSH Access

```bash
mkdir -p /root/.ssh && \
chmod 700 /root/.ssh && \
echo "<your-ssh-public-key>" > /root/.ssh/authorized_keys && \
chmod 600 /root/.ssh/authorized_keys
```

### 4.3. Install Java

Nexus requires Java 11+. Install OpenJDK 17:

```bash
apt-get install -y openjdk-17-jdk-headless && java -version
```

---

## 5. Prepare PostgreSQL Database

Nexus CE supports an external PostgreSQL database (added in v3.77.0). Ensure the database and user exist on `database.salad.com` (see `docs/09-database-setup.md`).

On `database.salad.com`, connect as `postgres` and run:

```sql
CREATE USER nexus WITH PASSWORD 'nexus_password';
CREATE DATABASE nexus OWNER nexus;
GRANT ALL PRIVILEGES ON DATABASE nexus TO nexus;
```

---

## 6. Install Sonatype Nexus Repository CE

Nexus is distributed as a self-contained tarball — no package manager required.

### 6.1. Download and Extract

```bash
NEXUS_VERSION="3.77.0-08"
wget -q "https://download.sonatype.com/nexus/3/nexus-${NEXUS_VERSION}-unix.tar.gz" -O /tmp/nexus.tar.gz
tar -xzf /tmp/nexus.tar.gz -C /opt
mv /opt/nexus-${NEXUS_VERSION} /opt/nexus
mv /opt/sonatype-work /opt/nexus-data
rm /tmp/nexus.tar.gz
```

### 6.2. Create a Dedicated System User

```bash
useradd --system --no-create-home --shell /bin/false nexus
chown -R nexus:nexus /opt/nexus /opt/nexus-data
```

### 6.3. Configure Nexus to Run as the `nexus` User

```bash
echo 'run_as_user="nexus"' > /opt/nexus/bin/nexus.rc
```

### 6.4. Configure JVM Memory

The default `nexus.vmoptions` ships with required `--add-opens` flags for Java 17 module access. **Do not replace the file** — append our custom settings on top of the defaults:

```bash
cat >> /opt/nexus/bin/nexus.vmoptions << 'EOF'

# Custom: tuned for 3 GB VM
-Xms512m
-Xmx1536m
-XX:MaxDirectMemorySize=512m
-Dkaraf.data=/opt/nexus-data
-Dkaraf.log=/opt/nexus-data/log
-Djava.io.tmpdir=/opt/nexus-data/tmp
EOF
```

> **Memory budget for a 3 GB VM:** 1536m heap + 512m direct memory ≈ 2 GB for Nexus, leaving ~1 GB for the OS and kernel buffers.
>
> ⚠️ **Do not use `cat >`** (overwrite) here — the default vmoptions contains `--add-opens` flags required by Apache Felix/Karaf on Java 17. Removing them causes an `InaccessibleObjectException` crash at startup.

### 6.5. Configure External PostgreSQL

Edit `/opt/nexus-data/etc/nexus.properties` (create if it doesn't exist yet — Nexus creates it on first start, so start once, stop, then edit):

```bash
# First start to generate config files
# Nexus 3.77 takes 2-4 minutes before nexus-data/etc/ is written
su - nexus -s /bin/bash -c "/opt/nexus/bin/nexus start"

# Watch karaf.log — nexus.log doesn't appear until fully started
tail -f /opt/nexus-data/log/karaf.log
# Wait until you see: "nexus/bundles" activity settles, then Ctrl+C

# Verify nexus-data/etc/ was created
ls /opt/nexus-data/etc/

# Then stop
su - nexus -s /bin/bash -c "/opt/nexus/bin/nexus stop"
sleep 10
```

Now edit the properties file:

```bash
cat >> /opt/nexus-data/etc/nexus.properties << 'EOF'

# External PostgreSQL (Nexus CE >= 3.77.0)
nexus.datastore.enabled=true
nexus.datastore.nexus.jdbcUrl=jdbc:postgresql://database.salad.com:5432/nexus
nexus.datastore.nexus.username=nexus
nexus.datastore.nexus.password=nexus_password
nexus.datastore.nexus.maximumPoolSize=50
EOF
```

### 6.6. Create systemd Service

```bash
cat > /etc/systemd/system/nexus.service << 'EOF'
[Unit]
Description=Sonatype Nexus Repository CE
After=network-online.target

[Service]
Type=forking
LimitNOFILE=65536
ExecStart=/opt/nexus/bin/nexus start
ExecStop=/opt/nexus/bin/nexus stop
User=nexus
Group=nexus
Restart=on-failure
RestartSec=10
TimeoutStartSec=180
TimeoutStopSec=60

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now nexus
```

### 6.7. Monitor Startup

Nexus takes **2–4 minutes** to fully start. Monitor startup progress in two phases:

**Phase 1 — early boot** (before nexus.log exists, use karaf.log):
```bash
tail -f /opt/nexus-data/log/karaf.log
```

**Phase 2 — once nexus.log appears**, switch to it:
```bash
tail -f /opt/nexus-data/log/nexus.log
```

Wait until you see: `Started Sonatype Nexus`

Check the port is listening:
```bash
ss -tlnp | grep java
```
Expected: Nexus on port `8081`.

---

## 7. Initial Nexus Configuration

Access via SSH tunnel from your host machine:

```bash
ssh -L 8081:localhost:8081 root@artifact-repository.salad.com
```

Open `http://localhost:8081` in your browser.

### 7.1. First Login

Default credentials: **admin** / *(printed in file)*:

```bash
cat /opt/nexus-data/admin.password
```

Log in and complete the setup wizard:
1. Change the admin password → set a strong password
2. Choose **"Disable anonymous access"** (tighter security)

---

## 8. Configure LDAP Authentication

Nexus CE supports LDAP natively. Configure it to authenticate against `ldap.salad.com`.

1. Go to **Administration** (gear icon) → **Security** → **LDAP**.
2. Click **Create connection**.
3. Configure:
   - **Name:** `salad-ldap`
   - **Protocol:** `ldap`
   - **Hostname:** `ldap.salad.com`
   - **Port:** `389`
   - **Search base:** `dc=salad,dc=com`
   - **Authentication:** `Simple`
   - **Username:** `cn=admin,dc=salad,dc=com`
   - **Password:** *(LDAP admin password)*
4. Click **Verify connection** → should show `Connected`.
5. Under **User and group settings**:
   - **User base DN:** `ou=people`
   - **User object class:** `inetOrgPerson`
   - **User ID attribute:** `uid`
   - **User real name attribute:** `cn`
   - **User email attribute:** `mail`
   - **Group base DN:** `ou=groups`
   - **Group object class:** `groupOfNames`
   - **Group type:** `Static` *(groups use explicit `member` DN attributes, not dynamic URL filters)*
   - **Group ID attribute:** `cn`
   - **Group member attribute:** `member`
   - **Group member format:** `${dn}`
   - **Map LDAP groups as roles:** ✅ enabled *(maps each LDAP group to a Nexus role so you can assign privileges to them)*
6. Click **Verify user mapping** → confirm your LDAP users appear.
7. Click **Save**.

---

## 9. Create Repositories

### Port Assignment Strategy

Nexus serves the Docker registry on **dedicated ports** (one per group/hosted repo) because Docker requires V2 API at the root path. Plan:

| Port | Repo | Direction |
|------|------|-----------|
| `8081` | Nexus UI + Maven | All |
| `8082` | `docker-hosted` (push) | CI/CD push |
| `8083` | `docker-group` (pull) | Swarm pull |

### 9.1. Maven Repositories

#### Maven Hosted (local artifacts)

1. Go to **Administration** → **Repository** → **Repositories** → **Create repository**.
2. Select **maven2 (hosted)**.
3. Configure:
   - **Name:** `maven-hosted`
   - **Version policy:** `Release`
   - **Deployment policy:** `Allow redeploy`
4. Click **Create repository**.

#### Maven Proxy (Maven Central)

1. Create repository → **maven2 (proxy)**.
2. Configure:
   - **Name:** `maven-central`
   - **Remote storage URL:** `https://repo1.maven.org/maven2`
   - **Blob store:** `default`
3. Click **Create repository**.

#### Maven Group (unified endpoint)

1. Create repository → **maven2 (group)**.
2. Configure:
   - **Name:** `maven-group`
   - **Member repositories:** add `maven-hosted`, `maven-central`
3. Click **Create repository**.

### 9.2. Docker Repositories

#### Docker Hosted (push target for CI/CD)

1. Create repository → **docker (hosted)**.
2. Configure:
   - **Name:** `docker-hosted`
   - **HTTP port:** `8082`
   - **Allow anonymous docker pull:** disabled
   - **Blob store:** `default`
3. Click **Create repository**.

#### Docker Proxy (Docker Hub cache)

1. Create repository → **docker (proxy)**.
2. Configure:
   - **Name:** `docker-hub`
   - **Remote storage URL:** `https://registry-1.docker.io`
   - **Docker Registry API Support:** enable **"Use Docker Hub"**
   - **Blob store:** `default`
3. Click **Create repository**.

#### Docker Group (pull endpoint for Swarm)

1. Create repository → **docker (group)**.
2. Configure:
   - **Name:** `docker-group`
   - **HTTP port:** `8083`
   - **Member repositories:** add `docker-hosted`, `docker-hub`
3. Click **Create repository**.

---

## 10. Create a CI/CD Service User

Create a dedicated user for GitLab CI.

1. Go to **Administration** → **Security** → **Users** → **Create local user**.
2. Configure:
   - **Username:** `gitlab-ci`
   - **First name / Last name:** GitLab CI
   - **Email:** `gitlab-ci@salad.com`
   - **Password:** set a strong password
   - **Status:** Active
   - **Roles:** `nx-anonymous` as a base, then assign a custom role (see below)
3. Click **Create user**.

#### Create a Deploy Role

1. Go to **Administration** → **Security** → **Roles** → **Create role**.
2. Configure:
   - **Role ID:** `ci-deploy`
   - **Role name:** CI Deploy
   - **Privileges:** add:
     - `nx-repository-view-maven2-maven-hosted-*`
     - `nx-repository-view-docker-docker-hosted-*`
     - `nx-repository-view-docker-docker-group-read`
3. Click **Create role**.

Assign the `ci-deploy` role to `gitlab-ci`.

---

## 11. Configure Nginx Proxy (on proxy.salad.com)

Add the following blocks to `services.conf` on the proxy VM. Nexus needs three virtual hosts: one for the UI/Maven and two for Docker (push port + pull port).

```nginx
# --- Nexus UI + Maven ---
server {
    listen 443 ssl;
    server_name nexus.salad.com;

    ssl_certificate /etc/nginx/ssl/salad.com.crt;
    ssl_certificate_key /etc/nginx/ssl/salad.com.key;

    client_max_body_size 0;

    location / {
        set $backend_nexus artifact-repository.salad.com:8081;
        proxy_pass http://$backend_nexus;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 900;
    }
}

server {
    listen 80;
    server_name nexus.salad.com;
    return 301 https://$host$request_uri;
}

# --- Docker Push (CI/CD → hosted repo) ---
server {
    listen 443 ssl;
    server_name registry-push.salad.com;

    ssl_certificate /etc/nginx/ssl/salad.com.crt;
    ssl_certificate_key /etc/nginx/ssl/salad.com.key;

    client_max_body_size 0;
    chunked_transfer_encoding on;

    location / {
        set $backend_docker_push artifact-repository.salad.com:8082;
        proxy_pass http://$backend_docker_push;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 900;
    }
}

server {
    listen 80;
    server_name registry-push.salad.com;
    return 301 https://$host$request_uri;
}

# --- Docker Pull (Swarm → group repo) ---
server {
    listen 443 ssl;
    server_name registry.salad.com;

    ssl_certificate /etc/nginx/ssl/salad.com.crt;
    ssl_certificate_key /etc/nginx/ssl/salad.com.key;

    client_max_body_size 0;
    chunked_transfer_encoding on;

    location / {
        set $backend_docker_pull artifact-repository.salad.com:8083;
        proxy_pass http://$backend_docker_pull;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 900;
    }
}

server {
    listen 80;
    server_name registry.salad.com;
    return 301 https://$host$request_uri;
}
```

Reload Nginx:
```bash
nginx -t && nginx -s reload
```

### DNS Entries

On `dns.salad.com`, add:

```bash
echo "address=/registry-push.salad.com/192.168.123.30" >> /etc/dnsmasq.conf
echo "address=/registry.salad.com/192.168.123.30" >> /etc/dnsmasq.conf
service dnsmasq restart
```

> `nexus.salad.com` DNS entry already points to `192.168.123.30` (Nginx) — no change needed.

---

## 12. Configure Maven in GitLab Projects

Create `.mvn/settings.xml` in the project:

```xml
<settings>
  <servers>
    <server>
      <id>nexus</id>
      <username>${env.NEXUS_USER}</username>
      <password>${env.NEXUS_PASSWORD}</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nexus.salad.com/repository/maven-group/</url>
    </mirror>
  </mirrors>
  <profiles>
    <profile>
      <id>nexus</id>
      <repositories>
        <repository>
          <id>nexus</id>
          <url>https://nexus.salad.com/repository/maven-group/</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
      </repositories>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>nexus</activeProfile>
  </activeProfiles>
</settings>
```

In `.gitlab-ci.yml`:

```yaml
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2"

build:
  stage: build
  image: maven:3.9.6-eclipse-temurin-17
  script:
    - mvn package -s .mvn/settings.xml
  variables:
    NEXUS_USER: gitlab-ci
    NEXUS_PASSWORD: $NEXUS_CI_PASSWORD   # set in GitLab CI/CD variables
```

---

## 13. Configure Docker in GitLab CI

To push Docker images to Nexus from GitLab CI:

```yaml
build-image:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_TLS_CERTDIR: "/certs"
    # Push to the hosted repo (port 8082 / registry-push)
    IMAGE_NAME: registry-push.salad.com/my-app
  script:
    - docker login registry-push.salad.com -u gitlab-ci -p $NEXUS_CI_PASSWORD
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
```

Docker Swarm pulls from the group (port 8083 / registry):

```yaml
# In your stack deploy or compose file:
image: registry.salad.com/my-app:1.0.0
```

---

## 14. Install Node Exporter

```bash
apt-get install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
```

---

## 15. Verification

From your host machine:

1. **Nexus UI:**
   ```bash
   curl -k https://nexus.salad.com/service/rest/v1/status
   ```
   Expected: HTTP 200

2. **Maven repository accessible:**
   ```bash
   curl -k -u admin:<password> https://nexus.salad.com/repository/maven-group/
   ```

3. **Docker push (CI → hosted):**
   ```bash
   docker login registry-push.salad.com -u gitlab-ci -p <password>
   docker pull hello-world
   docker tag hello-world registry-push.salad.com/hello-world:test
   docker push registry-push.salad.com/hello-world:test
   ```

4. **Docker pull (Swarm → group):**
   ```bash
   docker login registry.salad.com -u gitlab-ci -p <password>
   docker pull registry.salad.com/hello-world:test
   ```

5. **Node Exporter:**
   ```bash
   curl http://192.168.123.23:9100/metrics | grep node_uname
   ```
