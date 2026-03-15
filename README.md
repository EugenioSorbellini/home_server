# Home Server
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

## My Hardware
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
| Fans/Misc | Noctua fans, cables, etc... | 80€ |
| **Total** | | **1625€** |

## Monthly Costs
I would expect the HW to last 10 years or at least this was my experience with a previous server (Xeon E3-1240 v3) bought in 2013 and replaced in 2025. This makes the HW cost 14€/month. The server consumes 45W/h in idle and is on 24/7, the electricity cost is 10€/month (assuming 0.3€/kWh). I use also AWS S3 for offsite backups, which costs 3€/month and a domain name for 1€/month.

So the **total monthly cost is ~28€/month**.
Considering that 2TB on Google Drive costs 10€/month, this is totally not worth it from a purely economical point of view, this makes sense only if you would anyway build a similar machine for other purposes, like I do, using it also to experiment with new technologies and services, create VMs for development, etc... And of course if you value your privacy and want to keep your data under your control and avoid vendors lock-in. 

Of course you can reduce the costs by using cheaper components, avoiding a UPS, choosing a more power efficient CPU, not using an external cloud for backups, etc... Anyway the HW/SW setup I have built is most likely the bare minimum to have a decent degree of redundancy and reliability, not making you regret the convenience of cloud services when the first disk fails or a SW update breaks your configuration.

## Server Setup

The setup I describe here is based on my hardware configuration, it assumes you have a similar hardware setup so 3 sets of disks for OS, main storage and backups. And a domain name to access your services from the internet.

### Proxmox Installation

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

### Data and Backup Pools Setup

Check which drives you want to include in the data pool:

~~~shell
fdisk -l
~~~

Get the devices IDs

~~~shell
ls -la /dev/disk/by-id/
~~~

In my case the disks IDs are something like:

- nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXX63
- nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXXD0
- ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXUF
- ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXN1

Respectively for the data pool (nvme disks) and the backup pool (mechanical disks)

Create a mirrored pool named `tank` for the data pool and avoid setting a mount folder (`-m none`):

~~~Shell
zpool create -O compression=on -m none tank mirror /dev/disk/by-id/nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXX63 /dev/disk/by-id/nvme-KINGSTON_SNV3S2000G_XXXXXXXXXXXXXXD0
~~~

Create a mirrored pool named `archive` for the backup pool and avoid setting a mount folder (`-m none`):

~~~Shell
zpool create -O compression=on -m none archive mirror /dev/disk/by-id/ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXUF /dev/disk/by-id/ata-WDC_WD30PURZ-XXXXXXXXXXXXXXXXXXXXXN1
~~~

>NOTE
>
>Avoid using names such `/dev/sdc` and `/dev/sdd` in place of disk IDs because these identifier are NOT stable between reboots if new devices are added or removed.

Check the pools status:

~~~Shell
zpool status
~~~

It should be `ONLINE` for all your pools:
- tank
- archive
- rpool (Proxmox default pool created during installation)

By default the compression algorithm should be lz4 (that is good):

~~~Shell
zpool get feature@lz4_compress tank
zpool get feature@lz4_compress archive
zpool get feature@lz4_compress rpool
~~~

should return `active` for all pools.

Since compression is enabled at pool level this setting is inherited by all datasets created below the pools.

Periodically it is suggested to scrub the pool, the command `zpool scrub <pool_name>` examines all data in the specified pool to verify that it checksums correctly, it also repairs recoverable errors if found. This operation is I/O intensive, I decided to schedule it once a week late at night (every Tuesday at 2 am).

To setup the `cron` job give the following command:

~~~shell
crontab -e
~~~

Add the following lines to the cron table, the `MAILTO=""` line avoids that extra mails are sent by cron, the `zpool scrub` command already report the outcome by mail:

~~~cron
MAILTO=""
0 2 * * 2 /sbin/zpool scrub tank
0 2 * * 2 /sbin/zpool scrub rpool
0 2 * * 2 /sbin/zpool scrub archive
~~~

Save and close and verify that the cron table is updated:

~~~shell
crontab -l
~~~

### Dataset Encryption

I want all my data to be encrypted, so I decided to create an encrypted dataset below the `tank` pool and the `archive` pool. This means that whenever the server is powered off, the data will be encrypted and inaccessible without the passphrase, the price to pay is that I have to type the passphrase every time I reboot the server and want to restart VMs or containers. This is not a big deal for me since the server is in my home and I have also remote access to it (via VPN).

Encryption is not supported at pool level, it can be enabled in child datasets at creation time. Be aware that encryption cannot be enabled at later stages, the parameter is read-only.

Create the encrypted dataset below the `tank` and `archive` pool:

~~~Shell
zfs create tank/encrypted -o encryption=on -o keyformat=passphrase
zfs create archive/encrypted -o encryption=on -o keyformat=passphrase
~~~

Check the used cipher:

~~~Shell
zfs get encryption tank/encrypted
zfs get encryption archive/encrypted
~~~

The default should be `aes-256-gcm` that is considered secure and also the best cipher you can currently get from ZFS

To unlock/lock the created datasets use the following commands:

~~~Shell
zfs load-key tank/encrypted
zfs load-key archive/encrypted
zfs unload-key tank/encrypted
zfs unload-key archive/encrypted
~~~

If you want to use the dataset a mount point shall be specified in the dataset parameters. In this case I want `tank/encrypted` to be mounted in `/data` and `archive/encrypted` to be mounted in `/archive`

~~~Shell
zfs set mountpoint=/data tank/encrypted
zfs set mountpoint=/archive archive/encrypted
~~~

Then to mount/umount use the following commands:

~~~Shell
zfs mount -l tank/encrypted
zfs mount -l archive/encrypted
zfs umount -u /data
zfs umount -u /archive
~~~

The above commands also load(`-l`)/unload(`-u`) the encryption key (they ask to type the pool key when executed).

Using the proxmox GUI go to `Datacenter/Storage` and add 2 zfs storage, one with ID `tank-encrypted` pointing to the `tank/encrypted` pool and one with ID `archive-encrypted` pointing to the `archive/encrypted` pool
enable also the `thin provision` option for both.

### RAID array recovery

It may happen that one of the disks in the RAID array fails, in that case the array will be in a degraded state.

To check the pool status:

~~~shell
zpool status
~~~

Put offline the failed disk:

~~~shell
zpool offline <pool_name> <disk_id>
~~~

Power off the NAS and attach the new disk, use `sudo fdisk -l` to get all the disk info

Wipe the new disk (e.g. `/dev/sdd`):

~~~shell
wipefs -a -f /dev/sdd
~~~

Copy the partition scheme from the working disk (e.g. from `/dev/sdc`) and randomize the disk identifier:

~~~shell
sgdisk --backup=table /dev/sdc
sgdisk --load-backup=table /dev/sdd
sgdisk -G /dev/sdd
rm -rf table
~~~

Get the disk id:

~~~shell
ls -la /dev/disk/by-id/
~~~

Replace the disk:

~~~shell
zpool replace <pool_name> <old_disk_id> <new_disk_id> 
~~~

The rebuilding process can be monitored with `zpool status` command

After the disks are synced it is also possible to force a coherency check:

~~~shell
zpool scrub <pool_name>
~~~

#### OS disk recovery

In case the failed disk was part of the OS pool (rpool) the procedure is the same but further steps are needed to ensure the OS bootability from the replaced disk.


Check that the boot list covers both boot partitions identifiers:

~~~Shell
efibootmgr -v
~~~

You should see 2 entries similar to these ones:

~~~Shell
BootCurrent: 0001
Timeout: 1 seconds
BootOrder: 0001,0002
Boot0001* UEFI OS ....
Boot0002* UEFI OS ....
~~~

If these is not the case, you need to update the boot entries and then check again,
the following command should fix the issue with an interactive prompt:

~~~Shell
dpkg-reconfigure grub-efi-amd64
~~~ 

### Enable Email Alerts

The server should be able to send alerts via mail whenever something bad happens, a SMTP "send-only" server needs to be configured for this purpose.

Most ISP to prevent spam do not allow anymore to send emails via port 25, you can check if port 25 is blocked by your ISP by using this command `telnet <mailserver_address> 25`.
The alternative is to use port 587 but this is a *client* reserved port, so it only works for users subscribed to a mail server and requires authentication with username and password. The mail server will then forward the message to other mail servers using port 25 (without authentication).
Example: tom@foo.com send a mail to carl@bar.info

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

~~~ini
<server_ip> <server_name>.<domain> <server_name>
~~~

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

### Mail notifications for ZFS pools

The `zed` service can be used to monitor the data pool.

Mails are sent whenever a problem is detected and whenever a scrub process complete and a report is available:

~~~Shell
apt install zfs-zed
~~~

Then edit the `/etc/zfs/zed.d/zed.rc` uncommenting/changing the following settings:

~~~ini
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
~~~

#### Fix for zed bug

Unfortunately a bug prevent zed from sending notification if a drive fails when the NAS is off. In that case when the server is started the pool status is correctly set to `DEGRADED` however no mail notifications are sent by zed.

A periodic `cron` job can be scheduled to fix this issue, create a file `zpool-check.sh` in the user directory with the following content:

~~~bash
#! /bin/bash
ZPOOL_STATUS=$(/usr/sbin/zpool status -x)
echo "$ZPOOL_STATUS" | grep -v "all pools are healthy" && mail -r root -s "ZPOOL DEGRADED" root <<< $ZPOOL_STATUS
~~~

Make the file executable
~~~bash
chmod +x zpool-check.sh
~~~

Add the cron job for the user:

~~~shell
crontab -e
~~~

Paste the following line to execute the script every hour:

~~~cron
0 * * * * ./zpool-check.sh
~~~

Check that the new cron table is installed:

~~~shell
crontab -l
~~

