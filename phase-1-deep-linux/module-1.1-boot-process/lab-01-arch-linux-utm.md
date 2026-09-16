# Lab 01 — Manual Arch Linux Installation on UTM

## Objective

Install Arch Linux manually in an x86_64 UTM virtual machine, boot it through UEFI and systemd-boot, establish secure remote administration, and verify the resulting boot chain and service health.

## Environment

- Host platform: Apple Mac running UTM
- UTM mode: **Emulate** (x86_64 guest)
- Machine: Intel ICH9-based PC (2009, x86_64)
- Firmware: UEFI 2.70 (EDK II 1.00)
- vCPU: 2
- RAM: 4 GiB
- Disk: 40 GiB
- Installer: `archlinux-2026.09.01-x86_64.iso`
- Guest hostname: `arch-lab`
- Guest user: `josemanco`

The disk was increased from 20 GiB to 40 GiB before partitioning because 20 GiB was below the planned capacity for later labs.

## Storage layout

| Device | Size | Type | Filesystem | Mount point |
|---|---:|---|---|---|
| `/dev/sda1` | 1 GiB | EFI System | FAT32 | `/boot` |
| `/dev/sda2` | 39 GiB | Linux filesystem | ext4 | `/` |

Identifiers captured during the installation:

- EFI filesystem UUID: `1359-7C9D`
- Root filesystem UUID: `905421a2-6060-4b64-b5f8-c2a0cb0721d8`
- EFI partition PARTUUID: `2877263b-3d73-4bd4-9af6-950f03a0bf81`

The EFI partition is mounted with `fmask=0077,dmask=0077` so systemd-boot's random seed is not exposed to unprivileged users.

## Installation decisions

- GPT partition table
- systemd-boot as the UEFI boot manager
- EFI System Partition mounted at `/boot`
- No swap partition; swap can be added later as a file if a workload requires it
- `en_GB.UTF-8` as the system locale; `es_ES.UTF-8` also generated
- `Europe/Madrid` timezone
- NetworkManager for network configuration
- OpenSSH for remote administration
- Vim as the preferred editor
- Normal administrative account in the `wheel` group; direct routine use of root avoided

## Installed base packages

```text
base
linux
linux-firmware
networkmanager
sudo
nano
man-db
man-pages
git
openssh
qemu-guest-agent
vim
inetutils
```

## Boot configuration

`/boot/loader/loader.conf`:

```text
default arch.conf
timeout 3
console-mode keep
editor no
```

`/boot/loader/entries/arch.conf`:

```text
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=UUID=905421a2-6060-4b64-b5f8-c2a0cb0721d8 rw
```

The first `bootctl install` ran inside the chroot and could not write EFI variables, but it installed the standard removable-media fallback at `/EFI/BOOT/BOOTX64.EFI`. After the installed system booted, `bootctl install` was run again. UEFI then registered:

```text
Title: Linux Boot Manager
ID: 0x0004
Status: active, boot-order
File: /EFI/systemd/systemd-bootx64.efi
```

A final reboot confirmed that the active loader changed from the fallback path to:

```text
/EFI/systemd/systemd-bootx64.efi
```

## Accounts and SSH security

- User `josemanco` has UID 1000 and primary GID 1000.
- The account belongs to the `wheel` group.
- The sudoers configuration was edited through `visudo` and validated with `visudo -c`.
- NetworkManager and sshd are enabled at boot.
- An Ed25519 public key from the Mac was installed with `ssh-copy-id`.
- Key-only login was tested successfully before password authentication was disabled.

`/etc/ssh/sshd_config.d/10-hardening.conf`:

```text
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

The configuration was checked with `sshd -t`, reloaded, and verified from a second Mac terminal before the original SSH session was closed.

## Verification evidence

### Firmware and boot loader

```text
Firmware: UEFI 2.70 (EDK II 1.00)
Firmware Arch: x64
Product: systemd-boot 261.3-1-arch
Current Entry: arch.conf
Loader: /boot//EFI/systemd/systemd-bootx64.efi
```

### Startup timing

```text
Startup finished in 1.735s (kernel) + 7.971s (initrd) + 17.975s (userspace) = 27.682s
graphical.target reached after 17.970s in userspace.
```

The VM is emulating x86_64, so these timings include emulation overhead.

### Failed units

```text
0 loaded units listed.
```

### Boot journal errors

```text
virt/tdx: TDX not supported by the host platform
```

This is benign in the UTM-emulated environment: the emulated host does not expose Intel TDX confidential-computing support.

### Network and remote access

```text
Hostname: arch-lab
Interface: enp0s1
State: UP
IPv4: 192.168.64.4/24
NetworkManager: active
sshd: active
```

SSH from the Mac was verified with:

```text
ssh josemanco@192.168.64.4
[josemanco@arch-lab ~]$
```

## Result

The VM boots from its registered UEFI systemd-boot entry, reaches the installed Arch login prompt, has no failed systemd units, and accepts hardened SSH key authentication from the Mac host.
