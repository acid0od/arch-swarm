# Install Arch linux

## Boot and create hard disk partitions 

### Preparing the hard disk (UEFI with no encryption)

See partitions/drives on the system (find the name of your hard drive)

```
 fdisk -l
```

Start the partitioner (fdisk)

```
 fdisk /dev/nvme0n1

 or 

 fdisk /dev/sda
```
Show current partitions

```
 p
```

Create EFI partition

```
 g (to create an empty GPT partition table)
 n
 enter
 enter
 +512M
 t
 1 (For EFI)
```

Create boot partition

```
 n
 enter
 enter
 +512M
```

Create LVM partition
 ```
 n
 enter
 enter
 enter
 t
 enter
 30
```
Show current partitions again

```
 p
```

Finalize partition changes

```
 w
```

### Create physical volume


```
pvcreate /dev/sda2
vgcreate vg_arch /dev/sda2


```

Remove OLD physical volume

```
pvremove /dev/sdb1 
```

### Logical volumes
```
lvcreate --name root --size 40G vg_arch
lvcreate -C y --name swap -L 4G vg_arch
lvcreate --name home -l 100%FREE vg_arch

modprobe dm_mod

vgscan
vgchange -ay
```

### Format volumes

Format the EFI partition

 mkfs.fat -F32 /dev/sda1

Format the boot partition

 mkfs.ext2 /dev/sda2

Format the root partition

 mkfs.ext4 /dev/vg_arch/root
 mount /dev/vg_arch/root /mnt

Format the swap partition
 mkswap /dev/vg_arch/swap

 swapon /dev/vg_arch/swap

Format the home partition

 mkfs.ext4 /dev/vg_arch/home
 mkdir /mnt/home
 mount /dev/vg_arch/home /mnt/home

Mount boot partition

 mount /dev/sda2 /mnt/boot

Create the /etc/fstab file

 mkdir /mnt/etc
 genfstab -U -p /mnt >> /mnt/etc/fstab

## Install Arch base 

reflector --country "UA" --country "PL" --sort rate --save /etc/pacman.d/mirrorlist

pacstrap -i /mnt base base-devel

arch-chroot /mnt

Install a kernel and headers
 
 pacman -S linux linux-headers

 pacman -S netctl dhcpcd inetutils diffutils e2fsprogs less linux-firmware 
 pacman -S logrotate man-db man-pages nano perl sysfsutils texinfo usbutils 
 
 pacman -S vi which networkmanager dialog
 systemctl enable NetworkManager 

Add LVM support
 pacman -S lvm2

Edit /etc/mkinitcpio.conf
 nano /etc/mkinitcpio.conf

On the "HOOKS" line, add support for lvm2
Add "lvm2" in between "block" and "filesystems"
 
 HOOKS=(base udev autodetect modconf block lvm2 filesystems keyboard fsck)

Uncomment the line from the /etc/locale.gen file that corresponds to your locale
 nano /etc/locale.gen (uncomment en_US.UTF-8)

Generate the locale
 locale-gen

Set the root password
 passwd

Add console font

 pacman -S terminus-font

/etc/vconsole.conf

LOCALE="en_US.UTF-8"
HARDWARECLOCK="UTC"
TIMEZONE="Europe/Kiev"
KEYMAP=ru
FONT=ter-u16n
USECOLOR="yes"

Add "consolefont" in between "base" and "keymap"

  HOOKS="base consolefont keymap udev..."

Create the initial ramdisk for the main kernel
 mkinitcpio -p linux

/etc/locale.conf

LANG=en_US.UTF-8
LC_MESSAGES=en_US.UTF-8


Create a user for yourself
 useradd -m -g users -G wheel <username>
Set your password
 passwd <username>

Install sudo (may already be installed)
 pacman -S sudo

Allow users in the 'wheel' group to use sudo
 EDITOR=nano visudo

Uncomment:
 %wheel ALL=(ALL) NOPASSWD: ALL


### Installing GRUB for UEFI, with no encryption

Install the required packages for GRUB:
 pacman -S grub efibootmgr dosfstools os-prober mtools
Create the EFI directory:
 mkdir /boot/EFI
Mount the EFI partition:
 mount /dev/<DEVICE PARTITION 1> /boot/EFI

Install GRUB:
 grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck

Create the locale directory for GRUB
 mkdir /boot/grub/locale

Copy the locale file to locale directory
 cp /usr/share/locale/en\@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo

Generate GRUB's config file
 grub-mkconfig -o /boot/grub/grub.cfg

### Post-Install Tweaks/Enhancements

Set time zone
List time zones:

 timedatectl list-timezones

Set your time zone:

 timedatectl set-timezone Europe/Kiev
Enable time synchronization via systemd:

 systemctl enable systemd-timesyncd

To change the hardware clock time standard to localtime, use:

 timedatectl set-local-rtc 1

Set the hostname
Consider setting the hostname of your new installation. You can do so with the following command:

 hostnamectl set-hostname myhostname

Also, make the same change in /etc/hosts:

 nano /etc/hosts
Example lines to add:

 127.0.0.1 localhost
 127.0.1.1 myhostname

Install CPU Microde files (Intel CPU)
 pacman -S intel-ucode


Reboot your machine
Exit the chroot environment
 exit
Unmount everything (some errors are okay here)
 umount -a
Reboot the machine
 reboot


Post install

wget https://github.com/helmuthdu/aui/tarball/master -O - | tar xz

# arch-swarm
Tool for Arch linux

wget https://github.com/acid0od/arch-swarm/tarball/master -O - | tar xz
