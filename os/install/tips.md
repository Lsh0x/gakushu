## Table of Content

- [Adding more luks](#Opening-luks-disk-at-boot-time)
- [Luks keys](#Embed-a-keyfile-in-initramfs)
- [Share data between arch and window](#Disk-shared-between-window-and-linux)
- [Other luks](#Opening-luks-disk-at-boot-time)
- [Login customization](#Lightdm-customization)
- [Terminal](#termite)
- [Sound](#Sound)
- [Pacman hooks](#Pacman-hooks)
- [Automated screen display](#Xrandr-to-setup-automatically-the-screen)
- [GRUB](#Grub-configuration)
- [Picom](#Picom)
- [Conky](#Conky)
- [Desktop background](#nitrogen)
- [process isolation](../../security/sandbox/firejail.md)

## Tips

### Opening luks disk at boot time

This section is used only for those who have more disk, like a seperate usr
They will need to use the `systemd` and `sd-encrypt` hooks for multiple disks and the `usr` hooks if usr is in an other partition

Since we are using multiple luks, we need set them in the `/etc/crypttab` if the disk in not required at boot time
If it's needed you need to add it in `/etc/crypttab.initramfs`, it will be added as `/etc/crypttab` in your image.
For usr because it's needed at bootime we use the properties of the mkinitcpio HOOKS
`usr` to open the luks partition by adding the entries in `/etc/crypttab.initramfs`
Use the lsblk to get the UUID and add them to the `/etc/crypttab.initramfs` file
The syntax is the same either in `/etc/crypttab` and `/etc/crypttab.initramfs`

```
# <NAME>               UUID=<ID find with blkid>          none    luks
# With vim you can do:
# :read ! blkid <PATH OF THE DISK>

rootfs                  UUID=                             none    luks
usr                     UUID=                             none    luks
```

Source:
* https://wiki.archlinux.fr/mkinitcpio#Hooks
* https://wiki.archlinux.org/index.php/Dm-crypt/System_configuration#Using_sd-encrypt_hook

### Embed a keyfile in initramfs

This is a copy paste from : 
* https://gist.github.com/huntrar/e42aee630bee3295b2c671d098c81268#recommended-embed-a-keyfile-in-initramfs

#### Create a keyfile and add it as LUKS key
```
mkdir /root/secrets && chmod 700 /root/secrets
head -c 64 /dev/urandom > /root/secrets/crypto_keyfile.bin && chmod 600 /root/secrets/crypto_keyfile.bin
cryptsetup -v luksAddKey -i 1 /dev/nvme0n1p3 /root/secrets/crypto_keyfile.bin
```

#### Add the keyfile to the initramfs image
```/etc/mkinitcpio.conf```
```
FILES=(/root/secrets/crypto_keyfile.bin)
```

#### Recreate the initramfs image
```
mkinitcpio -p linux
```

#### Set kernel parameters to unlock the LUKS partition with the keyfile using ```encrypt``` hook
```/etc/default/grub```
```
GRUB_CMDLINE_LINUX="... cryptkey=rootfs:/root/secrets/crypto_keyfile.bin"
```

#### Regenerate GRUB's configuration file
```
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Restrict ```/boot``` permissions
```
chmod 700 /boot
```

### Lightdm customization

Lightdm allow you to use theme to custumize the login interface
Here an example to change the greetier session and use a theme 
```
# change greeter to use lightdm-webkit2-greeter
pacman -Sy ligthdm-webkit2-greeters
# Edit /etc/ligthdm/lightdm.conf
# Change greeter-session to greeter-session=lightdm-webkit2-greeter

# Download and install a theme 
git clone https://aur.archlinux.org/lightdm-webkit2-theme-glorious.git
cd lightdm-webkit2-theme-glorious && makepkg -rsi
#Edit: /etc/lightdm/lightdm-webkit2-greeter.conf
# Change webkit-theme to webkit-theme=glorious
```
Sources:

* https://www.addictivetips.com/ubuntu-linux-tips/set-up-lightdm-on-arch-linux/
* https://github.com/Litarvan/lightdm-webkit-theme-litarvan

### Termite

Termite is a low ressource terminal emulator
You can install it with 
```
pacman -Sy termite
```

Sources:
* https://www.linuxsecrets.com/archlinux-wiki/wiki.archlinux.org/index.php/Termite.html

### Sound

You should have alsa installed, complete it with pulse-audio
```
pacman -Sy pulseaudio-alsa
# add a graphical interface if needed.
pacman -Sy pavucontrol
```

### Pacman hooks
Let create a backup of installed package with the date.
The following command give you the installed packages on you system.
```
pacman -Qqe
```
Pacman allow you to create hooks related to its own use.
Pacman read from `/etc/pacman.d/hooks the .hook`

For example we can create a hooks to generate the installed program after each packet installation.
It will create a file with the date in the /var/pkgs directory like `2020-11-20_19:21:50-current-pkgs.txt`
And copy it to `/var/pkgs/prev/2020-11-20_19:21:50-pkgs.txt` allowing to keep trace on time.

```
# For the hooks file
mkdir /etc/pacman.d/hooks
# For the saved pkgs list
mkdir -p /var/pkgs/prev
```

Create a file inside this directory ending with .hook extension
ex: `/etc/pacman.d/hooks/current-pkgs.hook`
```
[Trigger]
Operation = Install
Operation = Remove
Type = Package
Target = *

[Action]
When = PostTransaction
Exec = /bin/sh -c '/usr/bin/pacman -Qqe > /var/pkgs/current-pkgs.txt && cp /var/pkgs/current-pkgs.txt /var/pkgs/prev/$(date +"%F_%X")-pkgs.txt'
```

See:
* https://wiki.archlinux.org/index.php/Pacman#Hooks

### Xrandr to setup automatically the screen

Since everything need to be done manually this part too
Fortunatly the package `autoxrandr` do that for us.
It's using the `xrandr` tools from `xorg`
Install the package and enable it at startup.

```
pacman -Sy autorandr
systemctl enable autorandr.service
```

### Disk shared between window and linux

If you need a disk to be shared between your window and linux
You need a partition for it with a `NTFS` file system
Window need to create it to assign it
Format the partition with window, disk management, see sources.
Then it should be reconize by your window
Now we need to mount it with linux, rebbot and boot on arch linux
```
# Change the label of the disk
e2label <DISK> shared
# You need the ntfs-3g to be able to mount the ntfs partition
pacman -Sy ntfs-3g
mkdir <MOUNTPOINT>
# Finally lets add it to our fstab with options to have permission for everyone
# (Do not store sensitive information here)
echo "LABEL=shared	        <MOUNTPOINT>	        ntfs-3g	        relatime,uid=1000,gid=1000	0 2" >> /etc/fstab
```

Sources:
* https://wiki.archlinux.org/index.php/NTFS-3G
* https://helpdesk.originpc.com/support/solutions/articles/9000124011-how-to-add-a-hard-drive-to-windows-10-

[Table of Contents](#Table-of-Content)

### Grub configuration

#### Chainloading window
Now we have multiple boot posssibilities but there are not in the same place
We have the window bootloader and grub for arch linux.
Why not choosing from grub if we want to boot window or linux.
Do do that, grub will need to chain to the window boot.
Make sure you boot partition and efi are open and mount.
Then we need to specify to grub that we want chain to window, and regenerate the configuration
Let's add a menuentry for window in `/etc/grub.d/40_custom`
```
# ex:
# Since we set the label of the efi partition to EFI we can search on it
menuentry "Windows" {
	insmod fat
	search --label --set=root EFI
	chainloader (${root})/EFI/Microsoft/Boot/bootmgfw.efi
	boot
}
```

Regenerate the grub configuration
```
grub-mkconfig -o /boot/grub/grub.cfg
```

Now we should be able to select `Windows` in you grub menu

Sources:
* https://wiki.archlinux.org/index.php/GRUB#Chainloading_Windows/Linux_installed_in_UEFI_mode


#### Changing boot order

`efibootmgr` is the utilily that handler the UEFI boot.
It let you interact with it
Try

```
efibootmgr -v
```
For know we keep the default window entry, but let's make grub the default boot
Changing the boot order will do that.
We set the arch linux to be the default one.
```
efibootmgr -o 0001,0000
```

Depending on firmware you can have some issue.
For my computer i had to delete the window boot entry
then create a new one with the same parameters than the arh linux one using: 
```
efibootmgr -b 0000 -B
# Will take the 0000 slot and then the firmare added the window boot
efibootmgr -c -l "Arch Linux" -d /dev/nvme0n1p1 -l "\EFI\ARCH\GRUBX64.EFI"
```

Sources:

https://github.com/Lsh0x/gakushu/edit/main/arch.md

#### Chaning theme 

You can change the grub theme.
For that choose the one you like and install it.
Be sure to install in the `/boot/grub/themes` directory and not in the rootfs

Sources: 
https://www.gnome-look.org/browse/cat/109/order/latest/
https://wiki.archlinux.org/index.php/GRUB/Tips_and_tricks#Hidden_menu


### Picom

Picom is a standalone compositor for Xorg, suitable for use with window managers that do not provide compositing.
This will be helpful for powering tools, like the conky monitoring system

#### Installation
```
pacman -Sy picom
```

#### Auto start


Add in your i3 configuration file, `$HOME/.config/i3/config` the following line
```
exec --no-startup-id picom
```


Source:
* https://wiki.archlinux.org/index.php/Picom#Conky

### Conky

#### Installation
Conky is a monitoring system hight configurable
You can install it simply by
```
pacman -Sy conky
```

Then you can find some theme with already configuration.
Install the configuration file in`$HOME/.config/conky/conky.conf`

#### Auto start

Add in your i3 configuration file, `$HOME/.config/i3/config` the following line
```
exec --no-startup-id conky
```

Sources:
* https://wiki.archlinux.org/index.php/conky
* https://github.com/brndnmtthws/conky/wiki/Configs
* http://conky.sourceforge.net/variables.html

### Nitrogen

You may want a background images for you desktop
For that you can use `feh` or `nitrogen` for multiple screen support

#### Installation
```
pacman -Sy nitrogen
```

#### Using it with i3 
Just add in you `$HOME/.config/i3/config`

```
exec --no-startup-id nitrogen -set-scaled <PATH TO WALLPAPER>
```

Sources: 

* https://wiki.archlinux.org/index.php/feh
