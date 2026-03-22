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
      - [1.4.7.1. Fix for zed bug](#1471-fix-for-zed-bug)
    - [1.4.8. UPS configuration](#148-ups-configuration)
      - [1.4.8.1. UPS Mail Notifications](#1481-ups-mail-notifications)

This repo contains scripts and configurations to setup an home server based on [Proxmox VE](https://www.proxmox.com/en/)
and [Proxmox Backup Server](https://www.proxmox.com/en/products/proxmox-backup-server/overview).

The goal is to keep all the services decoupled as much as possible from the hardware, so when something will break
you are be able to restore all your services without having to bother with settings you may have forgotten.

In the worst case scenario the restore procedure should be as simple as:

- Install Proxmox on a new machine
- Recreate the Backup Server instance
- Import all your VMs/Containers from local backups or remote backups.

This also applyes to SW issues in your running services (e.g. an update that broke the configuration), but in this case the recovery is juts one click away (restore a VM/Container snapshot).

The home server provides the following services:

- A cloud storage for files [Nextcloud](https://nextcloud.com/).
- A photo gallery [Memories](https://apps.nextcloud.com/apps/memories).
- A password [Passbolt](https://www.passbolt.com/).
- [Pi-hole](https://pi-hole.net/) with [Unbound](https://nlnetlabs.nl/projects/unbound/about/)
  running locally as reverse DNS, to block adds and protect your DNS queries.
- [HomeAssistant](https://www.home-assistant.io/) to manage all your home automations from a single interface.

All the services have both Desktop and Mobile clients.

Backups are performed automatically every night and uploaded (encrypted) on an external cloud service. Reports of backups jobs and periodic HW health checks are sent by mail.

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

I would expect the HW to last 10 years or at least this was my experience with a previous server (Xeon E3-1240 v3) bought in 2013 and replaced in 2025. This makes the HW cost ~13€/month. The server consumes 45W/h in idle and is on 24/7, the electricity cost is 10€/month (assuming 0.3€/kWh). I use also AWS S3 for offsite backups, which costs 3€/month and a domain name for 1€/month.

So the **total monthly cost is ~27€/month**.
Considering that 2TB on Google Drive costs 10€/month, this is totally not worth it from a purely economical point of view, this makes sense only if you would anyway build a similar machine for other purposes, like I do, using it also to experiment with new technologies and services, create VMs for development, etc... And of course if you value your privacy and want to keep your data under your control and avoid vendors lock-in.

Of course you can reduce the costs by using cheaper components, avoiding a UPS, choosing a more power efficient CPU, not using an external cloud for backups, etc... Anyway the HW/SW setup I have built is most likely the bare minimum to have a decent degree of redundancy and reliability, not making you regret the convenience of cloud services when the first disk fails or a SW update breaks your configuration.

## 1.3. Router Setup

My ISP provides a modem/router combo that I cannot replace because it integrates an Optical Network Terminal (ONT) for my FTTH connection—a feature not easily found in consumer routers. It supports a 5Gbps connection divided across multiple interfaces: one 2.5Gbps port, two 1Gbps ports, and an internal WiFi 6 access point. I have connected one of the 1Gbps ports to my server, and the remaining ports serve my home network.

The router is also connected to the UPS to ensure it remains operational during a local power outage. Unfortunately, my ISP contract does not include a cellular fallback (e.g., 4G/5G). As a result, my services will become inaccessible from the internet if the FTTH connection drops or if a broader power outage affects the ISP's infrastructure.

While I have a domain name to access my Proxmox services, this is not enough if something breaks while I am away from home; I also need an SSH connection to reach the hypervisor.

Because I do not want to expose an SSH service directly to the internet, I use a VPN. Fortunately, the ISP router supports configuring a WireGuard endpoint, allowing me to securely connect to my home network at any time and fix whatever is broken.

My ISP router has also another pitfall: it has a shared IPv4 address on which I have a limited number of ports I can use, for instance I cannot use port 80 and 443 to expose web services to the internet. I have also a pool of IPv6 addresses that can be potentially used to expose services to the internet, but unfortunately sometimes the router drops IPv6 connectivity and I have to reboot it to restore it.

I found out that I can use free services such as cloudflare tunnel (cloudflared) or Tailscale to expose my web services to the internet without having to deal with IP addresses. This is a practical solution also if you cannot request a static IP address from your ISP.

This guide covers the setup of a cloudflared tunnel, I choosed this option because Cloudflare is also the registrar of my domain name and I already use it to manage my DNS records.

## 1.4. Server Setup

The setup I describe here is based on my hardware configuration, it assumes you have a similar hardware setup so 3 sets of disks for OS, main storage and backups. And a domain name to access your services from the internet.

### 1.4.1. Smart Plug and Wake on LAN

In the BIOS/UEFI the server is set to power on whenever the power is restored after an outage, I also have a smart plug that can be controlled via Home Assistant or directly by a web interface, so I can power off the server and power it on again when needed. The smart plug is connected to the UPS, so it will not be affected by a power outage. This is a preventive measure if for whatever reason the server is not reachable on the network, I can power cycle it via the smart plug and it will come back online.

Just for extra safety, I have also configured Wake on LAN (WoL). This allows me to remotely power on the server using a WoL tool if it do not start automatically after a power outage.

Ensure WOL is enabled in BIOS settings then on Proxmox list
all network interfaces:

```bash
ip a
```

Check if WOL is enabled on the ethernet interface (example):

```bash
ethtool nic0
```

if not edit `/etc/network/interfaces`

```ini
iface nic0 inet manual
        post-up ethtool -s nic0 wol g
        pre-down ethtool -s nic0 wol g
```

### 1.4.2. Proxmox Installation

During the installation of Proxmox VE, remember to:

- Enable ZFS RAID 1 for the OS disks (2x 256GB SSDs)
- Set server name according to your domain name (e.g. pve01.domain.com)
- Assign a static IP address to the server

At the end of the installation, you should be able to access the Proxmox VE web interface at https://[your_server_ip]:8006.

Run the post installation script from [Proxmox Community Scripts](https://community-scripts.org/)

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

Set the following options:

- Disable 'pve-enterprise' repository
- Disable 'ceph enterprise' repository
- Add 'pve-no-subscription' repository
- Add 'ceph' package repository (no-subscription)
- Add 'pve-test' repository (but disabled)
- Disable subscription nag
- Disable high availability (no-replicas)
- Disable Corosync (no-replicas)

### 1.4.3. Data and Backup Pools Setup

Check which drives you want to include in the data pool:

```shell
fdisk -l
```

Get the devices IDs

```shell
ls -la /dev/disk/by-id/
```

In my case the disks IDs are something like:

- nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXX63
- nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXXD0
- ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXUF
- ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXN1

Respectively for the data pool (nvme disks) and the backup pool (mechanical disks)

Create a mirrored pool named `tank` for the data pool and avoid setting a mount folder (`-m none`):

```Shell
zpool create -O compression=on -m none tank mirror /dev/disk/by-id/nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXX63 /dev/disk/by-id/nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXXD0
```

Create a mirrored pool named `archive` for the backup pool and avoid setting a mount folder (`-m none`):

```Shell
zpool create -O compression=on -m none archive mirror /dev/disk/by-id/ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXUF /dev/disk/by-id/ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXN1
```

>NOTE
>
>Avoid using names such `/dev/sdc` and `/dev/sdd` in place of disk IDs because these identifier are NOT stable between reboots if new devices are added or removed.

Check the pools status:

```Shell
zpool status
```

It should be `ONLINE` for all your pools:

- tank
- archive
- rpool (Proxmox default pool created during installation)

By default the compression algorithm should be lz4 (that is good):

```Shell
zpool get feature@lz4_compress tank
zpool get feature@lz4_compress archive
zpool get feature@lz4_compress rpool
```

should return `active` for all pools.

Since compression is enabled at pool level this setting is inherited by all datasets created below the pools.

Periodically it is suggested to scrub the pool, the command `zpool scrub <pool_name>` examines all data in the specified pool to verify that it checksums correctly, it also repairs recoverable errors if found. This operation is I/O intensive, I decided to schedule it once a week late at night (every Tuesday at 2 am).

To setup the `cron` job give the following command:

```shell
crontab -e
```

Add the following lines to the cron table, the `MAILTO=""` line avoids that extra mails are sent by cron, the `zpool scrub` command already report the outcome by mail:

```cron
MAILTO=""
0 2 * * 2 /sbin/zpool scrub tank
0 2 * * 2 /sbin/zpool scrub rpool
0 2 * * 2 /sbin/zpool scrub archive
```

Save and close and verify that the cron table is updated:

```shell
crontab -l
```

### 1.4.4. Dataset Encryption

I want all my data to be encrypted, so I decided to create an encrypted dataset below the `tank` pool and the `archive` pool. This means that whenever the server is powered off, the data will be encrypted and inaccessible without the passphrase, the price to pay is that I have to type the passphrase every time I reboot the server and want to restart VMs or containers. This is not a big deal for me since the server is in my home and I have also remote access to it (via VPN).

Encryption is not supported at pool level, it can be enabled in child datasets at creation time. Be aware that encryption cannot be enabled at later stages, the parameter is read-only.

Create the encrypted dataset below the `tank` and `archive` pool:

```Shell
zfs create tank/encrypted -o encryption=on -o keyformat=passphrase
zfs create archive/encrypted -o encryption=on -o keyformat=passphrase
```

Check the used cipher:

```Shell
zfs get encryption tank/encrypted
zfs get encryption archive/encrypted
```

The default should be `aes-256-gcm` that is considered secure and also the best cipher you can currently get from ZFS

To unlock/lock the created datasets use the following commands:

```Shell
zfs load-key tank/encrypted
zfs load-key archive/encrypted
zfs unload-key tank/encrypted
zfs unload-key archive/encrypted
```

If you want to use the dataset a mount point shall be specified in the dataset parameters. In this case I want `tank/encrypted` to be mounted in `/data` and `archive/encrypted` to be mounted in `/archive`

```Shell
zfs set mountpoint=/data tank/encrypted
zfs set mountpoint=/archive archive/encrypted
```

Then to mount/umount use the following commands:

```Shell
zfs mount -l tank/encrypted
zfs mount -l archive/encrypted
zfs umount -u /data
zfs umount -u /archive
```

The above commands also load(`-l`)/unload(`-u`) the encryption key (they ask to type the pool key when executed).

Using the proxmox GUI go to `Datacenter/Storage` and add 2 zfs storage, one with ID `tank-encrypted` pointing to the `tank/encrypted` pool and one with ID `archive-encrypted` pointing to the `archive/encrypted` pool
enable also the `thin provision` option for both.

### 1.4.5. RAID array recovery

It may happen that one of the disks in the RAID array fails, in that case the array will be in a degraded state.

To check the pool status:

```shell
zpool status
```

Put offline the failed disk:

```shell
zpool offline <pool_name> <disk_id>
```

Power off the NAS and attach the new disk, use `sudo fdisk -l` to get all the disk info

Wipe the new disk (e.g. `/dev/sdd`):

```shell
wipefs -a -f /dev/sdd
```

Copy the partition scheme from the working disk (e.g. from `/dev/sdc`) and randomize the disk identifier:

```shell
sgdisk --backup=table /dev/sdc
sgdisk --load-backup=table /dev/sdd
sgdisk -G /dev/sdd
rm -rf table
```

Get the disk id:

```shell
ls -la /dev/disk/by-id/
```

Replace the disk:

```shell
zpool replace <pool_name> <old_disk_id> <new_disk_id> 
```

The rebuilding process can be monitored with `zpool status` command

After the disks are synced it is also possible to force a coherency check:

```shell
zpool scrub <pool_name>
```

#### 1.4.5.1. OS disk recovery

In case the failed disk was part of the OS pool (rpool) the procedure is the same but further steps are needed to ensure the OS bootability from the replaced disk.

Check that the boot list covers both boot partitions identifiers:

```Shell
efibootmgr -v
```

You should see 2 entries similar to these ones:

```Shell
BootCurrent: 0001
Timeout: 1 seconds
BootOrder: 0001,0002
Boot0001* UEFI OS ....
Boot0002* UEFI OS ....
```

If these is not the case, you need to update the boot entries and then check again,
the following command should fix the issue with an interactive prompt:

```Shell
dpkg-reconfigure grub-efi-amd64
```

### 1.4.6. Enable Email Alerts

The server should be able to send alerts via mail whenever something bad happens, a SMTP "send-only" server needs to be configured for this purpose.

Most ISP to prevent spam do not allow anymore to send emails via port 25, you can check if port 25 is blocked by your ISP by using this command `telnet <mailserver_address> 25`.
The alternative is to use port 587 but this is a *client* reserved port, so it only works for users subscribed to a mail server and requires authentication with username and password. The mail server will then forward the message to other mail servers using port 25 (without authentication).
Example: <tom@foo.com> send a mail to <carl@bar.info>

1. Tom send the mail by using the mail submission port (587) so by authenticating to the foo.com SMTP server (e.g. smtp.foo.com) whit his credentials
2. smtp.foo.com recognize that bar.info is an external domain and forward the mail to the destination SMTP server (e.g. smtp.bar.info) using port 25 (no authentication required, port 25 should be used only for mail exchanges between SMTP servers).
3. Carl open his mailbox at bar.info (POP3 or IMAP protocol) to read the received mail

The conclusion is that if your port 25 is blocked you cannot setup a standalone SMTP server. You have to leverage on an external mail provider to send mails outside of your domain.
Fortunately if you have a Google account you can use the Gmail SMTP server for free (limited to 500 mails per day).

What I did was to create a new Gmail account to act only as a mail relay, with
an app password to be used by the proxmox server.

The steps below cover the setup of `smtp.gmail.com` as a mail relay, but the procedure is similar for other mail providers.

Change the server hostname to be coherent with your server name by editing `/etc/hostname`.
Edit also the `/etc/hosts` to be coherent with the specified server name and the server assigned IP address

```ini
<server_ip> <server_name>.<domain> <server_name>
```

Reboot to change settings and then do the following procedure:

Ensure the proxmox default user (e.g. root@pam) has a valid email address set, this can be different from the mail address used for the SMTP relay. You can check this from `Datacenter -> Permissions -> Users` section

Go to `Datacenter -> Notifications -> Notification Targets`
Add a new SMTP target with the following settings:

- end point name:     gmail
- server:             smtp.gmail.com
- encryption          STARTTLS
- username:           gmail relay address
- password:           app password of gmail relay
- from adress:        gmail relay address
- recipients:         proxmox user (e.g. root@pam)
- Author:             name you want in the mail title

Modify the default Notification Matcher in `Datacenter -> Notifications -> Matchers` to route all mails to the newly created target,
you can also set the default `mail-to-root` target to disabled.

### 1.4.7. Mail notifications for ZFS pools

The `zed` service can be used to monitor the data pool.

Mails are sent whenever a problem is detected and whenever a scrub process complete and a report is available:

```Shell
apt install zfs-zed
```

Then edit the `/etc/zfs/zed.d/zed.rc` uncommenting/changing the following settings:

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

#### 1.4.7.1. Fix for zed bug

Unfortunately a bug prevent zed from sending notification if a drive fails when the NAS is off. In that case when the server is started the pool status is correctly set to `DEGRADED` however no mail notifications are sent by zed.

A periodic `cron` job can be scheduled to fix this issue, create a file `zpool-check.sh` in the user directory with the following content:

```bash
#! /bin/bash
ZPOOL_STATUS=$(/usr/sbin/zpool status -x)
echo "$ZPOOL_STATUS" | grep -v "all pools are healthy" && mail -r root -s "ZPOOL DEGRADED" root <<< $ZPOOL_STATUS
```

Make the file executable

```bash
chmod +x zpool-check.sh
```

Add the cron job for the user:

```shell
crontab -e
```

Paste the following line to execute the script every hour:

```cron
0 * * * * ./zpool-check.sh
```

Check that the new cron table is installed:

```shell
crontab -l
```

### 1.4.8. UPS configuration

This configuration is specific for the CyberPower CP1500PFCLCD UPS model, anyway the procedure should be similar for other models or brands supported by NUT (Network UPS Tools).

In my case the UPS is connected via USB and the `nut` server is not exposed to other untrusted clients (localhost only). Below steps are not safe if the UPS is exposed to other untrusted clients.

Install nut

```bash
apt-get install nut
```

list USB devices

```bash
lsusb

Bus 001 Device 002: ID 0764:0501 Cyber Power System, Inc. CP1500 AVR UPS
```

Use `nut-scanner` to find the UPS properties

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

Add the device to `/etc/nut/ups.conf` avoid setting also
`bus`, `device`, and `busport` properties as they may change between reboots

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

Edit `/etc/nut/nut.conf`

```ini
MODE=standalone
```

Edit `/etc/nut/upsd.users`

```ini
   [monuser]
       password = <your_password>
       upsmon primary
       actions = SET
       instcmds = ALL
```

Edit `/etc/nut/upsmon.conf`

```ini
MONITOR cyberpower@localhost 1 monuser <your_password> primary
```

Add a rule for nut in `/etc/udev/rules.d/50-ups.rules` take care
`idVendor` and `idProduct` are matching with the `lsusb` output

```ini
SUBSYSTEM=="usb", ATTR{idVendor}=="0764", ATTR{idProduct}=="0501", MODE="664", GROUP="nut"
```

Reload the rules

```bash
udevadm control --reload
udevadm trigger
```

Check if the driver start without errors

```bash
upsdrvctl start
```

Enable all required nut services and check the status

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

Edit `/etc/nut/upsmon.conf`

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

Create the notification script `/etc/nut/notify.sh`

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

Assign the correct permission to the script and restart the `nut` service:

```bash
chown :nut /etc/nut/notify.sh
chmod 774 /etc/nut/notify.sh

systemctl restart nut-server.service
systemctl restart nut-driver.service
systemctl restart nut-monitor.service
```

