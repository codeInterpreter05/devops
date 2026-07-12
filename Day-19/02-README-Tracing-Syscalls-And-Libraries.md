# Day 19 — Performance & Debugging: Tracing Syscalls and Library Calls

**Phase:** 0 – Foundation | **Week:** W3 | **Domain:** Linux | **Flag:** ⚡ Interview-critical

## Brief

Counters like `free` and `vmstat` tell you *that* something is spending time on I/O or CPU; they don't tell you *which specific call*, on *which specific file or socket*, is responsible. `strace` and `ltrace` close that gap by intercepting the boundary between a program and the kernel (or a program and its shared libraries) and showing you exactly what's being asked for, how often, and how long each call took. This is the tool that turns "the script feels slow" into "the script calls `read()` 40,000 times on a 4KB buffer against a file it opened without buffering" — a diagnosis you can actually act on. It's also directly the mechanism behind today's hands-on lab: profiling a slow Python script with `strace` to find and fix an I/O bottleneck.

## `strace` — tracing syscalls

Every time a program does something that requires the kernel — open a file, read/write, allocate memory via `brk`/`mmap`, make a network connection, fork a process — it issues a **system call**. `strace` uses `ptrace(2)` to attach to a process and log every syscall it makes: the call name, its arguments, its return value, and (with the right flag) its duration.

### Basic usage

```bash
strace ls                      # trace a fresh command from the start
strace -p 4213                 # attach to an already-running process by PID
strace -o out.txt ls           # write trace output to a file instead of stderr
```

Raw output looks like this:

```
execve("/bin/ls", ["ls"], 0x7ffc... /* 22 vars */) = 0
brk(NULL)                              = 0x55f2a1b3e000
access("/etc/ld.so.preload", R_OK)     = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=87652}, ...) = 0
mmap(NULL, 87652, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f8a...
close(3)                                = 0
openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
getdents64(3, 0x55f2..., 32768)        = 152
write(1, "file1.txt  file2.txt\n", 21)  = 21
```

Reading this: each line is `syscall(args) = return_value`. `-1 ENOENT` is a failed call with the errno name spelled out — no need to look up numeric errno codes. `openat` returning `3` is a file descriptor; subsequent calls referencing `3` are operating on that same open file — following the fd number across lines is how you tie a sequence of calls back to one specific file.

### The flags that matter

```bash
strace -c -p 4213               # summary mode: aggregate counts/time per syscall, not a line-by-line trace
strace -f -p 4213               # follow forks/threads (-ff writes each to a separate file: trace.<pid>)
strace -e trace=open,read,write -p 4213   # only trace specific syscalls (much less noise)
strace -e trace=network -p 4213           # trace only network-related syscalls (connect, send, recv, ...)
strace -T -p 4213               # show time spent inside each syscall
strace -tt -p 4213              # show wall-clock timestamp (with microseconds) before each line
strace -s 200 -p 4213           # print up to 200 chars of string args (default truncates at 32)
```

`-c` is the one you reach for first on an unfamiliar slow process — it answers "which syscall is eating the time" in one shot:

```bash
strace -c -o /dev/null python3 slow_script.py
```

```
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 92.31    0.184220           4     46123           read
  4.02    0.008010           8      1002           write
  2.10    0.004190          41       102           openat
  1.02    0.002031          20       102           close
  0.55    0.001098           1       780           fstat
------ ----------- ----------- --------- --------- ----------------
100.00    0.199549                 48109         5 total
```

This single table is the whole diagnosis for the day's hands-on lab: 46,123 calls to `read()` immediately tells you the script is issuing an enormous number of tiny reads instead of a small number of large, buffered ones — a classic unbuffered-I/O bottleneck. From here you'd drop `-c` and run a full trace (or `-e trace=read`) to see the actual buffer sizes being requested (`read(3, "x", 1) = 1` repeated thousands of times is the smoking gun for byte-at-a-time I/O).

### Finding I/O bottlenecks specifically

```bash
strace -f -T -e trace=file,read,write,network -p 4213
```

Combine `-T` (per-call duration) with a narrowed `-e trace=` filter and watch for: calls that individually take milliseconds instead of microseconds (`<0.004512>` at the end of a line — strace appends this with `-T`), the same file descriptor being read/written thousands of times in small chunks, or repeated `stat`/`access` calls against the same path (often a sign of an application doing filesystem existence checks in a hot loop instead of caching the result). `-e trace=network` narrows to `socket`, `connect`, `sendto`, `recvfrom` etc. when you suspect the bottleneck is a slow downstream service rather than local disk — a `connect()` call taking 3 seconds tells you immediately the problem isn't your code, it's what your code is waiting on.

### Overhead — the caveat you must know

`ptrace`-based tracing stops the target process at every single syscall entry and exit, which can slow a syscall-heavy program down by **2x–20x** or more. This matters two ways: first, don't panic if a traced program appears far slower than in normal operation — that's `strace` overhead, not a new finding. Second, and more subtly, this overhead can **mask or shift timing-sensitive bugs** — a race condition or a timeout-dependent failure that reproduces normally may disappear (or a new one may appear) once `strace` slows every syscall down, because the relative timing between threads/processes has changed. If you need low-overhead tracing for a timing-sensitive production issue, look toward `perf trace` or eBPF-based tools (`bpftrace`, part of the same performance-engineering toolkit) which impose far less overhead than `ptrace`.

## `ltrace` — tracing library calls

`ltrace` intercepts calls into shared libraries (`libc`, `libssl`, any dynamically linked `.so`) rather than kernel syscalls — think `malloc()`, `strcpy()`, `printf()`, or an application-specific library function, instead of `read()`/`write()`.

```bash
ltrace ./myapp
ltrace -c ./myapp          # summary mode, same idea as strace -c
ltrace -p 4213              # attach to a running process
ltrace -e malloc+free ./myapp   # trace only specific library calls
```

Sample output:

```
malloc(256)                                     = 0x55d2a0f1c260
strlen("hello world")                           = 11
strcpy(0x55d2a0f1c260, "hello world")            = 0x55d2a0f1c260
free(0x55d2a0f1c260)                            = <void>
```

### `strace` vs `ltrace` — when to use which

- **`strace`** shows the boundary between your program and the *kernel* — always meaningful, because syscalls are the only way any program interacts with files, network, memory mapping, or other processes, regardless of language or how it was compiled.
- **`ltrace`** shows the boundary between your program and *dynamically linked libraries* — useful when you suspect the bottleneck or bug is inside library-level logic (excessive string operations, a hot `malloc`/`free` churn suggesting a memory-allocation bottleneck, an unexpectedly-called library function) rather than in kernel-level I/O.

Use `ltrace` when a program is CPU-bound in userspace (so `strace` shows little — few syscalls, all time between them) and you need to see *which library function* is consuming the gap. Use `strace` for anything that smells like I/O, networking, or process/signal behavior.

### `ltrace`'s real limitations

- **Overhead is often higher than `strace`'s**, because library calls are far more frequent than syscalls in most programs (a single syscall might be preceded by dozens of library-internal calls) — treat `ltrace` timing data as even more approximate than `strace`'s.
- **It cannot trace statically linked binaries** (or calls resolved at compile-time rather than through the dynamic linker/PLT). `ltrace` works by intercepting the dynamic symbol resolution mechanism (PLT/GOT) — a Go binary, a `musl`-static binary, or any executable built with `-static` has no such indirection for `ltrace` to hook into, so it reports nothing useful (often just the process's syscalls-via-libc calls it can still see, or nothing at all). This is an increasingly common trap in containerized environments, where minimal/distroless images frequently ship statically linked binaries specifically to avoid needing a libc in the image.
- It is **not a substitute for `strace`** on such binaries — if `ltrace ./binary` returns nothing meaningful, the correct move is to fall back to `strace` (which always works, since syscalls happen regardless of linking) or a debugger/profiler that understands the binary's actual symbol table (`perf record` with debug symbols, or `gdb`).
- Development on `ltrace` itself has been largely dormant for years compared to `strace` — expect rougher edges (crashes on complex binaries, incomplete argument decoding for less common library calls).

## Points to Remember

- `strace` traces kernel syscalls; `ltrace` traces dynamic library calls — pick based on whether the suspected bottleneck is I/O/kernel-level or userspace/library-level.
- `strace -c` (summary mode) is the fastest way to find *which* syscall dominates time/count before drilling into a full trace — this is the direct tool for today's hands-on lab.
- `-f` follows forks/threads (essential for multi-process/multi-threaded targets); `-e trace=` filters noise; `-T` shows per-call duration; `-tt` shows wall-clock timestamps.
- A `read()`/`write()` call count in the tens of thousands against a small file is the signature of unbuffered, byte-at-a-time I/O — the fix is almost always adding a buffer (`open(..., buffering=...)` in Python, `BufferedReader`/`BufferedWriter`, or batching writes).
- `strace`/`ltrace` overhead can be large enough (2x–20x) to mask or alter timing-sensitive bugs — don't trust absolute timing under trace for anything race-condition-adjacent; use it for call patterns and relative comparisons instead.
- `ltrace` cannot see into statically linked binaries — it hooks the dynamic linker's symbol resolution, which doesn't exist for static binaries. `strace` still works on them because syscalls aren't a linking concern.

## Common Mistakes

- Reaching for `ltrace` on a Go binary, a `musl`-static Alpine binary, or anything built with `-static`, getting empty/useless output, and concluding "there's nothing to trace" instead of recognizing this as `ltrace`'s static-linking limitation and switching to `strace`.
- Running a full (non `-c`, non-filtered) `strace` on a busy production process and being surprised by a huge performance hit — always start with `-c` or a narrow `-e trace=` filter, especially outside a sandboxed reproduction.
- Treating `strace -T` timings as authoritative absolute latencies for a system under normal load — the tracing overhead itself inflates every call's measured duration; use it to compare *relative* time between calls, not as production-representative numbers.
- Forgetting `-f` when tracing a process that forks (e.g., a shell script, a server that forks workers) and wondering why the trace looks incomplete — without `-f`, only the top-level process is traced, not its children.
- Trying to `strace`/`ltrace` a process you don't have permission to attach to and getting a silent or permission-denied failure without realizing `ptrace` attachment is restricted by `ptrace_scope` (see `/proc/sys/kernel/yama/ptrace_scope`) or requires the same UID / `CAP_SYS_PTRACE`.
- Confusing `ltrace`'s library-call view with a full performance profile — it shows *that* `malloc` was called 50,000 times, not *why* (that needs `perf record` with a call graph, or reading the calling code).
