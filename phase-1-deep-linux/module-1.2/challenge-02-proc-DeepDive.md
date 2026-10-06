# Challenge 2: /proc Deep Dive Report

This report analyzes the runtime state of a controlled process via the `/proc` virtual filesystem on Arch Linux.

---

## 1. Process Setup & Isolation

To ensure sensitive credentials and API tokens were not exposed via the process environment, the target process was launched inside an empty environment using `env -i`:

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
  In Linux, environment variables are stored in process memory as a contiguous null-byte (`\0`) delimited block. By using `env -i`, inherited parent variables (`$PATH`, `$USER`, `$SSH_AUTH_SOCK`, etc.) were stripped. Only the explicit test keys are visible, confirming that no ambient secrets leak through `/proc/850/environ`.

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
* **Interpretation:**
  The `fd/` directory contains symbolic links to files currently held open by the process. 
  - Standard input (`0`), standard output (`1`), and standard error (`2`) all point directly to `/dev/pts/1` (the interactive pseudo-terminal slave device of the launching shell session).
  - The permissions (`lrwx------`) demonstrate read-write access to the terminal session. The process holds no open file handles, sockets, or pipes beyond standard streams.

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
  Each line defines an allocated virtual address range, permission flags (`r`ead, `w`rite, e`x`ecute, `p`rivate), file offset, device major:minor (`08:02`), inode number, and file path.
  - **Executable & Data Segments:** `/usr/bin/sleep` code is mapped executable (`r-xp`), read-only data is mapped `r--p`, and initialized globals are `rw-p`.
  - **Heap:** The dynamically allocated memory segment `[heap]` starts at address `5583fc390000`.
  - **Libraries:** Shared runtime dependencies (`libc.so.6`, `ld-linux-x86-64.so.2`) and `/etc/ld.so.cache` maintain strict W^X separation.
  - **Kernel Special Mappings:** `[vdso]` and `[vvar]` permit high-frequency clock and timing operations directly in user space without triggering kernel context switch traps.

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
  - The leading `0::` indicates that the host system is running under the **cgroups v2** unified hierarchy.
  - The process is scoped within `session-6.scope`, which is nested inside `user-1000.slice` under the general `user.slice`.
  - Any resource quotas (CPU limits, memory caps, I/O throttling) set by `systemd` on your interactive user session cascade directly to this process.

---

### 5. Namespaces (`/proc/<pid>/ns/`)

* **Command:**
  ```bash
  ls -l /proc/850/ns
  ```
* **Observed Output:**
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
* **Interpretation:**
  Each symlink targets a kernel namespace identified by type and a unique inode identifier (e.g., `net:[4026531833]`).
  - Because PID 850 was executed directly from the host user shell rather than an isolated container runtime (like Docker or Podman), all inode numbers match the initial host root namespaces (`init`).
  - When container engines create isolated network or mount environments, they allocate new namespace inodes and attach child processes accordingly.
