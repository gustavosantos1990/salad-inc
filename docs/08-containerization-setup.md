# 08 - Provisioning Containerization Host (Docker Swarm + Traefik + Portainer)

We will provision the **Containerization** VM (`containerization.salad.local`). This is a **Debian 12** instance that serves as our primary Docker Swarm application node. It will run:
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
1.  **Hostname:** `containerization.salad.local`
2.  **Domain:** `salad.local`
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
127.0.1.1       containerization.salad.local  containerization
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
virsh console containerization
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
    image: traefik:v2.10
    command:
      - "--providers.docker=true"
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      # Nginx terminates SSL, so Traefik just listens on HTTP
    ports:
      - "80:80"
      - "8080:8080" # Dashboard (internal access)
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    networks:
      - traefik-public
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager

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

Create `portainer.yml`. This configuration uses Traefik labels so we can access it via `portainer.salad.local`.

```yaml
version: '3.2'

services:
  agent:
    image: portainer/agent:2.19.4
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
    image: portainer/portainer-ce:2.19.4
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
      - "database.salad.local:192.168.123.21"
    environment:
      PGADMIN_DEFAULT_EMAIL: "admin@salad.local"
      PGADMIN_DEFAULT_PASSWORD: "admin"
    ports:
      - "5050:80"
    networks:
      - traefik-public
    deploy:
      mode: replicated
      replicas: 1

networks:
  traefik-public:
    external: true
```

Deploy the stack:
```bash
docker stack deploy -c pgadmin.yml pgadmin
```

## 8. Update Nginx Proxy (On Proxy VM)

For `portainer.salad.local` to work, we need to add a rule on the Nginx VM (`proxy.salad.local`).

1. SSH into Proxy VM (`192.168.123.30`).
2. Edit `/etc/nginx/conf.d/services.conf`.
3. Add the following block:

```nginx
# Portainer
server {
    listen 80;
    server_name portainer.salad.local;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name portainer.salad.local;

    ssl_certificate /etc/nginx/ssl/salad.local.crt;
    ssl_certificate_key /etc/nginx/ssl/salad.local.key;

    location / {
        proxy_pass http://containerization.salad.local:9000;
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

1.  **Ping**: `ping -c 2 containerization.salad.local` (Should return 192.168.123.22).
2.  **Access Portainer**: Open `https://portainer.salad.local`.
    *   Create the admin user.
    *   Connect to the "local" environment (via Agent).
