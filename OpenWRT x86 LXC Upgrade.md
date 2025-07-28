# OpenWRT x86 Upgrade on Proxmox LXC container

## Prerequisites
- Proxmox VE installed and accessible.
- Sufficient disk space and memory for new OpenWRT container.
- Backup of your current OpenWRT configuration.
- Familiarity with Proxmox LXC management (web UI and CLI).

## Container IDs
- Old OpenWRT container: `202`
- New OpenWRT container: `203`

## Download OpenWRT x86-64 Rootfs
Download the OpenWRT x86-64 rootfs tarball `rootfs.tar.gz` from the official OpenWRT
https://downloads.openwrt.org/releases/24.10.2/targets/x86/64/

## Create Container in Proxmox
Create a new LXC container in Proxmox using the downloaded OpenWRT rootfs tarball.

### Upload the OpenWRT rootfs tarball
Upload the `openwrt-24.10.2-x86-64-rootfs.tar.gz` file to the Proxmox server, typically to the directory `/var/lib/vz/template/cache/`.

#### Create CT Templates via Proxmox Web Interface:
1. Log in to the Proxmox web interface.
2. Navigate to **Datacenter** > **pve** > **local (pve)**.
3. Click on **CT Templates**.
4. Click on **Download from URL** and enter the URL of the OpenWRT rootfs tarball (https://downloads.openwrt.org/releases/24.10.2/targets/x86/64/openwrt-24.10.2-x86-64-rootfs.tar.gz)
5. Click on **Download** to start the download process.
6. Once the download is complete, the OpenWRT template will be available in the **CT Templates** section.

### Create a new LXC container

#### Example command to create the container with network interfaces:

``` bash
pct create 203 /var/lib/vz/template/cache/openwrt-24.10.2-x86-64-rootfs.tar.gz --arch amd64 --hostname OpenWrt-24.10 --rootfs local-lvm:2 --memory 512 --cores 4 --ostype unmanaged --unprivileged 1 --features nesting=1 --net0 name=eth0,bridge=vmbr0,ip=192.168.1.5/24,gw=192.168.1.1,type=veth,firewall=1 --net1 name=eth1,bridge=vmbr1,type=veth,firewall=1
```

#### Configure Network Interfaces with Static IP

Set up the network interfaces as follows:
- `eth0`: Assign a static IP address of `192.168.1.5` with gateway `192.168.1.1` and enable the firewall.
- `eth1`: Attach to bridge `vmbr1` and enable the firewall.

Example commands:
```bash
pct set 203 --net0 name=eth0,bridge=vmbr0,ip=192.168.1.5/24,gw=192.168.1.1,type=veth,firewall=1
pct set 203 --net1 name=eth1,bridge=vmbr1,type=veth,firewall=1
```


### Start the OpenWRT Container and Open the Console
Start the newly created OpenWRT container using the Proxmox web interface or via command line:
``` bash
pct start 203
```
Then, open the console of the container:
``` bash
pct enter 203
```

## Fix DNS problem in OpenWRT x86-64 LXC Container

quick fix about 'failed to seed the random number generator' problem:
edit /etc/init.d/dnsmasq
add /dev/urandom at the end of this line procd_add_jail_mount /etc/passwd /etc/group /etc/TZ /etc/hosts /etc/ethers
final line looks like:
procd_add_jail_mount /etc/passwd /etc/group /etc/TZ /etc/hosts /etc/ethers /dev/urandom

Create `fix_dns_add_urandom.sh` file in /root/ and paste the following script into it:
``` bash
#!/bin/sh

# Check if the script is running with root privileges
if [ "$(id -u)" -ne 0 ]; then
    echo "This script must be run as root."
    exit 1
fi

# Define the file to be modified
FILE="/etc/init.d/dnsmasq"

# Inform the user of the action
echo "Adding /dev/urandom to procd_add_jail_mount in $FILE"

# Use sed to append /dev/urandom to the exact line with leading whitespace,
# only if /dev/urandom is not already present; create a backup with -i.bak
sed -i.bak '/^[ \t]*procd_add_jail_mount \/etc\/passwd \/etc\/group \/etc\/TZ \/etc\/hosts \/etc\/ethers/ { /\/dev\/urandom/! s/$/ \/dev\/urandom/ }' "$FILE"

# Confirm completion and notify about the backup
echo "Modification complete. Original file backed up to $FILE.bak"
```

Make the script executable:
``` bash
chmod +x fix_dns_add_urandom.sh
```
Run the script to apply the changes:
``` bash
./fix_dns_add_urandom.sh
```


## Network Configuration during Upgrade
- Change LAN IP to 192.168.1.5 and gateway to 192.168.1.1 for upgrade
- Update LAN DNS to 1.1.1.1 and 8.8.8.8
- Disable DHCP server to avoid conflicts during upgrade
- Restart the network service after making changes

UCI commands to set the network configuration:
``` bash
uci set network.lan.ipaddr='192.168.1.5'
uci set network.lan.gateway='192.168.1.1'
uci set network.lan.dns='1.1.1.1 8.8.8.8'
uci commit network

uci set dhcp.lan.ignore='1'
uci commit dhcp
```

Restart the network service:
``` bash
/etc/init.d/network restart
```

## Install packages
``` bash
opkg update
opkg install luci-app-acme luci-app-banip luci-app-ddns luci-app-samba4 luci-app-uhttpd luci-app-upnp luci-app-wol luci-app-vnstat2 luci-proto-wireguard ddns-scripts-noip acme-acmesh-dnsapi wireguard-tools htop iperf3 kmod-fs-ext4 block-mount qrencode
```

## Change the default password in web interface
Access the OpenWRT web interface at http://192.168.1.5
1. Go to **System** > **Administration**.
2. Under the **Password** section, enter a new password.
3. Click on **Save & Apply** to set the new password.


## Backup Configuration from old OpenWRT
Backup the configuration from the old OpenWRT LUCI interface (http://192.168.1.1):
1. Go to **System** > **Backup / Flash Firmware**.
2. Click on **Generate Archive** to create a backup of the current configuration.
3. Download the generated archive to your local machine.

## Restore Configuration to new OpenWRT
To restore the configuration to the new OpenWRT installation:
1. In the new OpenWRT installation (http://192.168.1.5), go to **System** > **Backup / Flash Firmware**.
2. Click on **Restore Backup** and upload the previously downloaded archive.
3. Confirm the restoration process and wait for it to complete.
4. After the restoration, reboot the OpenWRT system to apply the changes.

### Post-Restoration Steps
After restoring the configuration, you may need to adjust the network settings again to ensure proper connectivity. Use the following commands to set the LAN IP, gateway, and DNS settings:
``` bash
uci set network.lan.ipaddr='192.168.1.5'
uci set network.lan.gateway='192.168.1.1'
uci set network.lan.dns='1.1.1.1 8.8.8.8'
uci commit network

uci set dhcp.lan.ignore='1'
uci commit dhcp
/etc/init.d/network restart
```


## Stop old OpenWRT container
Stop the old OpenWRT container using the Proxmox web interface or via command line:
``` bash
pct stop 202
```
Disconnect the old OpenWRT container from the wan network to avoid conflicts:
``` bash
pct set 202 --net1 link_down=1
```

## Set up the new OpenWRT container with network configuration
Set up the new OpenWRT container with the desired network configuration. Use the following command to set the network interface for the new container:

``` bash
pct set 203 --net0 name=eth0,bridge=vmbr0,ip=192.168.1.1/24,gw=192.168.1.1,type=veth,firewall=1
```

### Configure the new OpenWRT container to use the same network settings as the old one
Ensure that the new OpenWRT container has the same network settings as the old one (192.168.1.1 and remove DNS and enable DHCP). Use the following commands to set the LAN IP and gateway settings:

``` bash
uci set network.lan.ipaddr='192.168.1.1'
uci set network.lan.gateway='192.168.1.1'
uci set network.lan.dns=''
uci commit network

uci set dhcp.lan.ignore='0'
uci commit dhcp
/etc/init.d/network restart
```

### Restart the new OpenWRT container
Restart the new OpenWRT container to apply the changes:
``` bash
pct reboot 203
```

### Verify the new OpenWRT container is running and WAN is accessible
After the new OpenWRT container has restarted, verify that it is running correctly and that the WAN connection is accessible. You can do this by checking the network status in the Proxmox web interface or by using the following command:
``` bash
pct status 203
```
If the status shows that the container is running, you can also check the network connectivity by pinging an external address from within the container:
``` bash
pct exec 203 -- ping -c 4 8.8.8.8
```

Verify WAN IP in the OpenWRT web interface:
1. Access the OpenWRT web interface at http://192.168.1.1
2. Go to **Status** > **Overview** to check the WAN IP address.
3. Ensure that the WAN IP is correctly assigned and that the internet connection is working.


## Post-Upgrade Steps
Post-configuration in Proxmox after upgrading to the new OpenWRT container.

### Disable auto start of the old OpenWRT container
To prevent the old OpenWRT container from starting automatically, you can disable it using the following command:
``` bash
pct set 202 --onboot 0
```

### Enable auto start of the new OpenWRT container and set priority to 1
To ensure the new OpenWRT container starts automatically on boot, use the following command:
``` bash
pct set 203 --onboot 1 --startup order=1
```
