# 08 - Provisioning Containerization Host (Docker Swarm + Traefik + Portainer)

We will provision the **Containerization** VM (`containerization.salad.com`). This is a **Debian 12** instance that serves as our primary Docker Swarm application node. It will run:
*   **Docker Swarm**: For orchestration.
*   **Traefik**: As the ingress controller/router for Swarm services.
*   **Portainer**: For easy container management via UI.
*   **GitLab Runners**: To execute CI/CD jobs (installed in a later step).

## 1. VM Directory Structure

Create the directory for the VM:

```bash
mkdir -p infrastructure/vms/containerization
```

## 2. Create Virtual Disks

Create OS Disk (10G):
```bash
qemu-img create -f qcow2 infrastructure/vms/containerization/containerization-os.qcow2 10G
```

Create Docker Data Disk (30G):
```bash
qemu-img create -f qcow2 infrastructure/vms/containerization/containerization-data.qcow2 30G
```

## 3. Install Debian (ISO)

```bash
sudo virt-install \
  --name containerization \
  --ram 2048 \
  --vcpus 2 \
  --network network=salad-net,mac=52:54:00:00:00:22 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/containerization/containerization-os.qcow2,size=10 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/containerization/containerization-data.qcow2,size=30 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/debian-12.12.0-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics spice \
  --noautoconsole
```

### 3.1. Connect to Console

Since `noautoconsole` is set, use this to open the display:

```bash
virt-viewer --connect qemu:///system --wait containerization
```

### Installation Pointers
1.  **Hostname:** `containerization.salad.com`
2.  **Domain:** `salad.com`
3.  **Partitioning:** Standard / (Root).
4.  **Software Selection:**
    *   [X] SSH server
    *   [X] Standard system utilities
    *   [ ] (Uncheck everything else)

## 4. Post-Install Configuration

Login as `root` (or `sudo -i`).

### 1. System Prep
```bash
apt-get update && apt-get install -y curl openssh-server net-tools dnsutils vim git
```

**Disable Swap** (Critical for Kube/Swarm performance):
```bash
swapoff -a
sed -i '/swap/d' /etc/fstab
```

**Configure DNS**:
```bash
echo "nameserver 192.168.123.10" > /etc/resolv.conf
```

**Fix /etc/hosts Conflict**:
*Debian defaults map the hostname to 127.0.1.1, which breaks Swarm advertising.*

Edit `/etc/hosts`:
```bash
vi /etc/hosts
```
Change from:
```
127.0.0.1       localhost
127.0.1.1       containerization.salad.com  containerization
```
To:
```
127.0.0.1       localhost
```
(Remove the 127.0.1.1 line entirely).

**Enable Serial Console**:
```bash
systemctl enable --now serial-getty@ttyS0.service
```

*To connect via serial console from host:*
```bash
sudo virsh console containerization
```
*(Press Enter to see the login prompt. Use `Ctrl+]` to exit).*

### 2. Configure Data Disk (Docker Storage)

We need to format the 2nd disk and mount it for Docker data storage **before** installing Docker.

Identify the disk:
```bash
lsblk
# Should see vdb with 30G
```

Format as ext4:
```bash
mkfs.ext4 /dev/vdb
```

Create mount point:
```bash
mkdir -p /var/lib/docker
```

Mount and persist in `/etc/fstab`:
```bash
echo "/dev/vdb /var/lib/docker ext4 defaults 0 2" >> /etc/fstab
```

Mount it now:
```bash
mount -a
```

Verify:
```bash
df -h | grep docker
```

### 3. Configure SSH Access (Host to VM)
```bash
mkdir -p /root/.ssh && \
chmod 700 /root/.ssh && \
echo "<your-ssh-public-key>" > /root/.ssh/authorized_keys && \
chmod 600 /root/.ssh/authorized_keys
```

### 4. Install Docker Engine

Add Docker's official GPG key:
```bash
apt-get update
apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
```

Set up the repository:
```bash
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker:
```bash
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 5. Initialize Docker Swarm

Initialize Swarm, advertising the correct IP:
```bash
docker swarm init --advertise-addr 192.168.123.22
```

Create the overlay network for public traffic (Traefik will use this):
```bash
docker network create --driver=overlay --attachable traefik-public
```

### 6. Install Node Exporter
```bash
apt-get install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
```

## 5. Deploy Traefik (Reverse Proxy)

We will deploy Traefik as a global Swarm service to handle routing for containers. It will sit behind our Nginx VM.

Create directory for stacks:
```bash
mkdir -p /opt/stacks/traefik
cd /opt/stacks/traefik
```

Create `traefik.yml`:

```yaml
version: '3.8'

services:
  traefik:
    image: traefik:v2.11       # v2.11 negotiates Docker API version correctly with Engine 26+
    command:
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedByDefault=false"
      - "--providers.docker.network=traefik-public"
      - "--entrypoints.web.address=:80"
      - "--api.dashboard=true"
      - "--api.insecure=true"
    ports:
      - "8090:80"    # Nginx forwards *.app.salad.com → containerization:8090
      - "8080:8080"  # Dashboard — Nginx forwards traefik.salad.com → containerization:8080
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - traefik-public
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  traefik-public:
    external: true
```

Deploy the stack:
```bash
docker stack deploy -c traefik.yml traefik
```

## 6. Deploy Portainer (Management UI)

Create directory:
```bash
mkdir -p /opt/stacks/portainer
cd /opt/stacks/portainer
```

Create `portainer.yml`. This configuration uses Traefik labels so we can access it via `portainer.salad.com`.

```yaml
version: '3.2'

services:
  agent:
    image: portainer/agent:2.21.4
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker/volumes:/var/lib/docker/volumes
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    image: portainer/portainer-ce:2.21.4
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    volumes:
      - portainer_data:/data
    networks:
      - agent_network
    ports:
      - "9000:9000"
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

networks:
  agent_network:
    driver: overlay
    attachable: true

volumes:
  portainer_data:
```

Deploy the stack:
```bash
docker stack deploy -c portainer.yml portainer
docker stack deploy -c portainer.yml portainer
```

## 7. Deploy pgAdmin 4 (Database Management)

Create directory:
```bash
mkdir -p /opt/stacks/pgadmin
cd /opt/stacks/pgadmin
```

Create `pgadmin.yml`:
```yaml
version: '3.8'

services:
  pgadmin:
    image: dpage/pgadmin4:latest
    extra_hosts:
      - "database.salad.com:192.168.123.21"
    environment:
      PGADMIN_DEFAULT_EMAIL: "admin@salad.com"
      PGADMIN_DEFAULT_PASSWORD: "admin"
    ports:
      - "5050:80"
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    networks:
      - traefik-public
    deploy:
      mode: replicated
      replicas: 1

networks:
  traefik-public:
    external: true

volumes:
  pgadmin_data:
```

Deploy the stack:
```bash
docker stack deploy -c pgadmin.yml pgadmin
```

## 8. Update Nginx Proxy (On Proxy VM)

For `portainer.salad.com` to work, we need to add a rule on the Nginx VM (`proxy.salad.com`).

1. SSH into Proxy VM (`192.168.123.30`).
2. Edit `/etc/nginx/conf.d/services.conf`.
3. Add the following block:

```nginx
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
        proxy_pass http://containerization.salad.com:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support (Required for Portainer console)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

4. Reload Nginx: `service nginx reload`

## 8. Verification

From your host machine:

1.  **Ping**: `ping -c 2 containerization.salad.com` (Should return 192.168.123.22).
2.  **Confirm services are running**:
    ```bash
    docker stack services portainer
    # portainer_agent       global       1/1   portainer/agent:2.21.4
    # portainer_portainer   replicated   1/1   portainer/portainer-ce:2.21.4
    ```
3.  **Connect Portainer to the Docker Swarm**: Open `https://portainer.salad.com`.
    - Create the admin user and log in.
    - The setup wizard will prompt you to add an environment. Choose:
      - **Environment type:** Docker Swarm
      - **Connection method:** Agent
      - **Name:** `salad-swarm` (or any label)
      - **Environment Address:** `tasks.portainer_agent:9001`
    - Click **Connect**. Portainer will reach the agent over the `agent_network` overlay, discover all Swarm nodes, and display the cluster dashboard.

> **Version note:** Portainer agent `2.21.4`+ is required for Docker Engine 26+. Versions ≤ 2.19.x use a Docker SDK that only speaks API 1.42, which is below the minimum (1.44) enforced by Docker Engine 26+. Always keep Portainer in sync with your Docker Engine version.
