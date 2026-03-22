# 1. Home Server

- [1. Home Server](#1-home-server)
  - [1.1. My Hardware](#11-my-hardware)
  - [1.2. Monthly Costs](#12-monthly-costs)
  - [1.3. Router Setup](#13-router-setup)
  - [1.4. Server Setup](#14-server-setup)
    - [1.4.1. Smart Plug and Wake on LAN](#141-smart-plug-and-wake-on-lan)
    - [1.4.2. Proxmox Installation](#142-proxmox-installation)
    - [1.4.3. Data and Backup Pools Setup](#143-data-and-backup-pools-setup)
    - [1.4.4. Dataset Encryption](#144-dataset-encryption)
    - [1.4.5. RAID array recovery](#145-raid-array-recovery)
      - [1.4.5.1. OS disk recovery](#1451-os-disk-recovery)
    - [1.4.6. Enable Email Alerts](#146-enable-email-alerts)
    - [1.4.7. Mail notifications for ZFS pools](#147-mail-notifications-for-zfs-pools)
      - [1.4.7.1. Fix for ZED Bug](#1471-fix-for-zed-bug)
    - [1.4.8. UPS configuration](#148-ups-configuration)
      - [1.4.8.1. UPS Mail Notifications](#1481-ups-mail-notifications)
  - [1.5. Proxmox Containers and VMs](#15-proxmox-containers-and-vms)

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

- Cloud storage for files using [Nextcloud](https://nextcloud.com/).
- A photo gallery using [Memories](https://apps.nextcloud.com/apps/memories).
- A password manager using [Passbolt](https://www.passbolt.com/).
- [Pi-hole](https://pi-hole.net/) with [Unbound](https://nlnetlabs.nl/projects/unbound/about/)
  running locally as a recursive DNS solver, to block ads and protect your DNS queries.
- [HomeAssistant](https://www.home-assistant.io/) to manage all your home automations from a single interface.

All these services have both desktop and mobile clients available.

Backups are performed automatically every night and uploaded (encrypted) to an external cloud service. Reports for backup jobs and periodic hardware health checks are sent via email.

## 1.1. My Hardware

| Component | Description | Price |
| :--- | :--- | :--- |
| CASE | Fractal Design Define 7 Compact | 140€ |
| MB | Gigabyte B550M DS3H AC R2 rev. 1.0 | 91€ |
| RAM | 128GB (4x Silicon Power SP032GBLFU320X02 32GB DDR4 3200 1.2V) | 286€ |
| CPU | AMD Ryzen 7 5700G 8C/16T 3.8/4.6 GHz | 134€ |
| NVME | 2x Kingston 2TB SNV3S/2000G (Main storage for VMs/Containers - RAID1) | 202€ |
| SSD | 2x LNS100-256RB (OS and applications - RAID1) | 42€ |
| HDD | 2x Western Digital 3 TB WD30PURZ-85GR (Storage for backups - RAID1) | 240€ |
| PSU | Cooler Master G650M | 120€ |
| UPS | CyberPower CP900EPFCLCD | 210€ |
| Fans/Misc | Noctua fans, cables, etc... | 110€ |
| Smart Plug | Shelly Plug S Gen3 | 25€ |
| **Total** | | **1600€** |

## 1.2. Monthly Costs

I expect the hardware to last 10 years, as that was my experience with a previous server (Xeon E3-1240 v3) bought in 2013 and replaced in 2025. This makes the amortized hardware cost ~13€/month. The server consumes 45W in idle and runs 24/7, making the electricity cost roughly 10€/month (assuming a rate of 0.30€/kWh). I also use AWS S3 for offsite backups, which costs 3€/month, and a domain name for 1€/month.

So, the **total monthly cost is ~27€/month**.
Considering that 2TB of Google Drive storage costs 10€/month, building a home server is not strictly worth it from a purely economical point of view. It makes sense mainly if you plan to build a similar machine for other purposes—as I do—such as experimenting with new technologies and services, or creating VMs for development. It is also worthwhile if you value your privacy, want to keep your data under your own control, and wish to avoid vendor lock-in.

Of course, you can reduce these costs by using cheaper components, omitting a UPS, choosing a more power-efficient CPU, or not using an external cloud for backups. However, the hardware and software setup I have documented here is likely the bare minimum required to achieve a decent degree of redundancy and reliability. This ensures you won't regret skipping the convenience of cloud services when the first disk fails or a software update breaks your configuration.

## 1.3. Router Setup

My ISP provides a modem/router combo that I cannot replace because it integrates an Optical Network Terminal (ONT) for my FTTH connection—a feature not easily found in consumer routers. It supports a 5Gbps connection divided across multiple interfaces: one 2.5Gbps port, two 1Gbps ports, and an internal WiFi 6 access point. I have connected one of the 1Gbps ports to my server, and the remaining ports serve my home network.

The router is also connected to the UPS to ensure it remains operational during a local power outage. Unfortunately, my ISP contract does not include a cellular fallback (e.g., 4G/5G). As a result, my services will become inaccessible from the internet if the FTTH connection drops or if a broader power outage affects the ISP's infrastructure.

While I have a domain name to access my Proxmox services, this is not enough if something breaks while I am away from home; I also need an SSH connection to reach the hypervisor.

Because I do not want to expose an SSH service directly to the internet, I use a VPN. Fortunately, the ISP router supports configuring a WireGuard endpoint, allowing me to securely connect to my home network at any time and fix whatever is broken.

My ISP router has another limitation: it assigns a shared IPv4 address with a restricted number of available ports. For instance, I cannot use ports 80 and 443 to expose web services to the internet. While I do have a pool of IPv6 addresses that could potentially be used to expose services entirely over IPv6, the router unfortunately drops IPv6 connectivity on occasion and requires a reboot to restore it.

I discovered that I can use free services like Cloudflare Tunnels (`cloudflared`) or Tailscale to expose my web services securely without having to deal with IP addresses or port forwarding. This is a practical solution, particularly if you cannot request a static IP address from your ISP.

This guide covers the setup of a Cloudflare Tunnel. I chose this option because Cloudflare is already the registrar for my domain name, and I use it to manage my DNS records.

## 1.4. Server Setup

The setup I describe here is based on my hardware configuration, it assumes you have a similar hardware setup so 3 sets of disks for OS, main storage and backups. And a domain name to access your services from the internet.

### 1.4.1. Smart Plug and Wake on LAN

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

### 1.4.2. Proxmox Installation

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

### 1.4.3. Data and Backup Pools Setup

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

Add the following lines to the cron table. The `MAILTO=""` option prevents `cron` from sending redundant emails, as the `zpool scrub` command via ZED will already send an email report containing the final outcome:

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

### 1.4.4. Dataset Encryption

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

### 1.4.5. RAID array recovery

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

#### 1.4.5.1. OS disk recovery

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

### 1.4.6. Enable Email Alerts

The server needs to be able to send alert emails whenever an error occurs. To facilitate this, a "send-only" SMTP server must be configured.

To combat spam, most ISPs completely block outbound traffic on port 25. You can verify whether port 25 is blocked by your ISP using the command `telnet <external_mailserver_address> 25`.
The standard alternative is to use port 587. However, port 587 is a client submission port requiring authenticated credentials (username and password) to an existing mail server. Once authenticated, that mail server will forward your message to external servers over port 25.
As an example, if `<tom@foo.com>` sends an email to `<carl@bar.info>`:

1. Tom connects to his submission port (587) and authenticates identically to his `foo.com` SMTP server (`smtp.foo.com`) using his credentials.
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

### 1.4.7. Mail notifications for ZFS pools

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

#### 1.4.7.1. Fix for ZED Bug

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

### 1.4.8. UPS configuration

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

#### 1.4.8.1. UPS Mail Notifications

We need to configure NUT to execute a shell script that pushes an email whenever a power event happens. Edit `/etc/nut/upsmon.conf` and append these lines:

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

## 1.5. Proxmox Containers and VMs

The server hosts the following containers and VMs, each having an ID that reflects its last IP address octet:

**Containers:**

- **109 (pbs01):** An instance of Proxmox Backup Server used to perform backups of all other containers and VMs.
  - **110 (nas):** A Debian container initially named "nas" because it solely hosted Nextcloud. It now hosts multiple services:
    - Cloudflared
    - Traefik
    - Nextcloud
    - Passbolt
  - **120 (pihole):** Pi-hole runs in a dedicated container, generated by a Proxmox Community script, since it is only used on the local network.

**VMs:**

- **115 (homeassistant):** Home Assistant is installed in a dedicated VM, as running it in a container is not recommended by the HA community and poses certain limitations.

Cloudflared and Traefik provide secure remote access to all services. They could potentially be moved to a dedicated container, as Home Assistant also depends on them for remote access. However, I prefer to keep them in the "nas" container for simplicity. If I add more services in the future, I will consider moving them to a standalone container.

The following sections describe the configuration and setup process for each container and VM.
