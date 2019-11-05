# Setup arch linux on a raspberry pi

Setup and configuration of arch linux ARM on a raspberry pi 3 model B (+).

## Prerequisites

- sd card with at least 8G of space
- a raspberry pi 3 model B (+)
- an hdmi monitor and cable
- a linux machine to prepare the sd card with
- a working internet connection

## Preparing the sd card

### Set partition table and format sd card

Insert the sd card into a linux machine and get the cards device name using `lsblk`, which should yield a similar output to the following

```bash
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sdx      8:0    0 238,5G  0 disk 
├─sdx1   8:1    0   512M  0 part /boot
├─sdx2   8:2    0   232G  0 part /
└─...
sdz      8:16   1   7,9G  0 disk 
└─sdz1   8:17   1   7,9G  0 part /run/media/...
```

Find the device with the correct size in the list and note the device name, which in this example output is `sdz`.

Next, format the sd card and set an new partition table using [fdisk][1]. 

```bash
sudo fdisk /dev/sdz
```

#### Set partition table

Follow the steps below

1. `o`. This will clear out any partitions on the drive.
2. `p` to list partitions. There should be no partitions left.
3. `n`, then p for primary, `1` for the first partition on the drive, press ENTER to accept the default first sector, then type +100M for the last sector.
4. `t`, then `c` to set the first partition to type W95 FAT32 (LBA).
5. `n`, then `p` for primary, `2` for the second partition on the drive, and then press ENTER twice to accept the default first and last sector.
6. `w`, Write the partition table and exit with `q`

#### Format the partitions

```bash
sudo mkfs.vfat /dev/sdz1
# Create boot folder in ~/
mkdir boot
# Mount boot partition to boot folder
sudo mount /dev/sdz1 boot

sudo mkfs.ext4 /dev/sdz2
# Create root folder in ~/
mkdir root
# Mount root partition to root folder
sudo mount /dev/sdz2 root
```

### Write os to sd card

The version of arch linux used here is [arch linux ARM][2], an ARM optimized port of the official [arch linux][3] distribution. Download the distribution and write it to the sd card

```bash
wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-2-latest.tar.gz
sudo bsdtar -xpf ArchLinuxARM-rpi-2-latest.tar.gz -C root
sync

sudo mv root/boot/* boot
sync

sudo umount boot root

# clean up root and boot folders
rm -rf ~/root
rm -rf ~/boot
```

> The 2 in the tarball name `ArchLinuxARM-rpi-2-latest.tar.gz` is not related to the raspberry pi version 2 and is valid for a raspberry pi 3 model B (+) as well.

After this is done, unmount the sd card, plug it into the raspberry pi and boot it up.

## Arch linux configuration on raspberry pi

The default user is `alarm` with the password being the user name. the root user is `root` with the password being the user name as well. Login as root.

Depending on your keyboard layout, it may be advisable to map the correct keys

```bash
# Load german keymap
loadkeys de
```

### User managment

```bash
# Set new root password
passwd
# Delete default user
userdel -r alarm
# Add new user
useradd -m -g users -s /bin/bash <username>
# Set user password
passwd <username>

# Set hostname, ie. name of the raspberry pi
echo <hostname> > /etc/hostname
```

### Set locale information, eg. time zone and keymap

```bash
rm /etc/localtime
ln -s /usr/share/zoneinfo/Europe/Berlin /etc/localtime

# Keyboard
echo KEYMAP=de-latin1 > /etc/vconsole.conf

nano /etc/locale.gen

# Find and uncomment the following entries:
#en_US.UTF-8 UTF-8
#en_US ISO-8859-1

# Generate new locale
locale-gen
```

## Add swapfile

Adding a swapfile is not absolutely necessary. Based on the type of work the raspberry pi will be used for a swapfile is advisable. A swap partition is another option that can be implemented at [this](#set-partition-table) stage of the process.

```bash
# Allocation 3G of space to swapfile
fallocate -l 3G /swapfile
# On Error
#dd if=/dev/zero of=/swapfile bs=1M count=3072

# Set correct permissions
chmod 600 /swapfile
# Format swapfile and start it
mkswap /swapfile
swapon /swapfile

# Append swapfile entry to /etc/fstab
echo "/swapfile none swap defaults 0 0" >> /etc/fstab
```

## Configure wifi

Configure `wpa_supplicant` to use wifi instead of wired lan.

```bash
# Get interface name
# usually wlan0
ip link

# Activate interface
ip link set <interface-name> up

# Find router
iw dev <interface-name> scan | less

# Connect to router using SSID
iw <interface-name> connect <router-ssid>

# Configure wpa_supplicant
wpa_passphrase \
	"<router-ssid>" \
	"<p<ssword>" \
	> /etc/wpa_supplicant/wpa_supplicant.conf
wpa_supplicant \
	-i <interface-name> \
	-D wext \
	-c /etc/wpa_supplicant/wpa_supplicant.conf \
	-B

# Run dhcp
dhcpcd <interface-name>
```

The device might not be able to get a connection and need access to wifi be granted by router entry. This can be done by adding a wifi device in wifi security options in the router management console.

After the `dhcpcd <interface-name>` command yields a valid connection it is useful to have the pi connect on startup

```bash
# Copy and edit example config file
cp /etc/netctl/example/wireless-wpa /etc/netctl/auto-connect
nano /etc/netctl/auto-connect
# Use wpa_passphrase PSK instead of key
# Key='<psk>'
systemctl enable netctl-auto@wlan0.service
```

The _wpa_passphrase PSK_ can be found in `/etc/wpa_supplicant/wpa_supplicant.conf`.

## Update repositories and enable services

Install relevant software and enable time synchronisation service

```bash
pacman -Syyu
pacman -S sudo
pacman -S dnsutils
pacman -S ntp
pacman -S vim

systemctl start ntpd.service
systemctl enable ntpd.service
```

This may require the pacman keys to be initialized and populated

```bash
pacman-key --init
pacman-key --populate archlinuxarm
```

Since sudo is now installed, add [previously](#user-managment) created users to sudo group wheel

```bash
nano /etc/sudoers

# Uncomment
#%wheel ALL=(ALL) ALL

gpasswd -a <username> wheel
```

## Configure SSH

For secure remote connections to the raspberry pi, set up [openssh][4] using public key cryptography.

Get the lan address of the raspberry pi using `ip addr` on pi or by looking up the correct entry in the router.

```bash
# Simple keyless ssh
ssh <username>@<pi-lan-address>
```

On the linux machine used to connect to remote raspberry pi, generate a new keypair

```bash
ssh-keygen -t <keytype>
# Changing the passphrase for an existing key:
#ssh-keygen -f ~/.ssh/<keyname> -p
```

Check for correct permissions for the key

```bash
sudo chmod 600 ~/.ssh/<keyname>
```

Copy the key from client to pi

```bash
ssh-copy-id -i ~/.ssh/<keyname>.pub <username>@<pi-lan-address>
```

Lock `authorized_keys` file to prevent manipulation

```bash
sudo chmod 400 ~/.ssh/authorized_keys
echo "VisualHostKey=yes" > ~/.ssh/config
```

Login to remote raspberry pi via pkc

```bash
ssh -i ~/.ssh/<keyname>.pub <username>@<pi-lan-address>
# Login with encrypted SOCKS tunnel
ssh -i ~/.ssh/<keyname>.pub -TND 4711 <username>@<pi-lan-address>
```

On pi, make sure permissions for ssh folder are correct

```bash
sudo chmod 700 ~/.ssh
```

Prevent simple password login on pi

```bash
sudo nano /etc/ssh/sshd_config
# Find the following line, set to no and uncomment
#PasswordAuthentication no
```

Customize the welcome text and banner shown when logging into remote machine using ssh

```bash
# Edit welcome text
sudo vim /etc/motd
# Add or edit banner
sudo vim /etc/ssh/sshd_config
# Change line 'Banner none' to 'Banner /etc/banner'
```


[1]: http://tldp.org/HOWTO/Partition/fdisk_partitioning.html
[2]: https://archlinuxarm.org/
[3]: https://www.archlinux.org/
[4]: https://www.openssh.com/manual.html