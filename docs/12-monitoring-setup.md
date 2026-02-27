# 12 - Provisioning Monitoring Server (Prometheus + Grafana + Loki)

We will provision the **Monitor** VM (`monitor.salad.local`). This is a **Debian 12** instance running **Prometheus** (metrics collection), **Grafana** (visualization), and **Loki** (log aggregation) for full observability of the Salad.Inc infrastructure.

## 1. VM Directory Structure

Create the directory for the VM:

```bash
mkdir -p infrastructure/vms/monitor
```

## 2. Create Virtual Disk

We will create a single disk for the OS and all monitoring data:

```bash
qemu-img create -f qcow2 infrastructure/vms/monitor/monitor-os.qcow2 15G
```

## 3. Install Debian (ISO)

```bash
sudo virt-install \
  --name monitor \
  --ram 1536 \
  --vcpus 2 \
  --network network=salad-net,mac=52:54:00:00:00:31 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/monitor/monitor-os.qcow2,size=15 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/debian-12.12.0-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics spice \
  --noautoconsole
```

### 3.1. Connect to Console

```bash
virt-viewer --connect qemu:///system --wait monitor
```

### Installation Pointers
1.  **Hostname:** `monitor.salad.local`
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
apt-get update && apt-get install -y curl openssh-server net-tools dnsutils vim wget apt-transport-https software-properties-common gnupg
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
Edit `/etc/hosts` and remove/change the `127.0.1.1` line:
```bash
vi /etc/hosts
```
Remove/comment: `127.0.1.1 monitor.salad.local monitor`

**Enable Serial Console**:
```bash
systemctl enable --now serial-getty@ttyS0.service
```

### 2. Configure SSH Access

```bash
mkdir -p /root/.ssh && \
chmod 700 /root/.ssh && \
echo "<your-ssh-public-key>" > /root/.ssh/authorized_keys && \
chmod 600 /root/.ssh/authorized_keys
```

## 5. Install Prometheus

### 1. Create Prometheus User

```bash
groupadd --system prometheus
useradd --system --no-create-home --shell /bin/false -g prometheus prometheus
```

### 2. Download and Install

```bash
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.51.2/prometheus-2.51.2.linux-amd64.tar.gz
tar -xvzf prometheus-2.51.2.linux-amd64.tar.gz
mv prometheus-2.51.2.linux-amd64 prometheus-files
```

Install binaries:
```bash
cp prometheus-files/prometheus /usr/local/bin/
cp prometheus-files/promtool /usr/local/bin/
chown prometheus:prometheus /usr/local/bin/prometheus
chown prometheus:prometheus /usr/local/bin/promtool
```

Install console files:
```bash
mkdir -p /etc/prometheus /var/lib/prometheus
cp -r prometheus-files/consoles /etc/prometheus
cp -r prometheus-files/console_libraries /etc/prometheus
chown -R prometheus:prometheus /etc/prometheus
chown -R prometheus:prometheus /var/lib/prometheus
```

Clean up:
```bash
rm -rf /tmp/prometheus-files /tmp/prometheus-2.51.2.linux-amd64.tar.gz
```

### 3. Configure Prometheus

Create `/etc/prometheus/prometheus.yml`:

```bash
cat > /etc/prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: []

rule_files: []

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "dns"
    static_configs:
      - targets: ["dns.salad.local:9100"]

  - job_name: "repository"
    static_configs:
      - targets: ["repository.salad.local:9100"]

  - job_name: "database"
    static_configs:
      - targets: ["database.salad.local:9100"]

  - job_name: "containerization"
    static_configs:
      - targets: ["containerization.salad.local:9100"]

  - job_name: "artifact-repository"
    static_configs:
      - targets: ["artifact-repository.salad.local:9100"]

  - job_name: "proxy"
    static_configs:
      - targets: ["proxy.salad.local:9100"]

  - job_name: "monitor"
    static_configs:
      - targets: ["localhost:9100"]

  - job_name: "sso"
    static_configs:
      - targets: ["sso.salad.local:9100"]

  - job_name: "mail"
    static_configs:
      - targets: ["mail.salad.local:9100"]

  - job_name: "ldap"
    static_configs:
      - targets: ["ldap.salad.local:9100"]
EOF
```

Set permissions:
```bash
chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

### 4. Create Systemd Service

Create `/etc/systemd/system/prometheus.service`:

```bash
cat > /etc/systemd/system/prometheus.service << 'EOF'
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start:
```bash
systemctl daemon-reload
systemctl enable --now prometheus
```

## 6. Install Node Exporter (on the Monitor VM itself)

```bash
apt-get install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
```

## 6.1. Install Process Exporter (Optional — per-process metrics)

**Process Exporter** exposes per-process CPU and memory metrics to Prometheus, enabling Grafana dashboards that show which processes are consuming the most resources on each VM.

Install on any VM where you want per-process visibility (e.g. `artifact-repository` where Artifactory's JVM consumes most RAM):

```bash
cd /tmp
wget https://github.com/ncabatoff/process-exporter/releases/download/v0.8.7/process-exporter-0.8.7.linux-amd64.tar.gz
tar -xvzf process-exporter-0.8.7.linux-amd64.tar.gz
mv process-exporter-0.8.7.linux-amd64/process-exporter /usr/local/bin/
rm -rf process-exporter-0.8.7.linux-amd64 process-exporter-0.8.7.linux-amd64.tar.gz
```

Create config:
```bash
mkdir -p /etc/process-exporter
cat > /etc/process-exporter/config.yml << 'EOF'
process_names:
  - name: "{{.Comm}}"
    cmdline:
      - '.+'
EOF
```

Create systemd service:
```bash
cat > /etc/systemd/system/process-exporter.service << 'EOF'
[Unit]
Description=Process Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/process-exporter \
  --config.path=/etc/process-exporter/config.yml \
  --web.listen-address=0.0.0.0:9256
Restart=always

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable --now process-exporter
```

Then add the scrape target to `/etc/prometheus/prometheus.yml` on the monitor VM:

```yaml
  - job_name: "process-artifact-repository"
    static_configs:
      - targets: ["artifact-repository.salad.local:9256"]
```

Reload Prometheus:
```bash
systemctl reload prometheus
```

In Grafana, import dashboard **ID `249`** (Named Process Exporter) and filter by instance to see per-process CPU and RAM.

## 7. Install Loki

### 1. Create Loki User

```bash
groupadd --system loki
useradd --system --no-create-home --shell /bin/false -g loki loki
```

### 2. Download and Install

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v3.1.0/loki-linux-amd64.zip
apt-get install -y unzip
unzip loki-linux-amd64.zip
mv loki-linux-amd64 /usr/local/bin/loki
chmod +x /usr/local/bin/loki
chown loki:loki /usr/local/bin/loki
rm loki-linux-amd64.zip
```

### 3. Create Loki Data Directories

```bash
mkdir -p /var/lib/loki/chunks /var/lib/loki/rules /etc/loki
chown -R loki:loki /var/lib/loki /etc/loki
```

### 4. Configure Loki

Create `/etc/loki/loki-config.yml`:

```bash
cat > /etc/loki/loki-config.yml << 'EOF'
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

limits_config:
  retention_period: 744h

compactor:
  working_directory: /var/lib/loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150
  delete_request_store: filesystem
EOF
```

Set permissions:
```bash
chown loki:loki /etc/loki/loki-config.yml
```

### 5. Create Systemd Service

Create `/etc/systemd/system/loki.service`:

```bash
cat > /etc/systemd/system/loki.service << 'EOF'
[Unit]
Description=Loki Log Aggregation
Wants=network-online.target
After=network-online.target

[Service]
User=loki
Group=loki
Type=simple
ExecStart=/usr/local/bin/loki -config.file=/etc/loki/loki-config.yml
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start:
```bash
systemctl daemon-reload
systemctl enable --now loki
```

## 8. Install Grafana

### 1. Add Grafana APT Repository

```bash
wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" > /etc/apt/sources.list.d/grafana.list
apt-get update
```

### 2. Install Grafana

```bash
apt-get install -y grafana
```

### 3. Configure Grafana

**Prerequisite:** Ensure the `grafana` database and user have been created on `database.salad.local` as described in `docs/09-database-setup.md` (section 6).

Edit `/etc/grafana/grafana.ini`:

```bash
vi /etc/grafana/grafana.ini
```

Update the following settings:

```ini
[server]
http_addr = 0.0.0.0
http_port = 3000
domain = grafana.salad.local
root_url = https://grafana.salad.local/

[database]
type = postgres
host = database.salad.local:5432
name = grafana
user = grafana
password = grafana_password
ssl_mode = disable

[security]
admin_user = admin
admin_password = <user-password>

[users]
allow_sign_up = false
```

### 4. Enable and Start Grafana

```bash
systemctl daemon-reload
systemctl enable --now grafana-server
```

## 9. Configure Grafana Data Sources

Once Grafana is running, access it at `https://grafana.salad.local` (through Nginx proxy) or directly at `http://monitor.salad.local:3000`.

### 9.1. Add Prometheus Data Source

1.  Log in with `admin` / `<user-password>`.
2.  Go to **Connections** → **Data sources** → **Add data source**.
3.  Select **Prometheus**.
4.  Configure:
    -   **Name**: `Prometheus`
    -   **URL**: `http://localhost:9090`
    -   **Access**: `Server (default)`
5.  Click **Save & Test**.

### 9.2. Add Loki Data Source

1.  Go to **Connections** → **Data sources** → **Add data source**.
2.  Select **Loki**.
3.  Configure:
    -   **Name**: `Loki`
    -   **URL**: `http://localhost:3100`
    -   **Access**: `Server (default)`
4.  Click **Save & Test**.

## 10. Install Promtail (Log Shipper) on Each VM

**Promtail** is Loki's log shipper agent. It runs on each VM and sends logs to Loki on the Monitor VM.

### Install on Each Debian VM

Run the following on **each Debian VM** (repository, database, containerization, artifact-repository, sso, mail, ldap, monitor):

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v3.1.0/promtail-linux-amd64.zip
apt-get install -y unzip
unzip promtail-linux-amd64.zip
mv promtail-linux-amd64 /usr/local/bin/promtail
chmod +x /usr/local/bin/promtail
rm promtail-linux-amd64.zip
```

Create config directory:
```bash
mkdir -p /etc/promtail
```

Create `/etc/promtail/promtail-config.yml` (replace `<hostname>` with the actual hostname, e.g., `database`):

```bash
HOSTNAME=$(hostname -s)
cat > /etc/promtail/promtail-config.yml << EOF
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://192.168.123.31:3100/loki/api/v1/push

scrape_configs:
  # job_name must match the job label value so {job="varlogs"} queries work in Grafana
  - job_name: varlogs
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          host: ${HOSTNAME}.salad.local
          __path__: /var/log/*log
EOF
```

Create Promtail systemd service:
```bash
cat > /etc/systemd/system/promtail.service << 'EOF'
[Unit]
Description=Promtail Log Shipper
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/promtail-config.yml
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start:
```bash
systemctl daemon-reload
systemctl enable --now promtail
```

### Install on Alpine VMs (dns, proxy, mail)

On Alpine VMs, install Promtail using the `apk` package or binary:

```bash
cd /tmp
wget https://github.com/grafana/loki/releases/download/v3.1.0/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
mv promtail-linux-amd64 /usr/local/bin/promtail
chmod +x /usr/local/bin/promtail
rm promtail-linux-amd64.zip
```

Create config (replace `<hostname>` as needed):
```bash
HOSTNAME=$(hostname -s)
mkdir -p /etc/promtail
cat > /etc/promtail/promtail-config.yml << EOF
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://192.168.123.31:3100/loki/api/v1/push

scrape_configs:
  # job_name must match the job label value so {job="varlogs"} queries work in Grafana
  - job_name: varlogs
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          host: ${HOSTNAME}.salad.local
          __path__: /var/log/*log
EOF
```

Create OpenRC service `/etc/init.d/promtail`:
```bash
cat > /etc/init.d/promtail << 'EOF'
#!/sbin/openrc-run
description="Promtail Log Shipper"

command="/usr/local/bin/promtail"
command_args="-config.file=/etc/promtail/promtail-config.yml"
command_background=true
pidfile="/run/promtail.pid"

depend() {
    need net
}
EOF
chmod +x /etc/init.d/promtail
rc-update add promtail default
rc-service promtail start
```

## 11. Configure Nginx Proxy (on proxy.salad.local)

The Grafana block is **already included** in `services.conf` (see `docs/03-proxy-config.md`). No separate file is needed.

> **Why not use a separate `upstream` block?**
> Nginx resolves hostnames in `upstream` blocks **once at startup**. If the monitor VM is not yet running when Nginx starts, you get:
> `nginx: [emerg] host not found in upstream "monitor.salad.local"`
> The `set $backend ...` variable pattern (used throughout `services.conf`) forces Nginx to resolve the hostname **at request time**, so Nginx starts successfully even if the backend VM is temporarily down.

The relevant block already in `services.conf`:

```nginx
# DNS resolver - runtime resolution (prevents startup failures if backend is down)
resolver 192.168.123.10 valid=30s ipv6=off;

# Grafana
server {
    listen 80;
    server_name grafana.salad.local;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name grafana.salad.local;

    ssl_certificate /etc/nginx/ssl/salad.local.crt;
    ssl_certificate_key /etc/nginx/ssl/salad.local.key;

    location / {
        set $backend_grafana monitor.salad.local:3000;
        proxy_pass http://$backend_grafana;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket support (required for Grafana Live)
    location /api/live/ {
        set $backend_grafana monitor.salad.local:3000;
        proxy_pass http://$backend_grafana;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

After any change to `services.conf`, reload Nginx:
```bash
nginx -t && nginx -s reload
```


## 12. Import Grafana Dashboards

Grafana has a rich library of community dashboards. Import these recommended dashboards:

1.  Go to **Dashboards** → **Import**.
2.  Enter the dashboard ID and click **Load**.
3.  Select **Prometheus** as the data source.
4.  Click **Import**.

| Dashboard | ID | Purpose |
|-----------|-----|---------|
| Node Exporter Full | `1860` | VM CPU/RAM/Disk/Network metrics |
| Loki Logs | `13639` | Log exploration dashboard |
| Prometheus Stats | `2` | Prometheus internal metrics |

## 13. Keycloak SSO Integration

Once Keycloak is set up (see `docs/10-sso-keycloak-setup.md`), configure Grafana to authenticate via Keycloak.

### Key Concepts Before You Start

- **`client_id = grafana`** — This is the **name of the application registration** in Keycloak, created in `docs/10-sso-keycloak-setup.md` (section 11.2). It identifies *Grafana the application* to Keycloak, and has **nothing to do with Keycloak users**.
- **`client_secret`** — A secret token **auto-generated by Keycloak** when you create the `grafana` client. It is Keycloak's way of verifying that requests come from your Grafana instance and not an impersonator. You retrieve it from the Keycloak Admin Console.

### 13.1. Get the Client Secret from Keycloak

1. Log in to the Keycloak Admin Console (`https://keycloak.salad.local/admin`).
2. Select the **salad** realm.
3. Go to **Clients** → click on **grafana**.
4. Click the **Credentials** tab.
5. Copy the value under **Client secret** — this is what goes into `client_secret` below.

### 13.2. Configure grafana.ini

Edit `/etc/grafana/grafana.ini`:

```ini
[auth.generic_oauth]
enabled = true
name = Salad SSO
allow_sign_up = true
; client_id is the application name registered in Keycloak (not a user)
client_id = grafana
; client_secret is copied from: Keycloak Admin → salad realm → Clients → grafana → Credentials tab
client_secret = <paste_secret_from_keycloak_here>
scopes = openid profile email
auth_url = https://keycloak.salad.local/realms/salad/protocol/openid-connect/auth
token_url = https://keycloak.salad.local/realms/salad/protocol/openid-connect/token
api_url = https://keycloak.salad.local/realms/salad/protocol/openid-connect/userinfo
; Maps Keycloak roles (from LDAP groups) to Grafana roles
role_attribute_path = contains(roles[*], 'grafana-admin') && 'Admin' || contains(roles[*], 'grafana-user') && 'Viewer'
```

Restart Grafana:
```bash
systemctl restart grafana-server
```


## 14. Verification

From your host machine:

1.  **Prometheus UI** (if exposed or via SSH tunnel):
    ```bash
    curl http://192.168.123.31:9090/-/ready
    ```

2.  **Check targets** (all VMs should show as "UP"):
    ```bash
    curl http://192.168.123.31:9090/api/v1/targets | python3 -m json.tool | grep '"health"'
    ```

3.  **Loki health check**:
    ```bash
    curl http://192.168.123.31:3100/ready
    ```

4.  **Grafana UI**:
    Open `https://grafana.salad.local` in your browser and log in.
