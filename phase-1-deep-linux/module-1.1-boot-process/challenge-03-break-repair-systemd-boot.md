# Challenge 03 — Break and Repair systemd-boot

## Objective

The goal of this challenge was to deliberately break a systemd-boot entry, observe what happened, and recover the machine from an Arch Linux live ISO. I wanted to understand where the boot process stopped instead of simply running repair commands until the VM started again.

The boot chain involved in this exercise was:

1. UEFI loads systemd-boot.
2. systemd-boot reads the selected boot entry.
3. The entry points to the kernel and initramfs on the EFI System Partition (ESP).
4. systemd-boot loads the kernel and hands over control.
5. The kernel starts the initramfs.
6. The initramfs mounts the real root filesystem.
7. systemd starts as PID 1 and brings up userspace.

## Environment and storage layout

This challenge was completed in an Arch Linux virtual machine running under UTM with UEFI firmware and systemd-boot.

The relevant disk layout was:

```text
/dev/sda1  -> /boot  -> EFI System Partition (vfat)
/dev/sda2  -> /      -> Arch Linux root filesystem (ext4)
```

The Type #1 systemd-boot entry was stored here:

```text
/boot/loader/entries/arch.conf
```

Its normal contents were:

```text
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

One detail that initially caused some confusion was the path shown by `bootctl`. The entry itself contains:

```text
linux /vmlinuz-linux
```

It does not contain `/boot//vmlinuz-linux`. `bootctl` displays `/boot//vmlinuz-linux` because it joins the ESP mount point, `/boot`, with the absolute path stored in the entry, `/vmlinuz-linux`. The double slash in that output was not the problem.

## Healthy state

Before changing anything, I checked the working configuration with `bootctl status` and `bootctl list`. The important parts of the output were:

```text
Current Boot Loader:
        Product: systemd-boot 261.3-1-arch
      Partition: /dev/disk/by-partuuid/2877263b-3d73-4bd4-9af6-950f03a0bf81
         Loader: /boot//EFI/systemd/systemd-bootx64.efi
  Current Entry: arch.conf
```

The selected entry correctly referenced the kernel and initramfs:

```text
type: Boot Loader Specification Type #1 (.conf)
title: Arch Linux
id: arch.conf
source: /boot//loader/entries/arch.conf
linux: /boot//vmlinuz-linux
initrd: /boot//initramfs-linux.img
options: root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

This gave me a known healthy baseline: UEFI could load systemd-boot, systemd-boot could read `arch.conf`, and the installed operating system could boot normally.

### Backup evidence

Before introducing the fault, I saved a known-good copy of the entry as:

```text
/root/arch.conf.known-good
```

After the repair, I compared the SHA-256 hashes:

```text
7070628204ae5e617ba8f519bb62c00434cf63615888b0ffaed3cf06004bd743  /boot/loader/entries/arch.conf
7070628204ae5e617ba8f519bb62c00434cf63615888b0ffaed3cf06004bd743  /root/arch.conf.known-good
```

The hashes match, which proves that the repaired entry and the known-good backup were byte-for-byte identical at the time of verification. I used the following command to compare them:

```bash
sha256sum /boot/loader/entries/arch.conf /root/arch.conf.known-good
```

## Fault introduced

I introduced the failure by editing the `linux` line in `arch.conf`. I changed it from:

```text
linux /vmlinuz-linux
```

to:

```text
linux /vmlinuz-linux.broken
```

I did not change, rename, delete, or damage the kernel. The real kernel remained intact at `/boot/vmlinuz-linux`. The only change was the path stored in the systemd-boot entry, which now pointed to a file that did not exist.

After making the change, `bootctl list` showed:

```text
type: Boot Loader Specification Type #1 (.conf)
title: Arch Linux (default) (selected)
id: arch.conf
source: /boot//loader/entries/arch.conf (on the EFI System Partition)
linux: /boot//vmlinuz-linux.broken (No such file or directory)
initrd: /boot//initramfs-linux.img
options: root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

This was useful evidence because it showed that systemd-boot could still find and parse the entry, but the kernel path inside it was invalid.

## Failure observed

When I rebooted the VM, the boot failure passed too quickly for me to capture a screenshot. The installed Arch system did not start, and UTM/UEFI fell back to the Firmware Interface. From there, I booted the Arch ISO to recover the installation.

Although I do not have a screenshot of the brief error message, the failure is supported by this evidence chain:

1. `bootctl list` showed that the selected entry referenced a missing kernel path.
2. The VM failed to reach the installed operating system.
3. UTM/UEFI fell back to the Firmware Interface.
4. I had to boot the Arch live ISO to perform the recovery.
5. Once the path was repaired, the installed system booted normally again.

## Diagnosis

The key diagnostic line was:

```text
linux: /boot//vmlinuz-linux.broken (No such file or directory)
```

This narrowed the problem down to the boot entry. UEFI had already done its job because systemd-boot was running. systemd-boot had also found and parsed `arch.conf`. The failure happened when it tried to load the kernel file named by the `linux` directive.

The expected path was `/vmlinuz-linux`, but the entry requested `/vmlinuz-linux.broken`. Since the real kernel still existed, there was no reason to replace or rebuild it. The entry simply needed to point to the correct file again.

## Repair

Because the installed OS could not boot, I started the VM from the Arch live ISO. I mounted the installed root filesystem and then mounted the ESP in its expected location:

```bash
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```

I then corrected the entry at:

```text
/mnt/boot/loader/entries/arch.conf
```

The essential repair was changing:

```text
linux /vmlinuz-linux.broken
```

back to:

```text
linux /vmlinuz-linux
```

The repaired entry was:

```text
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

Before rebooting, I could verify the entry and confirm that the real kernel was present with:

```bash
grep -nE '^(title|linux|initrd|options)' /mnt/boot/loader/entries/arch.conf
ls -l /mnt/boot/vmlinuz-linux
```

I then unmounted the filesystems and rebooted:

```bash
umount -R /mnt
reboot
```

After removing the live ISO, the VM booted from the virtual disk normally.

## Root cause

The root cause was an incorrect kernel pathname in the Type #1 systemd-boot entry. The failure can be followed through the boot chain:

1. UEFI successfully loaded systemd-boot.
2. systemd-boot successfully found and read `arch.conf`.
3. The entry requested `/vmlinuz-linux.broken`.
4. That file did not exist on the ESP.
5. The real `/vmlinuz-linux` file still existed and had not been modified.
6. systemd-boot therefore could not load the requested kernel or transfer control to it.
7. Since the kernel never started, the initramfs never ran.
8. Because the kernel and initramfs did not run, systemd never started as PID 1.
9. Restoring `/vmlinuz-linux` in the entry repaired the boot chain.

The failure was between systemd-boot reading the entry and loading the kernel. It was not a systemd service failure, an initramfs problem, a damaged kernel, or a broken installation of systemd-boot.

## Why `bootctl install` and `mkinitcpio` were unnecessary

### `bootctl install`

I did not run `bootctl install` because systemd-boot was already installed and working. UEFI loaded it successfully, and it was able to find and parse `arch.conf`. Reinstalling the boot-loader executable would not have corrected the bad `linux` line inside the entry.

The smallest relevant repair was to fix the path, not reinstall a component that had already proved it was functioning.

### `mkinitcpio -P`

I also avoided `mkinitcpio -P`. That command regenerates the initramfs images, but there was no evidence that the initramfs was missing or corrupt. The entry still referenced `/initramfs-linux.img`; the boot process failed earlier, while systemd-boot was trying to load the nonexistent kernel path.

Regenerating the initramfs would not have fixed `/vmlinuz-linux.broken`. It would also have introduced an unrelated change during recovery, making the diagnosis less precise.

## Post-repair verification

After booting the installed system again, I checked the repaired entry:

```text
1:title Arch Linux
2:linux /vmlinuz-linux
3:initrd /initramfs-linux.img
4:options root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

`bootctl list` no longer reported a missing file:

```text
linux: /boot//vmlinuz-linux
initrd: /boot//initramfs-linux.img
```

The mounted filesystems were also correct:

```text
TARGET SOURCE    FSTYPE OPTIONS
/      /dev/sda2 ext4   rw,relatime

TARGET SOURCE    FSTYPE OPTIONS
/boot  /dev/sda1 vfat   rw,relatime
```

The running kernel was:

```text
7.2.6-arch2-1
```

`systemd-analyze` confirmed that all three main stages completed:

```text
Startup finished in 1.652s (kernel) + 7.773s (initrd) + 15.189s (userspace) = 24.615s
graphical.target reached after 15.181s in userspace.
```

Finally, `systemctl --failed` returned:

```text
0 loaded units listed.
```

Together, these checks show that systemd-boot loaded the repaired entry, the kernel and initramfs ran, the root filesystem mounted correctly, systemd started, and the machine reached `graphical.target` without failed units.

## What I learned

The main lesson from this challenge was to identify the exact point where the boot chain stops. A systemd-boot entry does not contain or modify the kernel; it only tells the boot loader where the kernel is located. A one-line path error was enough to prevent the whole operating system from starting even though the kernel itself was still healthy.

I also learned that recovery evidence does not always come from a perfect screenshot. The error disappeared quickly in this case, so I had to connect several observations: the missing path reported by `bootctl`, the VM falling back to firmware, the live-ISO recovery, and the successful boot after restoring the entry.

Most importantly, I learned not to use broad repair commands without a reason. Neither systemd-boot nor the initramfs was broken. Fixing the incorrect path was the smallest change and directly addressed the actual cause of the failure.
