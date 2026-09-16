# Challenge 2 — Boot Chain From Memory

## Submission

```text
Power-on → UEFI firmware → Bootloader (EFI app on the ESP) → Linux kernel → initramfs (early userspace) → real root filesystem → PID 1 (systemd) → targets → services → login prompt
```

When the UTM VM starts, firmware executes before Linux. UTM provides EDK II UEFI firmware. It initializes the virtual platform and checks NVRAM boot entries, each of which identifies an EFI executable on an EFI System Partition. In this VM the ESP is `/dev/sda1`, a FAT32 filesystem mounted at `/boot`. If no normal boot entry works, x86_64 UEFI can use the fallback executable at `\EFI\BOOT\BOOTX64.EFI`.

The bootloader is an EFI application. This VM uses `systemd-bootx64.efi`. systemd-boot reads its loader configuration and Boot Loader Specification entry, optionally displays a menu, loads `/vmlinuz-linux` and `/initramfs-linux.img` into memory, and supplies the kernel command line. The `root=` parameter identifies the real root filesystem. Control then passes to the kernel and the bootloader is no longer involved.

The kernel decompresses and initializes core facilities such as memory management, scheduling, and built-in drivers. It unpacks the initramfs cpio archive into a temporary RAM-backed root filesystem and executes `/init` from that environment. This early userspace process runs as PID 1 until it replaces itself with the final init process.

On Arch, mkinitcpio builds the initramfs. It contains the tools, hooks, and modules needed to discover the real root filesystem. This can include storage and filesystem drivers, udev, LUKS unlocking, or LVM assembly. It resolves the root filesystem requested on the kernel command line, mounts it, and uses `switch_root` to make it `/`. It then executes `/sbin/init` from the installed system while retaining PID 1. On this Arch system, `/sbin/init` resolves to systemd.

The real root filesystem is `/dev/sda2`, ext4, mounted at `/`. It provides `/usr`, `/etc`, binaries, libraries, and systemd units. systemd generators translate configuration such as `/etc/fstab` into runtime units. The EFI partition is mounted at `/boot` with restrictive FAT masks. Editing `fstab` alone does not change an already-active mount; the installation therefore used an explicit remount to apply `fmask=0077,dmask=0077`, while a normal reboot applies the saved options automatically.

systemd remains PID 1. It runs generators, builds a dependency graph, and activates mounts, sockets, targets, and services. Targets are synchronization and grouping units, not processes. `sysinit.target` leads into `basic.target`; `multi-user.target` represents a non-graphical multi-user service state, while `graphical.target` builds on it and additionally pulls in graphical-login functionality when configured. This VM actually reported that `graphical.target` was reached, even though no display manager was installed.

Enabled services include NetworkManager, which configures `enp0s1`, and sshd, which provides remote administration. Independent units start in parallel according to their ordering and dependency relationships.

A console login is provided through getty units such as `getty@tty1.service`. `agetty` presents the login prompt and invokes `/bin/login`; PAM performs authentication and session setup. After successful authentication, login changes to the user's identity and starts the configured shell.

The short mental model is:

```text
firmware finds the ESP → EFI bootloader loads kernel and initramfs →
kernel starts early userspace → initramfs discovers and switches to the real root →
systemd activates targets and services → getty/login starts the user session
```

Useful evidence commands include:

```bash
bootctl status
systemd-analyze
systemd-analyze critical-chain
journalctl -b
systemctl list-dependencies
```

## Assessment

**Grade: Pass**

The explanation correctly covered every control handoff and connected the model to evidence from the installed VM.

Corrections recorded during review:

1. Initramfs is not required because the kernel can *never* access the real disk directly. A simple system can boot without one when every required storage/filesystem driver is built into the kernel and no early userspace setup is needed. It is normally used because modular drivers, UUID discovery, encryption, LVM, RAID, and similar setup must happen before the real root can be mounted.
2. The observed system reached `graphical.target`, not `multi-user.target`. With no display manager, `graphical.target` can still become active because it builds on the multi-user state; it does not itself prove that a graphical desktop exists.
3. During the lab, the changed FAT masks were applied with an explicit remount rather than a full unmount/mount cycle.
