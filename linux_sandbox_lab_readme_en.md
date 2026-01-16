# Linux Sandbox Experiment: PID and `/proc` Isolation

![Linux Sandbox](https://img.shields.io/badge/Linux-Sandbox-blue?style=flat-square)

## Overview

This project explores **process isolation, PID namespaces, and `/proc` filesystem inconsistencies** using Linux. The goal is to understand how processes are managed and exposed inside lightweight sandboxes, leveraging **Bubblewrap (`bwrap`)**, `unshare`, and `chroot`.

By creating isolated environments, this experiment allows developers and system engineers to investigate:

- PID differences between host and sandbox
- Visibility of processes via `/proc` vs `ps`
- Root filesystem changes within a sandbox
- Security implications of process isolation

This project is **educational**, designed to deepen understanding of Linux internals, containers, and process namespace behavior.

---

## Tools and Requirements

- Linux host (Debian/Ubuntu recommended)
- `bubblewrap` (`bwrap`) — for lightweight sandboxing
- `unshare` — for namespace isolation
- `chroot` (optional, for legacy isolation experiments)
- `bash` — for automation scripts
- Basic Linux utilities: `ps`, `ls`, `mount`, `echo`, `grep`, `wc`

---

## Setup Instructions

### 1. Prepare the Environment

```bash
mkdir -p ~/security-lab/chroot
cd ~/security-lab
```

### 2. Install Required Packages

```bash
sudo apt update
sudo apt install -y debootstrap util-linux procps iproute2 net-tools sudo bwrap
```

### 3. Bootstrap Minimal Debian Environment (Optional)

```bash
sudo debootstrap --variant=minbase trixie ~/security-lab/chroot http://deb.debian.org/debian
```

### 4. Enter Isolated Shell with Bubblewrap

```bash
bwrap \
  --dev-bind / / \
  --proc /proc \
  --tmpfs /tmp \
  --unshare-pid \
  --unshare-uts \
  --unshare-ipc \
  /bin/bash
```

> This command starts a sandboxed shell with isolated PID, mount, UTS, and IPC namespaces. `/proc` is mounted inside the sandbox, allowing experimentation without affecting the host.

---

## Experiment Steps

### Step 1: Baseline Environment

```bash
mkdir -p /tmp/lab
cd /tmp/lab
ps aux > baseline_ps.txt
ls -la / > baseline_rootfs.txt
```

- Captures initial processes and root filesystem structure.
- PID inside sandbox differs from host PID.

### Step 2: Sandbox Validation

```bash
echo "PID inside sandbox: $$"
ps -p 1
```

- Confirms that the shell PID inside sandbox is **isolated**.
- `ps -p 1` shows the sandbox init process (`bwrap`).

### Step 3: Compare `/proc` vs `ps`

```bash
ls /proc | grep -E '^[0-9]+$' > proc_pids.txt
ps -eo pid > ps_pids.txt
diff proc_pids.txt ps_pids.txt
```

- Demonstrates **differences between `/proc` entries and `ps` output**.
- Key insight: not all host PIDs are visible in the sandbox.

### Step 4: Test Filesystem Isolation

```bash
touch /test-host
touch /tmp/sandbox-ok
ls /tmp
```

- Confirms that changes inside `/tmp` are sandbox-specific.
- `/home` or other host directories are not visible unless explicitly bound.

---

## Observations

- PIDs inside sandbox are **namespace-isolated**, starting from `1`.
- `/proc` reflects **only processes in the namespace**, while host processes remain hidden.
- Bubblewrap (`bwrap`) provides **Docker-like isolation** without root privileges.
- `chroot` can be used, but requires careful mounting of `/proc`, `/dev`, and `/sys`.

---

## Common Issues

- `chroot: cannot change root` — often caused by incorrect directory path or missing mounts.
- Host PIDs not visible — expected due to PID namespace isolation.
- Bash may not show typed passwords when using `sudo` — this is normal behavior in Linux.

---

## Scripts

- `compare_procs.sh` — compares `/proc` PIDs with `ps` output
- `capture_baseline.sh` — captures host rootfs and process state
- `sandbox_test.sh` — starts a `bwrap` sandbox and performs initial validation

---

## Technical Insights

- PID namespaces isolate processes; PID `1` inside sandbox is not host `init`.
- `/proc` provides process information per namespace.
- Bubblewrap enables **secure, minimal sandboxes** for testing, pentesting, or education.
- Understanding these mechanics is critical for **SRE, containerization, and Linux security**.

---

## References

- [Bubblewrap GitHub](https://github.com/containers/bubblewrap)
- [Linux Namespaces Documentation](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Debian debootstrap](https://wiki.debian.org/Debootstrap)
- [Proc Filesystem (`/proc`)](https://man7.org/linux/man-pages/man5/proc.5.html)

---

## Author

**Marcel Fernandes Ribeiro**  
Engineering student and Linux enthusiast exploring **system internals, sandboxing, and security engineering**.

