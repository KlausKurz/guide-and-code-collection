# backup a running proxmox zfs bare metal host<br>and restore or clone to new hardware
This is a proof of concept for backup and restore or clone a proxmox zfs bare metal host on a home lab system.<br>This is not about VM backup.<br> 
**For production environments more tests are necessary.**

## TL;TR
````
# backup running system to <destination>
zfs snapshot -r  rpool@export
zfs send -R rpool@export | gzip  > <destination>/backup-pve-rpool@export.gz
sgdisk --backup=<destination>/sgdisk-dev-sda /dev/sda
dd bs=64M if=/dev/sda1 | gzip  > <destination>/dev-sda1.gz
dd bs=64M if=/dev/sda2 | gzip  > <destination>/dev-sda2.gz
````
````
# restore with rescue console on new hardware from <source>
sgdisk --load-backup=<source>/sgdisk-dev-sda /dev/sda
gzip -c -d <source>/dev-sda1.gz | dd of=/dev/sda1
gzip -c -d <source>/dev-sda2.gz | dd of=/dev/sda2
zpool create -f -o ashift=12 rpool /dev/disk/by-id/<ata-xxxxxxxxxxxx>-part3 
gzip -c -d <source>/backup-pve-rpool@export.gz | zfs receive -u -v -F rpool
zfs set mountpoint=/rpool       rpool
zfs set mountpoint=/rpool/ROOT  rpool/ROOT
zfs set mountpoint=/            rpool/ROOT/pve-1
zfs set mountpoint=/rpool/data  rpool/data
zfs set mountpoint=/var/lib/vz  rpool/var-lib-vz
zpool set cachefile=none rpool
zpool export rpool
````

## test scenario
````
  2 x # Intel® NUC-Kit NUC6i3SYH
    each 1 x SATA and 1 x m.2 SATA disk, 265 GB each
    8 GB RAM
    latest bios v 0073
````
## source server configuration
````
vanilla pve layout definded by pve installer
    proxmox-ve: 8.1.0 (running kernel: 6.5.11-7-pve)
    pve-manager: 8.1.3 (running version: 8.1.3/b46aac3b42da5d15)
ZFS mirror on root device
no VM, no LXC, no cluster
bios boot
````
````
zfs list | grep "^rpool"
````
````
rpool                          1.95G   221G    96K  /rpool
rpool/ROOT                     1.93G   221G    96K  /rpool/ROOT
rpool/ROOT/pve-1               1.93G   221G  1.93G  /
rpool/data                       96K   221G    96K  /rpool/data
rpool/var-lib-vz                 96K   221G    96K  /var/lib/vz
````
````
zpool status rpool
````
````
  pool: rpool
 state: ONLINE
config:

        NAME                                               STATE     READ WRITE CKSUM
        rpool                                              ONLINE       0     0     0
          mirror-0                                         ONLINE       0     0     0
            ata-WDC_WDSxxxxxxxxxxxxxxxxxxxxxxxxxxxx-part3  ONLINE       0     0     0
            ata-SanDisk_SDSSDHP256G_xxxxxxxxxxxx-part3     ONLINE       0     0     0
````
````
fdisk -l /dev/sd{a,b}
````
````
  Device       Start       End   Sectors  Size Type
  /dev/sda1       34      2047      2014 1007K BIOS boot
  /dev/sda2     2048   2099199   2097152    1G EFI System
  /dev/sda3  2099200 486539264 484440065  231G Solaris /usr & Apple ZFS

  Device       Start       End   Sectors  Size Type
  /dev/sdb1       34      2047      2014 1007K BIOS boot
  /dev/sdb2     2048   2099199   2097152    1G EFI System
  /dev/sdb3  2099200 486539264 484440065  231G Solaris /usr & Apple ZFS
````
## steps taken
````
1. backup source server (ZFS mirror, rpool, boot devices)
	  zfs create snapshot
	  zfs snapshot send
	  disklayout copy
	  dd image boot devices
            to transfer disk
2. restore target server (same disks like source server)
      restore disk layout
      restore dd images
      create pool
      restore zfs receive
      resilver 2nd disk
      check mountpoints
````
## backup
An USB stick is used for the filetransfer.
### stop the necessary proxmox services
Before taking a snapshot, stop as many running services as possible to ensure consistency.
````
systemctl stop pve-cluster
systemctl stop pvedaemon
systemctl stop pveproxy
````
### make snapshot
````
zfs snapshot -r  rpool@export
````

### start the necessary proxmox services
````
systemctl start pve-cluster
systemctl start pvedaemon
systemctl start pveproxy
````
### check snapshot creation
````
zfs list -t snapshot | grep "^rpool"
````
````
  rpool@export                        0B      -    96K  -
  rpool/ROOT@export                   0B      -    96K  -
  rpool/ROOT/pve-1@export          36.9M      -  1.93G  -
  rpool/data@export                   0B      -    96K  -
  rpool/var-lib-vz@export             0B      -    96K  -
````
### backup to file
````
zfs send -R rpool@export | gzip  > <destination>/backup-pve-rpool@export.gz
````
### backup partition table
````
sgdisk --backup=<destination>/sgdisk-dev-sda /dev/sda
sgdisk --backup=<destination>/sgdisk-dev-sdb /dev/sdb
````
### backup sd{a,b}{1,2}
#### trim (optional)
````
mkidr /media/sda2
mount /dev/sda2 /media/sda2
fstrim -v /media/sda2

mkidr /media/sdb2
mount /dev/sdb2 /media/sdb2
fstrim -v /media/sdb2

df -h /media/sd{a,b}2
````
````
  Filesystem      Size  Used Avail Use% Mounted on
  /dev/sda2      1022M  150M  873M  15% /media/sda2
  /dev/sdb2      1022M  150M  873M  15% /media/sdb2
umount /media/sda2
umount /media/sdb2
````
#### backup to file
````
dd bs=64M if=/dev/sda1 | gzip  > <destination>/dev-sda1.gz
dd bs=64M if=/dev/sda2 | gzip  > <destination>/dev-sda2.gz
dd bs=64M if=/dev/sdb1 | gzip  > <destination>/dev-sdb1.gz
dd bs=64M if=/dev/sdb2 | gzip  > <destination>/dev-sdb2.gz
````
#### files to transfer
````
backup-pve-rpool@export.gz
dev-sda1.gz
dev-sda2.gz
dev-sdb1.gz
dev-sdb2.gz
sgdisk-dev-sda
sgdisk-dev-sdb
````
## restore target system
 boot proxmox installer on tartget system
 go to shell with \<ALT F3> while in installer
### setup ssh to make easy copy/paste
#### check network
````
ping zfsonlinux.org
````
#### update apt
````
apt update
````
#### install ssh
````
apt install ssh
 # let root login
nano /etc/ssh/sshd_config
  PermitRootLogin yes
/etc/init.d/ssh start
 # set password
passwd
````
### login with ssh on target system
#### get host information
````
fdisk -l
lsblk
ls -l -r /dev/disk/by-*
````
#### delete disk layout (be careful, check twice)
````
sgdisk  -o /dev/sda
sgdisk  -o /dev/sdb
````
#### restore disk (be careful, check twice)
````
sgdisk --load-backup=<source>/sgdisk-dev-sda /dev/sda
sgdisk --load-backup=<source>sgdisk-dev-sdb /dev/sdb
````
#### look at the layout
````
sgdisk  -p /dev/sda
sgdisk  -p /dev/sdb
````
#### restore content
````
gzip -c -d <source>/dev-sda1.gz | dd of=/dev/sda1
gzip -c -d <source>/dev-sdb1.gz | dd of=/dev/sdb1

gzip -c -d <source>/dev-sda2.gz | dd of=/dev/sda2
gzip -c -d <source>/dev-sdb2.gz | dd of=/dev/sdb2
````
#### create pool 
````
zpool create -f -o ashift=12 rpool mirror /dev/disk/by-id/ata-Micron_1100_SATA_256GB_xxxxxxxxxxxx-part3 /dev/disk/by-id/ata-Samsung_SSD_860_EVO_xxx_xxxxxxxxxxxxxxx-part3
````
#### get all properties
````
zfs get all rpool
````
#### restore pool
dry run
````
gzip -c -d <source>/backup-pve-rpool@export.gz | zfs receive -u -v -F -n rpool
````
do it
````
gzip -c -d <source>/backup-pve-rpool@export.gz | zfs receive -u -v -F rpool
````
#### check
````
zpool status
zfs list
````
#### prepare for boot
````
zfs set mountpoint=/rpool       rpool
zfs set mountpoint=/rpool/ROOT  rpool/ROOT
zfs set mountpoint=/            rpool/ROOT/pve-1
zfs set mountpoint=/rpool/data  rpool/data
zfs set mountpoint=/var/lib/vz  rpool/var-lib-vz
 # check
zpool status
zfs list
  # export the pool
zpool set cachefile=none rpool
zpool export  rpool
````
#### poweroff
````
poweroff -f
````
#### start server
import rpool (sometimes neccecary if in initramfs console)
````
zpool import rpool -f
````
#### set ip
````
nano /etc/network/interfaces
systemctl restart networking.service
````
#### rename node
https://pve.proxmox.com/wiki/Renaming_a_PVE_node
checck boot failures
````
journalctl -b
````
check host settings for ip change
````
nano /etc/hosts  /etc/hostname  /etc/mailname /etc/postfix/main.cf
````

## other methodes
Install a running proxmox on the target server and then apply the zfs backup via rescue console. The disk layout can be diferent on the target system.


