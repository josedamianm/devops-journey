# Challenge 03 — Break and Repair systemd-boot

## Objective
The objective of this challenge is to break and repair the boot process (systemd-boot), to undertand completely it.

## Environment and storage layout
josemanco@arch-lab ~]$ sudo bootctl status
sudo bootctl list
System:
      Firmware: UEFI 2.70 (EDK II 1.00)
 Firmware Arch: x64
   Secure Boot: disabled (unsupported)
  TPM2 Support: no
  Measured UKI: no
   Measured OS: no
  Boot into FW: supported
 Platform Lang: n/a

Current Boot Loader:
        Product: systemd-boot 261.3-1-arch
       Features: ✓ Boot counting
                 ✓ Menu timeout control
                 ✓ One-shot menu timeout control
                 ✓ Default entry control
                 ✓ One-shot entry control
                 ✓ Support for XBOOTLDR partition
                 ✓ Support for passing random seed to OS
                 ✓ Load drop-in drivers
                 ✓ Support Type #1 sort-key field
                 ✓ Support @saved pseudo-entry
                 ✓ Support Type #1 devicetree field
                 ✓ Enroll SecureBoot keys
                 ✓ Retain SHIM protocols
                 ✓ Menu can be disabled
                 ✓ Multi-Profile UKIs are supported
                 ✓ Loader reports network boot URL
                 ✓ Support Type #1 uki field
                 ✓ Support Type #1 uki-url field
                 ✓ Loader reports active TPM2 PCR banks
                 ✓ Loader reports firmware keyboard layout
                 ✓ Loader measures SMBIOS information
      Partition: /dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81
         Loader: └─/boot//EFI/systemd/systemd-bootx64.efi
Keyboard Layout: en-US
  Current Entry: arch.conf

Random Seed:
 System Token: set
       Exists: yes

Available Boot Loaders on ESP:
          ESP: /boot (/dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81)
         File: ├─/boot//EFI/systemd/systemd-bootx64.efi (systemd-boot 261.3-1-arch)
               └─/boot//EFI/BOOT/BOOTX64.EFI (systemd-boot 261.3-1-arch)

Boot Loaders Listed in EFI Variables:
        Title: Linux Boot Manager
           ID: 0x0004
       Status: active, boot-order
    Partition: /dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81
         File: └─/boot//EFI/systemd/systemd-bootx64.efi

Boot Loader Entry Locations:
          ESP: /boot (/dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81, $BOOT)
       config: /boot//loader/loader.conf
        token: arch

Default Boot Loader Entry:
         type: Boot Loader Specification Type #1 (.conf)
        title: Arch Linux
           id: arch.conf
       source: /boot//loader/entries/arch.conf (on the EFI System Partition)
        linux: /boot//vmlinuz-linux
       initrd: /boot//initramfs-linux.img
      options: root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
         type: Boot Loader Specification Type #1 (.conf)
        title: Arch Linux (default) (selected)
           id: arch.conf
       source: /boot//loader/entries/arch.conf (on the EFI System Partition)
        linux: /boot//vmlinuz-linux
       initrd: /boot//initramfs-linux.img
      options: root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw

         type: Automatic
        title: Reboot Into Firmware Interface
           id: auto-reboot-to-firmware-setup
       source: /sys/firmware/efi/efivars/LoaderEntries-4a67b082-0a4c-41cf-b6c7-440b29bb8c4f (on the EFI System Partition)

## Fault introduced
For making the boot fail, we change the kernel, so the boot proccess wont be able to find it. we change from /boot//vmlinuz-linux to /boot//vmlinuz-linux.broken

```
[josemanco@arch-lab ~]$ sudo bootctl list
         type: Boot Loader Specification Type #1 (.conf)
        title: Arch Linux (default) (selected)
           id: arch.conf
       source: /boot//loader/entries/arch.conf (on the EFI System Partition)
        linux: /boot//vmlinuz-linux.broken (No such file or directory)
       initrd: /boot//initramfs-linux.img
      options: root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw

         type: Automatic
        title: Reboot Into Firmware Interface
           id: auto-reboot-to-firmware-setup
       source: /sys/firmware/efi/efivars/LoaderEntries-4a67b082-0a4c-41cf-b6c7-440b29bb8c4f (on the EFI System Partition)
```
As you can see the linux kernel says Not such file or directory for the bootctl list command

## Live ISO recovery
<img width="1278" height="839" alt="Evidence3_1" src="https://github.com/user-attachments/assets/a06c3d9b-7a19-40d7-960e-1debc2a5235f" />
<img width="958" height="420" alt="Evidence3_2" src="https://github.com/user-attachments/assets/81dfba8f-afb5-47f6-9c30-e88ecb857034" />
<img width="813" height="843" alt="Evidence3_3" src="https://github.com/user-attachments/assets/97ac326e-74bc-44e9-9dfe-c7b774d433f2" />

## Post-repair verification
```
-----------------------------------------
Evidence checkpoint 4:
-----------------------------------------
[josemanco@arch-lab ~]$ sudo bootctl status
sudo bootctl list

sudo grep -nE '^(title|linux|initrd|options)' \
  /boot/loader/entries/arch.conf

findmnt /
findmnt /boot

systemd-analyze
systemctl --failed
uname -r
[sudo] password for josemanco:
System:
      Firmware: UEFI 2.70 (EDK II 1.00)
 Firmware Arch: x64
   Secure Boot: disabled (unsupported)
  TPM2 Support: no
  Measured UKI: no
   Measured OS: no
  Boot into FW: supported
 Platform Lang: n/a

Current Boot Loader:
        Product: systemd-boot 261.3-1-arch
       Features: ✓ Boot counting
                 ✓ Menu timeout control
                 ✓ One-shot menu timeout control
                 ✓ Default entry control
                 ✓ One-shot entry control
                 ✓ Support for XBOOTLDR partition
                 ✓ Support for passing random seed to OS
                 ✓ Load drop-in drivers
                 ✓ Support Type #1 sort-key field
                 ✓ Support @saved pseudo-entry
                 ✓ Support Type #1 devicetree field
                 ✓ Enroll SecureBoot keys
                 ✓ Retain SHIM protocols
                 ✓ Menu can be disabled
                 ✓ Multi-Profile UKIs are supported
                 ✓ Loader reports network boot URL
                 ✓ Support Type #1 uki field
                 ✓ Support Type #1 uki-url field
                 ✓ Loader reports active TPM2 PCR banks
                 ✓ Loader reports firmware keyboard layout
                 ✓ Loader measures SMBIOS information
      Partition: /dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81
         Loader: └─/boot//EFI/systemd/systemd-bootx64.efi
Keyboard Layout: en-US
  Current Entry: arch.conf

Random Seed:
 System Token: set
       Exists: yes

Available Boot Loaders on ESP:
          ESP: /boot (/dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81)
         File: ├─/boot//EFI/systemd/systemd-bootx64.efi (systemd-boot 261.3-1-arch)
               └─/boot//EFI/BOOT/BOOTX64.EFI (systemd-boot 261.3-1-arch)

Boot Loaders Listed in EFI Variables:
        Title: Linux Boot Manager
           ID: 0x0004
       Status: active, boot-order
    Partition: /dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81
         File: └─/boot//EFI/systemd/systemd-bootx64.efi

Boot Loader Entry Locations:
          ESP: /boot (/dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81, $BOOT)
       config: /boot//loader/loader.conf
        token: arch

Default Boot Loader Entry:
         type: Boot Loader Specification Type #1 (.conf)
        title: Arch Linux
           id: arch.conf
       source: /boot//loader/entries/arch.conf (on the EFI System Partition)
        linux: /boot//vmlinuz-linux
       initrd: /boot//initramfs-linux.img
      options: root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
         type: Boot Loader Specification Type #1 (.conf)
        title: Arch Linux (default) (selected)
           id: arch.conf
       source: /boot//loader/entries/arch.conf (on the EFI System Partition)
        linux: /boot//vmlinuz-linux
       initrd: /boot//initramfs-linux.img
      options: root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw

         type: Automatic
        title: Reboot Into Firmware Interface
           id: auto-reboot-to-firmware-setup
       source: /sys/firmware/efi/efivars/LoaderEntries-4a67b082-0a4c-41cf-b6c7-440b29bb8c4f (on the EFI System Partition)
1:title Arch Linux
2:linux /vmlinuz-linux
3:initrd /initramfs-linux.img
4:options root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
TARGET SOURCE    FSTYPE OPTIONS
/      /dev/sda2 ext4   rw,relatime
TARGET SOURCE    FSTYPE OPTIONS
/boot  /dev/sda1 vfat   rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro
Startup finished in 1.652s (kernel) + 7.773s (initrd) + 15.189s (userspace) = 24.615s
graphical.target reached after 15.181s in userspace.
  UNIT LOAD ACTIVE SUB DESCRIPTION

0 loaded units listed.
7.2.6-arch2-1
[josemanco@arch-lab ~]$
[josemanco@arch-lab ~]$
[josemanco@arch-lab ~]$ who -b
journalctl -b --no-pager -n 30
         system boot  2026-09-18 11:06
Sep 18 11:07:10 arch-lab systemd[323]: Starting D-Bus User Message Bus Socket...
Sep 18 11:07:10 arch-lab systemd[323]: Listening on GnuPG network certificate management daemon.
Sep 18 11:07:10 arch-lab systemd[323]: Listening on GnuPG cryptographic agent and passphrase cache (access for web browsers).
Sep 18 11:07:10 arch-lab systemd[323]: Listening on GnuPG cryptographic agent and passphrase cache (restricted).
Sep 18 11:07:10 arch-lab systemd[323]: Listening on GnuPG cryptographic agent (ssh-agent emulation).
Sep 18 11:07:10 arch-lab systemd[323]: Listening on GnuPG cryptographic agent and passphrase cache.
Sep 18 11:07:10 arch-lab systemd[323]: Listening on GnuPG public key management service.
Sep 18 11:07:10 arch-lab systemd[323]: Listening on p11-kit server.
Sep 18 11:07:10 arch-lab systemd[323]: Listening on Query the User Interactively for a Password.
Sep 18 11:07:10 arch-lab systemd[323]: Listening on Disk Image Download Service Socket.
Sep 18 11:07:10 arch-lab systemd[323]: Listening on Journal Log Access Socket.
Sep 18 11:07:10 arch-lab systemd[323]: Listening on Virtual Machine and Container Registration Service Socket.
Sep 18 11:07:10 arch-lab systemd[323]: Listening on Simple File System Backed Storage Provider.
Sep 18 11:07:10 arch-lab systemd[323]: Listening on D-Bus User Message Bus Socket.
Sep 18 11:07:10 arch-lab systemd[323]: Reached target Sockets.
Sep 18 11:07:10 arch-lab systemd[323]: Reached target Basic System.
Sep 18 11:07:10 arch-lab systemd[323]: Reached target Main User Target.
Sep 18 11:07:10 arch-lab systemd[323]: Startup finished in 828ms.
Sep 18 11:07:10 arch-lab systemd[1]: Started User Manager for UID 1000.
Sep 18 11:07:10 arch-lab systemd[1]: Started Session 1 of User josemanco.
Sep 18 11:07:13 arch-lab kernel: clocksource: Watchdog remote CPU 1 read timed out
Sep 18 11:07:17 arch-lab sudo[349]: josemanco : TTY=pts/0 ; PWD=/home/josemanco ; USER=root ; COMMAND=/usr/bin/bootctl status
Sep 18 11:07:17 arch-lab sudo[349]: pam_unix(sudo:session): session opened for user root(uid=0) by josemanco(uid=1000)
Sep 18 11:07:18 arch-lab sudo[349]: pam_unix(sudo:session): session closed for user root
Sep 18 11:07:18 arch-lab sudo[358]: josemanco : TTY=pts/0 ; PWD=/home/josemanco ; USER=root ; COMMAND=/usr/bin/bootctl list
Sep 18 11:07:18 arch-lab sudo[358]: pam_unix(sudo:session): session opened for user root(uid=0) by josemanco(uid=1000)
Sep 18 11:07:18 arch-lab sudo[358]: pam_unix(sudo:session): session closed for user root
Sep 18 11:07:18 arch-lab sudo[365]: josemanco : TTY=pts/0 ; PWD=/home/josemanco ; USER=root ; COMMAND=/usr/bin/grep -nE ^(title|linux|initrd|options) /boot/loader/entries/arch.conf
Sep 18 11:07:18 arch-lab sudo[365]: pam_unix(sudo:session): session opened for user root(uid=0) by josemanco(uid=1000)
Sep 18 11:07:18 arch-lab sudo[365]: pam_unix(sudo:session): session closed for user root
[josemanco@arch-lab ~]$

```
