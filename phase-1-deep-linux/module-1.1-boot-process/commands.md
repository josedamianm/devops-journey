# Command Record — Arch Linux UTM Installation

Commands are grouped by installation phase. Device names are specific to this VM and must be rediscovered before reuse elsewhere.

## Live-environment checks

```bash
cat /sys/firmware/efi/fw_platform_size
ping -c 3 archlinux.org
timedatectl status
lsblk -o NAME,SIZE,TYPE,FSTYPE,PARTTYPENAME,MOUNTPOINTS /dev/sda
```

Observed UEFI bitness: `64`. Network test: three packets received with zero loss. The clock was synchronized in UTC.

## Partitioning

```bash
fdisk /dev/sda
```

Interactive plan:

```text
g                         # create GPT
n, defaults, +1G          # /dev/sda1
 t, partition 1, type 1   # EFI System
n, defaults               # /dev/sda2, remaining space
w                         # write table
```

The EFI type was initially left as `Linux filesystem`; it was corrected in a second `fdisk` session before formatting.

## Formatting and mounting

```bash
mkfs.fat -F 32 /dev/sda1
mkfs.ext4 /dev/sda2
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
lsblk -f /dev/sda
```

## Mirrors and base installation

```bash
reflector \
  --country Spain,France,Germany \
  --age 24 \
  --protocol https \
  --latest 20 \
  --sort rate \
  --save /etc/pacman.d/mirrorlist

pacstrap -K /mnt \
  base linux linux-firmware \
  networkmanager sudo nano \
  man-db man-pages git openssh \
  qemu-guest-agent

genfstab -U /mnt > /mnt/etc/fstab
arch-chroot /mnt
```

## Time, locale, and hostname

```bash
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
hwclock --systohc
locale-gen
printf 'LANG=en_GB.UTF-8\n' > /etc/locale.conf
printf 'arch-lab\n' > /etc/hostname
printf '127.0.0.1 localhost\n::1 localhost\n127.0.1.1 arch-lab.localdomain arch-lab\n' > /etc/hosts
```

Generated locales:

```text
en_GB.UTF-8
es_ES.UTF-8
```

## Users, sudo, and services

```bash
passwd
useradd -m -G wheel -s /bin/bash josemanco
passwd josemanco
id josemanco
pacman -S vim
EDITOR=vim visudo
visudo -c
systemctl enable NetworkManager sshd
systemctl is-enabled NetworkManager sshd
```

Sudo was tested before reboot:

```bash
su - josemanco
sudo -k
sudo whoami
exit
```

Observed result: `root`.

## systemd-boot

```bash
bootctl install
```

Files created on the EFI System Partition included:

```text
/EFI/systemd/systemd-bootx64.efi
/EFI/BOOT/BOOTX64.EFI
```

`/boot/loader/loader.conf` and `/boot/loader/entries/arch.conf` were then created as recorded in the lab report.

## EFI mount hardening

The generated vfat options were changed in `/etc/fstab`:

```text
fmask=0022,dmask=0022
```

to:

```text
fmask=0077,dmask=0077
```

The running mount was updated and verified:

```bash
mount -o remount,fmask=0077,dmask=0077 /dev/sda1 /boot
findmnt -no OPTIONS /boot
stat -c '%a %n' /boot /boot/loader/random-seed
```

Observed permissions: `700` for `/boot` and `/boot/loader/random-seed`.

## Unmounting and first boot

```bash
exit
findmnt -R /mnt
fuser -vm /mnt
# A leftover chroot bash process was identified and terminated.
umount -R /mnt
findmnt -R /mnt
poweroff
```

The ISO was detached in UTM before the VM was started again.

## Post-installation

```bash
sudo pacman -Syu
sudo pacman -S inetutils
hostname
hostnamectl --static
systemctl is-active NetworkManager sshd
ip -br addr
```

## SSH key installation and hardening

From macOS:

```bash
ssh-copy-id josemanco@192.168.64.4
ssh josemanco@192.168.64.4
```

On Arch:

```bash
sudo vim /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t
sudo systemctl reload sshd
sudo sshd -T
```

A second key-only SSH session was opened before the original session was closed.

## Final validation

```bash
sudo bootctl install
sudo reboot
sudo bootctl status
systemd-analyze
systemctl --failed
sudo journalctl -b -p err --no-pager
```
