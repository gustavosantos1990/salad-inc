# 02 - Provisioning Core DNS

In this step, we provision the **DNS** VM (`dns.salad.com`). This separate Alpine Linux instance runs `dnsmasq` to resolve internal domains for the infrastructure.

## 1. VM Directory Structure

Create the directory for the DNS VM:
```bash
mkdir -p infrastructure/vms/dns
```

## 2. Create Virtual Disk

We create a smaller 2GB disk image since this VM only does DNS.

Create the disk:
```bash
qemu-img create -f qcow2 infrastructure/vms/dns/dns.qcow2 2G
```

## 3. Provisioning

Install using `virt-install` with the Alpine ISO:

```bash
sudo virt-install \
  --name dns \
  --ram 256 \
  --vcpus 1 \
  --network network=salad-net,mac=52:54:00:00:00:10 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/dns/dns.qcow2,size=2 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/alpine-virt-3.19.1-x86_64.iso \
  --os-variant alpinelinux3.18 \
  --graphics spice \
  --noautoconsole
```

## 4. Connect to Console

Since `setup-alpine` requires keyboard interaction, you need to open the VM's graphical console.

**Option A: Using Virt-Viewer (Recommended)**
This launches a window directly to the VM's display:
```bash
virt-viewer --connect qemu:///system --wait dns
```

**Option B: Using Virt-Manager (GUI)**
1. Open the "Virtual Machine Manager" app.
2. Double-click the `dns` VM.

**Option C: Manual Port Finding**
If you need to connect from another tool, find the port:
```bash
sudo virsh domdisplay dns
```
*(Usually returns `spice://127.0.0.1:5900`)*

## 5. Install Alpine

Follow the `setup-alpine` prompts to install Alpine Linux. Here are the recommended settings:

1. **Keyboard Layout:** `br` (Brazilian)
2. **Keyboard Variant:** `br` (or press Enter for default)
3. **Hostname:** `dns.salad.com`
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

Once you have finished `setup-alpine` and rebooted into the fresh system, log in as root and run these commands to match the planned configuration:

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

    Ensure SSHd is running (installed by default):
    ```bash
    rc-update add sshd default
    ```
    ```bash
    service sshd start

1.  **Edit SSH Config: (Alternative??)**
    ```bash
    vi /etc/ssh/sshd_config
    ```

    Update
    ```text
    PermitRootLogin yes
    ```
    

    Restart SSH
    ```bash
    rc-service sshd restart
    ```
    



2.  **Install Packages:**

    Enable Community Repo (If not done):
    ```bash
    vi /etc/apk/repositories
    ```

    Uncomment community line
    ```bash
    apk update
    ```
    
    Install `dnsmasq`, `vim`, `curl` and `prometheus-node-exporter`:
    ```bash
    apk add doas nano dnsmasq curl prometheus-node-exporter
    ```

3.  **Disable Swap (if enabled):**

    Turn off swap:
    ```bash
    swapoff -a
    ```
    ```bash
    sed -i '/swap/d' /etc/fstab
    ```
    ```bash
    free -h  # Verify swap is 0
    ```

4.  **Configure Dnsmasq:**
    ```bash
    cat > /etc/dnsmasq.conf <<EOF
    # Salad Inc Internal DNS
    domain-needed
    bogus-priv
    no-resolv
    server=8.8.8.8
    local=/salad.com/
    domain=salad.com
    expand-hosts
    filter-AAAA
    
    # Static Records
    
    # Core Infrastructure
    address=/dns.salad.com/192.168.123.10
    address=/proxy.salad.com/192.168.123.30
    
    # Services (→ Nginx for SSL termination)
    address=/gitlab.salad.com/192.168.123.30
    address=/nexus.salad.com/192.168.123.30
    address=/lam.salad.com/192.168.123.30
    address=/grafana.salad.com/192.168.123.30
    address=/keycloak.salad.com/192.168.123.30
    address=/portainer.salad.com/192.168.123.30
    
    # Direct VM Access (for SSH, troubleshooting)
    address=/repository.salad.com/192.168.123.20
    address=/database.salad.com/192.168.123.21
    address=/containerization.salad.com/192.168.123.22
    address=/artifact-repository.salad.com/192.168.123.23
    address=/monitor.salad.com/192.168.123.31
    address=/sso.salad.com/192.168.123.41
    address=/mail.salad.com/192.168.123.45
    address=/ldap.salad.com/192.168.123.46
    
    # Wildcard for dynamic apps (→ Nginx → Traefik)
    address=/.app.salad.com/192.168.123.30
    
    # Logging (optional, useful for troubleshooting)
    log-queries
    EOF
    ```

    **Note:** The `filter-AAAA` option tells dnsmasq to filter out IPv6 (AAAA) DNS queries for domains that don't have IPv6 addresses. This prevents BusyBox ping (used in Alpine Linux) from failing with "bad address" errors when it receives NXDOMAIN for IPv6 queries.

5.  **Fix /etc/hosts Conflict:**

    During Alpine setup, the system automatically adds the hostname to `/etc/hosts` pointing to `127.0.0.1`. This causes `dns.salad.com` to resolve to `127.0.0.1` instead of `192.168.123.10`, overriding the dnsmasq configuration.

    Edit `/etc/hosts`:
    ```bash
    vi /etc/hosts
    ```

    Change the first line from:
    ```
    127.0.0.1   dns.salad.com dns localhost.localdomain localhost
    ```

    To:
    ```
    127.0.0.1   localhost localhost.localdomain
    ```

    This ensures that `dns.salad.com` is resolved by dnsmasq (to `192.168.123.10`) rather than by the local hosts file.

6.  **Enable and Start Services:**
    
    Enable Dnsmasq:
    ```bash
    rc-update add dnsmasq default
    ```
    
    Start Dnsmasq:
    ```bash
    service dnsmasq start
    ```

    Enable Node Exporter:
    ```bash
    rc-update add node-exporter default
    ```
    
    Start Node Exporter:

    ```bash
    rc-service node-exporter start
    ```

    Verify dnsmasq it's listening, should show listening on port 53 (DNS):

    ```bash
    netstat -tulnp | grep dnsmasq
    ```

    Test name resolution:

    ```bash
    nslookup containerization.salad.com 127.0.0.1
    ```

    Verify Node Exporter (Should listen on :::9100):
    
    ```bash
    netstat -tulnp | grep 9100
    ```