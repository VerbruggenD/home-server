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
- [Insert linux distro] for running all production docker services.
- [Insert linux distro] for testing and experimenting (staging before production).
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
