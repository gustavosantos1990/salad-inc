# 01 - Network Configuration

This step establishes the virtual network infrastructure for Salad Inc. We will create a NAT-based network bridge (`virbr1`) using Libvirt/QEMU and configure the host system to resolve the internal `.salad.com` domain.

## 0. Prerequisite: Cleanup (Optional)

If you have a previous version of `salad-net` and want to start fresh, remove the existing network first:

Stop the network (ignore error if not running):
```bash
sudo virsh net-destroy salad-net
```

Remove the definition (ignore error if not defined):
```bash
sudo virsh net-undefine salad-net
```

## 1. Network Definition

We define a network named `salad-net` with the CIDR `192.168.123.0/24`. We use distinct static IPs for our infrastructure components to ensure predictable addressing.

**File:** `infrastructure/libvirt/salad-net.xml`

```xml
<network>
  <name>salad-net</name>
  <bridge name='virbr1' stp='on' delay='0'/>
  <domain name='salad.com'/>
  <forward mode='nat'/>
  <dns>
    <!-- Static DNS entries for infrastructure services -->
    <host ip='192.168.123.10'>
      <hostname>dns</hostname>
      <hostname>dns.salad.com</hostname>
    </host>
    <host ip='192.168.123.20'>
      <hostname>repository</hostname>
      <hostname>repository.salad.com</hostname>
    </host>
    <host ip='192.168.123.21'>
      <hostname>database</hostname>
      <hostname>database.salad.com</hostname>
    </host>
    <host ip='192.168.123.22'>
      <hostname>containerization</hostname>
      <hostname>containerization.salad.com</hostname>
    </host>
    <host ip='192.168.123.23'>
      <hostname>artifactory</hostname>
      <hostname>nexus.salad.com</hostname>
    </host>
    <host ip='192.168.123.30'>
      <hostname>proxy</hostname>
      <hostname>proxy.salad.com</hostname>
    </host>
    <host ip='192.168.123.31'>
      <hostname>monitor</hostname>
      <hostname>monitor.salad.com</hostname>
    </host>
    <host ip='192.168.123.41'>
      <hostname>sso</hostname>
      <hostname>sso.salad.com</hostname>
    </host>
  </dns>
  <ip address='192.168.123.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.123.100' end='192.168.123.254'/>
      <host mac='52:54:00:00:00:10' name='dns' ip='192.168.123.10'/>
      <host mac='52:54:00:00:00:20' name='repository' ip='192.168.123.20'/>
      <host mac='52:54:00:00:00:21' name='database' ip='192.168.123.21'/>
      <host mac='52:54:00:00:00:22' name='containerization' ip='192.168.123.22'/>
      <host mac='52:54:00:00:00:23' name='artifactory' ip='192.168.123.23'/>
      <host mac='52:54:00:00:00:30' name='proxy' ip='192.168.123.30'/>
      <host mac='52:54:00:00:00:31' name='monitor' ip='192.168.123.31'/>
      <host mac='52:54:00:00:00:41' name='sso' ip='192.168.123.41'/>
    </dhcp>
  </ip>
</network>
```

## 2. Implementation Steps

### Apply Network Configuration
Execute the following commands to define, start, and autostart the network:

Define the network from the XML file:
```bash
sudo virsh net-define infrastructure/libvirt/salad-net.xml
```

Start the network:
```bash
sudo virsh net-start salad-net
```

**Tip:** If you updated the XML (e.g. adding DNS entries), you must destroy/undefine and then redefine/start for changes to take effect.


Mark it to verify autostart (optional, but recommended):
```bash
sudo virsh net-autostart salad-net
```

Verify status:
```bash
sudo virsh net-list --all
```