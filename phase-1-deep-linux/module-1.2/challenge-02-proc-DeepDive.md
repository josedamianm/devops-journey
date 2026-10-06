# Challenge 2: /proc Deep Dive Report

This report analyzes the runtime state of a controlled process via the `/proc` virtual filesystem on Arch Linux.

---

## 1. Process Setup & Isolation

To ensure sensitive credentials and ambient API tokens were not exposed via the process environment, the target process was launched inside an empty environment using `env -i`:

```bash
env -i APP_ENV=devops_challenge STAGE=sandbox /usr/bin/sleep 3600 &
PID=$! # Process spawned with PID 850
```

---

## 2. /proc Node Inspections & Analysis

### 1. Environment Variables (`/proc/<pid>/environ`)

* **Command:**
  ```bash
  tr '\0' '\n' < /proc/850/environ
  ```
* **Observed Output:**
  ```text
  APP_ENV=devops_challenge
  STAGE=sandbox
  ```
* **Interpretation:**
  In Linux, environment variables are stored in process virtual memory as a contiguous block of key-value pairs delimited by null bytes (`\0`). By using `env -i`, standard inherited parent environment variables (`$PATH`, `$USER`, `$SSH_AUTH_SOCK`, etc.) were stripped entirely. Only the explicitly passed dummy configuration keys are present, proving that ambient secrets are not leaked through `/proc/850/environ`.

---

### 2. File Descriptors (`/proc/<pid>/fd/`)

* **Command:**
  ```bash
  ls -l /proc/850/fd
  ```
* **Observed Output:**
  ```text
  total 0
  lrwx------ 1 josemanco josemanco 64 Oct  6 14:43 0 -> /dev/pts/1
  lrwx------ 1 josemanco josemanco 64 Oct  6 14:43 1 -> /dev/pts/1
  lrwx------ 1 josemanco josemanco 64 Oct  6 14:43 2 -> /dev/pts/1
  ```
* **Interpretation & Correction:**
  - **Symlink Metadata vs. Descriptor Access Mode:** The mode bits `lrwx------` shown in `ls -l` represent the file permissions of the virtual symlink entries within procfs itself (restricted so only the owning user can inspect or traverse the descriptors). They **do not** reflect whether the underlying file descriptor was opened read-only (`O_RDONLY`), write-only (`O_WRONLY`), or read-write (`O_RDWR`).
  - **Determining True Access Mode:** The actual access mode and status flags for a descriptor are stored in `/proc/<pid>/fdinfo/<fd>`. The `flags:` field in `fdinfo` encodes the octal open mode (for example, flag `02` indicates `O_RDWR`, while `00` denotes `O_RDONLY`).
  - **Streams:** Descriptors `0` (stdin), `1` (stdout), and `2` (stderr) all point directly to `/dev/pts/1`, the interactive pseudo-terminal slave allocated to the controlling shell session.

---

### 3. Virtual Memory Mappings (`/proc/<pid>/maps`)

* **Command:**
  ```bash
  head -n 20 /proc/850/maps
  ```
* **Observed Output:**
  ```text
  5583d41ee000-5583d41f0000 r--p 00000000 08:02 1459089                    /usr/bin/sleep
  5583d41f0000-5583d41f6000 r-xp 0002000 08:02 1459089                    /usr/bin/sleep
  5583d41f6000-5583d41f8000 r--p 00008000 08:02 1459089                    /usr/bin/sleep
  5583d41f8000-5583d41f9000 r--p 0000a000 08:02 1459089                    /usr/bin/sleep
  5583d41f9000-5583d41fa000 rw-p 0000b000 08:02 1459089                    /usr/bin/sleep
  5583fc390000-5583fc3b1000 rw-p 00000000 00:00 0                          [heap]
  7f808ce00000-7f808ce24000 r--p 00000000 08:02 1444868                    /usr/lib/libc.so.6
  7f808ce24000-7f808cf9f000 r-xp 00024000 08:02 1444868                    /usr/lib/libc.so.6
  7f808cf9f000-7f808d015000 r--p 0019f000 08:02 1444868                    /usr/lib/libc.so.6
  7f808d015000-7f808d019000 r--p 00214000 08:02 1444868                    /usr/lib/libc.so.6
  7f808d019000-7f808d01b000 rw-p 00218000 08:02 1444868                    /usr/lib/libc.so.6
  7f808d01b000-7f808d023000 rw-p 00000000 00:00 0 
  7f808d10a000-7f808d10f000 rw-p 00000000 00:00 0 
  7f808d10f000-7f808d115000 r--p 00000000 08:02 1311746                    /etc/ld.so.cache
  7f808d115000-7f808d119000 r--p 00000000 00:00 0                          [vvar]
  7f808d119000-7f808d11b000 r--p 00000000 00:00 0                          [vvar_vclock]
  7f808d11b000-7f808d11d000 r-xp 00000000 00:00 0                          [vdso]
  7f808d11d000-7f808d11e000 r--p 00000000 08:02 1444859                    /usr/lib/ld-linux-x86-64.so.2
  7f808d11e000-7f808d14b000 r-xp 00001000 08:02 1444859                    /usr/lib/ld-linux-x86-64.so.2
  7f808d14b000-7f808d15a000 r--p 0002e000 08:02 1444859                    /usr/lib/ld-linux-x86-64.so.2
  ```
* **Interpretation:**
  Each record specifies an address range, page permissions (`r`ead, `w`rite, e`x`ecute, `p`rivate copy-on-write), file offset, major:minor device (`08:02`), filesystem inode, and mapped file pathname.
  - **Binary Segments:** `/usr/bin/sleep` code is mapped with executable rights (`r-xp`), read-only constant data with `r--p`, and initialized data with `rw-p`.
  - **Dynamic Heap:** The runtime `[heap]` segment is mapped `rw-p` starting at address `5583fc390000`.
  - **Shared Libraries:** Shared objects (`libc.so.6`, `ld-linux-x86-64.so.2`) adhere to $W \oplus X$ (Write XOR Execute) memory protections.
  - **Kernel Acceleration Pages:** `[vdso]` (virtual dynamic shared object) and `[vvar]` permit userspace to obtain system time and clock values directly without making expensive kernel mode transitions.

---

### 4. Control Groups (`/proc/<pid>/cgroup`)

* **Command:**
  ```bash
  cat /proc/850/cgroup
  ```
* **Observed Output:**
  ```text
  0::/user.slice/user-1000.slice/session-6.scope
  ```
* **Interpretation:**
  - The leading `0::` indicates the system is running under the **cgroups v2** unified hierarchy (single hierarchy with unified controller tree).
  - The path shows process 850 is grouped under `session-6.scope`, which is nested inside `user-1000.slice` (systemd's per-user slice for UID 1000) under `user.slice`.
  - All resource quotas, memory constraints, and CPU limits assigned by systemd to login session 6 apply to this process.

---

### 5. Namespaces (`/proc/<pid>/ns/`) & Host PID 1 Verification

To verify whether PID 850 resides in the host's initial namespaces, the namespace inodes of PID 850 were compared against PID 1 (`systemd`), which always runs in the root/initial namespaces.

* **Commands:**
  ```bash
  ls -l /proc/850/ns/
  sudo ls -l /proc/1/ns/
  ```

* **Observed Output (PID 850):**
  ```text
  total 0
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 cgroup -> 'cgroup:[4026531835]'
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 ipc -> 'ipc:[4026531839]'
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 mnt -> 'mnt:[4026531832]'
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 net -> 'net:[4026531833]'
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 pid -> 'pid:[4026531836]'
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 pid_for_children -> 'pid:[4026531836]'
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 time -> 'time:[4026531834]'
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 time_for_children -> 'time:[4026531834]'
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 user -> 'user:[4026531837]'
  lrwxrwxrwx 1 josemanco josemanco 0 Oct  6 14:44 uts -> 'uts:[4026531838]'
  ```

* **Observed Output (PID 1 - Init/Host Root):**
  ```text
  total 0
  lrwxrwxrwx 1 root root 0 Oct  6 14:56 cgroup -> 'cgroup:[4026531835]'
  lrwxrwxrwx 1 root root 0 Oct  6 14:56 ipc -> 'ipc:[4026531839]'
  lrwxrwxrwx 1 root root 0 Oct  6 14:56 mnt -> 'mnt:[4026531832]'
  lrwxrwxrwx 1 root root 0 Oct  6 14:56 net -> 'net:[4026531833]'
  lrwxrwxrwx 1 root root 0 Oct  6 13:59 pid -> 'pid:[4026531836]'
  lrwxrwxrwx 1 root root 0 Oct  6 14:56 pid_for_children -> 'pid:[4026531836]'
  lrwxrwxrwx 1 root root 0 Oct  6 14:56 time -> 'time:[4026531834]'
  lrwxrwxrwx 1 root root 0 Oct  6 14:56 time_for_children -> 'time:[4026531834]'
  lrwxrwxrwx 1 root root 0 Oct  6 13:59 user -> 'user:[4026531837]'
  lrwxrwxrwx 1 root root 0 Oct  6 14:56 uts -> 'uts:[4026531838]'
  ```

* **Side-by-Side Namespace Comparison Table:**

  | Namespace | PID 850 Inode Identifier | PID 1 Inode Identifier | Status |
  | :--- | :--- | :--- | :--- |
  | **cgroup** | `cgroup:[4026531835]` | `cgroup:[4026531835]` | **Identical (Host root)** |
  | **ipc** | `ipc:[4026531839]` | `ipc:[4026531839]` | **Identical (Host root)** |
  | **mnt** | `mnt:[4026531832]` | `mnt:[4026531832]` | **Identical (Host root)** |
  | **net** | `net:[4026531833]` | `net:[4026531833]` | **Identical (Host root)** |
  | **pid** | `pid:[4026531836]` | `pid:[4026531836]` | **Identical (Host root)** |
  | **time** | `time:[4026531834]` | `time:[4026531834]` | **Identical (Host root)** |
  | **user** | `user:[4026531837]` | `user:[4026531837]` | **Identical (Host root)** |
  | **uts** | `uts:[4026531838]` | `uts:[4026531838]` | **Identical (Host root)** |

* **Interpretation & Proof:**
  In Linux, every namespace instance is assigned a unique inode number on the internal `nsfs` mount. Because every single namespace inode of PID 850 matches the inode numbers of PID 1 exactly, this conclusively proves that PID 850 was executed directly on the host without container isolation, network virtualization, or mount unsharing.
