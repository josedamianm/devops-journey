# Challenge 03 — Break and Repair systemd-boot

## Objective

Intentionally make the installed Arch Linux system unbootable by configuring its systemd-boot entry to reference a nonexistent kernel, then recover it from the Arch live ISO without reinstalling the bootloader, rebuilding the initramfs, restoring a VM snapshot, or reinstalling the operating system.

## Environment and storage layout

- Hypervisor: UTM on macOS
- Guest architecture: emulated x86_64
- Firmware: UEFI 2.70 (EDK II 1.00)
- Boot manager: systemd-boot 261.3-1-arch
- Boot entry: `/boot/loader/entries/arch.conf`
- EFI System Partition: `/dev/sda1`, FAT32, mounted at `/boot`
- Root filesystem: `/dev/sda2`, ext4, mounted at `/`
- Installed kernel: `/boot/vmlinuz-linux`
- Installed initramfs: `/boot/initramfs-linux.img`

## Healthy state

Before introducing the fault, `bootctl status` showed that UEFI had loaded systemd-boot from the EFI System Partition and that the current entry was `arch.conf`:

```text
Firmware: UEFI 2.70 (EDK II 1.00)
Product: systemd-boot 261.3-1-arch
Partition: /dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81
Loader: /boot//EFI/systemd/systemd-bootx64.efi
Current Entry: arch.conf
```

The healthy Type #1 boot entry resolved to the existing kernel and initramfs:

```text
title: Arch Linux
id: arch.conf
source: /boot//loader/entries/arch.conf
linux: /boot//vmlinuz-linux
initrd: /boot//initramfs-linux.img
options: root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

A protected known-good copy of the entry was saved at `/root/arch.conf.known-good`. Matching SHA-256 hashes proved that the backup was byte-for-byte identical to the healthy entry:

```text
backup-ready
7070628204ae5e617ba8f519bb62c00434cf63615888b0ffaed3cf06004bd743  /boot/loader/entries/arch.conf
7070628204ae5e617ba8f519bb62c00434cf63615888b0ffaed3cf06004bd743  /root/arch.conf.known-good
```

## Fault introduced

I did not change, rename, or delete the kernel. I changed only the `linux` path in `/boot/loader/entries/arch.conf`:

```diff
-linux /vmlinuz-linux
+linux /vmlinuz-linux.broken
```

The complete broken entry was:

```text
1:title Arch Linux
2:linux /vmlinuz-linux.broken
3:initrd /initramfs-linux.img
4:options root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

The following checks confirmed that the configured path was invalid while the real kernel remained intact:

```text
confirmed: configured kernel does not exist
real kernel remains intact
```

`bootctl list` detected the invalid reference before rebooting:

```text
type: Boot Loader Specification Type #1 (.conf)
title: Arch Linux (default) (selected)
id: arch.conf
source: /boot//loader/entries/arch.conf (on the EFI System Partition)
linux: /boot//vmlinuz-linux.broken (No such file or directory)
initrd: /boot//initramfs-linux.img
options: root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

The double slash in `/boot//vmlinuz-linux.broken` is how `bootctl` displays the ESP mount path joined to the absolute path stored in the entry. The entry itself contained `linux /vmlinuz-linux.broken`.

## Failure observed

After rebooting, the VM did not reach the installed kernel or login prompt. The failure message passed too quickly to capture, and UTM/UEFI fell back to the Firmware Interface.

The failure is supported by the complete evidence chain:

1. Before reboot, `bootctl list` reported `/vmlinuz-linux.broken` as missing.
2. The real `/vmlinuz-linux` remained present, proving that the fault was limited to the entry.
3. The installed system did not boot and UEFI returned to its Firmware Interface.
4. Recovery had to be performed from the Arch live ISO.
5. The system booted normally after only the entry path was corrected.

## Live ISO recovery

The Arch installation medium was booted in UEFI mode. `lsblk -f` identified `/dev/sda2` as the installed ext4 root filesystem and `/dev/sda1` as the FAT32 EFI System Partition.

The filesystems were mounted in the required order:

```bash
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```

The mounted layout was verified as:

```text
/dev/sda2 on /mnt      type ext4
/dev/sda1 on /mnt/boot type vfat
```

The first screenshot records the live ISO, storage discovery, mounts, and files present on both filesystems:

<img width="1278" height="839" alt="Arch live ISO storage discovery and mounts" src="https://github.com/user-attachments/assets/a06c3d9b-7a19-40d7-960e-1debc2a5235f" />

## Diagnosis

The mounted system showed both the broken configuration and the intact real kernel:

```bash
cat /mnt/boot/loader/entries/arch.conf
ls -lh /mnt/boot/vmlinuz*
test ! -e /mnt/boot/vmlinuz-linux.broken \
  && echo "diagnosis: configured kernel is missing"
test -s /mnt/boot/vmlinuz-linux \
  && echo "diagnosis: real kernel exists"
```

The checks returned:

```text
diagnosis: configured kernel is missing
diagnosis: real kernel exists
```

A comparison with the known-good backup isolated the fault to one line:

```diff
--- /mnt/root/arch.conf.known-good
+++ /mnt/boot/loader/entries/arch.conf
@@
-linux /vmlinuz-linux
+linux /vmlinuz-linux.broken
```

<img width="958" height="420" alt="Diagnosis of the broken systemd-boot entry" src="https://github.com/user-attachments/assets/81dfba8f-afb5-47f6-9c30-e88ecb857034" />

## Repair

I entered the installed system with:

```bash
arch-chroot /mnt
```

I edited `/boot/loader/entries/arch.conf` and restored:

```text
linux /vmlinuz-linux
```

The corrected entry was verified against the known-good backup. The configured kernel existed, and the entry matched the backup.

No bootloader installation or initramfs rebuild was performed. After leaving the chroot, I initially typed the wrong path in `umount -R /mount`; it failed without changing anything. I corrected it to:

```bash
umount -R /mnt
findmnt -R /mnt
```

`findmnt -R /mnt` returned no output, confirming that the recovered installation was fully unmounted before rebooting.

<img width="813" height="843" alt="Chroot repair and filesystem cleanup" src="https://github.com/user-attachments/assets/97ac326e-74bc-44e9-9dfe-c7b774d433f2" />

## Root cause

The boot chain failed at the handoff from systemd-boot to the Linux kernel:

1. UEFI successfully loaded the systemd-boot EFI executable.
2. systemd-boot successfully read the Type #1 entry `arch.conf`.
3. The entry requested `/vmlinuz-linux.broken` from the EFI System Partition.
4. That file did not exist.
5. The real `/vmlinuz-linux` and `/initramfs-linux.img` were still intact.
6. systemd-boot could not load the requested kernel, so it could not transfer control to it.
7. Because the kernel never started, the initramfs was not executed and systemd never started as PID 1.
8. Restoring the correct kernel path repaired the failed handoff.

## Why `bootctl install` and `mkinitcpio -P` were unnecessary

`bootctl install` was unnecessary because neither the systemd-boot EFI executable nor its UEFI registration was damaged. UEFI could still launch systemd-boot, and the fault was inside one entry file.

`mkinitcpio -P` was unnecessary because the existing initramfs was present and had not been changed or corrupted. The boot process failed before the initramfs could be loaded.

The smallest valid repair was therefore to correct one path in `arch.conf`. Reinstalling unrelated components would have added risk without addressing the root cause more directly.

## Post-repair verification

After removing the live ISO and booting from disk, `bootctl status` confirmed:

```text
Product: systemd-boot 261.3-1-arch
Current Entry: arch.conf
Loader: /boot//EFI/systemd/systemd-bootx64.efi
```

`bootctl list` and the entry file showed the corrected paths:

```text
linux: /boot//vmlinuz-linux
initrd: /boot//initramfs-linux.img
```

```text
1:title Arch Linux
2:linux /vmlinuz-linux
3:initrd /initramfs-linux.img
4:options root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

The installed filesystems were mounted correctly:

```text
TARGET SOURCE    FSTYPE OPTIONS
/      /dev/sda2 ext4   rw,relatime
/boot  /dev/sda1 vfat   rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=ascii,shortname=mixed,utf8,errors=remount-ro
```

The repaired system reached its target with no failed units:

```text
Startup finished in 1.652s (kernel) + 7.773s (initrd) + 15.189s (userspace) = 24.615s
graphical.target reached after 15.181s in userspace.

0 loaded units listed.
7.2.6-arch2-1
```

A new successful boot was recorded:

```text
system boot  2026-09-18 11:06
```

## What I learned

- A bootloader can run correctly while the operating system remains unbootable because a boot entry points to the wrong kernel path.
- A Type #1 systemd-boot entry describes the kernel, initramfs, and kernel command line; it does not contain the kernel itself.
- Diagnosis should identify the exact failed handoff before changing anything.
- The live ISO plus `mount` and `arch-chroot` provides a recovery environment for an installed system that cannot boot.
- The root filesystem must be mounted first and the EFI System Partition at `/mnt/boot` before editing this configuration.
- Recovery should make the smallest necessary change. Reinstalling systemd-boot or rebuilding initramfs would not have been justified for this fault.

## Result

The controlled fault was reproduced, diagnosed from the Arch live ISO, repaired inside a chroot, and verified by a successful boot of the installed system. The repair changed only the incorrect kernel path and preserved the existing bootloader, kernel, initramfs, filesystems, and UEFI registration.
