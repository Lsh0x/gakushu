# Dual boot window/arch with full encrypted linux disk

I recomand to read the tips section that are optionnal for an installation
but can add a lot of thing that you might want to install a the installation

* Boot will be in UEFI mode.
* Boot is encrypted and will ask for passphrase to unlock linux
* With the window pro version you can use bitgarden to encrypt the window partition
    * TODO See if we can use luks on the boot window with grub
* Linux filesystem will be on LVM over luks.
    * TODO use yubikey to unlock the LVM

## Table of Content
- [Downloads](#Download-the-arch-and-window-iso)
- [Partioning and format the disk](#Partioning-and-format-the-disk)
- [Install Window](#Install-Window)
- [Install Arch Linux](#Install-Arch-Linux)
   - [Synchronise clock](#Synchronise-clock)
   - [Preparing the disk](#Preparing-the-disk)
   - [Mounting the filesystem](#Mounting-the-filesystem)
   - [Install system and some needed package](#Install-system-and-some-needed-package)
   - [System configuration](#System-configuration)
   - [Chroot mnt](#Chroot)
   - [Update zone info and language](#Update-zone-info-and-language)
   - [Hostname](#Hostname)
   - [Opening luks disk at boot time](tips.md/#Opening-luks-disk-at-boot-time)
   - [Bootloader](#Install-and-setup)
   - [Initramfs](#Prepare-initramfs)
   - [Almost finished](#End-of-installation)
 - [Post Installation](#Post-installation)
   - [Network](#Enable-network)
   - [Graphical Interface](#Graphical-Interface-and-commodities)
   - [Display Server](#Display-Server)
   - [GPU Driver](#Install-GPU-Driver)
   - [Dislay Managment](#Display-managment)
   - [Window Managment](#Window-managment)
- [Tips](tips.md)

## Download the arch and window iso

ex:
```
dd if=<PATH TO ISO> of=<PATH TO USB> bs=4096 status=progress
```

```sh
dd if=$HOME/Downloads/archlinux-2020.11.01-x86_64.iso of=/dev/sdb1 status=progress
```

If you are on other machine check google with "creating bootable usb on <YOUR OS>"
Insert the usb drive and boot on it. It's better to have two usb drive
But one can be enough  if you use dd to make the bootable needed usb drive.


### Troubleshooting:

* Be carefull i saw some issue on OSX to create bootable image for window

Sources for help:

* https://www.howtogeek.com/howto/linux/create-a-bootable-ubuntu-usb-flash-drive-the-easy-way/
* https://ubuntu.com/tutorials/create-a-usb-stick-on-ubuntu#1-overview
* https://ubuntu.com/tutorials/create-a-usb-stick-on-macos#1-overview
* https://medium.com/@tbeach/use-unix-dd-command-to-os-bootable-on-usb-drive-6671945d95a6
* https://www.freecodecamp.org/news/how-make-a-windows-10-usb-using-your-mac-build-a-bootable-iso-from-your-macs-terminal/



The Window installer will only create a partition of 100M.
To avoid this, you can create the EFI partition before running the window installation.
I recommand to create your own before to size it as you whish

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

You can create a parititon with the window install and keep the end of the disk for linux
The window installer will use the EFI partition and add its own entry
This will allow you to define the size of the EFI partiion and not deal with the 100M from the windown installer
When window is installed, we are ready to process the installation.

Note: i created the boot and grub partition but can be probably done after the window installation.
It's a choice i did to have the boot efi and grub at the begininig of the disk.
Feel like you want, just after the installation do the GRUB and BOOT partionning/formattage of the disks.


## Install Arch Linux

### Synchronise clock

```
timedatectl set-ntp true
```

### Preparing the disk

```
#export DISK=<YOUR DISK AGAIN>
export DISK=/dev/nvme0n1

# lvm

sgdisk --new=5:0:0 $DISK
sgdisk --typecode=5:8301 $DISK
sgdisk --change-name=5:lvm $DISK
cryptsetup luksFormat --use-random -S 1 -s 512 -h sha512 -i 5000 ${DISK}p5
cryptsetup open ${DISK}p5 lvm

# setup lvm partition
pvcreate /dev/mapper/lvm
vgcreate arch /dev/mapper/lvm

# set swap
lvcreate -L 8G -n swap arch
mkswap /dev/mapper/arch-swap -L swap

# set /
lvcreate -L 20G -n rootfs
mkfs.ext4 /dev/mapper/arch-rootfs -L rootfs

# set /usr/local
lvcreate -L 10G -n local
mkfs.ext4 /dev/mapper/arch-local -L local

# set /home
lvcreate -l 80%FREE -n home arch
mkfs.ext4 /dev/mapper/arch-home -L home
```

Tips:
- [Share data between arch and window](#Disk-shared-between-window-and-linux)
- [Other luks](#Opening-luks-disk-at-boot-time)

### Mounting the filesystem

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

mkdir /mnt/efi
mount ${DISK}p1 /mnt/efi

# If you have an HDD it can be useful to mount /var/log on it
# mkdir /mnt/var
# mount /dev/sda1 /mnt/var

mkdir /mnt/home
mount /dev/mapper/arch-home /mnt/home

# Change pacman rootdir to install package here to keep the basic system independent
# If you have feedback about this go ahead :)
mkdir -p /mnt/usr/local
mount /dev/mapper/arch-local /mnt/usr/local


swapon /dev/mapper/arch-swap
```


### Install system and some needed package

```
pacstrap /mnt base base-devel linux linux-firmware lvm2 vim dhcpcd git
```

### System configuration

```
# You can also use the -L instead of -U to use label disk
genfstab -U /mnt >> /mnt/etc/fstab

```

### Chroot


```
arch-root /mnt
```

#### update zone info and language
```
ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
hwclock --systohc

# Localization
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen
locale-gen
# language
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
```

#### Hostname

```
echo '<YOUR HOSTNAME>' > /etc/hostname
```

```
echo "127.0.0.1       localhost" >> /etc/hosts
echo "::1             localhost" >> /etc/hosts
echo "127.0.0.1       domain.localdomain" >> /etc/hosts
```


### Bootloader
As usual arch allow you to choose the one you want.
You have multiple choices you can check the one correspond to you
In this example we use grub as it allow multiple luks at boot time

See: https://wiki.archlinux.org/index.php/Arch_boot_process

#### Install and setup

```
pacman -S grub
# Said to grub that we use have luks partition
sed -i 's/#GRUB_ENABLE_CRYPTODISK/GRUB_ENABLE_CRYPTODISK=y/g' /etc/default/grub

# use blkid /dev/nvme0n1p3 to find the UUID
# add this to your /etc/default/grub
# if you root is on a lvm it's /dev/<volume group>/<logical volume>
GRUB_CMDLINE_LINUX="... cryptdevice=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:lvm root=/dev/arch/rootfs ..."

# use for managing efi entries and order
# See: https://doc.ubuntu-fr.org/efibootmgr
pacman -S efibootmgr

# install the grub efi bootloader on the efi partition
grub-install --target=x86_64-efi --efi-directory=/efi
```
Sources:
* https://wiki.archlinux.org/index.php/GRUB


### Prepare initramfs

#### Install microcode for your processor

Source:
https://wiki.archlinux.org/index.php/microcode

```
# For AMD cpu use the amd-ucode
pacman -S intel-ucode
# grub-mkconfig will automatically detect the microcode update and configure GRUB appropriately.
# Activate loading of microcode and generate the grub configuration
mkdir /boot/grub
grub-mkconfig -o /boot/grub/grub.cfg

```

#### Configuration of mkinitcpio

mkinitcpio will load your configuration and apply them to create the initramfs
Ex: the hooks sd-encrypt will add the /etc/crypptab.initramfs as /etc/crypttab to load the luks partition (useful in our case)
Add them in the /etc/mkinitcpio.conf
```
HOOKS=(base udev autodetect  modconf block encrypt lvm2 filesystems keyboard fsck shutdown)
```
You also have the /etc/mkinitcpio.d/linux.present that represent all the image that mkinitcpio will build
The fallback for example.
It can be useful when everything work as expected to backup with this

Sources:
* https://wiki.archlinux.org/index.php/mkinitcpio

```
# build the initramfs eveything will be generated in /boot which is our encrypted disk
mkinitcpio -p linux
```

[Table of Contents](#Table-of-Content)

### End of installation

- [Pacman hooks](#Pacman-hooks)


#### Add root password
```
passwd
```

#### User
```
# Add user with wheel group
useradd -G wheel -m x
pacman -Sy sudo
passwd x
# uncomment the wheel line in the /etc/sudoers file allowing x to use sudo

```

#### Reboot
```
reboot
```

Now the system should boot
[Table of Contents](#Table-of-Content)

## Post installation

### Enable network

This will handle related connection stuff, wifi as ethernet.

As a network manager i choose the systemd-network manager
but feel free to choose the one corresponding to your needs
See: https://wiki.archlinux.org/index.php/Category:Networking


### Example with systemd-networkd

Source:
* https://wiki.archlinux.org/index.php/Network_configuration


```
# list interface
ip link show

# enable the interface
ip link set <INTERFACE> up

# run the dhcp client demon
systemctl start dhcpcd.service
# run at init
systemctl enable dhcpcd
# run the systemd-networkd. service
systemctl start systemd-networkd.service
# run in at init
systemctl enable systemd-networkd.service
```

## Graphical Interface and commodities

### Display Server

Provide a basic framework where application will work on top of it
It handle basic display windows and keyboard/mouse interaction.

Sources:
* https://wiki.archlinux.org/index.php/General_recommendations#Graphical_user_interface
* https://en.wikipedia.org/wiki/X_Window_System

For example install and configure the xorg server
```
pacman -Sy xorg
Xorg :0 -configure
cp /root/xorg.conf.new /etc/X11/org.conf
```

Source:
* https://wiki.archlinux.org/index.php/xorg


### Install GPU Driver

The default vesa display driver will work with most video cards, but performance can be significantly improved and additional features harnessed by installing the appropriate driver for AMD, Intel, or NVIDIA products.

```
pacman -Sy nvidia
# Generate the xorg config
nvidia-xconfig
```

Source:
* https://wiki.archlinux.org/index.php/


### Display managment

This will allow you to have a GUI to enter your login
There is different solutions, where you can configure the display

I choose a ligth one ligthdm, there is a already configure display called greeters.
Let's try it

```
# install ligthdm
pacman -Sy lightdm
systemctl enable ligthdm.service
```

Source:
* https://wiki.archlinux.org/index.php/Display_manager
* https://wiki.archlinux.org/index.php/LightDM



### Window managment

The is your graphical environment, i choose a really light one.


```
pacman -Sy i3
echo "exec i3 >> /home/x/.xinit"
mkdir -p /home/x/.config/i3
cp /etc/i3/config /home/x/.config/i3/config
```

Source: 
* https://www.youtube.com/watch?v=j1I63wGcvU4

### Finally

#### Recommandation

You should be able to have a bit plus of an minimal environment 
I stongly recommand to read the sources given to learn more about what you are using.
It also allow you to go deeper in the configuration.
Some others learning stuff will come.

### Tips and tricks

See [Here](tips.md)
