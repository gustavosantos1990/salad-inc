# Troubleshooting Guide

This document contains common issues and their solutions for the Salad.Inc infrastructure.

---

## Virtual Machine Issues

### Failed to start domain: network is not active

**Problem:**
When trying to start a VM, you get an error like:
```
$ sudo virsh start dns
error: Failed to start domain 'dns'
error: Requested operation is not valid: network 'salad-net' is not active
```

**Cause:**
The virtual network that the VM is attached to is not currently running.

**Solution:**

1. Check the status of all networks:
   ```bash
   sudo virsh net-list --all
   ```

2. Start the inactive network:
   ```bash
   sudo virsh net-start salad-net
   ```

3. (Optional) Enable autostart to prevent this issue on reboot:
   ```bash
   sudo virsh net-autostart salad-net
   ```

4. Now start your VM:
   ```bash
   sudo virsh start dns
   ```

**Prevention:**
Always ensure that virtual networks have autostart enabled so they activate automatically when the host boots.

---

### Cannot resolve *.salad.com domains from host

**Problem:**
When trying to ping or access services using `.salad.com` domains from the host machine, you get:
```
$ ping dns.salad.com
ping: dns.salad.com: Temporary failure in name resolution
```

**Cause:**
The host machine is not configured to use the DNS VM (192.168.123.10) for resolving `*.salad.com` domains.

**Solution:**

1. Verify the DNS VM is running and get its IP:
   ```bash
   sudo virsh list
   sudo virsh domifaddr dns
   ```
   *(Should show 192.168.123.10)*

2. Configure systemd-resolved to use the DNS VM for the salad.com domain:
   ```bash
   sudo resolvectl dns virbr1 192.168.123.10
   sudo resolvectl domain virbr1 ~salad.com
   ```

3. Verify the configuration:
   ```bash
   resolvectl status virbr1
   ```
   *(Should show DNS Server: 192.168.123.10 and DNS Domain: ~salad.com)*

4. Test DNS resolution:
   ```bash
   ping -c 2 dns.salad.com
   ```

5. **Make it persistent** (important - the above commands are temporary):
   ```bash
   sudo mkdir -p /etc/systemd/network
   sudo tee /etc/systemd/network/virbr1.network > /dev/null <<EOF
   [Match]
   Name=virbr1
   
   [Network]
   DNS=192.168.123.10
   Domains=~salad.com
   EOF
   sudo systemctl restart systemd-networkd
   ```

**Prevention:**
Always create the persistent configuration file when setting up DNS to avoid losing the configuration on reboot.

**Note:** If DNS resolution stops working after restarting the `salad-net` network, you may need to restart the DNS VM to properly reconnect it to the bridge:
```bash
sudo virsh shutdown dns
sleep 5
sudo virsh start dns
```
Then reapply the DNS configuration:
```bash
sudo resolvectl dns virbr1 192.168.123.10
sudo resolvectl domain virbr1 ~salad.com
```

---

### BusyBox ping fails with "bad address" on Alpine Linux VMs

**Problem:**
When trying to ping a `.salad.com` domain from an Alpine Linux VM, you get:
```
proxy:~# ping dns.salad.com
ping: bad address 'dns.salad.com'
```

**Cause:**
BusyBox's `ping` implementation (used in Alpine Linux) queries for both IPv4 (A) and IPv6 (AAAA) DNS records. When it receives an NXDOMAIN response for the IPv6 query (because we only have IPv4 addresses configured), it treats the entire resolution as failed.

**Solution:**

**Option 1:** Force IPv4-only resolution (recommended):
```bash
ping -4 -c 2 dns.salad.com
```

**Option 2:** Use `getent` to verify DNS resolution is working:
```bash
getent hosts dns.salad.com
```

**Option 3:** Install full `iputils` package (if needed):
```bash
apk add iputils
```
Then use `/usr/bin/ping` instead of the BusyBox version.

**Verification:**
DNS resolution is actually working correctly (you can verify with `getent hosts` or `nslookup`). The issue is specific to BusyBox ping's behavior with mixed IPv4/IPv6 responses.

---

## DNS Issues

*To be documented as issues arise*

---

## Proxy Issues

*To be documented as issues arise*

---

## Container/Docker Issues

*To be documented as issues arise*
