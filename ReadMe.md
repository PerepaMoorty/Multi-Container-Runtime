# Multi-Container Runtime with Kernel Memory Monitor

## Overview

This project implements a lightweight multi-container runtime in C with a long-running supervisor and a Linux kernel module for memory monitoring. The system supports concurrent container execution, structured logging, CLI-based control, and controlled experiments for analyzing Linux scheduling behavior.

The project integrates user-space process management with kernel-space resource enforcement, providing a hands-on exploration of operating system concepts such as process isolation, inter-process communication, synchronization, and scheduling.

This work was developed as part of the Operating Systems course (UE24CS242B) at PES University. :contentReference[oaicite:0]{index=0}

---

## Team Information

- Moorty Perepa — PES2UG24CS248  
- Kritheesh N V — PES2UG24CS293  

---

## Project Architecture

The system consists of two main components:

### 1. User-Space Runtime (`engine.c`)
- Acts as a supervisor daemon
- Manages multiple containers concurrently
- Provides CLI interface for container lifecycle operations
- Implements IPC for control and logging
- Maintains metadata for all containers

### 2. Kernel-Space Monitor (`monitor.c`)
- Linux Kernel Module (LKM)
- Tracks container processes via PID
- Enforces:
  - Soft memory limits (warning)
  - Hard memory limits (SIGKILL)
- Communicates with user-space using `ioctl`

---

## Key Features

- Multi-container execution with namespace isolation
- Long-running supervisor with CLI interaction
- Two IPC mechanisms:
  - UNIX domain sockets (control path)
  - Pipes with bounded buffer (logging path)
- Concurrent logging using producer-consumer model
- Kernel-level memory monitoring with soft and hard limits
- Scheduler experimentation using controlled workloads
- Clean resource management and shutdown

---

## Repository Structure

```
.
├── engine.c              # User-space runtime and supervisor
├── monitor.c             # Kernel memory monitor module
├── monitor_ioctl.h       # Shared ioctl interface
├── cpu_hog.c             # CPU-bound workload
├── io_pulse.c            # I/O-bound workload
├── memory_hog.c          # Memory stress workload
├── environment-check.sh  # Environment validation script
├── Makefile              # Build system
├── logs/                 # Generated log files
└── README.md
```

---

## Environment Requirements

- Ubuntu 22.04 or 24.04 (VM only)
- Secure Boot disabled
- Kernel headers installed
- Root privileges for module loading

Run preflight check:

```bash
chmod +x environment-check.sh
sudo ./environment-check.sh
```

This script verifies:
- OS compatibility
- VM environment
- Kernel headers
- Module loading capability :contentReference[oaicite:1]{index=1}

---

## Build Instructions

```bash
make
```

This builds:
- User-space runtime (`engine`)
- Kernel module (`monitor.ko`)
- Workload binaries

---

## Setup Instructions

### 1. Load Kernel Module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
```

---

### 2. Prepare Root Filesystem

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
```

Create per-container copies:

```bash
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
```

---

### 3. Start Supervisor

```bash
sudo ./engine supervisor ./rootfs-base
```

---

## CLI Usage

The runtime supports the following commands:

```bash
engine supervisor <base-rootfs>
engine start <id> <rootfs> <command> [--soft-mib N] [--hard-mib N] [--nice N]
engine run   <id> <rootfs> <command> [--soft-mib N] [--hard-mib N] [--nice N]
engine ps
engine logs <id>
engine stop <id>
```

### Command Description

- `start` — Launch container in background
- `run` — Launch and wait for completion
- `ps` — List all containers and metadata
- `logs` — View container logs
- `stop` — Gracefully terminate container

---

## Workloads

### CPU-bound (`cpu_hog`)
- Continuously performs computation
- Prints progress every second :contentReference[oaicite:2]{index=2}  

### I/O-bound (`io_pulse`)
- Writes to disk in bursts with delays
- Simulates I/O-heavy workload :contentReference[oaicite:3]{index=3}  

### Memory-bound (`memory_hog`)
- Allocates memory in chunks over time
- Triggers soft/hard limit enforcement :contentReference[oaicite:4]{index=4}  

---

## Logging System

- Each container’s stdout/stderr is piped to the supervisor
- A bounded buffer (size = 16) stores log chunks
- Producer threads read from pipes
- Consumer thread writes logs to files

### Properties

- No log loss on abrupt termination
- Deadlock-free buffer design
- Clean shutdown with buffer flush

---

## Kernel Memory Monitoring

- Containers are registered via `ioctl`
- Kernel module tracks RSS usage periodically
- Uses linked list with mutex protection

### Policies

- **Soft Limit**: Logs warning once
- **Hard Limit**: Sends `SIGKILL` and removes entry

Supervisor classifies termination as:
- `EXITED` — normal completion
- `STOPPED` — user-requested termination
- `KILLED` — hard limit violation

---

## Scheduler Experiments

The runtime supports controlled experiments using:

- CPU-bound vs CPU-bound (different priorities)
- CPU-bound vs I/O-bound workloads

Parameters:
- `nice` values (priority)
- Concurrent execution

### Observations

- Lower nice value results in higher CPU share
- I/O-bound processes remain responsive due to scheduler fairness
- CPU-bound processes compete for time slices

---

## OS Concepts Demonstrated

### 1. Isolation Mechanisms
- PID, UTS, and mount namespaces
- `chroot` for filesystem isolation
- Shared kernel across containers

### 2. Supervisor and Process Lifecycle
- Parent-child relationships via `clone()`
- SIGCHLD handling and reaping
- Metadata tracking for each container

### 3. IPC, Threads, and Synchronization
- UNIX sockets for CLI communication
- Pipes for logging
- Mutexes and condition variables for buffer control

### 4. Memory Management and Enforcement
- RSS tracking via kernel
- Soft vs hard enforcement policies
- Kernel-space enforcement for reliability

### 5. Scheduling Behavior
- Effects of priority (`nice`)
- Fair scheduling across workloads
- Trade-offs between responsiveness and throughput

---

## Cleanup Instructions

### Stop Supervisor
Press `Ctrl+C` or send SIGTERM

### Unload Kernel Module

```bash
sudo rmmod monitor
```

### Remove Temporary Files

```bash
rm -rf rootfs-*
rm -rf logs/
```

---

## Notes

- Do not run on WSL
- Ensure unique rootfs per container
- Kernel module must be loaded for memory enforcement
- Logging directory is created automatically

---

## Conclusion

This project demonstrates a practical integration of user-space container management with kernel-level resource monitoring. It provides a minimal yet functional container runtime that exposes core operating system concepts through implementation and experimentation.

---