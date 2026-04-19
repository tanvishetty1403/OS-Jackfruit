# Multi-Container Runtime — OS Project (Jackfruit)

## 1. Team Information

| Name | SRN |
|------|-----|
| Tanvi. B. Shetty | PES1UG24AM302 |
| Maansi. S     | PES1UG24AM302     |


---

## 2. Build, Load, and Run Instructions

### Prerequisites

Ubuntu 22.04 or 24.04 VM with Secure Boot **OFF**. WSL will not work.

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build Everything

```bash
make
```

This builds:
- `engine` — the user-space runtime binary
- `monitor.ko` — the kernel module
- `cpu_hog`, `io_pulse`, `memory_hog` — workload test programs

### Prepare the Root Filesystem

```bash
mkdir rootfs
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs
```

### Copy Workloads Into rootfs (needed to run them inside containers)

```bash
cp cpu_hog  ./rootfs/
cp io_pulse ./rootfs/
cp memory_hog ./rootfs/
```

### Full Demo Run Sequence

```bash
# 1. Load kernel module
sudo insmod monitor.ko

# 2. Verify device exists
ls -l /dev/container_monitor

# 3. Start supervisor in one terminal
sudo ./engine supervisor ./rootfs

# 4. In a second terminal — start containers
sudo ./engine start alpha ./rootfs "/cpu_hog"
sudo ./engine start beta  ./rootfs "/io_pulse"

# 5. List containers
sudo ./engine ps

# 6. View logs
sudo ./engine logs alpha

# 7. Start a container with memory limits (triggers soft/hard limit demo)
sudo ./engine start memtest ./rootfs "/memory_hog" --soft-mib 10 --hard-mib 20

# 8. Watch kernel messages
dmesg | tail -20

# 9. Scheduling experiment — different nice values
sudo ./engine start cpu1 ./rootfs "/cpu_hog" --nice 0
sudo ./engine start cpu2 ./rootfs "/cpu_hog" --nice 19
time sudo ./engine run cpu1r ./rootfs "/cpu_hog"
time sudo ./engine run cpu2r ./rootfs "/cpu_hog" --nice 19

# 10. Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta

# 11. Stop supervisor (Ctrl+C in its terminal, or):
sudo kill $(pgrep -f "engine supervisor")

# 12. Unload module
sudo rmmod monitor

# 13. Verify no zombies
ps aux | grep defunct
```

---

## 3. Demo with Screenshots

> **Replace the placeholders below with actual annotated screenshots before submission.**

### Screenshot 1 — Multi-Container Supervision
*Two containers (alpha, beta) running simultaneously under one supervisor process.*

`[INSERT SCREENSHOT: sudo ./engine start alpha ... && sudo ./engine start beta ... && ps aux | grep engine]`
<img width="1031" height="42" alt="image" src="https://github.com/user-attachments/assets/5a0df8d5-3dc5-48c7-b55c-c6bf853c8ef9" />
<img width="1010" height="48" alt="image" src="https://github.com/user-attachments/assets/8f4f3da2-21a7-4031-b549-e9a8f695e04e" />
<img width="904" height="70" alt="image" src="https://github.com/user-attachments/assets/0a299ed7-b530-4078-878a-2d1547954aed" />

### Screenshot 2 — Metadata Tracking
*Output of `sudo ./engine ps` showing ID, PID, state, start time, and memory limits.*

`[INSERT SCREENSHOT: sudo ./engine ps]`
<img width="932" height="131" alt="image" src="https://github.com/user-attachments/assets/9150c459-45c1-4a12-8c1c-db44bf3c41e9" />


### Screenshot 3 — Bounded-Buffer Logging
*Log file contents captured through the logging pipeline.*

`[INSERT SCREENSHOT: sudo ./engine logs alpha  &&  cat logs/alpha.log]`
<img width="814" height="205" alt="image" src="https://github.com/user-attachments/assets/c086ef25-4f87-46bc-abd3-c3508c72d164" />


### Screenshot 4 — CLI and IPC
*A CLI command being issued (start/stop/ps) and the supervisor responding via the UNIX domain socket.*

`[INSERT SCREENSHOT: sudo ./engine stop alpha  → supervisor terminal shows "Sent SIGTERM"]`
<img width="593" height="44" alt="image" src="https://github.com/user-attachments/assets/bff3e32c-5536-4c21-985d-d1b49a99501a" />

### Screenshot 5 — Soft-Limit Warning
*dmesg showing SOFT LIMIT event for memtest container.*

`[INSERT SCREENSHOT: dmesg | grep "SOFT LIMIT"]`
<img width="950" height="41" alt="image" src="https://github.com/user-attachments/assets/feeb8b65-0bac-425f-bcf0-b2cc9e7e0e3d" />

### Screenshot 6 — Hard-Limit Enforcement
*dmesg showing HARD LIMIT kill event; ps showing container state changed to "killed".*

`[INSERT SCREENSHOT: dmesg | grep "HARD LIMIT"  &&  sudo ./engine ps]`
<img width="1023" height="116" alt="image" src="https://github.com/user-attachments/assets/9b6d0b6a-aadb-4521-a05e-94c2873152e1" />


### Screenshot 7 — Scheduling Experiment
*Two CPU-bound containers at different nice values; measurable difference in completion time.*

`[INSERT SCREENSHOT: time output for nice=0 vs nice=19]`
<img width="730" height="292" alt="image" src="https://github.com/user-attachments/assets/3bbbf415-31dd-4512-b20c-fd113563988a" />

### Screenshot 8 — Clean Teardown
*No zombie processes after supervisor shutdown.*

`[INSERT SCREENSHOT: ps aux | grep defunct  (empty output)]`
<img width="1897" height="153" alt="image" src="https://github.com/user-attachments/assets/49e07d16-620b-4cae-aa01-a54818d26112" />


---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

The runtime uses Linux **namespaces** passed to `clone()` to isolate each container:

- **PID namespace** (`CLONE_NEWPID`): The container's init process sees itself as PID 1. The host kernel still assigns a real host PID, but the container cannot see or signal host processes.
- **UTS namespace** (`CLONE_NEWUTS`): Each container gets its own hostname (set to the container ID), independent of the host.
- **Mount namespace** (`CLONE_NEWNS`): Each container gets a private mount table. We mount `/proc` inside the container's rootfs and call `chroot()` to restrict filesystem view to the Alpine rootfs.

The host kernel still underlies everything: the same physical RAM, CPU scheduler, and kernel threads are shared. Namespaces only affect *visibility*, not resource enforcement — which is why the kernel module (cgroups-style enforcement) is needed for memory limits.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is essential because:

1. **Child reaping**: When a container (child) exits, its PCB remains in the kernel as a zombie until a parent calls `wait()`. A supervisor that stays alive can catch `SIGCHLD` and call `waitpid(-1, WNOHANG)` to reap all children, preventing zombie accumulation.
2. **State tracking**: A persistent supervisor maintains metadata (state, exit code, log path) that would be lost if the launcher exited immediately.
3. **Signal routing**: `SIGTERM`/`SIGINT` to the supervisor triggers orderly shutdown — stopping containers, joining threads, freeing memory — rather than an abrupt kill.

Process creation flow: `clone()` → child runs `child_fn()` → `chroot()` → `execv()` into the container command. The parent (supervisor) stores the host PID in `container_record_t` and registers it with the kernel monitor.

### 4.3 IPC, Threads, and Synchronization

The project uses two distinct IPC mechanisms:

**1. Pipe (log data path)**: Each container's `stdout`/`stderr` is connected to the supervisor via a `pipe()`. The supervisor's pipe-reader thread reads chunks and pushes them into the bounded buffer. This separates the fast container output path from the slower disk-write path.

**2. UNIX Domain Socket (control plane)**: CLI commands (`start`, `stop`, `ps`, `logs`) are sent as `control_request_t` structs over a `SOCK_STREAM` Unix socket at `/tmp/mini_runtime.sock`. The supervisor's event loop `accept()`s connections and `recv()`s requests. This is a second IPC mechanism distinct from the pipe.

**Bounded buffer synchronization**:

| Shared data | Protection | Why |
|-------------|-----------|-----|
| `bounded_buffer_t` items array | `pthread_mutex_t` + `pthread_cond_t not_full/not_empty` | Classic producer-consumer: prevents simultaneous read+write of the same slot; conditions allow threads to sleep instead of spin |
| `container_record_t *containers` linked list | `pthread_mutex_t metadata_lock` | Multiple threads (SIGCHLD handler, control request handler, logger) read/write the list; mutex prevents torn reads |

**Race conditions without synchronization**:
- Two pipe-reader threads could both write `tail` simultaneously, corrupting the buffer index.
- A SIGCHLD handler updating `c->state` while the `ps` handler reads it would produce garbage output.
- The logger thread could read a partially-written `log_item_t`.

### 4.4 Memory Management and Enforcement

**RSS (Resident Set Size)** measures the physical RAM pages currently mapped and present for a process. It does *not* include:
- Pages swapped out to disk
- Pages mapped but not yet faulted in (lazy allocation)
- Shared library pages counted once per process even if shared

**Why two limits?**
- **Soft limit**: An early warning. The process may be behaving normally with a temporary spike. A warning to `dmesg` lets an operator decide what to do without immediately killing the container.
- **Hard limit**: Absolute enforcement. If RSS exceeds this, the container is terminated with `SIGKILL`. This protects other containers and the host from memory exhaustion.

**Why kernel space?** A user-space monitor could be fooled: the container could `kill` the monitoring process, or the monitoring process itself might be descheduled. A kernel module runs at ring 0, cannot be killed by user processes, and has direct access to `mm_struct` RSS counters without needing `/proc` parsing. The kernel timer fires reliably every second regardless of user-space scheduling.

### 4.5 Scheduling Behavior

Linux uses the **Completely Fair Scheduler (CFS)** for normal processes. CFS assigns CPU time proportional to weight, where weight is derived from the `nice` value (nice=0 is baseline, nice=19 is lowest priority, nice=-20 is highest).

In our experiment (see Section 6), two identical CPU-bound containers ran simultaneously:
- Container `cpu1` at nice=0 (default weight)
- Container `cpu2` at nice=19 (lowest weight, ~1/5 of cpu1's share)

CFS gave `cpu1` roughly 5× more CPU time per scheduling period, resulting in measurably shorter completion time for cpu1. This demonstrates CFS's weighted fair queuing: fairness is relative to weight, not absolute equality.

For an I/O-bound workload, the process voluntarily yields CPU while waiting for I/O, so its vruntime advances slowly. CFS then schedules it more frequently when it becomes runnable — this is why I/O-bound processes feel "responsive" even at the same nice value as CPU-bound ones.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice**: `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` passed to `clone()`.
**Tradeoff**: We do not use `CLONE_NEWNET`, so containers share the host network stack. This simplifies the implementation (no veth pairs needed) but means containers can communicate freely on localhost.
**Justification**: The project spec requires PID, UTS, and mount isolation. Network isolation was not required and adds significant complexity.

### Supervisor Architecture
**Choice**: Single-threaded event loop with `accept()` + blocking `recv()` per connection, plus a detached per-container pipe-reader thread.
**Tradeoff**: One slow client can block the event loop momentarily. A thread-per-client design would avoid this but adds complexity.
**Justification**: For a lab demo with few containers, single-threaded is simpler to reason about for correctness and signal handling.

### IPC / Logging
**Choice**: Pipe for log data + UNIX domain socket for control.
**Tradeoff**: The socket is file-system-visible and must be cleaned up on exit. An anonymous socketpair would be cleaner but harder to connect to from a separate CLI invocation.
**Justification**: Named socket at a fixed path allows any number of CLI invocations to reach the supervisor without needing the supervisor's PID.

### Kernel Monitor
**Choice**: `mutex` (not `spinlock`) for the monitored list.
**Tradeoff**: Mutex cannot be held in hard-IRQ context. If we ever needed to call this from a hardware interrupt, we'd need a spinlock.
**Justification**: The timer callback runs in a softirq-safe but sleepable context on modern kernels (`timer_list` callbacks use `task_work` on recent kernels). `kmalloc(GFP_KERNEL)` in the ioctl path also requires a sleepable context, so mutex is correct here.

### Scheduling Experiments
**Choice**: `nice()` syscall for priority differentiation rather than `sched_setattr`.
**Tradeoff**: `nice` only affects CFS weight within the `SCHED_NORMAL` policy. Real-time policies (`SCHED_FIFO`, `SCHED_RR`) provide stronger guarantees but require root and bypass CFS entirely.
**Justification**: `nice` is the standard POSIX mechanism, universally available, and sufficient to demonstrate CFS weighting behavior.

---

## 6. Scheduler Experiment Results

### Experiment: CPU-bound containers at different priorities

Two containers running identical CPU-bound workload (`cpu_hog`, which computes in a tight loop for a fixed number of iterations).

| Container | nice value | Wall-clock time (seconds) |
|-----------|-----------|--------------------------|
| cpu1      | 0         | ~10s                     |
| cpu2      | 19        | ~48s                     |

*(Replace with your actual measured values)*

**What this shows**: CFS assigned cpu1 roughly 4–5× more CPU share than cpu2. The nice=19 weight is approximately 15 out of a total ~`15 + 1024 = 1039` weight units when combined with a nice=0 process, giving cpu2 about 1.4% of CPU. This matches the CFS weight table in the Linux kernel (`prio_to_weight[]`).

### Experiment: CPU-bound vs I/O-bound at same priority

| Container | workload  | nice | Observed behavior |
|-----------|-----------|------|------------------|
| cpu3      | cpu_hog   | 0    | High CPU, low I/O wait |
| io1       | io_pulse  | 0    | Low CPU, frequent I/O sleeps |

**What this shows**: Despite equal nice values, the I/O-bound container had lower vruntime accumulation (it sleeps most of the time), so CFS always scheduled it promptly when it woke up. The CPU-bound container got the remaining CPU time. This demonstrates CFS's inherent preference for interactive/I/O workloads: they naturally get low vruntime and thus high scheduling priority without any nice adjustment.

---

## Project Structure

```
.
├── engine.c           # User-space runtime and supervisor
├── monitor.c          # Kernel-space memory monitor (LKM)
├── monitor_ioctl.h    # Shared ioctl definitions
├── cpu_hog.c          # CPU-bound test workload
├── io_pulse.c         # I/O-bound test workload
├── memory_hog.c       # Memory-consuming test workload
├── Makefile           # Builds all of the above with `make`
├── README.md          # This file
└── logs/              # Per-container log files (created at runtime)
```
