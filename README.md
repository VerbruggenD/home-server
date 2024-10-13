# Home server setup

## Hardware

- **CPU**: [Intel Core i7-12700 2.1 GHz (4.9 GHz Turbo Boost)](https://www.alternate.be/Intel/Core-i7-12700-2-1-GHz-(4-9-GHz-Turbo-Boost)-socket-1700-processor/html/product/1778354)
- **Motherboard**: [ASUS PRIME Z790-P (Socket 1700)](https://www.alternate.be/ASUS/PRIME-Z790-P-socket-1700-moederbord/html/product/1868082)
- **Memory**: [Kingston FURY 64 GB DDR5-4800 Kit](https://www.alternate.be/Kingston-FURY/64-GB-DDR5-4800-Kit-werkgeheugen/html/product/1826099)
- **Power Supply**: [Corsair SF850 850 Watt](https://www.alternate.be/Corsair/SF850-850-Watt-voeding/html/product/100061554)
- **Boot SSD**: [Crucial P3 500 GB SSD](https://www.alternate.be/Crucial/P3-500-GB-SSD/html/product/1854379)
- **HDD**: [Seagate Exos 7E10 8 TB HDD](https://www.alternate.be/Seagate/Exos-7E10-8-TB-harde-schijf/html/product/1789459)

## Software
This is an overview of the software stack used on the server.
The bare metal is running Proxmox VE as a hypervisor.

These VM's are running on proxmox:
- Truenas scale for managing all HDD's.
- Debian for running all production docker services.
- Debian for testing and experimenting (staging before production).
- Possibly also windows or other VM's for desktop replacement and expanding the domotica setup.

This is a list of docker containers deployed on the server:
- Nextcloud
- PiHole as a dns server and add blocking (maybe test also adguard home)
- Vaultwarden
- Uptime Kuma
- NGINX reverse proxy manager: for internal redirects from the subdomains to the correct docker containers
- Jellyfin

### Proxmox installation
- Download proxmox VE iso (in this case 8.2-2)
- Download balena etcher if not installed yet
- Run balena etcher with admin rights to flash a usb drive
- Disconnect all storage except the boot drive and boot from usb

After installation update the os:
```
apt update
apt upgrade
```

You can set up cpu temp sensors to get the current temps of the CPU.
```
apt install lm-sensors
sensors-detect
```

Go through the detection process, you can skip the I2C scanning.

Create the service and request the temps using:
```
/etc/init.d/kmod start
sensors
```

### Setting up users
- Create a user group "Admins" under ``Datacenter >> Permissions >> Groups``
- Create the needed user accounts under ``Datacenter >> Permissions >> Users``
    E.g.: username: dieter, Realm: Proxmox authentication server, Group: Admins

### Setting up email notifications
TODO!!!
Still need to figure out how this would work.

## Setting up Truenas
TODO:
- pass through HDD's to VM and create ZFS pool
- setup network shares and permissions (users)
  - set a docker user
  - set a media-server user
  - set a nextcloud user
  - set personal users
 
### Setting up notifications using telegram
Already had botfather configured on my telegram.
Used ``/newbot`` to create the @nasnotifyalertbot, friendly chat name is Truenas-notifications.
Opened the conversation from the botfather prompt.
Used @userinfobot to get my ID and copied the token from the botfather prompt.

Tested the config through Truenas.

### Passing a sata drive to a VM
On the proxmox shell:
Install lshw
```
apt install lshw
```

``lshw -class disk -class storage`` command shows the drives, like show here

```
  *-sata
       description: SATA controller
       product: Intel Corporation
       vendor: Intel Corporation
       physical id: 17
       bus info: pci@0000:00:17.0
       logical name: scsi4
       version: 11
       width: 32 bits
       clock: 66MHz
       capabilities: sata msi pm ahci_1.0 bus_master cap_list emulated
       configuration: driver=ahci latency=0
       resources: irq:147 memory:86000000-86001fff memory:86003000-860030ff ioport:7090(size=8) ioport:7080(size=4) ioport:7060(size=32) memory:86002000-860027ff
     *-disk
          description: ATA Disk
          product: ST2000DM001-1ER1
          physical id: 0.0.0
          bus info: scsi@4:0.0.0
          logical name: /dev/sda
          version: CC26
          serial: Z56041K1
          size: 1863GiB (2TB)
          capabilities: gpt-1.00 partitioned partitioned:gpt
          configuration: ansiversion=5 guid=53be006c-8945-4f36-a0a9-74f9f36bb31b logicalsectorsize=512 sectorsize=4096
```

To get the ID for passing through by id instead of mount point do the following command: 
``ls -l /dev/disk/by-id | grep $serialNumber`` replace the ``$serialNumber`` with the number from earlier.

This gives something like this:
``
lrwxrwxrwx 1 root root  9 Oct  5 16:44 ata-ST2000DM001-1ER164_Z56041K1 -> ../../sda
``

Use the following to list all drives:
```
lsblk |awk 'NR==1{print $0" DEVICE-ID(S)"}NR>1{dev=$1;printf $0" ";system("find /dev/disk/by-id -lname \"*"dev"\" -printf \" %p\"");print "";}'|grep -v -E 'part|lvm'
```

Use the following command to attach the drive to the correct VM:
make sure to replace the VM id and drive id with the correct values.

!!! Make sure that scsi2 name is unique, so check first if it exists !!!

```
qm set $VM_ID -scsi2 /dev/disk/by-id/$DRIVE_ID
```

To disconnect the drive from the VM use:
```
qm unlink $VM_ID --idlist scsi2
```

For the truenas VM it currently has a VM-id of 100. The commands used are:
```
qm set 100 -scsi2 /dev/disk/by-id/$DRIVE_ID
qm unlink 100 --idlist scsi2
```
Do this for every drive you want access to in Truenas.

### Setting up zfs pool:
TODO

## Setting up production VM
Vm settings:
- start at boot
- VirtIO-GPU
- VirtIO SCSI
- assign resources (eg 4Gb ram, 2 cores)

Install debian on the vm.
I used user as new user.
Set regions and keyboard layout.
!! Make sure that SSH server is installed during install. This is then by default enabled.

Update the os first, it needs to run as root with ``su root``:
```
apt update
apt upgrade
```

### Installing docker
As root:
```
apt install apt-transport-https ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
sudo apt install docker-ce docker-ce-cli containerd.io
```

Check the installed version:
```
docker --version
```

Change user rights to run dockers from the "user"
```
sudo usermod -aG docker user
```

### Mounting AppData netshare at boot
Create an appdata dataset in truenas and a user "docker" in the usergroup "appUsers".
Create an SMB share (private) so it is undetected in the network unlike the other shares.

!!! Figure out how to only have access to the share with 1 user.

From root install cifs
```
apt install cifs-utils
```

Create the mount point like this: ``mkdir -p /mnt/appdata``

You can manually mount (once per boot) with
```
sudo mount -t cifs -o username=docker,password=$userpassword //truenas.local/AppData /mnt/appdata
```

To always mount at boot modify the file ``nano /etc/fstab`` to add the following at the end.
```
//truenas.local/AppData /mnt/appdata cifs username=docker,password=Nskctl10,uid=1000,gid=1000,iocharset=utf8 0 0
//truenas.local/media /mnt/media cifs username=docker,password=Nskctl10,uid=1000,gid=1000, iocharset=utf8 0 0
//truenas.local/nextcloud_db /mnt/nextcloud_db cifs username=docker,password=Nskctl10,uid=999,gid=999, iocharset=utf8 0 0
//truenas.local/nextcloud /mnt/nextcloud cifs  rw,mfsymlinks,seal,username=docker,password=Nskctl10,uid=33,gid=0,file_mode=0770,dir_mode=0770 0 0
```

Here the nextcloud instance has special parameters to enable symlinks and user the correct user (uid=33). The database share needs to use uid 999 for able to work.

### Installing nextcloud

As earlier mentioned, the nextcloud instance needs special permissions to use the net share as the data directory and database dir.

### Installing Jellyfin

The media share is mounted and the docker uses this as the data volumne for media.

