# What Broke and How It Was Fixed

## 1. Initial virtual disk was undersized

**Symptom:** The VM was created with a 20 GiB disk.

**Diagnosis:** This left little capacity for the base system and later labs.

**Fix:** The disk was increased to 40 GiB before partitioning. `lsblk` verified `/dev/sda` as 40 GiB and unpartitioned.

**Lesson:** Correct sizing is cheapest before filesystems and data exist.

## 2. EFI partition had the wrong GPT type

**Symptom:** `/dev/sda1` was 1 GiB but `lsblk` reported `Linux filesystem` rather than `EFI System`.

**Diagnosis:** Creating a partition does not automatically make it an EFI System Partition; its GPT type must be set explicitly.

**Fix:** In `fdisk`, partition 1 was selected and changed to type `1` (`EFI System`). Verification showed:

```text
sda1  1G   part  EFI System
sda2  39G  part  Linux filesystem
```

**Lesson:** Verify partition type, size, filesystem, and mount point independently.

## 3. systemd-boot random seed was exposed by vfat masks

**Symptom:** `bootctl install` warned that `/boot` and `/boot/loader/random-seed` were world-accessible and called this a security hole.

**Diagnosis:** `genfstab` had recorded `fmask=0022,dmask=0022` for the FAT EFI partition. FAT lacks Unix permission bits, so access is controlled by mount masks.

**Fix:** Both masks were changed to `0077` in `/etc/fstab`. Because the system was still in a chroot, `systemctl daemon-reload` could not apply anything. The current mount was explicitly remounted:

```bash
mount -o remount,fmask=0077,dmask=0077 /dev/sda1 /boot
```

`findmnt` and `stat` then confirmed the new masks and mode `700`.

**Lesson:** Treat boot random-seed warnings as security defects, not cosmetic messages.

## 4. EFI NVRAM was not updated from the chroot

**Symptom:** `bootctl install` copied boot files but reported that EFI variable modification was skipped. The first installed boot used `/EFI/BOOT/BOOTX64.EFI`.

**Diagnosis:** The command was running inside the installation chroot. The firmware fallback path allowed a successful first boot, but no normal firmware boot entry existed yet.

**Fix:** After booting the installed system under UEFI, `sudo bootctl install` was run again. It registered an active `Linux Boot Manager` entry. A second reboot confirmed the current loader as `/EFI/systemd/systemd-bootx64.efi`.

**Lesson:** Separate the files on the EFI System Partition from the NVRAM entries that point to those files; either one can exist without the other.

## 5. `/mnt` remained busy during shutdown preparation

**Symptom:** `umount -R /mnt` returned `target is busy`, even after the live shell changed directory to `/`.

**Diagnosis:** `findmnt -R /mnt` showed only the root mount remained. `fuser -vm /mnt` identified a leftover root-owned Bash process with PID 5956 whose root/current directory referenced `/mnt`.

**Fix:** A normal `SIGTERM` did not clear it, so the already-identified leftover chroot shell was terminated with `SIGKILL`. The filesystem then unmounted cleanly and `findmnt -R /mnt` returned no output.

**Lesson:** Diagnose a busy filesystem with `findmnt` and `fuser`; never jump directly to a lazy or forced unmount.

## 6. VM returned to the installer menu

**Symptom:** After installation, UTM displayed `Arch Linux install medium` again.

**Diagnosis:** The ISO remained attached, so the VM booted the installer rather than the virtual disk.

**Fix:** The VM was powered off and only the ISO was detached. The 40 GiB system disk remained attached. The next boot reached the installed `arch-lab login:` prompt.

**Lesson:** Installation media and installed storage are different boot targets; verify the active target after every first reboot.

## 7. Traditional `hostname` command was absent

**Symptom:** Bash returned `hostname: command not found`.

**Diagnosis:** Arch's minimal base installation provides `hostnamectl` through systemd but not the traditional `hostname` utility.

**Fix:** `inetutils` was installed. `hostname` then returned `arch-lab`.

**Lesson:** Minimal distributions intentionally omit familiar utilities; identify the owning package instead of assuming the installation is broken.

## 8. Kernel reported unsupported Intel TDX

**Symptom:** The boot error journal contained:

```text
virt/tdx: TDX not supported by the host platform
```

**Diagnosis:** UTM's emulated platform does not expose Intel TDX confidential-computing support.

**Fix:** No change was required. Boot, networking, storage, and systemd health checks all passed.

**Lesson:** Classify journal messages by operational impact. An error-priority message can still be benign for the current hardware model.
