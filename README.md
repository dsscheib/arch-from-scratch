# Arch Linux Installation Guide
> See [ArchWiki](https://wiki.archlinux.org/title/Installation_guide) for additional details

## WiFi

Enter iwctl

```sh
iwctl
```

When inside, check for the name of your wireless devices.

```sh
device list
```

If your device name is wlan0, connect using the following command

```sh
station wlan0 connect <SSID>
```

Make sure to enter in your password

exit when complete

```sh
exit
```

## SSH for remote installation

Enable sshd (should be done by default)

```sh
systemctl enable sshd
systemctl start sshd
```

Set a password for the current user

```sh
passwd
```

Get IP address

```sh
ip addr
```

## Prepare disk(s) for encryption

List blocks. Could be nvme drives, such as nvme0 or nvme1. Could be sdx drives, such as sda or sdb.

```sh
lsblk
```

### SSD memory cell clearing
> See [ArchWiki](https://wiki.archlinux.org/title/Solid_state_drive/Memory_cell_clearing) for additional details

Make sure disk supports nvme sanitize

```sh
nvme id-ctrl /dev/nvme0 -H | grep -E 'Format |Crypto Erase|Sanitize'
```

The *device* should be /dev/nvme0 and **not** /dev/nvme0n1, for example
```sh
nvme sanitize <device> -a start-block-erase
```

Check for completion

```sh
nvme sanitize-log /dev/nvme0
```

### Wipe disk(s)
> See [ArchWiki](https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation) for additional details

Create a temporary encrypted container

```sh
cryptsetup open --type plain --key-file /dev/urandom --sector-size 4096 /dev/nvme0 to_be_wiped
```

Verify that it exists

```sh
lsblk
```

Wipe the container with zeros

```sh
dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress bs=1M
```

Close the temporary container

```sh
cryptsetup close to_be_wiped
```

## Partitioning disk(s)

Get the names of the blocks

```sh
lsblk
```

For both partition setups, you'll want to setup a table on your primary drive.

```sh
gdisk /dev/nvme0n1
```

Inside of gdisk, you can print the table using the `p` command.

To create a new partition use the `n` command. The below table shows 
the disk setup I have for my primary drive

| partition | first sector | last sector | code |
|-----------|--------------|-------------|------|
| 1         | default      | +512M       | ef00 |
| 2         | default      | +4G         | ef02 |
| 3         | default      | default     | 8309 |

If you have a second drive for your home disk, then your table would be as 
follows.

| partition | first sector | last sector | code |
|-----------|--------------|-------------|------|
| 1         | default      | default     | 8302 |

Write changes to disk using the `w` command

## Encryption

Load the encryption modules to be safe.

```sh
modprobe dm-crypt
modprobe dm-mod
```

Setting up encryption on our luks lvm partiton

```sh
cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme0n1p3
```

Enter in your password and **Keep it safe**. There is no "forgot password" here.


If you have a home partition, then initialize this as well

```sh
cryptsetup luksFormat -v -s 512 -h sha512 /dev/nvme1n1p1
```

Mount the drives:

```sh
cryptsetup open /dev/nvme0n1p3 luks_lvm
```

If you have a home parition:

```sh
cryptsetup open /dev/nvme1n1p1 arch-home
```

## Volume setup

Create the volume and volume group

```sh
pvcreate /dev/mapper/luks_lvm
vgcreate arch /dev/mapper/luks_lvm
```

Create a volume for your swap space. A good size for this is your RAM size + 2GB.
In my case, 32GB of RAM + 2GB = 34G.

```sh
lvcreate -n swap -L 34G arch
```

Next you have a few options depending on your setup

### Single Disk
If you have a single disk, you can either have a single volume for your root 
and home, or two separate volumes.

#### Single volume 

Single volume is the most straightforward. To do this, just use the entire
disk space for your root volume

```sh
lvcreate -n root -l +100%FREE arch
```

#### Two volumes

For two volumes, you'll need to estimate the max size you want for either your
root or home volumes. With a root volume of 200G, this looks like:

```sh
lvcreate -n root -L 200G arch
```

Then use remaining disk space for home

```sh
lvcreate -n home -l +100%FREE arch
```

### Dual Disk

If you have two disks, then create a single volume on your LVM disk.

```sh
lvcreate -n root -l +100%FREE arch
```


## Filesystems

FAT32 on EFI partiton

```sh
mkfs.fat -F32 /dev/nvme0n1p1 
```

EXT4 on Boot partiton

```sh
mkfs.ext4 /dev/nvme0n1p2
```

BTRFS on root

```sh
mkfs.btrfs -L root /dev/mapper/arch-root
```

BTRFS on home if exists

```sh
mkfs.btrfs -L home /dev/mapper/arch-home
```

Setup swap device

```sh
mkswap /dev/mapper/arch-swap
```

## Mounting

Mount swap

```sh
swapon /dev/mapper/arch-swap
swapon -a
```

Mount root 

```sh
mount /dev/mapper/arch-root /mnt
```

Create home and boot

```sh
mkdir -p /mnt/{home,boot}
```

Mount the boot partiton

```sh
mount /dev/nvme0n1p2 /mnt/boot
```

Mount the home partition if you have one, otherwise skip this

```sh
mount /dev/mapper/arch-home /mnt/home
```

Create the efi directory

```sh
mkdir /mnt/boot/efi
```

Mount the EFI directory

```sh
mount /dev/nvme0n1p1 /mnt/boot/efi
```

## Install arch

```sh
pacstrap -K /mnt base linux linux-firmware
```

With base-devel

```sh
pacstrap -K /mnt base base-devel linux linux-firmware
```

Load the file table

```sh
genfstab -U -p /mnt > /mnt/etc/fstab
```

chroot into your installation

```sh
arch-chroot /mnt /bin/bash
```

## Configuring

### Text Editor

Install a text editor

```sh
pacman -S --noconfirm --needed neovim
```

### Decrypting volumes

Open up mkinitcpio.conf

```sh
nvim /etc/mkinitcpio.conf
```

add `encrypt` and `lvm2` into the hooks

```vim
HOOKS=(... block encrypt lvm2 filesystems fsck)
```

install lvm2

```sh
pacman -S --noconfirm --needed lvm2
```

### Bootloader

Install grub and efibootmgr

```sh
pacman -S --noconfirm --needed grub efibootmgr
```

Setup grub on efi partition

```sh
grub-install --efi-directory=/boot/efi
```

obtain your lvm partition device UUID

```sh
blkid /dev/nvme0n1p3
```

Copy this to your clipboard

```sh
nvim /etc/default/grub
```

Add in the following kernel parameters

```sh
root=/dev/mapper/arch-root cryptdevice=UUID=<uuid>:luks_lvm
```

### Keyfile

```sh
mkdir /secure
```

Root keyfile

```sh
dd if=/dev/random of=/secure/root_keyfile.bin bs=512 count=8
```

Home keyfile if home partition exists

```sh
dd if=/dev/random of=/secure/home_keyfile.bin bs=512 count=8
```

Change permissions

```sh
chmod 000 /secure/*
chmod 600 /boot/initramfs-linux*
```

Add to partitions

```sh
cryptsetup luksAddKey /dev/nvme0n1p3 /secure/root_keyfile.bin
# skip below if using single disk
cryptsetup luksAddKey /dev/nvme1n1p1 /secure/home_keyfile.bin
```

```sh
nvim /etc/mkinitcpio.conf
```

```vim
FILES=(/secure/root_keyfile.bin)
```

### Home Partition Crypttab (Skip if single disk)

Get uuid of home partition

```sh
blkid /dev/nvme1n1p1
```

Open up the crypt table.

```sh
nvim /etc/crypttab
```

Add in the following line at the bottom of the table

```vim
arch-home      UUID=<uuid>    /secure/home_keyfile.bin
```

Reload linux

```sh
mkinitcpio -p linux
```

## Grub

Create grub config

```sh
grub-mkconfig -o /boot/grub/grub.cfg
grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```

## System Configuration

### Timezone

```sh
ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime
```

### NTP

```sh
nvim /etc/systemd/timesyncd.conf
```

Add in the NTP servers

```vim
[Time]
NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org 
FallbackNTP=0.pool.ntp.org 1.pool.ntp.org
```

Enable timesyncd

```sh
systemctl enable systemd-timesyncd.service
```

### Locale

```sh
nvim /etc/locale.gen
```

uncomment the UTF8 lang you want

```vim
en_US.UTF-8 UTF-8
```

```sh
locale-gen
```

```sh
nvim /etc/locale.conf
```

```vim
LANG=en_US.UTF-8
```


### hostname

enter it into your /etc/hostname file

```sh
$ nvim /etc/hostname
```

or 

```sh
echo "mymachine" > /etc/hostname
```

### Users

First secure the root user by setting a password

```sh
passwd
```

Then install the shell you want

```sh
pacman -S --noconfirm --needed fish
```

Add a new user as follows

```sh
useradd -m -G wheel -s /usr/bin/fish <user>
```

set the password on the user

```sh
passwd <user>
```

Add the wheel group to sudoers

```sh
EDITOR=nvim visudo
```

```vim
%wheel ALL=(ALL:ALL) ALL
```

### Network Connectivity

```sh
pacman -S --noconfirm --needed networkmanager
systemctl enable NetworkManager
```

### Display Manager

Gnome

```sh
pacman -S --needed gnome
```

```sh
systemctl enable gdm
```

Hyprland

```sh
# Do not install sddm if you install both Gnome and Hyprland
pacman -S --needed hyprland uwsm sddm kitty
```

```sh
systemctl enable sddm
```

### Microcode

For AMD

```sh
pacman -S --noconfirm --needed amd-ucode
```

For Intel

```sh
pacman -S --noconfirm --needed intel-ucode
```

```sh
grub-mkconfig -o /boot/grub/grub.cfg
grub-mkconfig -o /boot/efi/EFI/arch/grub.cfg
```


## Reboot

```sh
exit
umount -R /mnt
reboot now
```
