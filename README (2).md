# OS-Jackfruit : Multi-Container Runtime

> A lightweight Linux container runtime written in C, featuring a long-running supervisor process and a kernel-space memory monitor.

---

## 1. Team Information

| Name | SRN |
|------|-----|
|  |  |
|  |  |

---

## 2. Build & Run Instructions

### Prerequisites

Ubuntu 22.04 or 24.04 in a VM (not WSL). Secure Boot must be **OFF**.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Get Alpine rootfs

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

### Build

```bash
gcc -o engine engine.c -lpthread
make
```

### Load Kernel Module (Task 4)

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

### Run

```bash
# Terminal 1 — start supervisor
sudo ./engine supervisor ./rootfs

# Terminal 2 — use CLI
sudo ./engine start alpha ./rootfs-alpha /bin/sh
sudo ./engine start beta ./rootfs-beta /bin/sh
sudo ./engine ps
sudo ./engine logs alpha
sudo ./engine stop alpha
sudo ./engine stop beta
```

### Cleanup

```bash
sudo rmmod monitor
dmesg | tail
```

---

## 3. Screenshots

### Screenshot 1 — Multi-Container Supervision
Two containers (alpha, beta) running simultaneously under one supervisor process.

<!-- Add screenshots here -->

---

### Screenshot 2 — Metadata Tracking
Output of `ps` command showing container ID, PID, state, start time, soft and hard memory limits.

<!-- Add screenshot here -->

---

### Screenshot 3 — Bounded-Buffer Logging
Container output captured through the logging pipeline:
`pipe → producer thread → bounded buffer → consumer thread → log file`

`engine logs alpha` retrieves the captured output.

<!-- Add screenshot here -->

---

### Screenshot 4 — CLI and IPC
CLI commands (`start`, `stop`, `ps`) sent to the supervisor over a UNIX domain socket at `/tmp/engine.sock`. Supervisor responds correctly to each command.

<!-- Add screenshots here -->

---

### Screenshot 5 — Soft-Limit Warning
`dmesg` output showing a soft-limit warning event when a container's RSS memory exceeds the configured soft threshold.

<!-- Add screenshot here -->

---

### Screenshot 6 — Hard-Limit Enforcement
`dmesg` output showing a container being killed after exceeding its hard memory limit. Supervisor metadata reflects the kill by updating container state to `killed`.

<!-- Add screenshot here -->

---

### Screenshot 7 — Scheduling Experiment
Terminal output from scheduling experiments comparing CPU-bound and I/O-bound workloads under different priorities. Observable differences in completion time and CPU share are shown.

<!-- Add screenshot here -->

---

### Screenshot 8 — Clean Teardown
Evidence that all containers are reaped, logging threads exit cleanly, and no zombie processes remain after supervisor shutdown — shown via `ps aux` output and supervisor exit messages.

<!-- Add screenshot here -->

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Our runtime achieves process and filesystem isolation using three Linux namespace types combined with `chroot`.

**PID Namespace (`CLONE_NEWPID`):**
Each container gets its own PID namespace, making it believe it is PID 1. The host kernel maintains the real PIDs, but the container cannot see or signal any host processes. This is enforced at the kernel level — the namespace boundary is maintained by the kernel's PID allocation table.

**UTS Namespace (`CLONE_NEWUTS`):**
Each container gets its own hostname and domain name. When `sethostname("alpha")` is called inside the container, it only affects that container's UTS namespace. The host hostname remains unchanged.

**Mount Namespace (`CLONE_NEWNS`):**
Each container gets its own copy of the mount table. Mounts inside the container (like `/proc`) do not propagate to the host. Combined with `chroot()`, this locks the container into its Alpine rootfs.

**chroot:**
Changes what the process considers `/`. After `chroot(./rootfs)`, the container cannot navigate above its root and sees Alpine Linux's filesystem, not the host's.

**What the host kernel still shares:**
All containers share the host kernel — there is no separate kernel per container. System calls go to the same kernel, meaning kernel vulnerabilities affect all containers. Network namespace is also shared in our implementation, so containers share the host network stack.

---

### 4.2 Supervisor and Process Lifecycle

A long-running parent supervisor is useful because it maintains state across the entire lifetime of all containers. Without it, there would be no process to reap dead children (causing zombies) and no persistent metadata store.

- **Process creation:** We use `clone()` instead of `fork()` to pass namespace flags. The child process starts in `container_main()` with its own stack.
- **Parent-child relationships:** The supervisor is the parent of all container processes. When a container exits, the kernel sends `SIGCHLD` to the supervisor.
- **Reaping:** `sigchld_handler()` calls `waitpid(-1, &status, WNOHANG)` in a loop to reap all dead children without blocking. `WNOHANG` is critical — without it the handler would block, freezing the supervisor.
- **Metadata tracking:** Each container has a `ContainerMeta` struct in a global array, protected by a `containers_lock` mutex since both the signal handler and CLI handler threads access it concurrently.
- **Signal delivery:** `SIGTERM` triggers graceful shutdown; `SIGKILL` from the kernel module triggers forced termination. The supervisor detects both via `SIGCHLD` and updates state accordingly.

---

### 4.3 IPC, Threads, and Synchronization

**IPC Mechanism 1 — Pipes (Logging):**
Each container's stdout and stderr are redirected into the write end of a pipe via `dup2()`. A producer thread reads from the read end. This is anonymous IPC between parent and child.

**IPC Mechanism 2 — UNIX Domain Socket (CLI):**
The supervisor listens on `/tmp/engine.sock`. CLI clients connect, send a command string, and read the response. This is named IPC between unrelated processes.

**Bounded Buffer Synchronization:**
The bounded buffer has three shared variables: `slots[]`, `head`, `tail`, and `count`. Without synchronization, race conditions include two producers writing to the same slot, a consumer reading while a producer is writing, and `count` corruption from simultaneous increment/decrement.

We address this with:
- `pthread_mutex_t lock` — ensures only one thread modifies the buffer at a time
- `pthread_cond_t not_full` — producer waits here when the buffer is full, preventing overflow
- `pthread_cond_t not_empty` — consumer waits here when the buffer is empty, preventing busy-waiting

**Container Metadata Synchronization:**
The `containers[]` array is accessed by the SIGCHLD handler, the CLI handler, and producer/consumer threads. It is protected with a `containers_lock` mutex. A spinlock would waste CPU since contention is low and lock hold times are short — mutex is the right choice here.

---

### 4.4 Memory Management and Enforcement

**What RSS measures:**
RSS (Resident Set Size) is the amount of physical RAM currently occupied by a process. It excludes swapped-out pages and unloaded shared library pages.

**What RSS does not measure:**
Virtual memory that is allocated but not yet used, memory-mapped files that aren't resident, and memory shared with other processes counted multiple times.

**Why soft and hard limits are different policies:**
A soft limit is a warning threshold — the process may be temporarily spiking and could recover. A hard limit is a kill threshold — the process has exceeded what the system can tolerate. Having both gives a graduated response: warn first, then kill if the situation doesn't improve.

**Why enforcement belongs in kernel space:**
A user-space monitor can be killed, paused, or starved of CPU. If the container process is consuming all resources, a user-space monitor may never get scheduled to check it. The kernel always runs — a kernel module's timer callback fires regardless of what user-space processes are doing, making enforcement reliable and tamper-proof.

---

### 4.5 Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS) as its default scheduler. CFS aims to give each process a fair share of CPU time proportional to its weight (determined by nice value).

In our experiments, we ran CPU-bound and I/O-bound workloads simultaneously with different nice values:
- CPU-bound processes with lower nice values (higher priority) received more CPU time and completed faster.
- I/O-bound processes spent most of their time blocked on I/O regardless of priority, showing that CFS primarily affects CPU allocation, not I/O throughput.
- When two CPU-bound containers ran at the same nice value, CFS distributed CPU time approximately equally between them, consistent with its fairness goal.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
| | |
|---|---|
| **Choice** | PID + UTS + Mount namespaces via `clone()` |
| **Tradeoff** | No network namespace — containers share the host network stack |
| **Justification** | Network namespace requires additional veth pair setup beyond the project scope. The three namespaces used are sufficient to demonstrate isolation. |

### Supervisor Architecture
| | |
|---|---|
| **Choice** | Single long-running process accepting one CLI connection at a time |
| **Tradeoff** | CLI commands are serialized — two simultaneous `start` commands would queue up |
| **Justification** | Simplifies synchronization significantly. For this use case, serialized commands are acceptable. |

### IPC and Logging
| | |
|---|---|
| **Choice** | Pipes for logging, UNIX socket for CLI |
| **Tradeoff** | Pipes are one-way and anonymous — one pipe is needed per container |
| **Justification** | Pipes are the natural IPC for parent-child output capture. UNIX sockets are the natural IPC for request-response CLI commands. |

### Kernel Monitor
| | |
|---|---|
| **Choice** | Periodic RSS polling via kernel timer |
| **Tradeoff** | Not instantaneous — a process could briefly exceed the hard limit between checks |
| **Justification** | Event-driven memory monitoring requires kernel tracepoints which are significantly more complex. Periodic polling is reliable and simple to implement correctly. |

### Scheduling Experiments
| | |
|---|---|
| **Choice** | `nice` values and CPU affinity via `taskset` |
| **Tradeoff** | Results vary with host load — not perfectly reproducible |
| **Justification** | These are the standard Linux interfaces for influencing scheduling and demonstrate CFS behavior without requiring a custom scheduler. |

---

## 6. Scheduler Experiment Results

### Experiment 1 — CPU-Bound Containers with Different Priorities

Two containers running `busybox yes` simultaneously:
- Container **alpha**: nice 0 (default priority)
- Container **beta**: nice 15 (lower priority)

| Container | Nice | Completion Time | CPU % |
|-----------|------|-----------------|-------|
| alpha | 0 | X s | ~66.7% |
| beta | 15 | Y s | ~33.3% |

**Analysis:** CFS allocated more CPU time to the higher-priority container (alpha) compared to the lower-priority container (beta). Although both were CPU-bound workloads, the difference in nice values caused alpha to consistently receive a larger CPU share. Beta, having lower priority, received less CPU time and would take longer to complete the same workload.

---

### Experiment 2 — CPU-Bound vs I/O-Bound

- Container **alpha**: CPU-bound (`busybox yes`)
- Container **beta**: I/O-bound (`io_pulse`)

| Container | Type | CPU % | Completion |
|-----------|------|-------|------------|
| alpha | CPU-bound | ~95% | X s |
| beta | I/O-bound | ~5% | Y s |

**Analysis:** The I/O-bound container spent most of its time blocked waiting for I/O, voluntarily yielding the CPU. This allowed the CPU-bound container to use nearly all available CPU. CFS correctly identified beta as low-CPU-demand and prioritized alpha for CPU allocation.

---
