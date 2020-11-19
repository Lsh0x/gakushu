# Dual boot window/arch with full encrypted linux disk


## Download the arch and window iso.

dd if=<PATH TO ISO> of=<PATH TO USB> bs=4096 status=progress

ex:

```sh
dd if=$HOME/Downloads/archlinux-2020.11.01-x86_64.iso of=/dev/sdb1 status=progress
```

If you are on other machine check google with "creating bootable usb on <YOUR OS>"
Insert the usb drive and boot on it. It's better to have two usb drive
But one can be enough  if you use dd to make the bootable needed usb drive.


Troubleshooting:

* Be carefull i saw some issue on OSX to create bootable image for window

Sources for help:

* https://www.howtogeek.com/howto/linux/create-a-bootable-ubuntu-usb-flash-drive-the-easy-way/ 
* https://ubuntu.com/tutorials/create-a-usb-stick-on-ubuntu#1-overview
* https://ubuntu.com/tutorials/create-a-usb-stick-on-macos#1-overview
* https://medium.com/@tbeach/use-unix-dd-command-to-os-bootable-on-usb-drive-6671945d95a6
* https://www.freecodecamp.org/news/how-make-a-windows-10-usb-using-your-mac-build-a-bootable-iso-from-your-macs-terminal/


## Prepare EFI

The Window installer will only create a partition of 100M.
I recommand to create your own before to size it as you whish 
To avoid this, can create the EFI partition before running the window installation.

## Partioning and format the disk


We boot the arch iso to access sgdisk

You have multiple command line help to work with disk.
I choose to work with sgdisk

Here a non exhautive list: 
* parted
* fdisk
* sgdisk

```
#export $DISK=/dev/<DISK>

# in this example we use nvme disk 
# so when going to partition we add DISKp<x> ex: nvme0n1p1
# for other just add the partition number sda1
export $DISK=/dev/nvme0n1

sgdisk --zap-all $DISK

# EFI

sgdisk --new=1:0:+500M $DISK
sgdisk --typecode=1:ef00 $DISK
sgdisk --change-name=1:EFI $DISK
mkfs.fat -F 32 ${DISK}p1 -n EFI

# GRUB

sgdisk --new=2:0:+2M $DISK
sgdisk --typecode=2:ef02 $DISK
sgdisk --change-name=2:GRUB

# BOOT

sgdisk --new=3:0:+500M $DISK
sgdisk --typecode=2:8301 $DISK
sgdisk --change-name=3:boot $DISK
cryptsetup luksFormat --type=luks1 --use-random -S 1 -s 512 -h sha512 -i 5000 ${DISK}p3
cryptsetup open ${DISK}p3 boot
mkfs.ext4 /dev/mapper/boot -L boot

```


## Install window 

Change a size at the end of the disk used.
The window installer will use the EFI partition and add its own entry
This will allow you to define the size of the EFI partiion and not deal with the 100M from the windown installer
When window is installed, we are ready to process the installation.

Note: i created the boot and grub partition but can be probably done after the window installation.
It's a choice i did to have the boot efi and grub at the begininig of the disk. 
Feel like you want, just after the installation do the GRUB and BOOT partionning/formattage of the disks.



## Install arch linux

## Setting environment 

```
timedatectl set-ntp true
```

### Preparing the disk (again)

```
#export DISK=<YOUR DISK AGAIN>
export DISK=/dev/nvme0n1

# Rootfs /
sgdisk --new=6:0:+50G $DISK
sgdisk --typecode=6:8301 $DISK
sgdisk --change-name=6:rootfs
cryptsetup luksFormat --use-random -S 1 -s 512 -h sha512 -i 5000 ${DISK}6
cryptsetup open ${DISK}p6 rootfs
mkfs.ext4 /dev/mapper/rootfs -L rootfs


# usr /usr
sgdisk --new=7:0:+50G $DISK
sgdisk --typecode=7:8301 $DISK
sgdisk --change-name=7:usr $DISK
cryptsetup luksFormat --use-random -S 1 -s 512 -h sha512 -i 5000 ${DISK}p7
cryptsetup open ${DISK}p7 usr


# var /var

sgdisk --new=8:0:+50G $DISK
sgdisk --typecode=8:8301 $DISK
sgdisk --change-name=8:var $DISK
cryptsetup luksFormat --use-random -S 1 -s 512 -h sha512 -i 5000 ${DISK}p8
cryptsetup open ${DISK}p8 var

# lvm for the rest

sgdisk --new=9:0:0 $DISK
sgdisk --typecode=9:8301 $DISK
sgdisk --change-name=9:lvm $DISK
cryptsetup luksFormat --use-random -S 1 -s 512 -h sha512 -i 5000 ${DISK}p9
cryptsetup open ${DISK}p9 lvm

# setup lvm partition
pvcreate /dev/mapper/lvm
vgcreate arch /dev/mapper/lvm

# set swap
lvcreate -L 8G -n swap arch
mkswap /dev/mapper/arch-swap -L swap

# set home
lvcreate -l 80%FREE -n home arch
mkfs.ext4 /dev/mapper/arch-home -L home 
```

## Mounting the filesystem

In order to generate the fstab locate in the /etc/fstab
We need to mount all partition 

```
mount /dev/mapper/rootfs /mnt

mkdir /efi
mount ${DISK}p1 /mnt/efi

# since we reboot for window installation
cryptopen open ${DISK}p3 boot
mkdir /mnt/boot
mount /dev/mapper/boot

mkdir /mnt/usr
mount /dev/mapper/usr /mnt/usr

mkdir /mnt/efi
mount ${DISK}p1 /mnt/efi

mkdir /mnt/var
mount /dev/mapper/var /mnt/var

mkdir /mnt/home
mount /dev/mapper/arch-home /mnt/home

swapon /dev/mapper/arch-swap
```

## Install system and some needed package

pacstrap /mnt base linux linux-firmware lvm2 vim dhcpcd 


## System configuration

```
# You can also use the -L instead of -U to use label disk
genfstab -U /mnt >> /mnt/etc/fstab

#(optional) Change relatime option to noatime
sed -i 's/relatime/noatime/g' /mnt/etc/fstab

```

## Enter inside 


```
arch-root /mnt
```



# update zone info 
```
ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
hwclock --systohc
```

## Set language to us utf-8

```
# Localization
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen
locale-gen
# language
echo "LANG=en_US.UTF-8" >> /etc/locale.conf

```

#  Network

## Hostname 

```
echo '<YOUR HOSTNAME>' > /etc/hostname
```

```
echo "127.0.0.1       localhost" >> /etc/hosts
echo "::1             localhost" >> /etc/hosts
echo "127.0.0.1       domain.localdomain" >> /etc/hosts
```

# Disk opening at boot time

Source:
* https://wiki.archlinux.fr/mkinitcpio#Hooks
* https://wiki.archlinux.org/index.php/Dm-crypt/System_configuration#Using_sd-encrypt_hook

Since we are using multiple luks we need set them in the /etc/crypttab 
And because the usr is needed at bootime we use the properties of the mkinitcpio HOOKS
sd-encrypt to open the luks partition by adding the entries in /etc/crypttab.initramfs
Use the lsblk to get the UUID and add them to the /etc/crypttab.initramfs file

```
# <NAME>               UUID=<ID find with blkid>          none    luks
# With vim you can do: 
# :read ! blkid <PATH OF THE DISK>

rootfs                  UUID=                             none    luks
usr                     UUID=                             none    luks
var                     UUID=                             none    luks
lvm                     UUID=                             none    luks

```


## Install a bootloader

In this example we use grub as it allow multiple luks at boot time


## Install and setup
```
pacman -S grub
# Said to grub that we use have luks partition
echo "GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub

# use blkid /dev/nvme0n1p3 to find the UUID
# add this to your /etc/default/grub
# if you root is on a lvm it's /dev/<volume group>/<logical volume>
GRUB_CMDLINE_LINUX="... cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:cryptlvm root=/dev/mapper/rootfs ..."

# use for managing efi entries and order
# See: https://doc.ubuntu-fr.org/efibootmgr
pacman -S efibootmgr

# install the grub efi bootloader on the efi partition
grub-install --target=x86_64-efi --efi-directory=/efi
```


## Prepare the parameters for the initramfs

### Install microcode for your processor

Source: 
https://wiki.archlinux.org/index.php/microcode

```
# For AMD cpu use the amd-ucode
pacman -S intel-ucode
# grub-mkconfig will automatically detect the microcode update and configure GRUB appropriately.
# Activate loading of microcode and generate the grub configuration 
grub-mkconfig -o /boot/grub/grub.cfg

```


## Configuration of mkinitcpio

mkinitcpio will load your configuration and apply them to create the initramfs
Ex: the hooks sd-encrypt will add the /etc/crypptab.initramfs as /etc/crypttab to load the luks partition (useful in our case)
Add them in the /etc/mkinitcpio.conf
HOOKS=(base udev autodetect keyboard modconf block encrypt usr shutdown lvm2 filesystems fsck)

You also have the /etc/mkinitcpio.d/linux.present that represent all the image that mkinitcpio will build
The fallback for example. 
It can be useful when everything work as expected to backup with this

Sources: 
* https://wiki.archlinux.org/index.php/mkinitcpio

```
# build the initramfs
mkinitcpio -p linux
```

## Add a root passwd to be able to debug if needed
```
passwd

```
reboot should be installed 


## Enable network
ex: systemd networkd
https://wiki.archlinux.org/index.php/Network_configuration

```
# list interface
ip link show

# enable the interface
ip link set <INTERFACE> up

# run the dhcp client demon
systemctl start dhcpcd.service
# run in at init
systemctl enable dhcpcd


# run the dhcp client demon
systemctl start systemd-networkd.service
# run in at init
systemctl enable systemd-networkd.service
s``

## Setup x server
###example with xorg


# Graphical Interface

## Install xorg-server
```
pacman -Sy xorg

Source: https://wiki.archlinux.fr/startx

## install driver
```
```
## Install window management

```
pacman -Sy i3
echo "exec i3 >> ~/.xinit"
```

## Install next package in an other directory to keep the minimal package here

In /etc/pacman.conf uncomment the RootDir and change it for /usr/local for example
Next package will be install there

Sources: 
* https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019#Detecting_UEFI_boot_mode
