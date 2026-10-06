# System Call Analysis Report

This document analyzes the `strace` log outputs provided for the `ls` and `curl` commands, mapping out the behavior of essential Linux system calls.

---

## 1. Executed Commands
Based on the trace logs, two commands were analyzed:
1. **`ls -l`** (Traced in `strace-ls.log`)
2. **`curl -I https://example.com`** (Traced in `strace-curl.log`)

---

## 2. System Call Documentation & Log Lines

### `execve`
* **Description:** Replaces the current process image with a new program image. It initializes the stack, loads the binary into virtual memory space, and jumps execution to the program's main entry point.
* **Trace Line (`strace-ls.log`):**
  ```c
  execve("/usr/bin/ls", ["ls"], 0x7ffe59939cd0 /* 21 vars */) = 0
  ```
* **Trace Line (`strace-curl.log`):**
  ```c
  execve("/usr/bin/curl", ["curl", "-I", "https://example.com"], 0x7ffc0ec37150 /* 21 vars */) = 0
  ```

### `openat`
* **Description:** Opens a file path relative to a directory file descriptor argument. When passed `AT_FDCWD`, it opens the target relative to the current working directory. It returns a non-negative file descriptor (like `3`) on success.
* **Trace Lines (`strace-ls.log`):**
  ```c
  openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
  openat(AT_FDCWD, "/usr/lib/libcap.so.2", O_RDONLY|O_CLOEXEC) = 3
  openat(AT_FDCWD, "/usr/lib/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
  openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
  openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
  ```
* **Trace Lines (`strace-curl.log`):**
  ```c
  openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
  openat(AT_FDCWD, "/usr/lib/libcurl.so.4", O_RDONLY|O_CLOEXEC) = 3
  openat(AT_FDCWD, "/usr/lib/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
  openat(AT_FDCWD, "/usr/lib/libnghttp3.so.9", O_RDONLY|O_CLOEXEC) = 3
  ```

### `mmap`
* **Description:** Maps files or hardware devices into the process's virtual memory address space. In these logs, the dynamic linker uses it to read dynamic configurations and map shared library code segments (`PROT_READ|PROT_EXEC`) and data segments directly into memory.
* **Selected Trace Lines (`strace-ls.log`):**
  ```c
  mmap(NULL, 24099, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fb2cbd4b000
  mmap(NULL, 2239312, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fb2cba00000
  mmap(0x7fb2cba24000, 1552384, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x24000) = 0x7fb2cba24000
  ```
* **Selected Trace Lines (`strace-curl.log`):**
  ```c
  mmap(NULL, 24099, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f036358c000
  mmap(NULL, 1086256, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0363480000
  ```

### `connect`
* **Description:** Initiates a connection on a network socket to a specified destination address.
* **Status:** **Unavailable in the provided log snippet.** The text block for `strace-curl.log` cuts off early during the runtime linking phase (while mapping `libnghttp3.so.9`). Because the network execution block is absent from the input context, no real `connect` system call trace lines can be documented.
