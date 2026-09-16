# Boot Process Notes

## Observed boot chain

```text
UTM starts the VM
  → EDK II UEFI firmware initializes emulated hardware
  → firmware reads its Linux Boot Manager NVRAM entry
  → systemd-boot is loaded from the EFI System Partition
  → systemd-boot reads loader.conf and arch.conf
  → the Linux kernel and initramfs are loaded from /boot
  → initramfs finds the ext4 root filesystem by UUID
  → control switches to the real root filesystem
  → systemd starts as PID 1
  → targets and services are activated
  → getty presents the tty1 login prompt
```

## UEFI firmware

UEFI runs before Linux. In this VM it is EDK II firmware. It discovers boot options from NVRAM and executable files on the FAT32 EFI System Partition. The final validation showed an active `Linux Boot Manager` NVRAM entry pointing to `systemd-bootx64.efi`.

The fallback executable at `/EFI/BOOT/BOOTX64.EFI` is useful when an NVRAM entry is missing. It enabled the first boot after the chroot installation, but the normal registered boot entry was created afterward.

## Boot loader

systemd-boot is a UEFI boot manager. It does not find the Linux root filesystem itself. It reads a Boot Loader Specification entry that identifies:

- the kernel image;
- the initramfs image;
- kernel command-line parameters, including the root filesystem UUID.

The boot entry used `root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw`, avoiding dependence on a potentially unstable device name such as `/dev/sda2`.

## Kernel and initramfs

The firmware and boot loader place the kernel and initramfs in memory. The kernel initializes CPU, memory, drivers, and the initial userspace. The initramfs contains enough userspace and driver support to discover and mount the real root filesystem. After that mount succeeds, the system switches from the temporary initramfs root to the installed ext4 root.

The measured startup split was:

```text
1.735s kernel + 7.971s initrd + 17.975s userspace = 27.682s
```

Because this guest uses x86_64 emulation, these values include emulation overhead and should not be treated as bare-metal performance.

## PID 1 and systemd

After the real root filesystem is available, systemd becomes PID 1. PID 1 is special because it:

- establishes the userspace service dependency graph;
- starts targets, sockets, mounts, timers, and services;
- adopts orphaned processes;
- reaps terminated child processes;
- coordinates shutdown and reboot.

`systemctl --failed` reported zero failed units after installation. NetworkManager and sshd were enabled so systemd starts them on future boots.

## Targets and services

A target groups units into a desired system state. The VM reached `graphical.target`; with no display manager installed, it still provides the multi-user service state and console login. A service such as `sshd.service` has lifecycle and dependency metadata that systemd applies during startup and shutdown.

## Journal and boot analysis

Useful commands demonstrated in this lab:

```bash
systemd-analyze
systemctl --failed
journalctl -b
journalctl -b -p err
bootctl status
```

`systemd-analyze` separates kernel, initramfs, and userspace time. `journalctl -b` scopes logs to the current boot. `bootctl status` connects the running system back to its firmware, EFI partition, current loader, and selected entry.

## Security observations

- The EFI System Partition uses restrictive masks because FAT does not implement Unix file ownership and modes.
- The systemd-boot random seed must not be world-readable.
- Routine administration uses a named user with sudo rather than direct root login.
- SSH key authentication was proven before password authentication was disabled.
- Root SSH login is disabled.

## Interview summary

The boot path is not one opaque action. Each stage hands control and state to the next:

1. UEFI chooses an EFI executable.
2. systemd-boot chooses a kernel entry.
3. The kernel initializes hardware and starts the initramfs environment.
4. The initramfs discovers and mounts the real root filesystem.
5. systemd as PID 1 builds userspace from units and targets.
6. A getty or display manager presents login.

A failure must be located at the correct boundary: firmware entry, boot-loader file, kernel parameters, initramfs storage discovery, root mount, PID 1, or later service activation.
