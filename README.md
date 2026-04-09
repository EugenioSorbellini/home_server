# 1. Home Server

- [1. Home Server](#1-home-server)
  - [1.1. My Hardware](#11-my-hardware)
    - [1.1.1. Some considerations on the HW](#111-some-considerations-on-the-hw)
  - [1.2. Router Setup](#12-router-setup)
  - [1.3. Server Setup](#13-server-setup)
    - [1.3.1. Proxmox Installation](#131-proxmox-installation)
    - [1.3.2. Smart Plug and Wake on LAN](#132-smart-plug-and-wake-on-lan)
    - [1.3.3. Data and Backup Pools Setup](#133-data-and-backup-pools-setup)
    - [1.3.4. Dataset Encryption](#134-dataset-encryption)
    - [1.3.5. RAID Array Recovery](#135-raid-array-recovery)
      - [1.3.5.1. OS Disk Recovery](#1351-os-disk-recovery)
    - [1.3.6. Enable Mail Alerts](#136-enable-mail-alerts)
    - [1.3.7. Mail notifications for ZFS pools](#137-mail-notifications-for-zfs-pools)
      - [1.3.7.1. Fix for ZED bug](#1371-fix-for-zed-bug)
    - [1.3.8. UPS Configuration](#138-ups-configuration)
      - [1.3.8.1. UPS mail notifications](#1381-ups-mail-notifications)
  - [1.4. Proxmox Containers and VMs](#14-proxmox-containers-and-vms)
  - [1.5. Pi-hole Container Setup](#15-pi-hole-container-setup)
  - [1.6. NAS Container Setup](#16-nas-container-setup)
    - [1.6.1. GPU Passthrough](#161-gpu-passthrough)
    - [1.6.2. Cloudflared](#162-cloudflared)
      - [1.6.2.1. DNS Challenge](#1621-dns-challenge)
    - [1.6.3. Traefik](#163-traefik)
    - [1.6.4. Passbolt](#164-passbolt)
    - [1.6.5. Nextcloud](#165-nextcloud)
      - [1.6.5.1. Nextcloud stop script](#1651-nextcloud-stop-script)
    - [1.6.6. Wetty](#166-wetty)
  - [1.7. Home Assistant VM](#17-home-assistant-vm)
  - [1.8. Backups](#18-backups)
    - [1.8.1. Remote Sync to S3 storage](#181-remote-sync-to-s3-storage)
    - [1.8.2. Recreate PBS LXC container](#182-recreate-pbs-lxc-container)

This repository contains scripts and configurations to set up a home server based on [Proxmox VE](https://www.proxmox.com/en/)
and [Proxmox Backup Server](https://www.proxmox.com/en/products/proxmox-backup-server/overview).

The goal is to keep all services decoupled from the hardware as much as possible, so that when a failure occurs,
you will be able to restore all your services without having to worry about forgotten settings.

In the worst-case scenario, the restore procedure should be as simple as:

- Installing Proxmox on a new machine
- Recreating the Backup Server instance
- Importing all your VMs/Containers from local or remote backups.

This also applies to software issues within your running services (e.g., an update that breaks a configuration), but in that case, recovery is just one click away (restoring a VM/Container snapshot).

The home server provides the following services:

- Cloud storage for files [Nextcloud](https://nextcloud.com/).
- A photo gallery [Memories](https://apps.nextcloud.com/apps/memories).
- A password manager [Passbolt](https://www.passbolt.com/).
- [Pi-hole](https://pi-hole.net/) with [Unbound](https://nlnetlabs.nl/projects/unbound/about/)
  running locally as a recursive DNS solver, to block ads and protect your DNS queries.
- [HomeAssistant](https://www.home-assistant.io/) to manage all your home automations from a single interface.

All these services have both desktop and mobile clients available.

Backups are performed automatically every night and uploaded (encrypted) to an external cloud service. Reports for backup jobs and periodic hardware health checks are sent via email.

## 1.1. My Hardware

| Component | Description |
| :--- | :--- |
| CASE | Fractal Design Define 7 Compact |
| MB | Gigabyte B550M DS3H AC R2 rev. 1.0 |
| RAM | 128GB (4x Silicon Power SP032GBLFU320X02 32GB DDR4 3200 1.2V) |
| CPU | AMD Ryzen 7 5700G 8C/16T 3.8/4.6 GHz |
| NVME | 2x Kingston 2TB SNV3S/2000G (Main storage for VMs/Containers - RAID1) |
| SSD | 2x LNS100-256RB (Hypervisor OS - RAID1) |
| HDD | 2x Western Digital 3 TB WD30PURZ-85GR (Storage for backups - RAID1) |
| PSU | Cooler Master G650M |
| UPS | CyberPower CP900EPFCLCD |
| Smart Plug | Shelly Plug S Gen3 (emergency power cycle) |

### 1.1.1. Some considerations on the HW

I expect the hardware to have a lifespan of 10 years, as that was my experience with a previous server (Xeon E3-1240 v3) bought in 2013 and replaced in 2025. All the components listed above consume approximately 45W at idle (measured with the smart plug).

Besides hardware and electricity costs, you should also consider additional recurring costs for offsite backups and a domain name. For example, I pay ~3€/month for AWS S3 storage and ~1€/month to Cloudflare for the domain.

Considering that 2TB of Google Drive storage costs 10€/month, building a home server is not strictly worth it from a purely economical point of view. It makes sense mainly if you plan to build a similar machine for other purposes—as I do—such as experimenting with new technologies and services, or creating VMs for development. It is also worthwhile if you value your privacy, want to keep your data under your own control, and wish to avoid vendor lock-in.

You can of course save some money by using less RAM, less storage etc...
What I would consider the bare minimum to protect your data and have a usable server are:

- 2x SSD/NVMe in RAID1 for OS, VMs and Containers
- 2x HDDs in RAID1 for backups
- 1x UPS to protect against power outages and power surges
- 1x Smart Plug to power cycle the server if it hangs
- 1x offsite backup to a cloud service for disaster recovery
- 1x domain name to easily access services from the internet

## 1.2. Router Setup

My ISP provides a modem/router combo that I cannot replace because it integrates an Optical Network Terminal (ONT) for my FTTH connection—a feature not easily found in consumer routers. It supports a 5Gbps connection divided across multiple interfaces: one 2.5Gbps port, two 1Gbps ports, and an internal WiFi 6 access point. I have connected one of the 1Gbps ports to my server, and the remaining ports serve my home network.

The router is also connected to the UPS to ensure it remains operational during a local power outage. Unfortunately, my ISP contract does not include a cellular fallback (e.g., 4G/5G). As a result, my services will become inaccessible from the internet if the FTTH connection drops or if a broader power outage affects the ISP's infrastructure.

While I have a domain name to access my Proxmox services, this is not enough if something breaks while I am away from home; I also need an SSH connection to reach the hypervisor.

Because I do not want to expose an SSH service directly to the internet, I use a VPN. Fortunately, the ISP router supports configuring a WireGuard endpoint, allowing me to securely connect to my home network at any time and fix whatever is broken.

My ISP router has another limitation: it assigns a shared IPv4 address with a restricted number of available ports. For instance, I cannot use ports 80 and 443 to expose web services to the internet. While I do have a pool of IPv6 addresses that could potentially be used to expose services entirely over IPv6, the router unfortunately drops IPv6 connectivity on occasion and requires a reboot to restore it.

I discovered that I can use free services like Cloudflare Tunnels (`cloudflared`) or Tailscale to expose my web services securely without having to deal with IP addresses or port forwarding. This is a practical solution, particularly if you cannot request a static IP address from your ISP.

This guide covers the setup of a Cloudflare Tunnel. I chose this option because Cloudflare is already the registrar for my domain name, and I use it to manage my DNS records.

Regarding the local network configuration, I would suggest using static IP addresses for all devices you want to regularly access, while keeping DHCP for smartphones and other devices that you do not need to log into remotely. I do not need multiple subnets; I just use one network with a /24 netmask. What I normally do is set the DHCP server to assign IP addresses in the range of 150–250. You may think that reserving almost 150 static IP addresses is a lot, but it is not, considering that you may have many CTs and VMs running on your server, each with its own local IP address, and you may also have many devices for your home automations (e.g., cameras, lights, blind actuators, sensors, etc.).

## 1.3. Server Setup

The setup I describe here is based on my hardware configuration. It assumes you have a similar hardware setup—three sets of disks for OS, main storage, and backups—and a domain name to access your services from the internet.

### 1.3.1. Proxmox Installation

During the initial installation of Proxmox VE, remember to:

- Enable ZFS RAID 1 for the OS disks (using the 2x 256GB SSDs).
- Set the server hostname according to your domain name (e.g., `pve01.domain.com`).
- Assign a static IP address to the server.

Once the installation completes, you should be able to access the Proxmox VE web interface at `https://[your_server_ip]:8006`.

Next, run the post-installation script from [Proxmox Community Scripts](https://community-scripts.org/):

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

When prompted, set the following options:

- Disable 'pve-enterprise' repository
- Disable 'ceph enterprise' repository
- Add 'pve-no-subscription' repository
- Add 'ceph' package repository (no-subscription)
- Add 'pve-test' repository (but disabled)
- Disable subscription nag
- Disable high availability (no-replicas)
- Disable Corosync (no-replicas)

### 1.3.2. Smart Plug and Wake on LAN

In the BIOS/UEFI, the server is set to power on automatically whenever power is restored after an outage. I also have a smart plug that can be controlled via Home Assistant or directly through its web interface, allowing me to remotely power cycle the server if needed. The smart plug is connected to the UPS, so it will not drop power during a utility outage. This setup is a preventive measure: if the server ever becomes unreachable on the network, I can forcefully power cycle it via the smart plug to bring it back online.

For extra safety, I have also configured Wake on LAN (WoL). This allows me to remotely wake the server using a WoL tool if it fails to start automatically after a power outage.

First, ensure WoL is enabled in your BIOS settings. Then, in Proxmox, list all available network interfaces:

```bash
ip a
```

Check if WoL is enabled on your primary Ethernet interface (for example, `nic0`):

```bash
ethtool nic0
```

If it is not enabled, edit `/etc/network/interfaces`:

```ini
iface nic0 inet manual
        post-up ethtool -s nic0 wol g
        pre-down ethtool -s nic0 wol g
```

### 1.3.3. Data and Backup Pools Setup

First, check which drives you want to include in the data pool:

```shell
fdisk -l
```

Retrieve the proper device IDs:

```shell
ls -la /dev/disk/by-id/
```

In my case, the disk IDs look something like this:

- nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXX63
- nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXXD0
- ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXUF
- ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXN1

These correspond to the data pool (NVMe disks) and the backup pool (mechanical disks), respectively.

Create a mirrored pool named `tank` for the data pool. We avoid setting a default mount folder by passing `-m none`:

```Shell
zpool create -O compression=on -m none tank mirror /dev/disk/by-id/nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXX63 /dev/disk/by-id/nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXXD0
```

Create a mirrored pool named `archive` for the backup pool, again passing `-m none`:

```Shell
zpool create -O compression=on -m none archive mirror /dev/disk/by-id/ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXUF /dev/disk/by-id/ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXN1
```

> [!NOTE]
> Avoid using standard device names like `/dev/sdc` or `/dev/sdd` in place of disk IDs. These identifiers are NOT stable between reboots and might change if standard devices are added or removed.

Check the status of your pools:

```Shell
zpool status
```

The status should be `ONLINE` for all your pools:

- tank
- archive
- rpool (Proxmox default pool created during installation)

By default, the compression algorithm should be set to `lz4` (which is ideal):

```Shell
zpool get feature@lz4_compress tank
zpool get feature@lz4_compress archive
zpool get feature@lz4_compress rpool
```

The command should return `active` for all pools.

Because compression is enabled at the pool level, this setting is inherited by all datasets created within these pools.

It is highly recommended to scrub your pools periodically. The command `zpool scrub <pool_name>` examines all data in the specified pool to verify that its checksums are correct, and actively repairs recoverable errors if found. Since scrub operations are I/O intensive, I schedule them to run once a week late at night (every Tuesday at 2:00 AM).

To set up the `cron` job, run the following command to edit your crontab:

```shell
crontab -e
```

Add the following lines to the crontab. The `MAILTO=""` option prevents `cron` from sending redundant emails, as `zpool scrub` via ZED will already send an email report containing the final outcome:

```cron
MAILTO=""
0 2 * * 2 /sbin/zpool scrub tank
0 2 * * 2 /sbin/zpool scrub rpool
0 2 * * 2 /sbin/zpool scrub archive
```

Save and close the editor, then verify that the cron table was successfully updated:

```shell
crontab -l
```

### 1.3.4. Dataset Encryption

I want all my data to be encrypted at rest. To achieve this, I created an encrypted dataset inside both the `tank` pool and the `archive` pool. This ensures that whenever the server is powered off, the data remains encrypted and inaccessible without the passphrase. The trade-off is that I must manually type the passphrase every time I reboot the server before the VMs and containers can start. This is acceptable for my use case since the server is physically in my home, and I can always reach it remotely via VPN.

Currently, ZFS does not support native encryption at the entire pool level; it must be enabled on child datasets at the time they are created. Be aware that encryption cannot be enabled retroactively on existing datasets—the parameter is read-only once created.

Create the encrypted datasets inside the `tank` and `archive` pools:

```Shell
zfs create tank/encrypted -o encryption=on -o keyformat=passphrase
zfs create archive/encrypted -o encryption=on -o keyformat=passphrase
```

Check which cipher was used for the encryption:

```Shell
zfs get encryption tank/encrypted
zfs get encryption archive/encrypted
```

The default cipher should be `aes-256-gcm`, which is highly secure and currently the most performant cipher offered by ZFS.

To unlock or lock the created datasets manually, use the following commands:

```Shell
zfs load-key tank/encrypted
zfs load-key archive/encrypted
zfs unload-key tank/encrypted
zfs unload-key archive/encrypted
```

If you want to use these datasets, you need to assign them a mount point. In my case, I want `tank/encrypted` to be mounted at `/data` and `archive/encrypted` to be mounted at `/archive`:

```Shell
zfs set mountpoint=/data tank/encrypted
zfs set mountpoint=/archive archive/encrypted
```

To conveniently mount or unmount these datasets, use the following commands:

```Shell
zfs mount -l tank/encrypted
zfs mount -l archive/encrypted
zfs umount -u /data
zfs umount -u /archive
```

These mount commands pass the `-l` (load key) and `-u` (unload key) flags, respectively, which will prompt you to type the decryption passphrase when executed.

Finally, log in to the Proxmox GUI, navigate to `Datacenter -> Storage`, and add two new ZFS storage units. Give them IDs like `tank-encrypted` (pointing to the `tank/encrypted` dataset) and `archive-encrypted` (pointing to the `archive/encrypted` dataset). Make sure to enable the `Thin provision` option for both.

### 1.3.5. RAID Array Recovery

It may happen that one of the disks in a RAID array fails. In that case, the pool will enter a `DEGRADED` state.

First, check the pool status to identify the failed disk:

```shell
zpool status
```

Take the failed disk offline:

```shell
zpool offline <pool_name> <disk_id>
```

Power off the server, physically remove the failed disk, and attach the replacement disk. Once powered back on, use `sudo fdisk -l` to identify the new disk's assigned device name.

Wipe the new disk (e.g., assuming the new disk is `/dev/sdd`):

```shell
wipefs -a -f /dev/sdd
```

Copy the partition scheme over from the working disk (e.g., from `/dev/sdc`) and randomize the new disk's GUIDs:

```shell
sgdisk --backup=table /dev/sdc
sgdisk --load-backup=table /dev/sdd
sgdisk -G /dev/sdd
rm -rf table
```

Find the new disk's stable `/dev/disk/by-id/` identifier:

```shell
ls -la /dev/disk/by-id/
```

Replace the old failed disk with the new disk in the array:

```shell
zpool replace <pool_name> <old_disk_id> <new_disk_id> 
```

The rebuilding (resilvering) process will begin immediately. You can monitor its progress using the `zpool status` command.

After the disks are completely synced, you can optionally force a manual coherency check to ensure everything is healthy:

```shell
zpool scrub <pool_name>
```

#### 1.3.5.1. OS Disk Recovery

If the failed disk was part of the OS pool (`rpool`), the recovery procedure is identical to the one above, but additional steps are required to ensure the system remains bootable from the replacement disk.

Check that the EFI boot list correctly references both boot partitions:

```Shell
efibootmgr -v
```

You should see two boot entries similar to these:

```Shell
BootCurrent: 0001
Timeout: 1 seconds
BootOrder: 0001,0002
Boot0001* UEFI OS ....
Boot0002* UEFI OS ....
```

If only one entry is listed, you need to update the boot loader entries. The following command should launch an interactive prompt to fix the issue and reinstall GRUB to both disks:

```Shell
dpkg-reconfigure grub-efi-amd64
```

### 1.3.6. Enable Mail Alerts

The server needs to be able to send alert emails whenever an error occurs. To facilitate this, a send-only SMTP relay must be configured.

To combat spam, most ISPs completely block outbound traffic on port 25. You can verify whether port 25 is blocked by your ISP using the command `telnet <external_mailserver_address> 25`.
The standard alternative is to use port 587. However, port 587 is a client submission port requiring authenticated credentials (username and password) to an existing mail server. Once authenticated, that mail server will forward your message to external servers over port 25.
For example, if `<tom@foo.com>` sends an email to `<carl@bar.info>`:

1. Tom connects to the submission port (587) and authenticates with his `foo.com` SMTP server (`smtp.foo.com`) using his credentials.
2. `smtp.foo.com` recognizes that `bar.info` is an external domain and forwards the email to the target SMTP server (`smtp.bar.info`) over port 25.
3. Carl subsequently opens his `bar.info` mailbox over POP3 or IMAP to read the message.

The key takeaway is that if your port 25 is blocked, you cannot host your own standalone outbound SMTP server. Instead, you must use an external mail provider as an SMTP relay to send emails externally.
Fortunately, if you have a Google account, you can use the Gmail SMTP server for free (with a generous limit of 500 emails per day).

I created a dedicated Gmail account strictly to act as a mail relay and generated a Google "App Password" to be used by the Proxmox server.

The steps below cover the setup process for using `smtp.gmail.com` as a mail relay, though the procedure is very similar for other providers.

First, ensure your server's hostname matches your fully qualified domain name by editing `/etc/hostname`.
Also, update `/etc/hosts` to ensure it reflects your server name and its assigned IP address:

```ini
<server_ip> <server_name>.<domain> <server_name>
```

Reboot the server to apply these settings, then proceed with the following steps:

Ensure the default Proxmox user (e.g., `root@pam`) has a valid email address configured. This address can be entirely different from the one used for the SMTP relay. You can check this in the Proxmox GUI under `Datacenter -> Permissions -> Users`.

Next, navigate to `Datacenter -> Notifications -> Notification Targets`.
Click to add a new `SMTP Target` and configure it with the following settings:

- **Endpoint Name:**     `gmail`
- **Server:**            `smtp.gmail.com`
- **Encryption:**        `STARTTLS`
- **Username:**          `<your_gmail_relay_address>`
- **Password:**          `<your_gmail_app_password>`
- **From Address:**      `<your_gmail_relay_address>`
- **Recipients:**        `<your_proxmox_user>` (e.g., `root@pam`)
- **Author:**            The sender name you want displayed in the email subject.

Finally, edit the default Notification Matcher under `Datacenter -> Notifications -> Matchers` to route all mail output to your newly created target. You can also safely set the default `mail-to-root` target to disabled.

### 1.3.7. Mail notifications for ZFS pools

The ZFS Event Daemon (`zed`) service can automatically monitor your data pools.

ZED is responsible for sending out email alerts whenever a hardware problem is detected or whenever a scrub process successfully completes and generates a report.

Install the `zfs-zed` package:

```Shell
apt install zfs-zed
```

Then, edit `/etc/zfs/zed.d/zed.rc` by uncommenting and modifying the following configuration lines:

```ini
ZED_EMAIL_ADDR="root"
ZED_EMAIL_PROG="mail"
ZED_EMAIL_OPTS="-s '@SUBJECT@' @ADDRESS@"
ZED_LOCKDIR="/var/lock"
ZED_NOTIFY_INTERVAL_SECS=3600
ZED_NOTIFY_VERBOSE=1
ZED_NOTIFY_DATA=1
ZED_USE_ENCLOSURE_LEDS=1
ZED_SYSLOG_PRIORITY="daemon.notice"
ZED_SYSLOG_TAG="zed"
ZED_SYSLOG_SUBCLASS_EXCLUDE="history_event"
```

#### 1.3.7.1. Fix for ZED bug

Unfortunately, a known bug prevents ZED from sending a notification if a drive completely fails while the server is powered off. In this edge case, when the server starts back up, the pool status will correctly display `DEGRADED`, but ZED will fail to trigger any email notifications alerting you of the failure.

A simple periodic `cron` script can be scheduled to patch over this issue. Create a file named `zpool-check.sh` in your user directory with the following content:

```bash
#! /bin/bash
ZPOOL_STATUS=$(/usr/sbin/zpool status -x)
echo "$ZPOOL_STATUS" | grep -v "all pools are healthy" && mail -r root -s "ZPOOL DEGRADED" root <<< $ZPOOL_STATUS
```

Make the script executable:

```bash
chmod +x zpool-check.sh
```

Add a background cron job for the root user:

```shell
crontab -e
```

Paste the following line to execute the check script silently every hour:

```cron
0 * * * * ./zpool-check.sh
```

Check that the new cron schedule is active:

```shell
crontab -l
```

### 1.3.8. UPS Configuration

This specific configuration is tailored for the CyberPower CP1500PFCLCD UPS model. However, the overall procedure will be very similar for other UPS models or brands supported by Network UPS Tools (NUT).

In my setup, the UPS is connected directly to the server via USB. The `nut` daemon is restricted exclusively to `localhost` and is not exposed to untrusted clients. Please note that the steps below are not considered secure if you plan to expose the UPS daemon over the network.

First, install the `nut` package:

```bash
apt-get install nut
```

List your connected USB devices:

```bash
lsusb

Bus 001 Device 002: ID 0764:0501 Cyber Power System, Inc. CP1500 AVR UPS
```

Use `nut-scanner` to auto-discover the USB UPS properties:

```bash
nut-scanner -U

[nutdev1]
        driver = "usbhid-ups"
        port = "auto"
        vendorid = "0764"
        productid = "0501"
        product = "CP900EPFCLCD"
        serial = "000000000000"
        vendor = "CPS"
        bus = "001"
        device = "002"
        busport = "006"
        ###NOTMATCHED-YET###bcdDevice = "0001"
```

Manually declare the device inside `/etc/nut/ups.conf`. Avoid including the `bus`, `device`, or `busport` parameters, as they are inherently dynamic and may randomly change between reboots.

```ini
[cyberpower]
        driver = "usbhid-ups"
        port = "auto"
        vendorid = "0764"
        productid = "0501"
        product = "CP900EPFCLCD"
        serial = "000000000000"
        vendor = "CPS"
```

Set the daemon mode to standalone in `/etc/nut/nut.conf`:

```ini
MODE=standalone
```

Set up a monitoring user and password in `/etc/nut/upsd.users`:

```ini
   [monuser]
       password = <your_password>
       upsmon primary
       actions = SET
       instcmds = ALL
```

Configure the local monitor agent in `/etc/nut/upsmon.conf`:

```ini
MONITOR cyberpower@localhost 1 monuser <your_password> primary
```

Ensure the daemon possesses the correct hardware permissions by adding a udev rule to `/etc/udev/rules.d/50-ups.rules`. Double-check that `idVendor` and `idProduct` accurately reflect your `lsusb` output:

```ini
SUBSYSTEM=="usb", ATTR{idVendor}=="0764", ATTR{idProduct}=="0501", MODE="664", GROUP="nut"
```

Reload the udev rules to apply the new permissions:

```bash
udevadm control --reload
udevadm trigger
```

Start the UPS driver to verify it launches without errors:

```bash
upsdrvctl start
```

Finally, enable and start all required NUT backend services, then check their status to ensure they are running properly:

```bash
systemctl enable nut-driver-enumerator.service
systemctl start nut-driver-enumerator.service
systemctl status nut-driver-enumerator.service

systemctl enable nut-server.service
systemctl start nut-server.service
systemctl status nut-server.service

systemctl enable nut-monitor.service
systemctl start nut-monitor.service
systemctl status nut-monitor.service

systemctl enable nut.target
systemctl enable nut-driver.target
```

#### 1.3.8.1. UPS mail notifications

NUT needs to be configured to execute a shell script that sends an email whenever a power event occurs. Edit `/etc/nut/upsmon.conf` and append these lines:

```ini
NOTIFYCMD "/etc/nut/notify.sh"
NOTIFYFLAG ONLINE    SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT    SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT   SYSLOG+WALL+EXEC
NOTIFYFLAG FSD       SYSLOG+WALL+EXEC
NOTIFYFLAG COMMOK    SYSLOG+WALL+EXEC
NOTIFYFLAG COMMBAD   SYSLOG+WALL+EXEC
NOTIFYFLAG SHUTDOWN  SYSLOG+WALL+EXEC
```

Create the notification script at `/etc/nut/notify.sh`:

```bash
#!/bin/bash

# Email settings
EMAIL_SUBJECT="<your_server_name> UPS"
EMAIL_RECIPIENT="root"

# UPS event data
UPSEVENT=$1

# Compose email message
MESSAGE="UPS Event: $UPSEVENT\n"

# Send email using mail command
echo -e "$MESSAGE" | mail -s "$EMAIL_SUBJECT" "$EMAIL_RECIPIENT"
```

Assign the correct executable permissions to the script, assign ownership to the `nut` user, and restart all `nut` services to apply the configuration changes:

```bash
chown :nut /etc/nut/notify.sh
chmod 774 /etc/nut/notify.sh

systemctl restart nut-server.service
systemctl restart nut-driver.service
systemctl restart nut-monitor.service
```

## 1.4. Proxmox Containers and VMs

The server hosts the following containers and VMs. Each ID reflects the last octet of the container's or VM's IP address:

**Containers:**

- **120 (pihole):** Pi-hole runs in a dedicated container, generated by a Proxmox Community script, since it is only used on the local network.
- **110 (nas):** A Debian container initially named "nas" because it solely hosted Nextcloud. It now hosts multiple services:
  - Cloudflared
  - Traefik
  - Nextcloud
  - Passbolt
  - Wetty
- **109 (pbs01):** An instance of Proxmox Backup Server used to perform backups of all other containers and VMs.

**VMs:**

- **115 (homeassistant):** Home Assistant is installed in a dedicated VM, as running it in a container is not recommended by the Home Assistant community and poses certain limitations.

Cloudflared and Traefik provide secure remote access to all services. They could potentially be moved to a dedicated container, as Home Assistant also depends on them for remote access. However, I prefer to keep them in the "nas" container for simplicity. If I add more services in the future, I will consider moving them to a standalone container.

The following sections describe the configuration and setup process for each container and VM.

> Note that in my `docker-compose` files, sensitive information (like tokens) is stored in a separate location (e.g. `/root/vault/`) from this repository. This is a security measure to prevent accidental exposure or scraping. You can, of course, place everything in a single `.env` file if you prefer.

## 1.5. Pi-hole Container Setup

`Pi-hole` is a DNS sinkhole that can be used to block ads and other unwanted content. It can be used together with `Unbound` to create a private DNS server.

`Pi-hole` is installed in a dedicated LXC container using the available Proxmox helper scripts:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/pihole.sh)"
```

During the setup phase, I specified the following settings for the container:

- CT ID:  120
- IP:     <your_local_subnet>.20
- RAM:    4GB
- Disk:   8GB
- Cores:  2
- Unbound: configured as recursive DNS server

To ensure all traffic from Unbound to the top-level DNS servers is also encrypted, I added the following lines to the Unbound configuration at the end of `/etc/unbound/unbound.conf.d/pi-hole.conf`:

```ini
# TLS upstream config
  tcp-upstream: yes
  tls-upstream: yes
  tls-cert-bundle: "/etc/ssl/certs/ca-certificates.crt"

forward-zone:
  name: "."
  forward-tls-upstream: yes
  forward-addr: 9.9.9.9@853#dns.quad9.net
  forward-addr: 149.112.112.112@853#dns.quad9.net
  forward-addr: 2620:fe::fe@853#dns.quad9.net
  forward-addr: 2620:fe::9@853#dns.quad9.net
```

After changing the configuration, remember to restart `Pi-hole` to apply
the new settings.

## 1.6. NAS Container Setup

The `nas` container was originally created to host Nextcloud, but it now hosts multiple services (and thus outgrew its name).

The container is based on Debian 13 with the following settings:

- Type:                 Privileged
- IP                    <your_local_subnet>.10
- CT ID                 110
- Hostname:             nas
- DiskSize:             1000G
- CPU:                  8
- RAM                   32768

In `Options -> Features`, enable nesting.

Install Docker following the official instructions.

Enable SSH login:

```shell
echo "PermitRootLogin yes" > /etc/ssh/sshd_config.d/nas.conf
```

The `nas` container runs several services using `docker-compose`:

- Cloudflared - to expose services to the internet
- Traefik - to route traffic to the correct service depending on the CNAME (e.g. vault.domain.com -> Passbolt)
- Nextcloud - file and media sharing
- Passbolt - password manager

Consistent with the structure of this repository, services are organized into separate directories. Each directory contains at least a `docker-compose.yml` file and a `.env` file (the `.env` files are excluded from version control as they contain sensitive secrets).

To start a service, navigate to its directory and run:

```shell
docker compose up -d
```

To stop a service, use:

```shell
docker compose down
```

Additional details for each service are provided in the following sections.

### 1.6.1. GPU Passthrough

My CPU features an integrated GPU that supports hardware acceleration for video encoding and decoding. I pass this GPU through to the `nas` container to make it available to Docker.

First, identify the GPU device on the host system:

```shell
lspci -v
08:00.0 VGA compatible controller: Advanced Micro Devices, Inc. [AMD/ATI] Cezanne [Radeon Vega Series / Radeon Vega Mobile Series] (rev c8) (prog-if 00 [VGA controller])
```

```shell
lspci -nn
08:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Cezanne [Radeon Vega Series / Radeon Vega Mobile Series] [1002:1638] (rev c8)
```

On the host, check which `/dev/dri/renderD*` and `/dev/dri/card*` devices correspond to the GPU (amdgpu driver):

```shell
ls -l /sys/class/drm/renderD*/device/driver
ls -l /sys/class/drm/card*/device/driver

/dev/dri/card1
/dev/dri/renderD129
```

On the LXC container:

```shell
apt install pciutils
apt install mesa-utils
```

Find the GID for the `render` and `video` groups:

```shell
cat /etc/group | grep render
cat /etc/group | grep video
```

In my case:

```shell
video 44
render 992
```

On the host, using the GUI, go to the LXC settings and navigate to `Resources -> Add -> Device Passthrough`.
Add the identified devices and set the correct `render` and `video` group IDs for the target.

For example:

- `/dev/dri/renderD129 gid=992`
- `/dev/dri/card1 gid=44`

### 1.6.2. Cloudflared

A Cloudflare Tunnel avoids exposing the NAS directly to the internet using a public IP address.
Instead, a tunnel is created between the NAS and the Cloudflare servers.

I opted for this service because my ISP frequently dropped the public IPv6 address assigned to the NAS, which required manual router restarts to restore public access.

To enable a Cloudflare Tunnel, the `cloudflared` service must run on the NAS.
It can be executed inside a Docker container.

To use a Cloudflare Tunnel, you must first create a Cloudflare account and use it to manage your DNS records.

To enable a Cloudflare Tunnel, first delete all existing A, AAAA, and CNAME
entries in your DNS records, then go to the `Zero Trust` section in your Cloudflare
account and subscribe to the free plan.

Then click on `networks` -> `connectors` and create a new tunnel.
Save the generated `token`, as it will be required later to start the `cloudflared` service.

In the generated connectors, publish all required applications.
For each application, specify the desired domain and subdomain. Cloudflare will automatically regenerate the necessary DNS rules.

You also need to specify the endpoint of the tunnel. If you plan to route requests through Traefik (as I do), choose `HTTPS` as the service and type `traefik` into the URL field. This will resolve on the local network to the Traefik container's address. I also had to enable the `Match SNI to Host` and `HTTP2 connection` options in the tunnel configuration to make it work with Traefik.

Ensure that the `cloudflared` service is started in a container on the NAS. For the complete compose configuration, refer to the `cloudflared` directory in this repository.

You will need to manually create an env file `/root/vault/cloudflared.env`containing your tunnel token:

```env
TUNNEL_TOKEN=your_token_here
```

To properly route traffic, the `cloudflared` container must be linked to the `traefik` network. In the `docker-compose.yml` file, the `traefik` network is declared as `external`. This means `docker compose` expects the network to already exist on the host rather than creating it automatically.

You can create this network beforehand by running:

```shell
docker network create traefik
```

The `cloudflared` service will establish a tunnel to Cloudflare, and all traffic will be conveyed through it.
No ports need to be exposed to the internet.

#### 1.6.2.1. DNS Challenge

Cloudflare Tunnels do not work with Traefik's HTTP or TLS challenge for HTTPS certificate validation and renewal, so a DNS challenge is needed instead.

You need to create an API token in Cloudflare using the `Edit zone DNS` template. Save the token, as it will be required later for the Traefik configuration.

### 1.6.3. Traefik

Traefik is a reverse proxy that can be used to route traffic to different services based on the domain name.

All the configuration is stored in the `traefik` directory in this repository.

Before starting the Traefik container, you need to create all the required networks. I chose to use separate networks to isolate the services from each other.

```shell
docker network create traefik
docker network create passbolt
docker network create passbolt_db
docker network create nextcloud-aio 
docker network create nextcloud-frontend
docker network create wetty
```

You need to create a `.env` file in the `traefik` directory containing the following secrets:

```env
TRAEFIK_WEBPORT=<web port for traefik dashboard default should be 8080>
ACME_EMAIL=<mail you want to use for acme certificates e.g. info@yourdomain.com>
CLOUDFLARE_DNS_TOKEN=<token for dns challenges you should setup this in cloudflare and copy it there>
CLOUDFLARE_EMAIL=<mail you used in cloudflare registration, is needed for authentication together with the token>
NEXTCLOUD_DOMAIN=<domain you want to use for nextcloud e.g. nextcloud.yourdomain.com>
HOMEASSISTANT_DOMAIN=<domain you want to use for your HomeAssistant instance e.g. homeassistant.yourdomain.com>
NEXTCLOUD_URL=<this is the url to the local nextcloud frontend container e.g. http://nextcloud-aio-apache:11000>
HOMEASSISTANT_URL=<this is the url to the homeassistant VM e.g. http://<your_local_subnet>.15:8123/>
```

You also need an additional env file `/root/vault/traefik.env` containing the following secrets:

```env
CF_API_EMAIL=<mail you used in cloudflare registration, is needed for authentication together with the token>
CF_DNS_API_TOKEN=<token for dns challenges you should setup this in cloudflare and copy it there>
```

Note that `NEXTCLOUD_URL` and `HOMEASSISTANT_URL` use HTTP, not HTTPS, as TLS termination is handled by Traefik. You could also avoid handling certificates in Traefik entirely, since the Cloudflare Tunnel is already encrypted. In that case, you would use the `HTTP` protocol in the tunnel configuration instead of `HTTPS` and remove the TLS configuration from Traefik. However, I have not tried this option, since having Traefik handle TLS termination provides an extra layer of security.

### 1.6.4. Passbolt

Passbolt is a web-based password manager that can be self-hosted. It also has a polished mobile app.

All the configuration is stored in the `passbolt` directory in this repository.

You need to create a `.env` file in that directory containing the following secrets:

```env
# db
MYSQL_DATABASE=<database name you want to use for passbolt>
MYSQL_USER=<db user you want to use for passbolt>
MYSQL_PASSWORD=<db password you want to use for passbolt>
# pasbolt
MAIL_HOST=<mail relay host e.g. smtp.gmail.com>
MAIL_PORT=<mail relay port e.g. 587>
MAIL_TLS=<true or false>
MAIL_FROM=<passbolt mails will be sent from this address e.g. passbolt@yourdomain.com>
MAIL_USERNAME=<your mail relay username>
MAIL_PASSWORD=<your mail relay password>
DOMAIN=<domain you want to use for passbolt e.g. vault.yourdomain.com>
BASE_URL=https://${DOMAIN}
KEY_NAME=<key name for passbolt e.g. vault>
KEY_EMAIL=${KEY_NAME}@${DOMAIN}
```

You also need an additional env file `/root/vault/passbolt.env` containing the following secrets:

```env
EMAIL_TRANSPORT_DEFAULT_USERNAME=<your mail relay username>
EMAIL_TRANSPORT_DEFAULT_PASSWORD=<your mail relay password>
```

### 1.6.5. Nextcloud

Nextcloud is an open-source cloud storage solution, similar to the Google suite (Drive, Photos, Calendar, Contacts, etc.).
You can deploy Nextcloud in many ways; I chose to use the `nextcloud-aio` Docker image, which is currently the simplest and recommended method.
Since the [nextcloud-aio](https://github.com/nextcloud/all-in-one) repository contains all the information you need to deploy it, I will not repeat it here.

All the configuration I use is stored in the `nextcloud` directory in this repository.

#### 1.6.5.1. Nextcloud stop script

Nextcloud is composed of multiple containers, so it is important to stop them before shutting down the `nas` LXC container to avoid data corruption.
You can use a `systemd` service to stop nextcloud containers before shutdown and restart them at startup. Place the following script under `/etc/systemd/system/nextcloud-manager.service`:

```ini
[Unit]
Description=Stop nextcloud containers before shutdown and restart them at startup
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c "docker exec --env START_CONTAINERS=1 nextcloud-aio-mastercontainer /daily-backup.sh"
ExecStop=/bin/bash -c "docker exec --env STOP_CONTAINERS=1 nextcloud-aio-mastercontainer /daily-backup.sh"

[Install]
WantedBy=multi-user.target
```

Then install the service:

```bash
systemctl enable nextcloud-manager.service
```

To check if the service is working properly try to reboot and then inspect the service logs:

```bash
journalctl -u nextcloud-manager.service
```

### 1.6.6. Wetty

Wetty is a web-based terminal that can be used to access the server's terminal from a web browser.
I primarily use it to access a Raspberry Pi 3, but you can configure it to access any machine on your network.

All the configuration I use is stored in the `wetty` directory in this repository.

You need to create a `.env` file in that directory containing the following secrets:

```env
DOMAIN=<domain you want to use for wetty e.g. wetty.yourdomain.com>
SSH_HOST=<local ip address of the machine you want to access>
```

## 1.7. Home Assistant VM

I use Home Assistant OS in a dedicated VM to manage my smart home devices. It is configured with the following settings:

VM ID: 115
Memory: 16G
CPU: 4
Disk: 32GB
IP: <your_local_subnet>.15

While the `traefik` container is already configured to forward external requests to this VM, you must also allow remote access within your Home Assistant configuration. Add the following lines to your `configuration.yaml` file:

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - <your_local_subnet>.15
```

Note: Using a dedicated VM is recommended for Home Assistant OS. While running it in an LXC container is possible, it restricts certain features.

## 1.8. Backups

I use Proxmox Backup Server (PBS) to back up my Proxmox VE (PVE) environment.
Since I do not have a second PVE server, I installed PBS in an LXC container.
This means that if the PVE server fails, the LXC container will fail with it
and must be recreated. However, the backup data remains safe, as it is
stored on a RAID array and cloned nightly to remote S3 storage.

PBS supports client-side encryption, meaning backups are stored securely (both locally and on S3).
To restore these backups, you must have the client encryption key.
It is critical to store this key in a secure, separate location so you can access
your S3 backups if your PVE server completely fails.

PBS allows you to restore either an entire VM/CT or just specific files.

To install Promox Backup Server (PBS) in a LXC container, you can use the following script:

```shell
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/proxmox-backup-server.sh)"
```

I used the following settings:

- Type:                 Unprivileged
- CT ID:                109
- Hostname:             pbs01
- DiskSize:             32G
- CPU:                  2
- RAM:                  8192
- IP static:            <your_local_subnet>.9/24
- Root SSH access:      no (you have the web interface to access it)

Then you can run PBS post install (in PBS terminal this time) to complete the setup

```shell
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pbs-install.sh)"
```

From the PVE interface, navigate to the LXC PBS container and select `Resource -> Add -> Mount Point`.
Add a disk to host backups and mount it to the `/mnt/backup` directory. Remember to uncheck the `Backup` option for this mount point.

From PBS interface add a new local datastore with Backing Path `/mnt/backup` and name `backup-local`
In the prune options select: keep daily 1, keep weekly 1, keep monthly 1

From the created datastore options select `Verify new Snapshot`

Add prune and GC jobs to be executed daily before the backup job (e.g. at midnight)

Go to access controll and create a new user `backup-agent`
Go to permissions tab and assign to the `backup-agent` user permission to path `/datastore` with role `DatastoreAdmin`

Go to PVE interface and select `Datacenter -> Storage -> Add -> Proxmox Backup Server`

Fill the settings as follows:

- ID:           pbs-backup
- Server:       <your_local_subnet>.9
- Username:     backup-agent@pbs
- Datastore:    backup-local
- Fingerprint:  `pbs -> certificates -> view certificate`
- Encryption:   `auto generate a client encryption key`

> [!NOTE]
> It is mandatory to store the client encryption key in a safe place,
> otherwise you will not be able to restore your backups in case you need to recreate the PBS LXC container.

Go to PVE `Datacenter -> backup` and add desired backups, my configuration is the following:

- VMs/CTs to backup: 110, 115 and 120 (nas, home-assistant, pi-hole)
- Schedule: daily at 1am
- Mode: stop (stop the VMs/CTs to minimize data corruption risk)
- Other options to default

You can configure the backup server to send mail notifications. The procedure is identical to the PVE mail configuration.

Once backups are configured, you can test the restore procedure. From the PVE interface, navigate to the `pbs-backup` storage, where you can restore entire VMs/CTs or single files.

### 1.8.1. Remote Sync to S3 storage

I use this feature to sync my backups to Amazon S3. This ensures an offsite copy is available in case the local PVE server (and its disks) fails completely.

The backups are uploaded to Amazon S3 in an encrypted format. PBS uses the AES-256-GCM cipher, which is highly secure, ensuring your data remains safe even in the event of a cloud provider data breach.

PBS backups are incremental: only the initial backup uploads all your data to S3.
Subsequent backups upload only the changes, which is highly efficient for both speed and cost.
Furthermore, data is compressed and deduplicated, so the total amount of data uploaded to S3 remains much smaller than the actual size of your local VMs and CTs, even with multiple snapshots.

First, create an Amazon S3 bucket for your backups. Note that PBS has naming restrictions: dashes (`-`) are allowed, but dots (`.`) are not.

I named my Amazon S3 bucket something like `my-domain-backup`. You can leave all the other settings as default for generic usage.
You must also navigate to the `IAM` section and create a user with access to this bucket.
Generate an `AWS CLI` access key for this user, as it will be required by PBS later.

You need to add some cache to the backup server, this is needed to optimize the number of API calls to S3.
go to `Resource -> add -> mount point` and add a new disk for the cache, I have set the size to 128G
and the mount point to `/mnt/cache_s3`. Remember to uncheck the `Backup` option for this mount point.

Then in your PBS console go to `Configuration -> S3 Endpoints` and add a new endpoint with the following settings:

- Endpoint: `s3.{{region}}.amazonaws.com`
- Region: <your_s3_region> (e.g. eu-central-1)
- Port: 443
- Access Key: <your_access_key>
- Path Style: checked

Then go to `Datastore -> Add Datastore` and add a new datastore with the following settings:

- Datastore Type: s3
- Local Cache: /mnt/cache_s3
- S3 Endpoint: <your_s3_endpoint>
- Bucket: <your_bucket_name> (the bucket list should be automatically populated from the endpoint)

From the created datastore options select `Verify new Snapshot`

Add prune and GC jobs to be executed daily before the backup job (e.g. at midnight)

Create a `Pull` Sync job to sync the local datastore to S3, I set it with the following settings:

- Local Owner: root@pam
- Sync Schedule: daily at 5am
- Sorce Datastore: backup-local
- Comment: Copy backups in s3
- Remove Vanished: checked
- Encrypted Only: checked
- Verified Only: checked

### 1.8.2. Recreate PBS LXC container

- Recreate the PBS LXC container using the Proxmox Backup Server script as above
- You should have locally the volume you used for the datastore, you can attach it to the new LXC container,
  if you do not have it anymore you can pull it from S3.
  You need to mount this volume to the new LXC container at `/mnt/backup`
- In PBS go to `Datastore -> Add Datastore` and recreate the datastore from the `/mnt/backup` directory
  > [!NOTE]
  > It is mandatory to click on `Advanced` and select `Reuse existing datastore`
  > otherwise you will create a new datastore and you will not be able to restore your backups.
- Recreate the `backup-agent` user as described above
- Recreate the `pbs-backup` storage in PVE as described above
  > [!NOTE]
  > Here you shall not create a new encryption key, you shall use the old one you saved previously.
