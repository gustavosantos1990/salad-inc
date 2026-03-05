# Configure Host DNS Resolution

For `*.salad.com` domains to resolve correctly without interfering with your internet, we need **Split DNS**. The best way to achieve this on Linux is using `systemd-resolved`.

## 1. Enable systemd-resolved (If not already active)
Since you are on Debian, you might need to install/enable it:

Install and enable:
```bash
sudo apt update && sudo apt install -y systemd-resolved
sudo systemctl enable --now systemd-resolved
```

Ensure it manages your DNS (Standard setup):
```bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

## 2. Configure the Link
Now we tell the system to send `salad.com` queries to our virtual network.

Set the DNS server for the virtual bridge. We point this to our **DNS VM (192.168.123.10)** so the host machine uses our custom dnsmasq configuration (which includes all our service records) instead of the default Libvirt DNS.
```bash
sudo resolvectl dns virbr1 192.168.123.10
```

Route the domain to this interface:
```bash
sudo resolvectl domain virbr1 ~salad.com
```

### Making Configuration Persistent

The `resolvectl` commands above are **temporary** and will be lost on reboot. Since Debian Desktop uses `NetworkManager` instead of `systemd-networkd` by default, interface-specific configurations (`/etc/systemd/network/*.network`) won't be applied.

The most robust way to make this persistent is to use a global `systemd-resolved` drop-in configuration. This tells the system resolver purely at the DNS level without fighting your interface manager:

Create the drop-in directory:
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
```

Create the configuration file for the `.salad.com` domain routing:
```bash
sudo tee /etc/systemd/resolved.conf.d/salad.conf > /dev/null <<EOF
[Resolve]
DNS=192.168.123.10
Domains=~salad.com
EOF
```

Restart systemd-resolved to apply the configuration immediately:
```bash
sudo systemctl restart systemd-resolved
```

**Note:** This configuration will instruct `systemd-resolved` to forward any query for `*.salad.com` directly to `192.168.123.10`. The OS networking layer will automatically route these packets through `virbr1`.

### Verification

Check that the domain resolves to the IP defined in the XML (192.168.123.20):

```bash
ping -c 1 repository.salad.com
```
*(If successful, it will try to ping 192.168.123.20. It will fail to connect because the VM isn't up, but it proves the NAME resolution worked!)*

You can also verify the configuration is active:
```bash
resolvectl status virbr1
```

This should show:
- DNS Servers: 192.168.123.10
- DNS Domain: ~salad.com

