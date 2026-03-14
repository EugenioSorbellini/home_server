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

