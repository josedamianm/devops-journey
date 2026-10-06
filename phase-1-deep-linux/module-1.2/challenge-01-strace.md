# System Call Analysis Report

This document analyzes the `strace` log outputs for the `ls` and `curl` commands, mapping the behavior of key Linux system calls during program initialization, dynamic library resolution, and network execution.

---

## 1. Executed Commands

Based on the trace logs, the commands were captured using the following invocations:

1. **`ls -l`** (Traced in `strace-ls.log`):
   ```bash
   strace -o strace-ls.log ls -l
   ```
2. **`curl -I https://example.com`** (Traced in `strace-curl.log`):
   ```bash
   strace -f -o strace-curl.log curl -I https://example.com
   ```
   *(Note: The `-f` flag tracks child threads created via `clone3` during asynchronous name resolution and connection workers).*

---

## 2. System Call Documentation & Log Lines

### `execve`
* **Description:** Replaces the current process address space with a new program binary. It sets up the execution stack, loads ELF binaries into memory, and branches execution to the program's dynamic linker or entry point.
* **Trace Line (`strace-ls.log`):**
  ```c
  execve("/usr/bin/ls", ["ls"], 0x7ffe59939cd0 /* 21 vars */) = 0
  ```
* **Trace Line (`strace-curl.log`):**
  ```c
  execve("/usr/bin/curl", ["curl", "-I", "https://example.com"], 0x7ffc0ec37150 /* 21 vars */) = 0
  ```

### `openat`
* **Description:** Opens a filesystem path relative to the directory file descriptor passed in the first parameter. When passed `AT_FDCWD`, paths are resolved relative to the current working directory. Returns an allocated non-negative file descriptor on success.
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
  openat(AT_FDCWD, "/etc/ssl/certs/ca-certificates.crt", O_RDONLY) = 6
  ```

### `mmap`
* **Description:** Creates memory mappings within the calling process's virtual address space. The dynamic linker uses `mmap` to load read-only code segments (`PROT_READ|PROT_EXEC`) and read/write data pages (`PROT_READ|PROT_WRITE`) from dynamic ELF libraries.
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
  mmap(0x7f036348d000, 815104, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xd000) = 0x7f036348d000
  ```

### `connect`
* **Description:** Initiates a connection on a network socket descriptor to an endpoint address specified via a `sockaddr` structure.

#### IPv6 to IPv4 Sequence Explanation
The trace shows `curl` executing Happy Eyeballs fallback behavior:

1. **IPv6 Attempt:** `curl` creates an IPv6 stream socket (`socket(AF_INET6, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, ...)` on file descriptor `5`) and issues an asynchronous non-blocking connection request via `connect()` targeting Cloudflare's IPv6 address `2606:4700:10::6814:179a` on port 443. This immediately returns `-1 EINPROGRESS`.
2. **Network Unreachable:** The socket is monitored with `poll()`. When polling returns `POLLERR|POLLHUP`, `curl` inspects the socket error using `getsockopt(5, SOL_SOCKET, SO_ERROR, ...)`, which yields `ENETUNREACH` (Network is unreachable).
3. **Socket Teardown:** The failed IPv6 socket descriptor `5` is discarded with `close(5)`.
4. **IPv4 Fallback:** `curl` immediately allocates a standard IPv4 socket (`socket(AF_INET, ...)` re-using fd `5`) and calls `connect()` targeting IPv4 address `172.66.147.243` on port 443. 
5. **Successful Handshake:** Subsequent `poll()` calls verify `POLLOUT` readiness and `getsockopt` confirms `SO_ERROR` is `0`, completing the TCP connection and allowing TLS negotiation (`sendto`/`recvfrom`) to proceed over IPv4.

* **Trace Lines (`strace-curl.log`):**
  ```c
  // 1. Initial IPv6 attempt
  socket(AF_INET6, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_TCP) = 5
  connect(5, {sa_family=AF_INET6, sin6_port=htons(443), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "2606:4700:10::6814:179a", &sin6_addr), sin6_scope_id=0}, 28) = -1 EINPROGRESS (Operation now in progress)
  poll([{fd=4, events=POLLIN}, {fd=5, events=POLLOUT}, {fd=3, events=POLLIN}], 3, 196) = 1 ([{fd=5, revents=POLLOUT|POLLERR|POLLHUP}])
  getsockopt(5, SOL_SOCKET, SO_ERROR, [ENETUNREACH], [4]) = 0
  close(5) = 0

  // 2. IPv4 fallback
  socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_TCP) = 5
  connect(5, {sa_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("172.66.147.243")}, 16) = -1 EINPROGRESS (Operation now in progress)
  poll([{fd=4, events=POLLIN}, {fd=5, events=POLLOUT}, {fd=3, events=POLLIN}], 3, 198) = 1 ([{fd=5, revents=POLLOUT}])
  getsockopt(5, SOL_SOCKET, SO_ERROR, [0], [4]) = 0
  ```
