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


### the curl complete strace:

The connect system call remains unavailable in your provided trace log.
The new log segment ends precisely at:
c
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 
Use code with caution.
Why it is Still Missing
Your trace is still capturing the early dynamic linking phase of the application. The program is sequentially opening and mapping shared runtime dependencies (such as libssh2.so.1 and libpsl.so.5) into its memory space via openat and mmap before the actual logic of the program runs.
Network sockets and connection handlers like connect(...) are only initialized after all shared libraries are safely loaded into memory and execution transfers to curl's main program logic.
Expected Network Sequence
Had the log continued past the library initializations, the real connect sequence would look resembling this:
c
// 1. A socket descriptor is created for IPv4/TCP
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 5 

// 2. The connection is initiated to the resolved IP address of example.com (e.g., 93.184.215.14) on port 443 (HTTPS)
connect(5, {sa_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("93.184.215.14")}, 16) = -1 EINPROGRESS (Operation now in progress)
Use code with caution.
Because this phase of execution was not captured in your output snippet, no real connect trace lines can be evaluated.

```
execve("/usr/bin/curl", ["curl", "-I", "https://example.com"], 0x7ffc0ec37150 /* 21 vars */) = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=24099, ...}) = 0
mmap(NULL, 24099, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f036358c000
close(3)                                = 0
brk(NULL)                               = 0x562396620000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f036358a000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libcurl.so.4", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=1082984, ...}) = 0
mmap(NULL, 1086256, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0363480000
mmap(0x7f036348d000, 815104, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xd000) = 0x7f036348d000
mmap(0x7f0363554000, 184320, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xd4000) = 0x7f0363554000
mmap(0x7f0363581000, 32768, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x100000) = 0x7f0363581000
mmap(0x7f0363589000, 816, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f0363589000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\300y\2\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=2215072, ...}) = 0
mmap(NULL, 2239312, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0363200000
mmap(0x7f0363224000, 1552384, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x24000) = 0x7f0363224000
mmap(0x7f036339f000, 483328, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x19f000) = 0x7f036339f000
mmap(0x7f0363415000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x214000) = 0x7f0363415000
mmap(0x7f036341b000, 31568, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f036341b000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libnghttp3.so.9", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=160768, ...}) = 0
mmap(NULL, 163200, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0363458000
mmap(0x7f036345b000, 86016, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7f036345b000
mmap(0x7f0363470000, 53248, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x18000) = 0x7f0363470000
mmap(0x7f036347d000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x24000) = 0x7f036347d000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libngtcp2_crypto_ossl.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=47064, ...}) = 0
mmap(NULL, 49248, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f036344b000
mmap(0x7f036344e000, 24576, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7f036344e000
mmap(0x7f0363454000, 8192, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x9000) = 0x7f0363454000
mmap(0x7f0363456000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xa000) = 0x7f0363456000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libngtcp2.so.16", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=374768, ...}) = 0
mmap(NULL, 373120, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f03631a4000
mmap(0x7f03631a8000, 290816, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4000) = 0x7f03631a8000
mmap(0x7f03631ef000, 61440, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4b000) = 0x7f03631ef000
mmap(0x7f03631fe000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x5a000) = 0x7f03631fe000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libnghttp2.so.14", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=165928, ...}) = 0
mmap(NULL, 168224, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f036317a000
mmap(0x7f036317e000, 90112, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4000) = 0x7f036317e000
mmap(0x7f0363194000, 49152, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1a000) = 0x7f0363194000
mmap(0x7f03631a0000, 16384, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x25000) = 0x7f03631a0000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libidn2.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=133064, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0363449000
mmap(NULL, 135184, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0363427000
mmap(0x7f0363429000, 16384, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2000) = 0x7f0363429000
mmap(0x7f036342d000, 106496, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x6000) = 0x7f036342d000
mmap(0x7f0363447000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1f000) = 0x7f0363447000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libssh2.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=317984, ...}) = 0
mmap(NULL, 316016, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f036312c000
mmap(0x7f0363132000, 229376, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x6000) = 0x7f0363132000
mmap(0x7f036316a000, 53248, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3e000) = 0x7f036316a000
mmap(0x7f0363177000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4b000) = 0x7f0363177000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libpsl.so.5", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=75640, ...}) = 0
mmap(NULL, 77840, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0363118000
mmap(0x7f036311a000, 8192, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2000) = 0x7f036311a000
mmap(0x7f036311c000, 57344, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4000) = 0x7f036311c000
mmap(0x7f036312a000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x11000) = 0x7f036312a000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libssl.so.3", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=1011672, ...}) = 0
mmap(NULL, 1009640, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0363021000
mmap(0x7f0363035000, 708608, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x14000) = 0x7f0363035000
mmap(0x7f03630e2000, 163840, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xc1000) = 0x7f03630e2000
mmap(0x7f036310a000, 57344, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xe9000) = 0x7f036310a000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libcrypto.so.3", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=5922312, ...}) = 0
mmap(NULL, 5933312, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362a00000
mmap(0x7f0362a53000, 3850240, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x53000) = 0x7f0362a53000
mmap(0x7f0362dff000, 1204224, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3ff000) = 0x7f0362dff000
mmap(0x7f0362f25000, 528384, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x525000) = 0x7f0362f25000
mmap(0x7f0362fa6000, 10496, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f0362fa6000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libgssapi_krb5.so.2", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=360736, ...}) = 0
mmap(NULL, 359176, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362fc9000
mmap(0x7f0362fd2000, 262144, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x9000) = 0x7f0362fd2000
mmap(0x7f0363012000, 49152, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x49000) = 0x7f0363012000
mmap(0x7f036301e000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x55000) = 0x7f036301e000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libzstd.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=939976, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0363425000
mmap(NULL, 938040, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f036291a000
mmap(0x7f0362926000, 815104, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xc000) = 0x7f0362926000
mmap(0x7f03629ed000, 69632, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xd3000) = 0x7f03629ed000
mmap(0x7f03629fe000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xe4000) = 0x7f03629fe000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libbrotlidec.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=55168, ...}) = 0
mmap(NULL, 57360, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362fba000
mmap(0x7f0362fbb000, 36864, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1000) = 0x7f0362fbb000
mmap(0x7f0362fc4000, 12288, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xa000) = 0x7f0362fc4000
mmap(0x7f0362fc7000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xc000) = 0x7f0362fc7000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libz.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=104464, ...}) = 0
mmap(NULL, 106512, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f03628ff000
mmap(0x7f0362902000, 65536, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7f0362902000
mmap(0x7f0362912000, 24576, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x13000) = 0x7f0362912000
mmap(0x7f0362918000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x18000) = 0x7f0362918000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libunistring.so.5", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=1992752, ...}) = 0
mmap(NULL, 1992968, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362718000
mmap(0x7f0362727000, 278528, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xf000) = 0x7f0362727000
mmap(0x7f036276b000, 1634304, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x53000) = 0x7f036276b000
mmap(0x7f03628fa000, 20480, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1e2000) = 0x7f03628fa000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libbrotlienc.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=780768, ...}) = 0
mmap(NULL, 782888, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362658000
mmap(0x7f0362659000, 446464, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1000) = 0x7f0362659000
mmap(0x7f03626c6000, 327680, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x6e000) = 0x7f03626c6000
mmap(0x7f0362716000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xbd000) = 0x7f0362716000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libkrb5.so.3", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=845896, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0363423000
mmap(NULL, 844632, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362589000
mmap(0x7f036259a000, 458752, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x11000) = 0x7f036259a000
mmap(0x7f036260a000, 253952, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x81000) = 0x7f036260a000
mmap(0x7f0362648000, 65536, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xbf000) = 0x7f0362648000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libk5crypto.so.3", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=182320, ...}) = 0
mmap(NULL, 184360, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f036255b000
mmap(0x7f036255e000, 114688, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7f036255e000
mmap(0x7f036257a000, 49152, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1f000) = 0x7f036257a000
mmap(0x7f0362586000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2a000) = 0x7f0362586000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libcom_err.so.2", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=18376, ...}) = 0
mmap(NULL, 20552, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362fb4000
mmap(0x7f0362fb6000, 4096, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2000) = 0x7f0362fb6000
mmap(0x7f0362fb7000, 4096, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7f0362fb7000
mmap(0x7f0362fb8000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7f0362fb8000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libkrb5support.so.0", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=55544, ...}) = 0
mmap(NULL, 57904, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f036254c000
mmap(0x7f036254f000, 28672, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7f036254f000
mmap(0x7f0362556000, 12288, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xa000) = 0x7f0362556000
mmap(0x7f0362559000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xc000) = 0x7f0362559000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libkeyutils.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=22480, ...}) = 0
mmap(NULL, 24592, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362fad000
mmap(0x7f0362faf000, 8192, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2000) = 0x7f0362faf000
mmap(0x7f0362fb1000, 4096, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4000) = 0x7f0362fb1000
mmap(0x7f0362fb2000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x4000) = 0x7f0362fb2000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libresolv.so.2", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=67904, ...}) = 0
mmap(NULL, 75912, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362539000
mmap(0x7f036253c000, 36864, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7f036253c000
mmap(0x7f0362545000, 12288, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xc000) = 0x7f0362545000
mmap(0x7f0362548000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xf000) = 0x7f0362548000
mmap(0x7f036254a000, 6280, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f036254a000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libbrotlicommon.so.1", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=141264, ...}) = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0362fab000
mmap(NULL, 139280, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f0362516000
mmap(0x7f0362517000, 4096, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1000) = 0x7f0362517000
mmap(0x7f0362518000, 126976, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2000) = 0x7f0362518000
mmap(0x7f0362537000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x21000) = 0x7f0362537000
close(3)                                = 0
newfstatat(AT_FDCWD, "/etc/ld.so.cache", {st_mode=S_IFREG|0644, st_size=24099, ...}, 0) = 0
openat(AT_FDCWD, "/usr/lib/libm.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
fstat(3, {st_mode=S_IFREG|0755, st_size=1268336, ...}) = 0
mmap(NULL, 1270088, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f03623df000
mmap(0x7f03623ef000, 638976, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x10000) = 0x7f03623ef000
mmap(0x7f036248b000, 561152, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0xac000) = 0x7f036248b000
mmap(0x7f0362514000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x134000) = 0x7f0362514000
close(3)                                = 0
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f0362fa9000
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f03623dc000
arch_prctl(ARCH_SET_FS, 0x7f03623dc780) = 0
set_tid_address(0x7f03623dcda8)         = 473
set_robust_list(0x7f03623dca60, 24)     = 0
rseq({cpu_id_start=0, cpu_id=RSEQ_CPU_ID_UNINITIALIZED, rseq_cs=NULL, flags=0, node_id=0, mm_cid=0, slice_ctrl={request=0, granted=0, __reserved=0}, __reserved=0}, 33, 0, 0x53053053) = 0
getrandom("\xf3\xcd\xc1\x25\x1b\xf0\xa9\x1f\x06\x6f\x78\x3a\x1d\xa1\x6d\xe4", 16, GRND_NONBLOCK) = 16
mprotect(0x7f0363415000, 16384, PROT_READ) = 0
mprotect(0x7f0362514000, 4096, PROT_READ) = 0
mprotect(0x7f0362537000, 4096, PROT_READ) = 0
mprotect(0x7f0362548000, 4096, PROT_READ) = 0
mprotect(0x7f0362fb2000, 4096, PROT_READ) = 0
mprotect(0x7f0362559000, 4096, PROT_READ) = 0
mprotect(0x7f0362fb8000, 4096, PROT_READ) = 0
mprotect(0x7f0362586000, 8192, PROT_READ) = 0
mprotect(0x7f0362648000, 57344, PROT_READ) = 0
mprotect(0x7f0362716000, 4096, PROT_READ) = 0
mprotect(0x7f03628fa000, 16384, PROT_READ) = 0
mprotect(0x7f0362918000, 4096, PROT_READ) = 0
mprotect(0x7f0362fc7000, 4096, PROT_READ) = 0
mprotect(0x7f03629fe000, 4096, PROT_READ) = 0
mprotect(0x7f036301e000, 8192, PROT_READ) = 0
mprotect(0x7f0362f25000, 516096, PROT_READ) = 0
mprotect(0x7f036310a000, 40960, PROT_READ) = 0
mprotect(0x7f0363447000, 4096, PROT_READ) = 0
mprotect(0x7f036312a000, 4096, PROT_READ) = 0
mprotect(0x7f0363177000, 8192, PROT_READ) = 0
mprotect(0x7f03631a0000, 12288, PROT_READ) = 0
mprotect(0x7f03631fe000, 4096, PROT_READ) = 0
mprotect(0x7f0363456000, 4096, PROT_READ) = 0
mprotect(0x7f036347d000, 4096, PROT_READ) = 0
mprotect(0x7f0363581000, 28672, PROT_READ) = 0
mprotect(0x562379306000, 20480, PROT_READ) = 0
mprotect(0x7f03635d7000, 8192, PROT_READ) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
getrandom("\xd0\x1d\x1b\x71\xfb\x68\xa0\x20", 8, GRND_NONBLOCK) = 8
fcntl(0, F_GETFD)                       = 0
fcntl(1, F_GETFD)                       = 0
fcntl(2, F_GETFD)                       = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
brk(NULL)                               = 0x562396620000
brk(0x562396641000)                     = 0x562396641000
futex(0x7f0362fa5d1c, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5cb8, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5cb4, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5cb0, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa60f8, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5ca4, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5ca0, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa561c, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa6064, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa60b0, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5c7c, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5880, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5cac, FUTEX_WAKE_PRIVATE, 2147483647) = 0
openat(AT_FDCWD, "/etc/ssl/openssl.cnf", O_RDONLY) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=12411, ...}) = 0
read(3, "#\n# OpenSSL example configuratio"..., 4096) = 4096
read(3, "d attributes must be the same, a"..., 4096) = 4096
read(3, "coding of an extension: beware e"..., 4096) = 4096
brk(0x562396662000)                     = 0x562396662000
read(3, "cert = $insta::certout # insta.c"..., 4096) = 123
read(3, "", 4096)                       = 0
close(3)                                = 0
futex(0x7f0362fa5658, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa56bc, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5c90, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5c8c, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5660, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5c9c, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f03631177dc, FUTEX_WAKE_PRIVATE, 2147483647) = 0
brk(0x562396683000)                     = 0x562396683000
openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=5655840, ...}) = 0
mmap(NULL, 5655840, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f0361e00000
close(3)                                = 0
openat(AT_FDCWD, "/home/josemanco/.curlrc", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/home/josemanco/.config/curlrc", O_RDONLY) = -1 ENOENT (No such file or directory)
geteuid()                               = 1000
newfstatat(AT_FDCWD, "/etc/nsswitch.conf", {st_mode=S_IFREG|0644, st_size=359, ...}, 0) = 0
newfstatat(AT_FDCWD, "/", {st_mode=S_IFDIR|0555, st_size=4096, ...}, 0) = 0
openat(AT_FDCWD, "/etc/nsswitch.conf", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=359, ...}) = 0
read(3, "# Name Service Switch configurat"..., 4096) = 359
read(3, "", 4096)                       = 0
fstat(3, {st_mode=S_IFREG|0644, st_size=359, ...}) = 0
close(3)                                = 0
openat(AT_FDCWD, "/etc/passwd", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=1293, ...}) = 0
lseek(3, 0, SEEK_SET)                   = 0
read(3, "root:x:0:0::/root:/usr/bin/bash\n"..., 4096) = 1293
lseek(3, 1228, SEEK_SET)                = 1228
close(3)                                = 0
openat(AT_FDCWD, "/home/josemanco/.curlrc", O_RDONLY) = -1 ENOENT (No such file or directory)
brk(0x5623966a4000)                     = 0x5623966a4000
ioctl(1, TCGETS2, {c_iflag=ICRNL|IXON|IXANY|IMAXBEL|IUTF8, c_oflag=NL0|CR0|TAB0|BS0|VT0|FF0|OPOST|ONLCR, c_cflag=B9600|CS8|CREAD, c_lflag=ISIG|ICANON|ECHO|ECHOE|IEXTEN|ECHOCTL|ECHOKE|PENDIN, ...}) = 0
eventfd2(0, EFD_CLOEXEC|EFD_NONBLOCK)   = 3
eventfd2(0, EFD_CLOEXEC|EFD_NONBLOCK)   = 4
socket(AF_INET6, SOCK_DGRAM, IPPROTO_IP) = 5
close(5)                                = 0
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=3, events=POLLIN}], 1, 0)     = 0 (Timeout)
read(4, 0x7ffe5b574c80, 64)             = -1 EAGAIN (Resource temporarily unavailable)
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
rt_sigaction(SIGRT_1, {sa_handler=0x7f0363295680, sa_mask=[], sa_flags=SA_RESTORER|SA_ONSTACK|SA_RESTART|SA_SIGINFO, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
rt_sigprocmask(SIG_UNBLOCK, [RTMIN RT_1], NULL, 8) = 0
mmap(NULL, 8392704, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7f03615ff000
madvise(0x7f03615ff000, 4096, MADV_GUARD_INSTALL) = 0
rt_sigprocmask(SIG_BLOCK, ~[], [], 8)   = 0
clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, child_tid=0x7f0361dffce8, parent_tid=0x7f0361dff990, exit_signal=0, stack=0x7f03615ff000, stack_size=0x7fff40, tls=0x7f0361dff6c0} => {parent_tid=[474]}, 88) = 474
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
futex(0x562396686a80, FUTEX_WAKE_PRIVATE, 1) = 1
mmap(NULL, 8392704, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7f0360dfe000
madvise(0x7f0360dfe000, 4096, MADV_GUARD_INSTALL) = 0
futex(0x7f03635d9758, FUTEX_WAIT_PRIVATE, 2, NULL) = 0
futex(0x7f03635d9758, FUTEX_WAKE_PRIVATE, 1) = 0
rt_sigprocmask(SIG_BLOCK, ~[], [], 8)   = 0
clone3({flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, child_tid=0x7f03615fece8, parent_tid=0x7f03615fe990, exit_signal=0, stack=0x7f0360dfe000, stack_size=0x7fff40, tls=0x7f03615fe6c0} => {parent_tid=[475]}, 88) = 475
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
futex(0x562396686a80, FUTEX_WAKE_PRIVATE, 1) = 1
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=4, events=POLLIN}, {fd=3, events=POLLIN}], 2, 15) = 0 (Timeout)
read(4, 0x7ffe5b574c80, 64)             = -1 EAGAIN (Resource temporarily unavailable)
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=4, events=POLLIN}, {fd=3, events=POLLIN}], 2, 1000) = 1 ([{fd=4, revents=POLLIN}])
read(4, "\1\0\0\0\0\0\0\0", 64)         = 8
read(4, "\1\0\0\0\0\0\0\0", 64)         = 8
read(4, 0x7ffe5b574c80, 64)             = -1 EAGAIN (Resource temporarily unavailable)
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
socket(AF_INET6, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_TCP) = 5
setsockopt(5, SOL_TCP, TCP_NODELAY, [1], 4) = 0
setsockopt(5, SOL_SOCKET, SO_KEEPALIVE, [1], 4) = 0
setsockopt(5, SOL_TCP, TCP_KEEPIDLE, [60], 4) = 0
setsockopt(5, SOL_TCP, TCP_KEEPINTVL, [60], 4) = 0
setsockopt(5, SOL_TCP, TCP_KEEPCNT, [9], 4) = 0
connect(5, {sa_family=AF_INET6, sin6_port=htons(443), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "2606:4700:10::6814:179a", &sin6_addr), sin6_scope_id=0}, 28) = -1 EINPROGRESS (Operation now in progress)
getsockname(5, {sa_family=AF_INET6, sin6_port=htons(45070), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "fdb8:bc1d:e667:ec1c:5a6f:254b:314d:714b", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=4, events=POLLIN}, {fd=5, events=POLLOUT}, {fd=3, events=POLLIN}], 3, 196) = 1 ([{fd=5, revents=POLLOUT|POLLERR|POLLHUP}])
read(4, 0x7ffe5b574c80, 64)             = -1 EAGAIN (Resource temporarily unavailable)
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=5, events=POLLPRI|POLLOUT|POLLWRNORM}], 1, 0) = 1 ([{fd=5, revents=POLLOUT|POLLERR|POLLHUP|POLLWRNORM}])
getsockopt(5, SOL_SOCKET, SO_ERROR, [ENETUNREACH], [4]) = 0
getsockname(5, {sa_family=AF_INET6, sin6_port=htons(45070), sin6_flowinfo=htonl(0), inet_pton(AF_INET6, "fdb8:bc1d:e667:ec1c:5a6f:254b:314d:714b", &sin6_addr), sin6_scope_id=0}, [128 => 28]) = 0
openat(AT_FDCWD, "/usr/share/locale/locale.alias", O_RDONLY|O_CLOEXEC) = 6
fstat(6, {st_mode=S_IFREG|0644, st_size=2998, ...}) = 0
read(6, "# Locale name alias data base.\n#"..., 4096) = 2998
read(6, "", 4096)                       = 0
close(6)                                = 0
openat(AT_FDCWD, "/usr/share/locale/en_GB.UTF-8/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale/en_GB.utf8/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale/en_GB/LC_MESSAGES/libc.mo", O_RDONLY) = 6
fstat(6, {st_mode=S_IFREG|0644, st_size=1430, ...}) = 0
mmap(NULL, 1430, PROT_READ, MAP_PRIVATE, 6, 0) = 0x7f0362367000
close(6)                                = 0
openat(AT_FDCWD, "/usr/share/locale/en.UTF-8/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale/en.utf8/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale/en/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
close(5)                                = 0
socket(AF_INET, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_TCP) = 5
setsockopt(5, SOL_TCP, TCP_NODELAY, [1], 4) = 0
setsockopt(5, SOL_SOCKET, SO_KEEPALIVE, [1], 4) = 0
setsockopt(5, SOL_TCP, TCP_KEEPIDLE, [60], 4) = 0
setsockopt(5, SOL_TCP, TCP_KEEPINTVL, [60], 4) = 0
setsockopt(5, SOL_TCP, TCP_KEEPCNT, [9], 4) = 0
connect(5, {sa_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("172.66.147.243")}, 16) = -1 EINPROGRESS (Operation now in progress)
getsockname(5, {sa_family=AF_INET, sin_port=htons(52082), sin_addr=inet_addr("192.168.64.4")}, [128 => 16]) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=4, events=POLLIN}, {fd=5, events=POLLOUT}, {fd=3, events=POLLIN}], 3, 198) = 1 ([{fd=5, revents=POLLOUT}])
read(4, 0x7ffe5b574c80, 64)             = -1 EAGAIN (Resource temporarily unavailable)
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=5, events=POLLPRI|POLLOUT|POLLWRNORM}], 1, 0) = 1 ([{fd=5, revents=POLLOUT|POLLWRNORM}])
getsockopt(5, SOL_SOCKET, SO_ERROR, [0], [4]) = 0
getsockname(5, {sa_family=AF_INET, sin_port=htons(52082), sin_addr=inet_addr("192.168.64.4")}, [128 => 16]) = 0
futex(0x7f0362fa6128, FUTEX_WAKE_PRIVATE, 2147483647) = 0
getpid()                                = 473
getrandom("\x08\x02\x7c\x07\xd1\x6c\xcb\x52\xf6\xfd\xb1\xd6\x47\x3b\x18\xfb\x6a\x91\x63\x41\x6b\xc9\x49\xd2\xc1\x3f\x74\x9d\x9d\x52\x00\x40"..., 48, 0) = 48
futex(0x7f0362fa5ca8, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f03631177d4, FUTEX_WAKE_PRIVATE, 2147483647) = 0
brk(0x5623966c5000)                     = 0x5623966c5000
futex(0x7f0362fa60e0, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa60c0, FUTEX_WAKE_PRIVATE, 2147483647) = 0
getpid()                                = 473
getpid()                                = 473
getpid()                                = 473
getpid()                                = 473
getpid()                                = 473
getpid()                                = 473
getpid()                                = 473
getpid()                                = 473
getpid()                                = 473
getpid()                                = 473
futex(0x7f0362fa5624, FUTEX_WAKE_PRIVATE, 2147483647) = 0
futex(0x7f0362fa5614, FUTEX_WAKE_PRIVATE, 2147483647) = 0
getpid()                                = 473
getpid()                                = 473
getpid()                                = 473
brk(0x5623966e7000)                     = 0x5623966e7000
getpid()                                = 473
getpid()                                = 473
sendto(5, "\26\3\1\6\36\1\0\6\32\3\3\210\37\370-:g\333\352|\16\312\230\337\360\245\337\244\371y^)"..., 1571, MSG_NOSIGNAL, NULL, 0) = 1571
recvfrom(5, 0x5623966cbdf3, 5, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
openat(AT_FDCWD, "/etc/ssl/certs/ca-certificates.crt", O_RDONLY) = 6
fstat(6, {st_mode=S_IFREG|0444, st_size=185311, ...}) = 0
read(6, "# ACCVRAIZ1\n-----BEGIN CERTIFICA"..., 4096) = 4096
brk(0x562396708000)                     = 0x562396708000
read(6, "b8EZ6WdmF/9ARP67Jpi6Yb+tmLSbkyU+"..., 4096) = 4096
read(6, "DjAMBgNVBAcMBU1pbGFuMSMwIQYDVQQK"..., 4096) = 4096
read(6, "MMPQFWAJI/TPlUq9LhONm\nUjANBgkqhk"..., 4096) = 4096
read(6, "V+xafJlrJaSQOoD0IJ2azsct+bJLKZWD"..., 4096) = 4096
read(6, "nxbkRs1CTqjSShGL+9V/6pmTW12xB3uD"..., 4096) = 4096
read(6, "Aw\nHgYDVQQDDBdCdXlwYXNzIENsYXNzI"..., 4096) = 4096
read(6, "s3ElweldPe6hL6P3KjzJIx1qqx2hp/Hz"..., 4096) = 4096
read(6, "uvTwJaP+EmzzV1gsD41eeFPfR60/IvYc"..., 4096) = 4096
brk(0x562396729000)                     = 0x562396729000
read(6, "5I7HX1eBYdpnDBfzwboZL7z8g81sWT\nC"..., 4096) = 4096
read(6, "nRp\nZmljYXRpb24gQXV0aG9yaXR5MSQw"..., 4096) = 4096
read(6, "MjAyMDB2MBAGByqGSM49AgEGBSuB\nBAA"..., 4096) = 4096
read(6, "AzM1oXDTM4MDUw\nOTA5MTAzMlowSDELM"..., 4096) = 4096
read(6, "2J87otTlZCpV6LqYQXY+U3EJ/pure351"..., 4096) = 4096
read(6, "eF22d+mQrvHRAiGfzZ0JFrabA0UWTW98"..., 4096) = 4096
read(6, "GlnaUNlcnQgSW5jMRkwFwYDVQQLExB3\n"..., 4096) = 4096
read(6, "wNjIyMDAw\nMDAwWjBHMQswCQYDVQQGEw"..., 4096) = 4096
brk(0x56239674a000)                     = 0x56239674a000
read(6, "CgYIKoZIzj0EAwMwUDEk\nMCIGA1UECxM"..., 4096) = 4096
read(6, "JBgNVBAYTAkJFMRkwFwYDVQQKExBHbG9"..., 4096) = 4096
read(6, "BAgIQZ3SdjXfYO2rbIvT/WeK/zjAKBgg"..., 4096) = 4096
read(6, "hMCR1Ix\nDzANBgNVBAcTBkF0aGVuczFE"..., 4096) = 4096
read(6, "egAwIBAgIUCBZfikyl7ADJk0DfxMauI7"..., 4096) = 4096
read(6, "Qsw\nCQYDVQQGEwJVUzEpMCcGA1UEChMg"..., 4096) = 4096
read(6, "c3z1i9kKlT/YPyNtGtEqJBnZhbMX73hu"..., 4096) = 4096
read(6, "1Rq41Bab2XD0h7lbwyYIiLXpUq3DDfSJ"..., 4096) = 4096
read(6, "zdLAIx6yo\n0es+nPxdGoMuK8u180SdOq"..., 4096) = 4096
brk(0x56239676b000)                     = 0x56239676b000
read(6, "J8bZT8R\ntOpXaZ+0AOuFJJkk9SGdl6r7"..., 4096) = 4096
read(6, "u4/sh6x/gpqG7D0DmVIB0jWe\nrNrwU8l"..., 4096) = 4096
read(6, "3emLoG+B01vr87ERROR\nFHAGjx+f+Idp"..., 4096) = 4096
read(6, "\nTdFap4Yb5aHglmMeNx+fAIhDluWVfxT"..., 4096) = 4096
read(6, "DPYbCWe+0F+S8T\nkdzt5fxQaxFGRrMcI"..., 4096) = 4096
read(6, "NtVcBdIKQXTbYxE3waWglksejBYS\nd66"..., 4096) = 4096
read(6, "YctyPDQ0RTp5A1NDvZdV3LFOxxHVp3i1"..., 4096) = 4096
read(6, "MSUwIwYDVQQKExxTRUNPTSBUcnVzdCBT"..., 4096) = 4096
brk(0x56239678c000)                     = 0x56239678c000
read(6, "oCggEBANUMOsQq+U7i9b4Zl1+OiFOxHz"..., 4096) = 4096
read(6, "R40cR3p1m0IvVVGb6\ng1XqfMIpiRvpb7"..., 4096) = 4096
read(6, "BL5hSmO68gnFSDA\nS9TMfAxsNAwmmyYx"..., 4096) = 4096
read(6, "nCWlWQtNoURi+VJq/REG6Sb4gumlc7rh"..., 4096) = 4096
read(6, "TE-----\nMIIFgjCCA2qgAwIBAgIPAYvS"..., 4096) = 4096
read(6, "YSBUZWNobm9sb2dp\nZXMsIEluYy4xJDA"..., 4096) = 4096
read(6, "5dZO9LnPWwvB0ZqB9WOwj0PBuwhaGnrh"..., 4096) = 4096
read(6, "IILvS\nsPGP2KxFRv+qZ2C0d35qHzwaUn"..., 4096) = 4096
read(6, "l23OA1xmNjmjAOBgNVHQ8BAf8EBAMCAQ"..., 4096) = 4096
brk(0x5623967ad000)                     = 0x5623967ad000
read(6, "XHmgwo38oZJar55CJD2AhZk\nPuXaTH4M"..., 4096) = 4096
read(6, "MzAw\nMFoXDTQzMDIxODE4MzAwMFowVjE"..., 4096) = 4096
read(6, "FFL2/s1m02I4zhKOQ\nUqqzApVg+QxMaP"..., 4096) = 991
read(6, "", 4096)                       = 0
close(6)                                = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=4, events=POLLIN}, {fd=5, events=POLLIN}, {fd=3, events=POLLIN}], 3, 1000) = 1 ([{fd=5, revents=POLLIN}])
read(4, 0x7ffe5b574c80, 64)             = -1 EAGAIN (Resource temporarily unavailable)
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
recvfrom(5, "\26\3\3\4\272", 5, 0, NULL, NULL) = 5
recvfrom(5, "\2\0\4\266\3\3\27\365d\7\357\276\30\210\233\316\220p\365\334\362\tM\3\317\236\323\250\220J\237\221"..., 1210, 0, NULL, NULL) = 1210
recvfrom(5, "\24\3\3\0\1", 5, 0, NULL, NULL) = 5
recvfrom(5, "\1", 1, 0, NULL, NULL)     = 1
recvfrom(5, "\27\3\3\t\366", 5, 0, NULL, NULL) = 5
recvfrom(5, "Tt\210G\37p\201\372\364^+O.Y\277\2532\214\272\345\224\\o#\305\372\17\26\217\253\f{"..., 2550, 0, NULL, NULL) = 2550
openat(AT_FDCWD, "/etc/localtime", O_RDONLY|O_CLOEXEC) = 6
fstat(6, {st_mode=S_IFREG|0644, st_size=2614, ...}) = 0
fstat(6, {st_mode=S_IFREG|0644, st_size=2614, ...}) = 0
read(6, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\v\0\0\0\v\0\0\0\0"..., 4096) = 2614
lseek(6, -1645, SEEK_CUR)               = 969
read(6, "TZif2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\v\0\0\0\v\0\0\0\0"..., 4096) = 1645
lseek(6, 2613, SEEK_SET)                = 2613
close(6)                                = 0
sendto(5, "\24\3\3\0\1\1\27\3\3\0E\361\255\242$\216V\242H\205\361r\0262k\225Na7x\220\247"..., 80, MSG_NOSIGNAL, NULL, 0) = 80
getsockname(5, {sa_family=AF_INET, sin_port=htons(52082), sin_addr=inet_addr("192.168.64.4")}, [128 => 16]) = 0
sendto(5, "\27\3\3\0{1\321\313=#\307\326\372\331\2463\215\f\366\375\320\260\2573\326\n\313\33\376\30\361O"..., 128, MSG_NOSIGNAL, NULL, 0) = 128
brk(0x5623967de000)                     = 0x5623967de000
recvfrom(5, 0x5623967bd0f3, 65653, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(5, "\27\3\3\0+\244\216G\217\216|Y\340\377\t\346N\327\315\263\261\37\212H\227\300\246\210\313\244P("..., 48, MSG_NOSIGNAL, NULL, 0) = 48
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=4, events=POLLIN}, {fd=5, events=POLLIN}, {fd=3, events=POLLIN}], 3, 1000) = 1 ([{fd=5, revents=POLLIN}])
read(4, 0x7ffe5b574c80, 64)             = -1 EAGAIN (Resource temporarily unavailable)
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
recvfrom(5, "\27\3\3\1\335OT\315F\260\33\\e\204w\307\356\244`\323\326\5\257\23Z%\374k$\321\341|"..., 65653, 0, NULL, NULL) = 575
recvfrom(5, 0x5623967bd0f3, 65653, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(5, "\27\3\3\0\32\227~\304\210V\266Oyku\227D\311\350\243\243\211\364\366\303\23\371\311u\236C", 31, MSG_NOSIGNAL, NULL, 0) = 31
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
poll([{fd=4, events=POLLIN}, {fd=5, events=POLLIN}, {fd=3, events=POLLIN}], 3, 1000) = 1 ([{fd=5, revents=POLLIN}])
read(4, 0x7ffe5b574c80, 64)             = -1 EAGAIN (Resource temporarily unavailable)
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
recvfrom(5, "\27\3\3\0\275\250EU\304Vt\253\270\202f\352\320\342n\33\231\334\220\306'U\263V\216*\7\n"..., 65653, 0, NULL, NULL) = 194
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
write(1, "HTTP/2 200 \r\n", 13)         = 13
write(1, "\33[1mdate\33[0m: Tue, 06 Oct 2026 1"..., 45) = 45
write(1, "\33[1mcontent-type\33[0m: text/html;"..., 48) = 48
write(1, "\33[1mserver\33[0m: cloudflare\r\n", 28) = 28
write(1, "\33[1mlast-modified\33[0m: Fri, 02 O"..., 54) = 54
write(1, "\33[1mallow\33[0m: GET, HEAD\r\n", 26) = 26
write(1, "\33[1maccept-ranges\33[0m: bytes\r\n", 30) = 30
write(1, "\33[1mage\33[0m: 13992\r\n", 20) = 20
write(1, "\33[1mcf-cache-status\33[0m: HIT\r\n", 30) = 30
write(1, "\33[1mcf-ray\33[0m: a4648850688d3ba5"..., 38) = 38
write(1, "\33[1malt-svc\33[0m: h3=\":443\"; ma=8"..., 38) = 38
write(1, "\r\n", 2)                     = 2
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
close(3)                                = 0
close(4)                                = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
rt_sigaction(SIGPIPE, NULL, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, 8) = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
sendto(5, "\27\3\3\0+\316\352\354\301\235\204\26\33>\v\207\321\30\356\302R\253\310V0f\256W0\r\27F"..., 48, MSG_NOSIGNAL, NULL, 0) = 48
recvfrom(5, 0x5623967bd0f3, 65653, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
sendto(5, "\27\3\3\0\23\222$\201\16(\245\265\276\20\232\245\333@N\237\230\227\244}", 24, MSG_NOSIGNAL, NULL, 0) = 24
recvfrom(5, 0x5623967bd0f3, 65653, 0, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable)
brk(0x5623967c5000)                     = 0x5623967c5000
brk(0x5623967c3000)                     = 0x5623967c3000
close(5)                                = 0
rt_sigaction(SIGPIPE, {sa_handler=SIG_IGN, sa_mask=[PIPE], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7f036323e6f0}, NULL, 8) = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```
