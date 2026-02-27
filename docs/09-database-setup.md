# 09 - Provisioning Database Server (PostgreSQL)

We will provision the **Database** VM (`database.salad.local`). This is a **Debian 12** instance that serves as the central relational database for our applications (Keycloak, SonarQube, etc.).

## 1. VM Directory Structure

Create the directory for the VM:

```bash
mkdir -p infrastructure/vms/database
```

## 2. Create Virtual Disks

We will create two disks:
1.  **OS Disk:** 10GB for the base system.
2.  **Data Disk:** 20GB for PostgreSQL data (`/var/lib/postgresql`).

```bash
qemu-img create -f qcow2 infrastructure/vms/database/database-os.qcow2 10G
qemu-img create -f qcow2 infrastructure/vms/database/database-data.qcow2 20G
```

## 3. Install Debian (ISO)

```bash
sudo virt-install \
  --name database \
  --ram 2048 \
  --vcpus 2 \
  --network network=salad-net,mac=52:54:00:00:00:21 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/database/database-os.qcow2,size=10 \
  --disk path=/home/$USER/workspace/salad.inc/infrastructure/vms/database/database-data.qcow2,size=20 \
  --cdrom /home/$USER/workspace/salad.inc/infrastructure/iso/debian-12.12.0-amd64-netinst.iso \
  --os-variant debian12 \
  --graphics spice \
  --noautoconsole
```

### 3.1. Connect to Console

```bash
virt-viewer --connect qemu:///system --wait database
```

### Installation Pointers
1.  **Hostname:** `database.salad.local`
2.  **Domain:** `salad.local`
3.  **Partitioning:** Standard / (Root). We will format the data disk later.
4.  **Software Selection:**
    *   [X] SSH server
    *   [X] Standard system utilities
    *   [ ] (Uncheck everything else)

## 4. Post-Install Configuration

Login as `root`.

### 1. System Prep
```bash
apt-get update && apt-get install -y curl openssh-server net-tools dnsutils vim git postgresql-common gnupg lsb-release
```

**Disable Swap** (Recommended for DB performance):
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
```
127.0.0.1       localhost
# 127.0.1.1     database.salad.local database  <-- Remove this line or comments
```

**Enable Serial Console** (Optional but good for troubleshooting):
```bash
systemctl enable --now serial-getty@ttyS0.service
```

### 2. Configure Data Disk (PostgreSQL Storage)

Identify the disk (usually `vdb`):
```bash
lsblk
```

Format and mount:
```bash
mkfs.ext4 /dev/vdb
mkdir -p /var/lib/postgresql
```

Persist in fstab:
```bash
echo "/dev/vdb /var/lib/postgresql ext4 defaults 0 2" >> /etc/fstab
mount -a
```

Ensure permissions (will be set by postgres installer, but good to check):
```bash
chown postgres:postgres /var/lib/postgresql
```

### 3. Configure SSH Access
```bash
mkdir -p /root/.ssh && \
chmod 700 /root/.ssh && \
echo "<your-ssh-public-key>" > /root/.ssh/authorized_keys && \
chmod 600 /root/.ssh/authorized_keys
```

### 4. Install PostgreSQL 15

We will use the official PostgreSQL repository for the latest stable version.

```bash
# Create the file repository configuration:
sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key:
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add -

# Update the package lists:
apt-get update

# Install the latest version of PostgreSQL.
# If you want a specific version, use 'postgresql-15' or similar instead of 'postgresql':
apt-get install -y postgresql-15
```

### 5. Configure PostgreSQL

**Allow Remote Connections:**

1.  Edit `postgresql.conf`:
    ```bash
    vi /etc/postgresql/15/main/postgresql.conf
    ```
    Uncomment/Change:
    ```ini
    listen_addresses = '*'
    ```

2.  Edit `pg_hba.conf` to allow access from local network:
    ```bash
    vi /etc/postgresql/15/main/pg_hba.conf
    ```
    Add to the end:
    ```
    # Allow connections from salad-net IPs
    host    all             all             192.168.123.0/24        scram-sha-256
    ```

3.  Restart PostgreSQL:
    ```bash
    systemctl restart postgresql
    ```

### 6. Create Users and Databases

We will create databases for our planned services.

Switch to postgres user:
```bash
su - postgres
```

**Keycloak Database:**
```bash
psql -c "CREATE DATABASE keycloak;"
psql -c "CREATE USER keycloak WITH ENCRYPTED PASSWORD 'keycloak_password';"
psql -c "GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;"
# For Postgres 15+, need to grant schema usage
psql -d keycloak -c "GRANT ALL ON SCHEMA public TO keycloak;"
```

**SonarQube Database (Future):**
```bash
psql -c "CREATE DATABASE sonarqube;"
psql -c "CREATE USER sonarqube WITH ENCRYPTED PASSWORD 'sonar_password';"
psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube TO sonarqube;"
psql -d sonarqube -c "GRANT ALL ON SCHEMA public TO sonarqube;"
```

**Grafana Database:**
```bash
psql -c "CREATE DATABASE grafana;"
psql -c "CREATE USER grafana WITH ENCRYPTED PASSWORD 'grafana_password';"
psql -c "GRANT ALL PRIVILEGES ON DATABASE grafana TO grafana;"
psql -d grafana -c "GRANT ALL ON SCHEMA public TO grafana;"
```

**Artifactory Database:**
```bash
psql -c "CREATE DATABASE artifactory;"
psql -c "CREATE USER artifactory WITH ENCRYPTED PASSWORD 'artifactory_password';"
psql -c "GRANT ALL PRIVILEGES ON DATABASE artifactory TO artifactory;"
psql -d artifactory -c "GRANT ALL ON SCHEMA public TO artifactory;"
```

Exit postgres user:
```bash
exit
```

## 5. Install Node Exporter (Monitoring)

```bash
apt-get install -y prometheus-node-exporter
systemctl enable --now prometheus-node-exporter
```

## 6. Verification

From your host machine:

1.  **Check Port Open:**
    ```bash
    nc -zv 192.168.123.21 5432
    ```
2.  **Test Connection** (requires `postgresql-client` on host, or test via another VM like containerization):
    ```bash
    psql -h 192.168.123.21 -U keycloak -d keycloak -W
    ```
