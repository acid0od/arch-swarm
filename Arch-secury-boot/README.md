# Arch Linux Installation Guide with Secure Boot

This guide provides a comprehensive, step-by-step process for installing Arch Linux with Secure Boot enabled, using a Unified Kernel Image (UKI) and LUKS2 encryption. It assumes you are booting from an Arch Linux live ISO and have basic familiarity with Linux commands.

## Prerequisites
- A USB drive with the Arch Linux ISO
- A computer with UEFI firmware and Secure Boot capability
- An internet connection (Wi-Fi or Ethernet)
- A disk to install Arch Linux (e.g., `/dev/sda`)

## Step 1: Connect to the Network
Ensure your system is connected to the internet. If using Wi-Fi, follow these steps:

1. **List Wi-Fi devices** to identify your wireless device name:
```bash
iwctl
[iwd]# device list
```
2. **Enable the device and adapter** if they are powered off (replace `name` and `adapter` with your device and adapter names):
```bash
[iwd]# device name set-property Powered on
[iwd]# adapter adapter set-property Powered on
```
3. **Scan for networks** (this command produces no output):
```bash
[iwd]# station name scan
```
4. **List available networks**:
```bash
[iwd]# station name get-networks
```
5. **Connect to a network** (replace `SSID` with your network's SSID):
```bash
[iwd]# station name connect SSID
```
If prompted for a passphrase, enter it. Alternatively, provide it directly:
```bash
iwctl --passphrase passphrase station name connect SSID
```
6. Exit `iwctl`:
```bash
[iwd]# exit
```
Verify connectivity:
```bash
ping -c 3 archlinux.org
```

## Step 2: Prepare Disks
Partition and format the disk for a UEFI system with LUKS2 encryption and Btrfs filesystem.

1. **Wipe and partition the disk** (replace `/dev/sda` with your target disk):
```bash
sgdisk -Z /dev/sda
sgdisk -n1:0:+512M -t1:ef00 -c1:EFI -N2 -t2:8304 -c2:LINUXROOT /dev/sda
partprobe -s /dev/sda
```
This creates:
- Partition 1: 512 MB EFI System Partition (ESP)
- Partition 2: Linux root partition (remainder of the disk)

2. **Set up LUKS2 encryption** on the root partition:
```bash
cryptsetup luksFormat --type luks2 /dev/sda2
cryptsetup luksOpen /dev/sda2 lvm
```
Enter a passphrase when prompted.

3. **Format partitions and create Btrfs subvolumes**:
```bash
mkfs.vfat -F32 -n EFI /dev/sda1
mkfs.btrfs -f -L linuxroot /dev/mapper/lvm
mount /dev/mapper/lvm /mnt
mkdir /mnt/efi
mount /dev/sda1 /mnt/efi
btrfs subvolume create /mnt/home
```

## Step 3: Install Base System
1. **Update mirrorlist** (e.g., for Ukraine; adjust `--country` as needed):
```bash
reflector --country UA --age 24 --protocol http,https --sort rate --save /etc/pacman.d/mirrorlist
```
2. **Install essential packages**:
```bash
pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode vim nano cryptsetup btrfs-progs dosfstools util-linux git unzip sbctl kitty networkmanager sudo
```

## Step 4: Configure the System
1. **Enable locale**:
```bash
sed -i -e '/^#en_US.UTF-8/s/^#//' /mnt/etc/locale.gen
```
2. **Set system configuration** (adjust keymap, timezone, and hostname as needed):
```bash
systemd-firstboot --root /mnt --prompt
```
Example prompts and responses:
```
‣ Please enter system keymap name or number (empty to skip, "list" to list options): uk
/mnt/etc/vconsole.conf written.
‣ Please enter timezone name or number (empty to skip, "list" to list options): Europe/London
/mnt/etc/localtime written.
‣ Please enter hostname for new system (empty to skip): archinstall9001
/mnt/etc/hostname written.
```
3. **Generate locale**:
```bash
arch-chroot /mnt locale-gen
```
Output:
```
Generating locales...
  en_US.UTF-8... done
Generation complete.
```
4. **Create a user** (replace `acid` with your username):
```bash
arch-chroot /mnt useradd -G wheel -m acid
arch-chroot /mnt passwd acid
```
Set a password when prompted.
5. **Grant sudo privileges** to the wheel group:
```bash
sed -i -e '/^# %wheel ALL=(ALL:ALL) NOPASSWD: ALL/s/^# //' /mnt/etc/sudoers
```

## Step 5: Configure Unified Kernel Image (UKI)
1. **Create kernel command line file**:
```bash
echo "quiet rw" > /mnt/etc/kernel/cmdline
```
2. **Set up EFI directory**:
```bash
mkdir -p /mnt/efi/EFI/Linux
```
3. **Edit `/mnt/etc/mkinitcpio.conf`** to use systemd hooks:
```bash
HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole sd-encrypt block filesystems fsck)
```
4. **Update `/mnt/etc/mkinitcpio.d/linux.preset`** to generate UKIs:
```bash
# mkinitcpio preset file to generate UKIs
ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"
ALL_microcode=(/boot/*-ucode.img)
PRESETS=('default' 'fallback')
default_uki="/efi/EFI/Linux/arch-linux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
fallback_uki="/efi/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```
5. **Generate UKIs**:
```bash
arch-chroot /mnt mkinitcpio -P
```
Expected output includes successful generation of `/efi/EFI/Linux/arch-linux.efi` and `/efi/EFI/Linux/arch-linux-fallback.efi`. Ignore warnings about missing firmware for unused modules.
6. **Verify UKIs**:
```bash
ls -lR /mnt/efi
```
Expected output:
```
/mnt/efi:
total 4
drwxr-xr-x 3 root root 4096 Aug 25 20:51 EFI
/mnt/efi/EFI:
total 4
drwxr-xr-x 2 root root 4096 Aug 25 21:02 Linux
/mnt/efi/EFI/Linux:
total 118668
-rwxr-xr-x 1 root root 29928960 Aug 9 14:07 arch-linux.efi
-rwxr-xr-x 1 root root 91586048 Aug 9 14:07 arch-linux-fallback.efi
```

## Step 6: Enable Services and Install Bootloader
1. **Enable essential services**:
```bash
systemctl --root /mnt enable systemd-resolved systemd-timesyncd NetworkManager
systemctl --root /mnt mask systemd-networkd
```
2. **Install systemd-boot**:
```bash
arch-chroot /mnt bootctl install --esp-path=/efi
```

## Step 7: Reboot and Configure Secure Boot
1. **Sync and reboot into UEFI/BIOS**:
```bash
sync
systemctl reboot --firmware-setup
```
2. **Enter UEFI/BIOS Setup Mode**:
   - Access your UEFI/BIOS settings (consult your motherboard manual).
   - Set Secure Boot to "Setup Mode" to allow enrolling keys.
3. **Enroll Secure Boot keys** (after rebooting into the new system):
   - Log in as the user created earlier (e.g., `acid`).
   - Use `sbctl` to create and enroll keys:
```bash
sudo sbctl create-keys
sudo sbctl enroll-keys
```
   - Sign the UKIs:
```bash
sudo sbctl sign -s /efi/EFI/Linux/arch-linux.efi
sudo sbctl sign -s /efi/EFI/Linux/arch-linux-fallback.efi
```
4. **Enable Secure Boot** in UEFI/BIOS and reboot.

## Step 8: Finalize Installation
- Boot into your new Arch Linux system.
- Verify Secure Boot is active:
```bash
sbctl status
```
- Ensure the system boots correctly and all services (e.g., NetworkManager) are running:
```bash
systemctl status NetworkManager
```

## Notes
- This guide uses `amd-ucode` for AMD processors. For Intel CPUs, replace `amd-ucode` with `intel-ucode` in the `pacstrap` command.
- Adjust the country code in the `reflector` command based on your location.
