# Arch Linux Installation & Maintenance Guide
*By: Matthew_Moore*

> Opinionated cheat-sheet for provisioning, configuring, and maintaining Arch Linux systems, from imaging media to desktop polish.

## Table of Contents
1. [Bootable Media Preparation](#bootable-media-preparation)
2. [Package Sources & AUR Workflows](#package-sources--aur-workflows)
3. [Mirror & Key Maintenance](#mirror--key-maintenance)
4. [Base Installation Recipes](#base-installation-recipes)
   - [Disk & LVM Quick Reference](#disk--lvm-quick-reference)
   - [Classic BIOS Install Flow](#classic-bios-install-flow)
   - [Extended Installation Checklist](#extended-installation-checklist)
   - [UEFI + LUKS Workflow (LearnLinux.tv)](#uefi--luks-workflow-learnlinuxtv)
   - [Unified Kernel Image (UKI) Deployment](#unified-kernel-image-uki-deployment)
5. [Desktop Environments & User Applications](#desktop-environments--user-applications)
6. [Audio, Fonts, and Media](#audio-fonts-and-media)
7. [Services, Hardware, & Virtualization](#services-hardware--virtualization)
8. [Troubleshooting & Quality-of-Life Tweaks](#troubleshooting--quality-of-life-tweaks)
9. [Docker & Container Notes](#docker--container-notes)
10. [Storage, Performance, & Package Audits](#storage-performance--package-audits)
11. [Reference Scripts & External Links](#reference-scripts--external-links)

---

## Bootable Media Preparation

- Identify disks:
  ```bash
  sudo fdisk -l
  ```

- Create a live USB (unmounted target):
  ```bash
  sudo dd bs=4M if=/path/to/archlinux.iso of=/dev/sdX status=progress oflag=sync
  sync
  ```

- Restore USB to general use:
  ```bash
  sudo mkfs.vfat -I /dev/sdX
  ```

- Clone CD/DVD media:
  ```bash
  cd /dev
  ls -ld sr* cdr* dvd*
  sudo dd if=/dev/cdrom of=~/dvdcopy.iso
  ```

---

## Package Sources & AUR Workflows

### Enable multilib + archlinuxfr
1. Edit `/etc/pacman.conf`, uncomment:
   ```
   [multilib]
   Include = /etc/pacman.d/mirrorlist
   ```
2. Append:
   ```
   [archlinuxfr]
   SigLevel = Never
   Server = http://repo.archlinux.fr/$arch
   ```
3. Install Yaourt:
   ```bash
   sudo pacman -Sy yaourt
   ```

### Yaourt from source (recommended)
```bash
sudo pacman -S git
sudo git clone https://aur.archlinux.org/package-query.git
cd package-query
sudo chown -R $(whoami) .
sudo chmod 775 .
makepkg -si
cd ..
sudo git clone https://aur.archlinux.org/yaourt.git
cd yaourt
sudo chown -R $(whoami) .
sudo chmod 775 .
makepkg -si
```

### Local AUR repository
`/etc/pacman.conf`:
```
[local]
SigLevel = Never
Server = file:///home/user/work/local/
```
Workflow:
```bash
mkdir -p /home/user/work/local
# build packages inside this folder
repo-add local.db.tar.gz *xz
```
Maintain an `AUR-Packages.txt` describing the desired bundles.

### Additional helpers & sources
```bash
git clone https://github.com/helmuthdu/aui
wget https://github.com/helmuthdu/aui/tarball/master -O - | tar xz
```
Trizen bootstrap:
```bash
cd /tmp
curl -o trizen.tar.gz https://aur.archlinux.org/cgit/aur.git/snapshot/trizen.tar.gz
tar zxvf trizen.tar.gz
cd trizen
makepkg -csi --noconfirm
```
Generic unattended AUR build snippet:
```bash
su - ${username} -c "
  [[ ! -d aui_packages ]] && mkdir aui_packages
  cd aui_packages
  curl -o ${PKG}.tar.gz https://aur.archlinux.org/cgit/aur.git/snapshot/${PKG}.tar.gz
  tar zxvf ${PKG}.tar.gz && rm ${PKG}.tar.gz
  cd ${PKG}
  makepkg -csi --noconfirm
"
```

---

## Mirror & Key Maintenance

1. Generate fresh mirrorlist:
   ```bash
   curl -so /tmp/mirrorlist https://www.archlinux.org/mirrorlist/?country=DE&country=UA&use_mirror_status=on
   sed -i 's/^#Server/Server/' /tmp/mirrorlist
   sudo install -m644 /tmp/mirrorlist /etc/pacman.d/mirrorlist
   ```

2. Rank mirrors (run as root):
   ```bash
   cd /etc/pacman.d
   for repo in *; do
     echo "Processing $repo..."
     mv $repo $repo.b4.rankmirrors
     rankmirrors -v $repo.b4.rankmirrors > $repo
   done
   ```

3. Refresh keys:
   ```bash
   sudo rm -r /etc/pacman.d/gnupg
   sudo pacman-key --init
   sudo pacman-key --populate archlinux
   sudo pacman-key --refresh-keys
   sudo pacman -S archlinux-keyring seahorse
   sudo pacman -Syyu
   ```

---

## Base Installation Recipes

### Disk & LVM Quick Reference
```bash
pvcreate /dev/sdb2
vgcreate vg_arch /dev/sdb2
lvcreate --name root --size 40G vg_arch
lvcreate -C y --name swap -L 4G vg_arch
lvcreate --name home -l 100%FREE vg_arch
dd if=/home/root.dd of=/dev/vg_arch/root   # restore backup
```
Common kernel flags for troublesome GPUs:
```
vga=normal  nofb  nomodeset  video=vesafb:off  i915.modeset=0
```
Btrfs SSD options: `noatime,discard,ssd,compress=lzo,space_cache`.

`smartctl --info /dev/sdX > SATA 3.1` for quick drive report.  
Reference: https://wiki.archlinux.org/index.php/SSD_memory_cell_clearing

### Classic BIOS Install Flow
```bash
pacstrap /mnt base base-devel
genfstab -U -p /mnt >> /mnt/etc/fstab
arch-chroot /mnt
ln -s /usr/share/zoneinfo/Europe/Kiev /etc/localtime
hwclock --systohc --utc
sed -i 's/^#\(ru_RU.UTF-8\)/\1/' /etc/locale.gen
locale-gen
pacman -S grub-bios os-prober
grub-mkconfig -o /boot/grub/grub.cfg
grub-install /dev/sda
echo 'nina.uptel.net' > /etc/hostname
```

`/etc/vconsole.conf`:
```
LOCALE="ru_RU.UTF-8"
HARDWARECLOCK="UTC"
TIMEZONE="Europe/Kiev"
KEYMAP=ru
FONT=ter-v16n
USECOLOR="yes"
```
`/etc/locale.conf`:
```
LANG=ru_RU.UTF-8
LC_MESSAGES=ru_RU.UTF-8
```
`/etc/mkinitcpio.conf` (add console font + keymap):
```
HOOKS="base consolefont keymap udev <your hooks>"
```
```bash
mkinitcpio -p linux
useradd -m -G users,wheel,audio,video -s /bin/bash acid
passwd acid
pacman -S sudo
reboot
```

### Extended Installation Checklist
```bash
pacman -S xorg-server xorg-server-utils xorg-xinit mesa xf86-video-intel
pacman -S alsa-lib alsa-utils alsa-oss alsa-plugins
pacman -S ttf-liberation ttf-droid ttf-dejavu ttf-mph-2b-damase-ib ttf-signika-family-ib \
           ttf-source-sans-pro-ibx ttf-triod-postnaja-ibx ttf-ubuntu-font-family-ib \
           ttf-vera-humana-95-ibx ttf-vollkorn-ibx terminus-font ttf-fira-mono ttf-fira-sans
pacman -S p7zip unace unarj unrar unzip
pacman -S xfwm4 xfwm4-themes xorg-luit xfce4-xkb-plugin
wget https://aur.archlinux.org/packages/ya/yaourt/yaourt.tar.gz
wget https://aur.archlinux.org/packages/pa/package-query/package-query.tar.gz
yaourt package-query
makepkg -csi --noconfirm
zukitwo-themes
sudo yaourt -S archey
sudo pacman -S slim
sudo systemctl enable slim.service
cp /etc/skel/.xinitrc ~acid/.xinitrc
sudo pacman -S firefox pcmanfm rxvt-unicode
mkdir -p ~acid/.config/awesome
cp /etc/xdg/awesome/ru.lua ~acid/.config/awesome
cp -r /usr/share/awesome/* ~acid/.config/awesome
sudo yaourt -S sublime-text
```
Wayland & GNOME:
```bash
sudo pacman -S --needed xorg-xwayland xorg-xlsclients glfw-wayland
sudo pacman -S --needed gnome gnome-tweaks nautilus-sendto gnome-nettool gnome-usage \
  gnome-multi-writer adwaita-icon-theme xdg-user-dirs-gtk fwupd arc-gtk-theme egl-wayland
```
XFCE goodies:
```bash
sudo pacman -S xfce4-goodies
```

### UEFI + LUKS Workflow (LearnLinux.tv)
```bash
cryptsetup luksFormat /dev/<DEVICE PARTITION 3>
cryptsetup open --type luks /dev/<DEVICE PARTITION 3> lvm
pvcreate --dataalignment 1m /dev/mapper/lvm
vgcreate volgroup0 /dev/sda2
lvcreate -L 30GB volgroup0 -n lv_root
lvcreate -l 100%FREE volgroup0 -n lv_home
modprobe dm_mod
vgscan
vgchange -ay
mkfs.ext4 /dev/volgroup0/lv_root
mount /dev/volgroup0/lv_root /mnt
mkfs.ext4 /dev/volgroup0/lv_home
mkdir /mnt/home
mount /dev/volgroup0/lv_home /mnt/home
mkdir /mnt/etc
genfstab -U -p /mnt >> /mnt/etc/fstab
pacstrap -i /mnt base
arch-chroot /mnt
pacman -S linux linux-headers       # or linux-lts*
pacman -S nano base-devel openssh
systemctl enable sshd
pacman -S networkmanager wpa_supplicant wireless_tools netctl dialog
systemctl enable NetworkManager
pacman -S lvm2
```
`/etc/mkinitcpio.conf` hooks:
- Unencrypted: `... block lvm2 filesystems ...`
- Encrypted: `... block encrypt lvm2 filesystems ...`

Locale & users:
```bash
nano /etc/locale.gen   # un-comment e.g. en_US.UTF-8
locale-gen
passwd
useradd -m -g users -G wheel <username>
passwd <username>
pacman -S sudo
EDITOR=nano visudo   # un-comment "%wheel ALL=(ALL) ALL"
```

GRUB variants:

| Scenario | Commands |
| --- | --- |
| BIOS (no encryption) | `pacman -S grub dosfstools os-prober mtools`<br>`grub-install --target=i386-pc --recheck /dev/sda`<br>`mkdir -p /boot/grub/locale`<br>`cp /usr/share/locale/en@quot/LC_MESSAGES/grub.mo /boot/grub/locale/en.mo`<br>`grub-mkconfig -o /boot/grub/grub.cfg` |
| UEFI (no encryption) | `pacman -S grub efibootmgr dosfstools os-prober mtools`<br>`mkdir -p /boot/EFI`<br>`mount /dev/<ESP> /boot/EFI`<br>`grub-install --target=x86_64-efi --bootloader-id=grub_uefi --recheck`<br>locale copy + `grub-mkconfig` |
| UEFI (with LUKS) | same as above plus `/etc/default/grub`:<br>`GRUB_ENABLE_CRYPTODISK=y`<br>`GRUB_CMDLINE_LINUX_DEFAULT="cryptdevice=/dev/sda3:volgroup0:allow-discards quiet"`<br>Regenerate config |

### Unified Kernel Image (UKI) Deployment
Disk prep:
```bash
sgdisk -Z /dev/sda
sgdisk -n1:0:+512M -t1:ef00 -c1:EFI -N2 -t2:8304 -c2:LINUXROOT /dev/sda
partprobe -s /dev/sda
cryptsetup luksFormat --type luks2 /dev/sda2
cryptsetup luksOpen /dev/sda2 lvm
mkfs.vfat -F32 -n EFI /dev/sda1
mkfs.btrfs -f -L linuxroot /dev/mapper/lvm
mount /dev/mapper/lvm /mnt
mkdir /mnt/efi
mount /dev/sda1 /mnt/efi
btrfs subvolume create /mnt/home
```
Bootstrap:
```bash
reflector --country UA --age 24 --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode vim nano \
  cryptsetup btrfs-progs dosfstools util-linux git unzip sbctl kitty networkmanager sudo
```
Locale & host:
```bash
sed -i -e '/^#\s*en_US.UTF-8/s/^#//' /mnt/etc/locale.gen
systemd-firstboot --root /mnt --prompt   # set keymap, timezone, hostname
arch-chroot /mnt locale-gen
arch-chroot /mnt useradd -G wheel -m acid
arch-chroot /mnt passwd acid
sed -i -e '/^# %wheel ALL=(ALL:ALL) NOPASSWD: ALL/s/^# //' /mnt/etc/sudoers
echo "quiet rw" > /mnt/etc/kernel/cmdline
mkdir -p /mnt/efi/EFI/Linux
```
`/mnt/etc/mkinitcpio.conf` (systemd hooks):
```
HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)
```
`/mnt/etc/mkinitcpio.d/linux.preset` (UKI outputs):
```
default_uki="/efi/EFI/Linux/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```
Build UKIs:
```bash
arch-chroot /mnt mkinitcpio -P
ls -lR /mnt/efi/EFI/Linux
```
Enable services + install bootloader:
```bash
systemctl --root /mnt enable systemd-resolved systemd-timesyncd NetworkManager
systemctl --root /mnt mask systemd-networkd
arch-chroot /mnt bootctl install --esp-path=/efi
sync
systemctl reboot --firmware-setup   # enter Secure Boot setup mode
```

---

## Desktop Environments & User Applications

- Optional fonts & themes:
  ```bash
  sudo pacman -S opendesktop-fonts
  yaourt -S fontconfig-ttf-ms-fonts ttf-google-fonts-git
  yaourt -S zukitwo-themes
  ```

- Flash & multimedia:
  ```bash
  sudo pacman -S a52dec faac faad2 flac jasper lame libdca libdv libmad libmpeg2 \
    libtheora libvorbis libxv wavpack x264 xvidcore gstreamer0.10-plugins flashplugin \
    libdvdcss libdvdread libdvdnav gecko-mediaplayer dvd+rw-tools dvdauthor dvgrab cdrdao
  ```

- Archive utilities:
  ```bash
  sudo pacman -S zlib p7zip unzip zip zziplib
  yaourt -S engrampa-thunar
  sudo pacman -S nemo-fileroller file-roller    # per DE preference
  ```

- XFCE extras:
  ```bash
  yaourt -S xfce4-volumed
  ```

- KDE stack:
  ```bash
  sudo pacman -S kdebase kde-l10n-ru
  sudo systemctl enable kdm.service
  sudo pacman -S kdemultimedia-kmix kdeutils-ark qtcurve kde-gtk-config oxygen plank
  yaourt -S kdeplasma-applets-homerun
  sudo fc-presets set
  sudo nano /etc/X11/xinit/xinitrc.d/infinality-settings
  # set: export INFINALITY_FT_BRIGHTNESS="-20"
  ```

- Optional GUI tools:
  ```bash
  yaourt -S pamac-aur octopi kalu blueman sublime-text archey
  sudo pacman -S firefox pcmanfm rxvt-unicode slim awesome
  sudo systemctl enable slim.service
  ```

- Fonts via Infinality bundles:
  ```
  [infinality-bundle]
  Server = http://bohoomil.com/repo/$arch
  [infinality-bundle-multilib]
  Server = http://bohoomil.com/repo/multilib/$arch
  [infinality-bundle-fonts]
  Server = http://bohoomil.com/repo/fonts
  ```
  ```bash
  pacman-key -r 962DDE58
  pacman-key --lsign-key 962DDE58
  pacman -Syyu
  pacman -S infinality-bundle
  ```

---

## Audio, Fonts, and Media

- PulseAudio tooling:
  ```bash
  sudo pacman -S pavucontrol pulseaudio-alsa
  yaourt -S xfce4-volumed
  ```
  *After reboot:* add XFCE mixer to panel → set card to *Playback: Built-in Audio*.  
  Launch mixer, set same card. Consider pinning `pavucontrol` for quick access.

- Optional notification for XFCE mail watcher:
  ```
  Run on click: thunderbird
  Run on new messages: notify-send "New mail" "You have new messages in your inbox" -i xfce-newmail
  ```

---

## Services, Hardware, & Virtualization

- Printing & scanning:
  ```bash
  sudo pacman -S libcups cups ghostscript gsfonts system-config-printer simple-scan
  sudo systemctl enable --now org.cups.cupsd.service cups-browsed.service
  sudo pacman -S hplip
  sudo pacman -Rdd foomatic-db foomatic-db-nonfree  # fix Manjaro/Antergos issues
  ```

- Bluetooth:
  ```bash
  sudo pacman -S bluez bluez-cups bluez-utils
  sudo modprobe btusb
  sudo systemctl enable --now bluetooth
  yaourt -S blueman
  ```

- VirtualBox:
  ```bash
  sudo pacman -S virtualbox virtualbox-host-dkms virtualbox-host-modules
  yaourt -S virtualbox-ext-oracle
  sudo modprobe vboxdrv
  sudo gpasswd -a $USER vboxusers
  echo vboxdrv | sudo tee /etc/modules-load.d/virtualbox.conf
  ```
  Load kernel module at boot per `Kernel_modules#Loading`.

- Autologin for LXDM (`/etc/lxdm/lxdm.conf`):
  ```
  [base]
  autologin=matt
  ```

- TeamViewer daemon:
  ```bash
  sudo systemctl enable --now teamviewerd
  sudo systemctl --system daemon-reload   # if restart needed
  ```

- Network tools:
  ```bash
  systemctl enable avahi-daemon.service
  nmcli con add type vpn con-name "Comcast" ifname "*" vpn-type openconnect \
    -- vpn.data "gateway=sslvpn.comcast.net,protocol=nc"
  ```

---

## Troubleshooting & Quality-of-Life Tweaks

- XFCE thumbnails reset:
  ```bash
  sudo pacman -S tumbler ffmpegthumbnailer gstreamer0.10 poppler-glib libgsf libopenraw
  sudo rm -rf ~/.thumbnails/
  mv ~/.config/Thunar ~/.config/Thunar.bak
  sudo update-mime-database /usr/share/mime
  ```

- Hide Windows partitions (`/etc/udev/rules.d/99-hide-partitions.rules`):
  ```
  KERNEL=="sda1",ENV{UDISKS_IGNORE}="1"
  KERNEL=="sda2",ENV{UDISKS_IGNORE}="1"
  ```
  Activate: `sudo udevadm trigger --verbose`.

- Trackpad palm-check autostart:
  ```bash
  syndaemon -k -i 2 &
  ```

- Disable screen blanking (startup commands):
  ```bash
  xset -dpms &
  xset s noblank &
  xset s off &
  ```

- GRUB text boot:
  ```
  GRUB_CMDLINE_LINUX_DEFAULT="text"
  ```

- Time & clock management:
  ```bash
  timedatectl status
  sudo pacman -S ntp
  sudo timedatectl set-ntp true
  timedatectl list-timezones
  sudo timedatectl set-timezone America/New_York
  sudo timedatectl set-time "2013-08-11 23:56:16"
  sudo timedatectl set-timezone UTC   # fix dual-boot issues
  ```

- Disable framebuffer hints (boot options enumerated earlier).

---

## Docker & Container Notes

- Remove stale Btrfs subvolumes:
  ```bash
  sudo btrfs subvolume delete /var/lib/docker/btrfs/subvolumes/*
  ```

- Force shell inside container:
  ```bash
  docker exec -it <containerIdOrName> bash
  ```

- Cleanup helpers:
  ```bash
  docker rm -f $(docker ps -aq)
  docker network rm $(docker network ls -q)
  docker run --rm -v /var/lib/docker/network/files:/network busybox rm /network/local-kv.db
  ```

- Known issue: “docker does not remove btrfs subvolumes when destroying container #9939”.

---

## Storage, Performance, & Package Audits

- Scheduler tweak (example for `/dev/sdb` SSD):
  ```bash
  echo 'ACTION=="add|change", KERNEL=="sdb", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="deadline"' \
    | sudo tee /etc/udev/rules.d/60-schedulers.rules
  cat /sys/block/sdb/queue/scheduler
  ```

- `tmpfs` sizing:
  ```
  tmpfs /tmp tmpfs nodev,nosuid,size=2G 0 0
  ```

- Package lists:
  ```bash
  pacman -Qqe | grep -vx "$(pacman -Qqm)" > Packages
  pacman -Qqm > Packages.aur
  ```

- Global toolchain (base station):
  ```bash
  pacman -S bc rsync mlocate bash-completion pkgstats arch-wiki-lite \
    zip unzip unrar p7zip lzop cpio avahi nss-mdns alsa-utils alsa-plugins \
    pulseaudio pulseaudio-alsa ntfs-3g dosfstools exfat-utils f2fs-tools \
    fuse fuse-exfat autofs mtpfs
  ```

---

## Reference Scripts & External Links

- Mirrorlist helper:
  ```bash
  #!/bin/sh
  url="https://www.archlinux.org/mirrorlist/?country=DE&country=UA&use_mirror_status=on"
  tmpfile=$(mktemp --suffix=-mirrorlist)
  curl -so ${tmpfile} ${url}
  sed -i 's/^#Server/Server/g' ${tmpfile}
  ```

- EFI layout reference:
  ```
  EFI/Boot
    bootx64.efi
    fbx64.efi
  EFI/ubuntu
    /fw/
    BOOTX64.CSV
    fbx64.efi
    fwupx64.efi
    grub.cfg
    grubx64.efi
    mmx64.efi
    shimx64.efi
  ```

- Quick commands:
  ```bash
  curl http://smart-ip.net/myip
  yaourt -Syu --devel --aur --noconfirm
  curl -o trizen.tar.gz https://aur.archlinux.org/cgit/aur.git/snapshot/trizen.tar.gz
  docker exec -it <container> bash
  ```

- Helpful links:
  - https://aur.archlinux.org
  - https://wiki.archlinux.org/index.php/Improving_performance
  - https://www.youtube.com/watch?v=yJOqiZEU-bg
  - http://www.noobslab.com/2014/11/mbuntu-macbuntu-1410-transformation.html
  - https://wiki.learnlinux.tv/index.php/Arch_Linux_-_Full_installation_Guide#Preparing_the_hard_disk_.28UEFI_with_encryption.29
  - https://www.debugpoint.com/wayland-arch-linux/

- Misc reminders:
  ```bash
  curl http://smart-ip.net/myip
  ```



