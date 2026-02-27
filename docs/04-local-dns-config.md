# Configure Host DNS Resolution

For `*.salad.local` domains to resolve correctly without interfering with your internet, we need **Split DNS**. The best way to achieve this on Linux is using `systemd-resolved`.

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
Now we tell the system to send `salad.local` queries to our virtual network.

Set the DNS server for the virtual bridge. We point this to our **DNS VM (192.168.123.10)** so the host machine uses our custom dnsmasq configuration (which includes all our service records) instead of the default Libvirt DNS.
```bash
sudo resolvectl dns virbr1 192.168.123.10
```

Route the domain to this interface:
```bash
sudo resolvectl domain virbr1 ~salad.local
```

### Making Configuration Persistent

The `resolvectl` commands above are **temporary** and will be lost on reboot. To make them persistent, create a systemd-resolved drop-in configuration file:

Create the directory:
```bash
sudo mkdir -p /etc/systemd/network
```

Create the configuration file for virbr1:
```bash
sudo tee /etc/systemd/network/virbr1.network > /dev/null <<EOF
[Match]
Name=virbr1

[Network]
DNS=192.168.123.10
Domains=~salad.local
EOF
```

Restart systemd-networkd to apply the configuration:
```bash
sudo systemctl restart systemd-networkd
```

**Note:** This configuration will automatically apply whenever the `virbr1` interface is brought up.

### Verification

Check that the domain resolves to the IP defined in the XML (192.168.123.20):

```bash
ping -c 1 repository.salad.local
```
*(If successful, it will try to ping 192.168.123.20. It will fail to connect because the VM isn't up, but it proves the NAME resolution worked!)*

You can also verify the configuration is active:
```bash
resolvectl status virbr1
```

This should show:
- DNS Servers: 192.168.123.10
- DNS Domain: ~salad.local

