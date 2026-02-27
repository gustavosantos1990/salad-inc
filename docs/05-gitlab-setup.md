# 04 - Provisioning GitLab (Repository & CI/CD)

We will provision the **Repository** VM (`repository.salad.local`). This is a **Debian 12** instance running **GitLab Omnibus**.

## 1. Strategy: Separate OS and Data

You suggested using **two separate disks**, which is an excellent architectural decision for a data-heavy application like GitLab.

*   **Disk 1 (`vda` - 10 GB):** The Operating System (Debian). If the OS breaks or we need to upgrade Debian versions, we can replace this disk without touching our data.
*   **Disk 2 (`vdb` - 20 GB):** The Data Partition. We will mount this at `/var/opt/gitlab` so that all repositories, databases, and artifacts live here.

## 2. Directory & Disks

Create the directory and both disk images:

```bash
mkdir -p infrastructure/vms/repository
```

Create OS Disk (10G):
```bash
qemu-img create -f qcow2 infrastructure/vms/repository/repository-os.qcow2 10G
```

Create Data Disk (20G):
```bash
qemu-img create -f qcow2 infrastructure/vms/repository/repository-data.qcow2 20G
```

## 3. Install Debian (ISO)

Since we are using the official ISO, we trigger a manual install. Note the `--disk` arguments attaching both drives.

```bash
sudo virt-install \
  --name repository \
  --ram 6144 \
  --vcpus 2 \
  --network network=salad-net,mac=52:54:00:00:00:20 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/repository/repository-os.qcow2,size=10 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/repository/repository-data.qcow2,size=20 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/debian-12.12.0-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics spice \
  --noautoconsole
```

### Installation Pointers
Connect via `virt-viewer` or Virt-Manager.
1.  **Hostname:** `repository.salad.local`
2.  **Domain:** `salad.local`
3.  **Root Password:** (Set securely)
4.  **User:** `salad-admin`
5.  **Partitioning:**
    *   Select **Manual**.
    *   **vda (10 GB):**
        *   `primary` partition, all space.
        *   Mount point: `/` (Root).
        *   Bootable: `yes`.
    *   **vdb (20 GB):**
        *   You can leave this **empty/unconfigured** for now. We will format and mount it manually post-install to be precise.
6.  **Software Selection:**
    *   [X] SSH server
    *   [X] Standard system utilities
    *   [ ] (Uncheck everything else, e.g., Desktop environment)

## 4. Post-Install Configuration

Login as `root` (or `sudo -i`).

### 1. Network & SSH Check
Verify you have the correct IP:
```bash
ip addr
# Should be 192.168.123.20 given strict MAC binding in salad-net
```

Configure SSH keys (Manual Step):
```bash
mkdir -p /root/.ssh && \
chmod 700 /root/.ssh && \
echo "<your-ssh-public-key>" > /root/.ssh/authorized_keys && \
chmod 600 /root/.ssh/authorized_keys
```

### 2. Configure Data Disk (vdb)
We need to format the 2nd disk and mount it for GitLab.

Identify the disk:
```bash
lsblk
# Should see vdb with 20G
```

Format as ext4:
```bash
mkfs.ext4 /dev/vdb
```

Create mount point:
```bash
mkdir -p /var/opt/gitlab
```

Mount and persist in `/etc/fstab`:
```bash
echo "/dev/vdb /var/opt/gitlab ext4 defaults 0 2" >> /etc/fstab
```

Mount it now:
```bash
mount -a
```

Verify:
```bash
df -h | grep gitlab
```

### 3. System Prep
Install dependencies:
```bash
apt-get update && apt-get install -y curl openssh-server ca-certificates perl net-tools dnsutils
```

*Note: If you missed checking "SSH server" during installation, the command above installs it (`openssh-server`). Ensure it is running with `systemctl enable --now ssh`.*

**Disable Swap** (Project Constraint):
```bash
swapoff -a
sed -i '/swap/d' /etc/fstab
```

**Configure DNS Resolution**:
Edit `/etc/resolv.conf` to use our custom DNS VM:
```bash
echo "nameserver 192.168.123.10" > /etc/resolv.conf
```
*(Long term: Configure systemd-resolved explicitly)*

**Fix /etc/hosts Conflict**:

During Debian installation, the system automatically adds the hostname to `/etc/hosts` pointing to `127.0.1.1`. This causes `repository.salad.local` to resolve to a loopback address instead of `192.168.123.20`, breaking communication with other VMs and the proxy.

Edit `/etc/hosts`:
```bash
vi /etc/hosts
```

Change from:
```
127.0.0.1       localhost
127.0.1.1       repository.salad.local  repository
```

To:
```
127.0.0.1       localhost
```

Remove the `127.0.1.1` line entirely. The hostname `repository.salad.local` should be resolved via DNS (which points to `192.168.123.20`).

Test DNS resolution:
```bash
nslookup repository.salad.local
```

Should return `192.168.123.20`, not `127.0.1.1`.

### 4. Enable Serial Console (Optional but Recommended)
To allow `virsh console repository` to work, enable the serial service:
```bash
systemctl enable --now serial-getty@ttyS0.service
```

### 5. Install GitLab CE
Add the repository:
```bash
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | bash
```

Install specifying the URL. This might take a while!
```bash
EXTERNAL_URL="http://repository.salad.local" apt-get install gitlab-ce
```

### 5. Optimize Configuration
To solve high CPU/RAM usage (`kswapd0`), we reduce unicorn workers and disable the bundled monitoring stack (since we have a separate node-exporter).

Edit the config:
```bash
vi /etc/gitlab/gitlab.rb
```

Add/Change these settings:
```ruby
# Reduce memory footprint
puma['worker_processes'] = 2
sidekiq['concurrency'] = 10
postgresql['shared_buffers'] = "256MB"

# Disable bundled monitoring (we use external Monitor VM)
prometheus_monitoring['enable'] = false
alertmanager['enable'] = false
grafana['enable'] = false
```

Apply changes (Takes time):
```bash
gitlab-ctl reconfigure
```

### 6. Install Node Exporter
```bash
apt-get install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
```

### 7. Configure SSL Certificate Trust (Required for SSO)

When integrating GitLab with Keycloak (or other external services via HTTPS), GitLab needs to trust the self-signed SSL certificate used by the Nginx proxy.

**Why this is needed:** GitLab makes backend HTTPS requests to Keycloak at `https://keycloak.salad.local`. Since we use a self-signed certificate on the Nginx proxy, GitLab will reject the connection unless we explicitly trust this certificate.

**Step 1:** Get the SSL certificate from the proxy VM

From the **proxy VM**, display the certificate:
```bash
cat /etc/nginx/ssl/salad.local.crt
```

**Step 2:** Add the certificate to GitLab's trusted certificates

On the **GitLab VM** (repository), create the certificate file:

```bash
cat > /etc/gitlab/trusted-certs/salad.local.crt << 'EOF'
-----BEGIN CERTIFICATE-----
MIIDkzCCAnugAwIBAgIUXPH51PKg4CXISj08Ua6arzL412swDQYJKoZIhvcNAQEL
BQAwWTELMAkGA1UEBhMCQlIxCzAJBgNVBAgMAlNQMREwDwYDVQQHDAhTYW9QYXVs
bzESMBAGA1UECgwJU2FsYWQgSW5jMRYwFAYDVQQDDA0qLnNhbGFkLmxvY2FsMB4X
DTI2MDEyMjIxMDMxMVoXDTI3MDEyMjIxMDMxMVowWTELMAkGA1UEBhMCQlIxCzAJ
BgNVBAgMAlNQMREwDwYDVQQHDAhTYW9QYXVsbzESMBAGA1UECgwJU2FsYWQgSW5j
MRYwFAYDVQQDDA0qLnNhbGFkLmxvY2FsMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
MIIBCgKCAQEAtVttYm602X9cCGsnNpzeX7Fn0ZLT22UW8JxL09zeYqwSTFqlR6WP
6GOsU3oIjtb62kgTQPoCuiq2uMeo0ZNQfwf0CfyKDYvJjtStLSsKLppCbxdf5EGs
l1VtdPgBWZmjKN5ZdOWbxvkMZkRzFAupgEUflrVsalNxeFpvJnE9z7MAA6/HxHom
eXDw5Wf00hjZu3AUCej423BWoMWo5dm/FQUISLUG5eCmDaQQ8H9+NMOvSY2W27+O
n8uIujT/h9S6elZws80ESMapF7bGbvo2p5Sb3AJrpGUfr4uGCxJEm1ibeH7kjzKH
RlD5bKgxpSm0SSZSsdp2KDPp6qOVCkLHMQIDAQABo1MwUTAdBgNVHQ4EFgQUW6Rn
JTjldQ31k9kZmpgwrx9+5qowHwYDVR0jBBgwFoAUW6RnJTjldQ31k9kZmpgwrx9+
5qowDwYDVR0TAQH/BAUwAwEB/zANBgkqhkiG9w0BAQsFAAOCAQEAUZ0I31k3P0u+
zloQ/H55Fn3MACkrnYrdiJA/pUeabCCX2uGIYUzKGO3A0vH/sQck0DIaDi0cIn5n
5NadGS/1gkUrTckV3KWzy/sV7cui8UT7B5oXviZVTirUplc0Eb/6N185U2M7CSIw
RYL1WODzxxeD/nDc2Kld2hnA9q+ZVnCh2PmRD6/w323yT5cAojpifYgzCzf+O+CT
qvtLy2f17fGDL7sd+CjhDIjOOKYhrOI8fu7ATWPbjW80KIr1LqgwGlnBdwXYWoew
GEe80t0sAAXVn09QdQy8yyylirP38wpsois3F/ZbuAlwuQbr5//3gEiMUisi2X3Q
rck3qpdm7A==
-----END CERTIFICATE-----
EOF
```

**Note:** Replace the certificate content above with your actual certificate from `/etc/nginx/ssl/salad.local.crt` on the proxy VM.

**Step 3:** Set proper permissions

```bash
chmod 644 /etc/gitlab/trusted-certs/salad.local.crt
```

**Step 4:** Reconfigure GitLab to pick up the certificate

```bash
gitlab-ctl reconfigure
```

**Verification:**

After reconfiguration, GitLab will trust the self-signed certificate. You can verify by checking:

```bash
# Check if the certificate is in the trusted store
ls -la /opt/gitlab/embedded/ssl/certs/ | grep salad
```

This step is **critical** for SSO integration with Keycloak to work properly. Without it, you'll see errors like:
```
Could not authenticate you from OpenIDConnect because "Ssl connect returned=1 errno=0 peeraddr=192.168.123.30:443 state=error: certificate verify failed (self-signed certificate)"
```

## 5. Verification
From your **Host Machine**:

1.  **Ping:**
    ```bash
    ping -c 2 repository.salad.local
    ```
2.  **Access Web Interface:**
    Open `http://repository.salad.local` in your browser (if you have one configured) or `curl` it:
    ```bash
    curl -I http://repository.salad.local
    ```

    *First load might be slow (502 Gateway) while Unicorn/Puma starts.*

3.  **Get Initial Root Password:**
    Inside the VM:
    ```bash
    cat /etc/gitlab/initial_root_password
    ```

3.  **Check Internal Services:**
    Ensure all components (Puma, Redis, Postgres) are up:
    ```bash
    gitlab-ctl status
    ```
