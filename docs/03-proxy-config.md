# 03 - Provisioning Nginx Proxy

In this step, we provision the **Proxy** VM (`proxy.salad.com`). This Alpine Linux instance runs Nginx as an edge gateway, handling SSL termination and reverse proxying to backend services.

## 1. VM Directory Structure

Create the directory for the Proxy VM:
```bash
mkdir -p infrastructure/vms/proxy
```

## 2. Create Virtual Disk

Create a 5GB disk image for the proxy VM:

```bash
qemu-img create -f qcow2 infrastructure/vms/proxy/proxy.qcow2 5G
```

## 3. Provisioning

Install using `virt-install` with the Alpine ISO:

```bash
sudo virt-install \
  --name proxy \
  --ram 512 \
  --vcpus 1 \
  --network network=salad-net,mac=52:54:00:00:00:30 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/proxy/proxy.qcow2,size=5 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/alpine-virt-3.19.1-x86_64.iso \
  --os-variant alpinelinux3.18 \
  --graphics spice \
  --noautoconsole
```

## 4. Connect to Console

Open the VM's graphical console:

```bash
virt-viewer --connect qemu:///system --wait proxy
```

## 5. Install Alpine

Follow the `setup-alpine` prompts:

1. **Keyboard Layout:** `br` (Brazilian)
2. **Keyboard Variant:** `br` (or press Enter for default)
3. **Hostname:** `proxy.salad.com`
4. **Interface:** `eth0`
5. **IP Address:** `dhcp`
6. **Do you want to do any manual network configuration?** `no`
7. **Timezone:** `America` then `Sao_Paulo`
8. **HTTP/FTP Proxy:** `none`
9. **APK Mirror:** `36`
10. **Create extra user:** `salad-maker`
11. **Enter ssh key or URL for salad-maker:** `<your-ssh-public-key>`
12. **SSH server:** `openssh`
13. **Disk to install:** `vda`
14. **How would you like to use it?** `sys`
15. `reboot`

**Note:** If you need to change the keyboard layout after installation, run:
```bash
setup-keymap
```
Then select `br` and `br` again.

## 6. Manual Configuration (Post-Install)

Once you have finished `setup-alpine` and rebooted, log in as root and run these commands:

1.  **Configure SSH Access:**
    
    Create the `.ssh` directory:
    ```bash
    mkdir -p /root/.ssh
    ```

    Set directory permissions:
    ```bash
    chmod 700 /root/.ssh
    ```

    Add the public key:
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

2.  **Install Packages:**

    Enable Community Repo:
    ```bash
    vi /etc/apk/repositories
    ```

    Uncomment community line, then update:
    ```bash
    apk update
    ```
    
    Install Nginx, OpenSSL, and monitoring tools:
    ```bash
    apk add nginx openssl curl prometheus-node-exporter
    ```

3.  **Disable Swap (if enabled):**

    Turn off swap:
    ```bash
    swapoff -a
    sed -i '/swap/d' /etc/fstab
    free -h
    ```

4.  **Fix /etc/hosts Conflict:**

    Edit `/etc/hosts`:
    ```bash
    vi /etc/hosts
    ```

    Change the first line from:
    ```
    127.0.0.1   proxy.salad.com proxy localhost.localdomain localhost
    ```

    To:
    ```
    127.0.0.1   localhost localhost.localdomain
    ```

    This ensures that `proxy.salad.com` is resolved by DNS (to `192.168.123.30`) rather than by the local hosts file.

5.  **Generate Self-Signed SSL Certificate:**

    Create SSL directory:
    ```bash
    mkdir -p /etc/nginx/ssl
    ```

    Generate wildcard certificate for `*.salad.com` and `*.app.salad.com` with Subject Alternative Name (SAN) and CA extensions:
    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout /etc/nginx/ssl/salad.com.key \
      -out /etc/nginx/ssl/salad.com.crt \
      -subj "/C=BR/ST=SP/L=SaoPaulo/O=Salad Inc/CN=*.salad.com" \
      -addext "subjectAltName = DNS:*.salad.com,DNS:salad.com,DNS:*.app.salad.com" \
      -addext "basicConstraints=critical,CA:TRUE"
    ```

    Set permissions:
    ```bash
    chmod 600 /etc/nginx/ssl/salad.com.key
    chmod 644 /etc/nginx/ssl/salad.com.crt
    ```

6.  **Configure Nginx:**

    Create the main Nginx configuration:
    ```bash
    cat > /etc/nginx/nginx.conf <<'EOF'
    user nginx;
    worker_processes auto;
    error_log /var/log/nginx/error.log warn;
    pid /run/nginx/nginx.pid;

    events {
        worker_connections 1024;
    }

    http {
        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

        access_log /var/log/nginx/access.log main;

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        keepalive_timeout 65;
        types_hash_max_size 2048;

        # SSL Settings
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # Include virtual host configs
        include /etc/nginx/conf.d/*.conf;
    }
    EOF
    ```

    Create configuration directory:
    ```bash
    mkdir -p /etc/nginx/conf.d
    ```

    Create a default server configuration:
    ```bash
    cat > /etc/nginx/conf.d/default.conf <<'EOF'
    # Default server - catch-all for undefined hosts
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;
        
        # Redirect all HTTP to HTTPS
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl default_server;
        listen [::]:443 ssl default_server;
        server_name _;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        location / {
            return 404;
        }
    }
    EOF
    ```

    Create reverse proxy configurations for services (we'll add more as we provision VMs):
    ```bash
    cat > /etc/nginx/conf.d/services.conf <<'EOF'
    # DNS resolver - use the DNS VM
    # ipv6=off prevents startup crashes/delays in IPv4-only environment
    resolver 192.168.123.10 valid=30s ipv6=off;

    # GitLab
    server {
        listen 80;
        server_name gitlab.salad.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name gitlab.salad.com;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        location / {
            # Use variable to force runtime DNS resolution
            # This allows Nginx to start even if backend VM is down
            set $backend_gitlab repository.salad.com:80;
            
            proxy_pass http://$backend_gitlab;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Ssl on;
            
            # WebSocket support for GitLab
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            
            # Timeouts for long-running operations
            proxy_connect_timeout 300;
            proxy_send_timeout 300;
            proxy_read_timeout 300;
        }
    }

    # Artifactory
    server {
        listen 80;
        server_name nexus.salad.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name nexus.salad.com;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        client_max_body_size 0;
        chunked_transfer_encoding on;

        location / {
            set $backend_artifactory artifact-repository.salad.com:8082;
            proxy_pass http://$backend_artifactory;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # Grafana
    server {
        listen 80;
        server_name grafana.salad.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name grafana.salad.com;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        location / {
            set $backend_grafana monitor.salad.com:3000;
            proxy_pass http://$backend_grafana;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # Keycloak
    server {
        listen 80;
        server_name keycloak.salad.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name keycloak.salad.com;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        location / {
            set $backend_keycloak sso.salad.com:8080;
            proxy_pass http://$backend_keycloak;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            proxy_buffer_size 128k;
            proxy_buffers 4 256k;
            proxy_busy_buffers_size 256k;
        }
    }

    # LDAP Account Manager
    server {
        listen 80;
        server_name lam.salad.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name lam.salad.com;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        # Redirect root to /lam to avoid confusing relative redirects
        location = / {
            return 301 https://lam.salad.com/lam/;
        }

        location / {
            # Construct full URL to preserve path structure
            set $target_url http://ldap.salad.com$request_uri;
            proxy_pass $target_url;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # pgAdmin 4
    server {
        listen 80;
        server_name pgadmin.salad.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name pgadmin.salad.com;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        location / {
            set $backend_pgadmin containerization.salad.com:5050;
            proxy_pass http://$backend_pgadmin;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # Portainer
    server {
        listen 80;
        server_name portainer.salad.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name portainer.salad.com;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        location / {
            set $backend_portainer containerization.salad.com:9000;
            proxy_pass http://$backend_portainer;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }

    # Traefik Dashboard
    server {
        listen 80;
        server_name traefik.salad.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name traefik.salad.com;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        location / {
            set $backend_traefik containerization.salad.com:8080;
            proxy_pass http://$backend_traefik;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # WebSocket support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }

    # Wildcard for dynamic applications (Traefik)
    # Traefik publishes port 8090:80 — Nginx forwards here and passes Host header
    # so Traefik can match the correct service router rule.
    server {
        listen 80;
        server_name *.app.salad.com;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name *.app.salad.com;

        ssl_certificate /etc/nginx/ssl/salad.com.crt;
        ssl_certificate_key /etc/nginx/ssl/salad.com.key;

        location / {
            set $upstream http://containerization.salad.com:8090;
            proxy_pass         $upstream;
            proxy_set_header   Host              $host;     # Traefik routes by this header
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
            proxy_read_timeout    60s;
            proxy_connect_timeout 10s;
        }
    }
    EOF
    ```

7.  **Enable and Start Services:**
    
    Enable Nginx:
    ```bash
    rc-update add nginx default
    ```
    
    Start Nginx:
    ```bash
    service nginx start
    ```

    Enable Node Exporter:
    ```bash
    rc-update add node-exporter default
    ```
    
    Start Node Exporter:
    ```bash
    rc-service node-exporter start
    ```

    Test Nginx configuration:
    ```bash
    nginx -t
    ```

    Verify Nginx is listening (should show ports 80 and 443):
    ```bash
    netstat -tulnp | grep nginx
    ```

    Verify Node Exporter (should listen on :::9100):
    ```bash
    netstat -tulnp | grep 9100
    ```

## 7. Testing

From your host machine, test the proxy:

Test HTTP to HTTPS redirect:
```bash
curl -I http://proxy.salad.com
```

Test SSL certificate (ignore self-signed warning):
```bash
curl -k -I https://proxy.salad.com
```

Test DNS resolution for service names:
```bash
ping -c 1 gitlab.salad.com
```

## 8. Install SSL Certificate in Host Browser

To avoid browser security warnings when accessing `*.salad.com` services, install the self-signed certificate as a trusted Certificate Authority.

### Copy Certificate from Proxy VM

From your host machine, copy the certificate (using the SSH key from infrastructure):

```bash
mkdir -p infrastructure/ssl
```
```bash
scp -i infrastructure/ssh/salad_key root@192.168.123.30:/etc/nginx/ssl/salad.com.crt infrastructure/ssl/
```

**Note:** If you don't want to specify `-i` every time, you can create an SSH config file:

```bash
cat >> ~/.ssh/config <<'EOF'
Host *.salad.com 192.168.123.*
    User root
    IdentityFile ~/workspace/salad.inc/infrastructure/ssh/salad_key
    StrictHostKeyChecking accept-new
EOF
```

Then you can use SCP without the `-i` flag:
```bash
scp root@192.168.123.30:/etc/nginx/ssl/salad.com.crt infrastructure/ssl/
```

### Install Certificate in Browser

#### For Firefox:

1. Open Firefox and go to **Settings** → **Privacy & Security**
2. Scroll down to **Certificates** → Click **View Certificates**
3. Go to the **Authorities** tab
4. Click **Import...**
5. Navigate to `~/workspace/salad.inc/infrastructure/ssl/salad.com.crt`
6. Check **Trust this CA to identify websites**
7. Click **OK**

#### For Chrome/Chromium:

1. Open Chrome and go to **Settings** → **Privacy and security** → **Security**
2. Scroll down to **Manage certificates**
3. Go to the **Authorities** tab
4. Click **Import**
5. Navigate to `~/workspace/salad.inc/infrastructure/ssl/salad.com.crt`
6. Check **Trust this certificate for identifying websites**
7. Click **OK**

#### For System-Wide Trust (Linux):

Install the certificate system-wide so all applications trust it:

```bash
sudo cp infrastructure/ssl/salad.com.crt /usr/local/share/ca-certificates/salad.com.crt
```
```bash
sudo update-ca-certificates
```

Verify the certificate was added (a symlink should exist):
```bash
ls -l /etc/ssl/certs/salad.com.pem
```

### Verify Certificate Trust

After installing the certificate, test in your browser:

1. Navigate to `https://proxy.salad.com`
2. You should see a **secure connection** (lock icon) without warnings
3. Click the lock icon to verify the certificate details

You can also test with curl (without `-k` flag):
```bash
curl -I https://proxy.salad.com
```

This should work without the `--insecure` flag if the system-wide trust was configured.

## 9. Next Steps

- Provision backend VMs (GitLab, Artifactory, etc.)
- Update Nginx configurations as new services come online
- Configure monitoring to scrape metrics from this VM
- Set up log forwarding to centralized logging system
