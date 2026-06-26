# Linux Advanced Concepts for DevOps

## 1. Union File System (UnionFS) - How Docker Images Work

### Traditional Filesystem
```
/app/
├── bin/
│   └── myapp
├── lib/
│   └── libc.so
└── etc/
    └── config.yaml

# Edit config.yaml
# Original file modified
```

### Union Filesystem (Docker)
```
Layer 1 (Base Image - Ubuntu)
├── bin/
│   ├── bash
│   ├── ls
│   └── cat
├── lib/
│   ├── libc.so
│   └── libm.so
└── etc/
    └── os-release

Layer 2 (Add Python)
├── bin/
│   └── python3
└── lib/
    └── libpython.so

Layer 3 (Application)
├── app/
│   ├── main.py
│   └── requirements.txt
└── etc/
    └── config.yaml

# When container runs:
# - All layers merged transparently
# - Write operations go to top layer (copy-on-write)
# - Original layers unchanged
```

### Copy-on-Write (CoW) Mechanism
```bash
# Image layers are read-only
# Container layer is read-write

Docker run ubuntu bash

# User runs: echo hello > /test.txt
# 1. Kernel doesn't find /test.txt in upper layer
# 2. Creates entry in container (upper) layer
# 3. Writes to container layer
# 4. Original image untouched

docker ps -s
# Shows size = base image + only changes made by container
```

### Benefits
```
10 containers from same image:
- Image stored once: 200MB
- Each container: 5MB (only changes)
- Total: 200MB + (10 × 5MB) = 250MB

Without UnionFS:
- 10 complete copies: 10 × 200MB = 2000MB
```

---

## 2. Virtual Ethernet (veth) - Container Networking

### How veth Works
```bash
# veth = virtual ethernet device pair
# Like a virtual cable connecting two endpoints

# Example: Container networking
Pod (network namespace)
├── eth0 (inside pod)
└── connected to veth0 (on host)

Host
├── veth1a2b3c (linux bridge slave)
└── connected to br0 (linux bridge)

# When pod sends packet:
# 1. Pod app -> socket -> eth0 (in pod namespace)
# 2. Packet travels through veth pair
# 3. Arrives at br0 on host
# 4. Bridge forwards based on MAC address
# 5. Reaches destination
```

### Kubernetes Service Networking with veth
```
Pod A (10.0.1.5)
    |
    veth0a
    |
    linux bridge (br0)
    |
    kube-proxy (iptables rules)
    |
Service IP: 10.1.0.1:80
    |
    Routes to:
    - Pod B (10.0.1.10)
    - Pod C (10.0.2.5)
    - Pod D (10.0.2.8)
```

---

## 3. iptables - Firewall Rules Inside Containers

### What is iptables?
Linux kernel firewall with rules for packet processing.

### iptables Chain
```
Packet enters Linux kernel
    |
    ├─ INPUT chain (packets destined for local machine)
    ├─ FORWARD chain (packets passing through
    └─ OUTPUT chain (packets from local machine)
```

### Docker Example
```bash
# When you run:
docker run -p 8080:80 nginx

# Docker creates iptables rule:
iptables -t nat -A DOCKER \
  -p tcp --dport 8080 \
  -j DNAT --to-destination 172.17.0.2:80

# Translation:
# Traffic on host:8080 -> redirects to container:80

# View rules:
iptables -t nat -L
# Shows all NAT rules (Docker, Kubernetes, etc.)
```

### Kubernetes Service Networking with iptables
```bash
# Service created: myservice:80 -> 10.1.0.1:80
# kube-proxy adds iptables rules:

# Rule 1: Redirect service traffic
iptables -A KUBE-SERVICES \
  -d 10.1.0.1/32 -p tcp -m tcp --dport 80 \
  -j KUBE-SVC-ABC123

# Rule 2: Load balance to endpoints
iptables -A KUBE-SVC-ABC123 \
  -j KUBE-EP-ABC123

# Rule 3: Actual destination (round-robin)
iptables -A KUBE-EP-ABC123 -m statistic --mode random --probability 0.5 \
  -j DNAT --to-destination 10.0.1.5:80
iptables -A KUBE-EP-ABC123 \
  -j DNAT --to-destination 10.0.1.10:80

# Result: Traffic to service:80 load balanced to both pods
```

### Debugging iptables
```bash
# See all rules
iptables -L -n -v

# See NAT rules
iptables -t nat -L -n -v

# Monitor packet flow
iptables -t nat -I DOCKER -d 127.0.0.1/32 -p tcp --dport 8080 -j LOG --log-prefix "CONTAINER: "

# Watch logs
tail -f /var/log/syslog | grep CONTAINER
```

---

## 4. Linux Load Balancing vs Service Mesh

### Linux iptables Load Balancing (K8s default)
```
┌─────────────────────────────────┐
│ Service 10.1.0.1:80             │
└────────────────┬────────────────┘
                 │
                 ├─ iptables rules
                 │  (stateless random)
                 │
        ┌────────┼────────┐
        ↓        ↓        ↓
     Pod A    Pod B    Pod C
   10.0.1.5 10.0.1.10 10.0.2.5

Limitations:
- No connection pooling
- Simple round-robin only
- No advanced traffic splitting
- Hard to debug
```

### Service Mesh (Istio, Linkerd)
```
┌──────────────────────────────┐
│ Istio Service Mesh           │
│ (runs as Linux process)      │
└────────────────┬─────────────┘
                 │
        ┌────────┼────────┐
        ↓        ↓        ↓
    Envoy    Envoy    Envoy  (sidecar proxies)
    10.0    10.0     10.0   (runs in pod namespace)

Features:
- Connection pooling
- Retry logic
- Circuit breaking
- Canary deployments
- Mutual TLS
- Advanced routing rules
```

---

## 5. Linux /proc Filesystem - Container Monitoring

### Understanding /proc
```bash
# /proc is virtual filesystem showing kernel state
ls /proc

1/              # Process 1 files
2/
3/
...
cmdline         # Kernel command line
cpuinfo         # CPU information
meminfo         # Memory usage
loadavg         # Load average
uptime          # System uptime
interrupts      # Interrupt count
```

### Container Process Information
```bash
# For running container:
CONTAINER_PID=$(docker inspect --format='{{.State.Pid}}' <container-id>)

# View container resource usage
cat /proc/$CONTAINER_PID/stat
# Shows CPU time, memory, etc.

cat /proc/$CONTAINER_PID/io
# Shows disk I/O statistics

cat /proc/$CONTAINER_PID/net/tcp
# Shows open TCP connections
```

### Kubernetes Monitoring from /proc
```bash
# Prometheus scrapes /proc to get metrics
# For each container:

# CPU usage (in cgroup interface)
cat /sys/fs/cgroup/cpuacct/docker/<id>/cpuacct.usage

# Memory usage
cat /sys/fs/cgroup/memory/docker/<id>/memory.usage_in_bytes

# Network stats (from pod namespace)
cat /proc/net/dev

# All fed to Prometheus -> Grafana dashboards
```

---

## 6. Linux Signals - Container Lifecycle

### Signal Flow
```bash
SIGHUP (1)    # Hang up - terminal closed
SIGINT (2)    # Interrupt - Ctrl+C
SIGQUIT (3)   # Quit - Ctrl+\
SIGKILL (9)   # Kill - cannot be caught
SIGTERM (15)  # Terminate - default
SIGSTOP (19)  # Stop - cannot be caught
SIGCONT (18)  # Continue
```

### Docker/Kubernetes Container Termination
```
1. User runs: kubectl delete pod myapp
   OR
   docker stop <container>

2. SIGTERM sent to container PID 1
   (Grace period: default 30 seconds)

3. Container receives SIGTERM
   (app should perform graceful shutdown)
   - Close database connections
   - Flush buffers
   - Save state
   - Exit

4a. If app exits: container stops ✓

4b. If app ignores SIGTERM:
    (After grace period)
    
5. SIGKILL sent (cannot be ignored)
   (Immediate, forceful termination)

6. Container stops

# Configuration:
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  terminationGracePeriodSeconds: 30  # Grace period
  containers:
  - name: app
    image: myapp
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # Extra cleanup time
```

### Dockerfile Signal Handling
```dockerfile
FROM ubuntu

# Use exec form (important for signal forwarding!)
CMD ["python", "app.py"]
# Good: Signals forwarded to Python process

# Bad: Shell form - signals not forwarded
CMD python app.py
# Signals go to /bin/sh, not Python
```

---

## 7. Linux Resource Limits - cgroup Deep Dive

### CPU Cgroups
```bash
# cpu.shares - CPU allocation (proportional)
# 1024 = 1 CPU share
# If two containers both at 1024:
# Each gets 50% of available CPU

# cpuset.cpus - Pin to specific CPU cores
cgroup_cpuset_cpus="0-3"  # Use CPU 0,1,2,3 only

# cpu.cfs_period_us - Scheduling period
# cpu.cfs_quota_us - CPU time allowed per period
# Together implement hard limits

cpu.cfs_period_us = 100000  # 100ms period
cpu.cfs_quota_us = 50000    # 50ms allowed per 100ms
# = 50% CPU hard limit
```

### Memory Cgroups
```bash
# memory.limit_in_bytes - Hard memory limit
# If exceeded: OOM killer triggers

# memory.memsw.limit_in_bytes - Memory + swap limit
# Prevents excessive swapping

# memory.swappiness - How much to use swap (0-100)
# 0 = never swap
# 100 = swap aggressively

# Kubernetes request/limit mapping:
spec:
  containers:
  - resources:
      requests:
        memory: "256Mi"   # scheduler consideration
      limits:
        memory: "512Mi"   # -> memory.limit_in_bytes
```

### I/O Cgroups (blkio)
```bash
# Limit disk I/O bandwidth
# blkio.throttle.read_bps_device - Read bytes/sec
# blkio.throttle.write_bps_device - Write bytes/sec

# Important for:
# - Preventing noisy neighbor problem
# - Fair disk access
# - Database containers don't starve others
```

---

## 8. Linux Kernel Tuning for Containers

### Connection Tracking
```bash
# netfilter connection tracking
# Important for iptables-based load balancing

# /proc/net/nf_conntrack - Connection table
cat /proc/net/nf_conntrack | head
# Shows established connections

# Tune for containers:
net.netfilter.nf_conntrack_max = 2000000
net.netfilter.nf_conntrack_tcp_timeout_established = 432000
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 60
```

### Socket Buffer Tuning
```bash
# TCP/UDP buffer sizes
# For high-throughput container networks

net.core.rmem_max=134217728          # Max receive buffer
net.core.wmem_max=134217728          # Max write buffer
net.ipv4.tcp_rmem=4096 87380 67108864   # Dynamic TCP receive
net.ipv4.tcp_wmem=4096 65536 67108864   # Dynamic TCP write
```

### Backlog Tuning
```bash
# For containers handling many connections

net.core.somaxconn=65535              # Listen queue
net.ipv4.tcp_max_syn_backlog=65535    # SYN queue
net.core.netdev_max_backlog=65535     # Network device queue
```

---

## 9: Linux Container Escape Vectors

### Common Attack Vectors

#### 1. Privileged Mode
```bash
# DANGEROUS: Don't use in production!
docker run --privileged ubuntu bash

# Inside container:
fs.show_hidden=0
ls -la /dev/
# Can see ALL host devices
# Can mount host filesystem
# Can manipulate kernel modules
```

#### 2. Capability Escalation
```bash
# If container has CAP_SYS_PTRACE:
# Can debug any process on system
# Can inject code into other containers!

getcap /bin/app
# = cap_sys_ptrace+ep
# Vulnerable!
```

#### 3. Kernel Vulnerability
```bash
# If kernel is unpatched:
# CVE-2021-22555 (iptables)
# Container escape via namespace confusion
```

#### 4. /proc/sysrq-b Exploitation
```bash
# If kernel.sysrq enabled:
# Container can crash entire host

echo b > /proc/sysrq-trigger
# Host reboots!
```

### Hardening Checklist
```bash
# Limit capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE ubuntu

# Read-only filesystem
docker run --read-only ubuntu

# Resource limits
docker run -m 512m --cpus=1 ubuntu

# No new privileges
docker run --security-opt=no-new-privileges ubuntu

# AppArmor or SELinux profile
docker run --security-opt apparmor=docker-default ubuntu
```

---

## 10. Linux System Calls (syscalls) - How Containers Talk to Kernel

### Important syscalls for Containers

```c
// Namespace creation
clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWUTS)
// Creates isolated namespace

// Capability manipulation
capset()
// Restricts container capabilities

// Memory mapping
mmap()
// Container memory allocation

// Network socket
socket(AF_INET, SOCK_STREAM)
// Container networking

// File operations
open(), read(), write(), close()
// Container file access
```

### Tracing syscalls
```bash
# See what syscalls container makes:
strace -p <container-pid>

# Example output:
clone(CLONE_VM|CLONE_FS|...) = 123
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fff2b3a0000
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
bind(3, {sa_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("0.0.0.0")}, 16) = 0

# Important for:
# - Understanding container behavior
# - Debugging permission issues
# - Security auditing
```

### Seccomp - Syscall Filtering
```json
{
  "defaultAction": "SCMP_ACT_ALLOW",
  "defaultErrnoRet": 1,
  "archMap": [
    {
      "architecture": "SCMP_ARCH_X86_64",
      "subArchitectures": [
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
      ]
    }
  ],
  "syscalls": [
    {
      "names": [
        "clone",
        "fork",
        "vfork"
      ],
      "action": "SCMP_ACT_ALLOW",
      "args": []
    },
    {
      "names": [
        "ptrace"
      ],
      "action": "SCMP_ACT_ERRNO",
      "errno": 1
    }
  ]
}
```

Blocks dangerous syscalls like ptrace, preventing container escape.

---

## 11. Linux Filesystem Types - Container Storage

### Overlay2 (Default Docker Storage)
```bash
# Most Linux distributions use overlay2

# Location:
/var/lib/docker/overlay2/

# Structure:
image-layers/
├── layer1/
│   ├── bin/
│   └── lib/
├── layer2/
│   ├── python/
│   └── lib/
└── layer3/
    ├── app/
    └── etc/

container-layer/
├── diff/          # Changes made by container
└── work/          # Temporary files

# Read-write layer stacked on read-only image layers
```

### Other Storage Drivers
```bash
# AUFS - Older (Ubuntu)
# BTRFS - Advanced copy-on-write
# ZFS - Enterprise filesystem
# devicemapper - Legacy Red Hat

# View current driver:
docker info | grep "Storage Driver"
```

---

## 12. Linux Boot Process & Init Systems

### Container init vs Host init
```
Host Boot:
1. Bootloader (GRUB)
2. Linux Kernel loads
3. Kernel initializes hardware
4. Init system starts (systemd)
5. Systemd starts services
6. System ready

Container Start:
1. Already in running kernel
2. New namespaces created
3. Container init process (PID 1) starts
4. Container app starts
5. Container ready
```

### Container Init Alternatives
```dockerfile
# Option 1: Direct app (minimal)
FROM ubuntu
CMD ["python", "app.py"]
# Pro: Small, fast
# Con: No signal handling, no orphan reaping

# Option 2: Shell (common)
FROM ubuntu
CMD /app/start.sh
# Pro: Easy
# Con: PID 1 is shell, bad for signals

# Option 3: Proper init system (best)
FROM ubuntu
RUN apt-get install -y tini
ENTRYPOINT ["/tini", "--"]
CMD ["python", "app.py"]
# Pro: Handles signals, reaps zombies
# Con: Tiny overhead
```

---

## Quick Reference: Linux Commands for DevOps

```bash
# Process monitoring
ps aux                          # All processes
htop                           # Interactive monitoring
top                            # CPU/memory top processes
pgrep -f "pattern"             # Find process by name

# Network
netstat -tlnp                  # Listening ports with process
ss -s                          # Socket statistics
ip addr show                   # IP addresses
ip route show                  # Routes
iptables -L -n                 # Firewall rules

# Filesystem
df -h                          # Disk usage
du -sh ./*                     # Directory sizes
mount                          # Mounted filesystems
ls -la                         # File permissions

# Memory
free -h                        # Memory usage
cat /proc/meminfo              # Detailed memory
swapoff -a                     # Disable swap

# Kernel
uname -a                       # Kernel info
cat /proc/version              # Kernel version
sysctl -a                      # Kernel parameters
dmesg                          # Kernel messages

# User/Permission
whoami                         # Current user
id                             # User/group IDs
sudo -l                        # Sudo permissions
chmod 755 file                 # Change permissions
chown user:group file          # Change ownership

# System
uptime                         # How long running
loadavg                        # Load average
w                              # Who's logged in
last                           # Login history
```

