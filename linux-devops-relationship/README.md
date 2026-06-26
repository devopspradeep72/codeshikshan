# Linux OS & DevOps Tools - Deep Relationship & Interview Guide

## Core Understanding: Why Linux is Foundation for All DevOps Tools

Linux OS is the **backbone** of modern DevOps. Every tool (Docker, Kubernetes, Terraform, Ansible) fundamentally relies on Linux concepts. Let's understand this relationship from the ground up.

---

## Part 1: Linux Kernel Fundamentals

### What is Linux Kernel?
The Linux kernel is the core that manages:
- **Process Management**: Running multiple programs simultaneously
- **Memory Management**: Allocating and freeing RAM
- **File System**: Organizing data on disk
- **Networking Stack**: TCP/IP protocols
- **Device Management**: Hardware access
- **Security**: User permissions, file access control

### Linux Process Model
```bash
# Every application runs as a process
# Process = Program in execution + its resources

# View running processes
ps aux

# Output shows:
USER       PID   CPU MEM    COMMAND
root       1     0.0 0.1    /sbin/init              # Init process (parent of all)
root       123   0.1 0.5    /usr/sbin/sshd          # SSH service
ubu        456   0.2 1.2    python app.py           # Your application
```

### Key Concept: Process Isolation
```bash
# Each process is isolated from others:
# - Memory: Process A can't access Process B's memory
# - File descriptors: Each has own open files
# - Network ports: Each can listen on different port
# - Users/Permissions: Different processes run as different users

# This isolation is critical for container security!
```

---

## Part 2: Linux Namespaces (Foundation of Containers)

### What are Namespaces?
Namespaces are Linux kernel features that provide **process isolation**. They partition kernel resources so that one group of processes sees one set, and another group sees a different set.

### 6 Main Namespaces

#### 1. **PID Namespace** (Process Isolation)
```bash
# Host OS sees:
$ ps aux
PID   COMMAND
1     /sbin/init
100   docker-containerd
101   application-in-container
102   another-app

# Container sees:
PID   COMMAND
1     application-in-container      # Thinks it's PID 1!
2     another-app

# Container's PID 1 ≠ Host's PID 1
# This is why docker exec container ps shows PID 1 inside, but it's actually PID 101 on host
```

**Interview Question:** "Why does a container's init process have PID 1?"
**Answer:** PID namespace isolates process IDs. Each namespace has its own PID numbering starting from 1.

---

#### 2. **Network Namespace** (Network Isolation)
```bash
# Host has network interface: eth0 with IP 192.168.1.100
# Container has own network interface: eth0 with IP 172.17.0.2

# They're completely separate:
# - Different IP addresses
# - Different routing tables
# - Different firewall rules
# - Different open ports

# Example: Two containers can both listen on port 80
Container 1: 172.17.0.2:80
Container 2: 172.17.0.3:80
# No conflict! Each thinks it owns port 80

# How they communicate:
Container 1 -> Docker Bridge (172.17.0.1) -> Container 2
```

**Interview Question:** "Can two containers listen on the same port?"
**Answer:** Yes! Each has its own network namespace. But on the host, they must map to different ports (e.g., -p 8001:80 and -p 8002:80).

---

#### 3. **IPC Namespace** (Inter-Process Communication)
```bash
# Host can use:
# - Shared Memory
# - Message Queues  
# - Semaphores

# Container has own isolated IPC
# Processes inside container can communicate with each other
# But can't communicate with host or other containers' IPC

# Example:
# App1 in Container A can use shared memory with App2 in Container A
# But App3 in Container B can't access that shared memory
```

---

#### 4. **Mount Namespace** (Filesystem Isolation)
```bash
# Host filesystem:
/
├── bin
├── etc
├── var
└── home

# Container sees only:
/
├── app
├── lib (copied from image)
├── bin (copied from image)
└── etc (copied from image)

# Docker Volume Mount is exception:
docker run -v /host/path:/container/path
# Shares /host/path from host into container
```

**Interview Question:** "How does Docker isolate the filesystem?"
**Answer:** Mount namespace shows container a different filesystem hierarchy. Combined with Union FS, each container sees only what's in its image + mounted volumes.

---

#### 5. **UTS Namespace** (Hostname Isolation)
```bash
# Host hostname: server-prod-01
hostname
# server-prod-01

# Container has different hostname:
docker run --hostname app-container ubuntu bash
hostname
# app-container

# Each container can have its own hostname
```

---

#### 6. **User Namespace** (User ID Isolation)
```bash
# Host user IDs:
# UID 0 = root
# UID 1000 = alice

# Container can remap user IDs:
# Container's UID 0 (container root) = Host's UID 1000 (alice)

# Security benefit:
# If container is compromised, attacker gets alice's permissions (UID 1000)
# Not root (UID 0) on host

# This is USER NAMESPACING - critical security feature!
```

---

## Part 3: Linux Cgroups (Resource Limits)

### What are Cgroups?
Control Groups (cgroups) limit and track resource usage.

### Memory Cgroups Example
```bash
# Without cgroups:
# Process can consume all RAM
# System freezes, OOM kills random process

# With cgroups:
docker run -m 512m ubuntu
# Container limited to 512MB RAM
# If exceeds, cgroup kills the process

# File location (on Linux host):
ls /sys/fs/cgroup/memory/docker/<container-id>/
# Shows exact memory usage
```

### CPU Cgroups Example
```bash
# Limit container to 1 CPU core:
docker run --cpus=1 ubuntu

# Limit container to 512MB of 1GB available CPU:
docker run --cpu-shares=512 ubuntu

# Under the hood:
cat /sys/fs/cgroup/cpuset/docker/<container-id>/cpuset.cpus
# Shows which CPUs container can use
```

**Interview Question:** "What happens if container exceeds memory limit?"
**Answer:** The cgroup enforces the limit. If container tries to use more, the process gets OOM (Out of Memory) killed. This is similar to K8s memory limits causing pod termination.

---

## Part 4: Docker & Linux Relationship

### How Docker Uses Linux Features

```
Docker Container = Namespaces + Cgroups + Union FS + Minimal Rootfs
         ↓
    Lightweight VM (but not a real VM)
```

### Docker Architecture
```
┌─────────────────────────────────────┐
│ Docker Client (docker CLI)          │
│ $ docker run -d myapp               │
└────────────────┬────────────────────┘
                 │
                 ↓ (API call)
┌─────────────────────────────────────┐
│ Docker Daemon (dockerd)             │
│ - Manages containers                │
│ - Talks to containerd               │
└────────────────┬────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────┐
│ containerd (Container Runtime)      │
│ - Creates namespaces                │
│ - Sets up cgroups                   │
└────────────────┬────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────┐
│ runc (OCI Runtime)                  │
│ - Actually calls Linux syscalls      │
│ - clone() for namespace creation    │
│ - cgroup setup                      │
└────────────────┬────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────┐
│ Linux Kernel                        │
│ - Creates container process         │
│ - Enforces isolation                │
│ - Manages resources                 │
└─────────────────────────────────────┘
```

### Linux Syscalls Used by Docker

```c
// When you run: docker run ubuntu bash
// Inside runc/containerd:

// 1. Create namespaces
clone(CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWUTS | ...)
// Creates isolated environment

// 2. Set up cgroups
cgroup_create(...)
// Limits resources

// 3. Change root filesystem
chroot() or pivot_root()
// Container sees only its rootfs

// 4. Set capabilities (minimal permissions)
capset(...)
// Container doesn't get CAP_SYS_ADMIN etc

// 5. Execute container init
exec()
// Starts the application
```

---

## Part 5: Kubernetes & Linux Relationship

### Kubernetes Architecture on Linux

```
┌──────────────────────────────────────────┐
│ Kubernetes Control Plane                 │
│ - etcd (state store on Linux filesystem) │
│ - API Server                             │
│ - Scheduler                              │
│ - Controller Manager                     │
└────────────────┬─────────────────────────┘
                 │
    ┌────────────┼────────────┐
    ↓            ↓            ↓
┌──────────┐ ┌──────────┐ ┌──────────┐
│  Linux   │ │  Linux   │ │  Linux   │
│ Worker 1 │ │ Worker 2 │ │ Worker 3 │
│   ├─────────────┐     │    │       │
│   │ Pod 1 (ns)  │     │    │       │  ns = namespace
│   │ Pod 2 (ns)  │     │    │       │
│   └─────────────┘     │    │       │
└──────────┘ └──────────┘ └──────────┘
```

### How Kubernetes Uses Linux

#### 1. **Pods are Linux Processes with Namespaces**
```bash
# When you create a pod:
kubectl run myapp --image=ubuntu

# Behind the scenes:
# 1. Kubernetes tells kubelet to create container
# 2. kubelet calls container runtime (Docker/containerd)
# 3. Container runtime creates Linux namespaces
# 4. Pod = group of Linux processes in shared namespaces
```

#### 2. **Pod Internal Network (Network Namespace)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
  - name: logging
    image: logger:latest
```

```
# Two containers in same pod:
# - Share SAME network namespace
# - Same IP address (e.g., 10.0.1.5)
# - Each container binds to different port
#   - app listens on 10.0.1.5:8080
#   - logger listens on 10.0.1.5:9000
# - They communicate via localhost:port

app container: curl http://localhost:9000  # Reaches logger container!
```

**Interview Question:** "Why can two containers in a pod communicate via localhost?"
**Answer:** They share the same network namespace, so they have the same IP and loopback interface.

---

#### 3. **Service Discovery (DNS on Linux)**
```bash
# Kubernetes uses CoreDNS (runs as Linux container)
# CoreDNS runs on every Linux node

# When pod needs to reach service:
# app-pod -> asks CoreDNS (10.96.0.10:53)
#         -> CoreDNS queries etcd
#         -> Returns service IP
#         -> iptables routes to actual pod

# All based on Linux DNS client, iptables firewall
```

#### 4. **Storage (Linux Filesystem)**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-pod
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: data
      mountPath: /data          # Inside container
  volumes:
  - name: data
    hostPath:
      path: /var/data           # On Linux node
      type: Directory
```

```
# Volume = mount namespace feature
# /var/data from Linux host -> /data in container
# Uses Linux mount() syscall under the hood
```

---

## Part 6: Terraform & Linux Relationship

### Terraform Provisions Linux Systems

```hcl
# This Terraform code:
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # Ubuntu Linux image
  instance_type = "t3.micro"
  
  user_data = base64encode("""
    #!/bin/bash
    apt-get update
    apt-get install -y docker.io
  """)
}

# Results in:
# 1. EC2 instance created with Linux OS
# 2. User data script runs (bash script)
# 3. Docker installed on Linux
```

### Infrastructure Dependencies on Linux

```
Terraform Code
     ↓
Cloud API (AWS, GCP, Azure)
     ↓
Provisioned Linux Instances
     ↓
Linux filesystems, networking, services
     ↓
Docker runs on Linux
     ↓
Kubernetes runs on Linux nodes
```

**Key Point:** Terraform doesn't create the Linux OS, it orchestrates cloud APIs to provision Linux instances. The Linux kernel then provides the environment for containers.

---

## Part 7: Ansible & Linux Relationship

### Ansible Configures Linux Systems

```yaml
- name: Configure Linux servers
  hosts: web_servers
  tasks:
    - name: Install package (Linux)
      apt:
        name: nginx
        state: present
    
    - name: Manage Linux users
      user:
        name: appuser
        shell: /bin/bash
        groups: docker,sudo
    
    - name: Manage Linux permissions
      file:
        path: /app
        owner: appuser
        group: appuser
        mode: '0755'
    
    - name: Manage Linux services
      systemd:
        name: docker
        enabled: yes
        state: started
```

### Ansible Communication with Linux

```
Ansible Control Node (usually Linux)
     ↓
SSH connection
     ↓
Linux Target Node
     ↓
Linux shell (bash)
     ↓
Linux commands execution
     ↓
Linux filesystem/services modified
```

---

## Part 8: Complete Linux-DevOps Workflow

### Real Production Scenario

```
1. INFRASTRUCTURE (Terraform)
   ├─ Provisions AWS EC2 instances (Linux OS)
   ├─ Creates VPC (networking on Linux kernel)
   ├─ Sets up security groups (Linux iptables)
   └─ Creates RDS (Linux database)

2. CONFIGURATION (Ansible)
   ├─ SSHes into Linux instances
   ├─ Installs Docker daemon (Linux service)
   ├─ Configures Linux kernel parameters
   ├─ Sets up Linux users and permissions
   └─ Enables Linux services (systemd)

3. CONTAINERIZATION (Docker)
   ├─ Builds image with Linux rootfs
   ├─ Uses Linux namespaces (PID, network, mount)
   ├─ Enforces limits with cgroups
   └─ Containers run as Linux processes

4. ORCHESTRATION (Kubernetes)
   ├─ Runs on Linux nodes
   ├─ kubelet manages Linux containers
   ├─ kube-proxy manages Linux iptables for networking
   ├─ etcd stores on Linux filesystem
   └─ All pods are Linux processes

5. MONITORING (Linux-based)
   ├─ Prometheus scrapes /proc filesystem
   ├─ Logs from Linux syslog
   ├─ metrics from cgroup interfaces
   └─ Traces from Linux perf tools
```

---

## Part 9: Essential Linux Concepts for DevOps

### 1. Process Management
```bash
# View processes
ps aux              # All processes
htop                # Interactive process viewer

# Process signals
kill -9 <PID>       # Forcefully kill process (used by K8s termination)
signal SIGTERM      # Graceful shutdown

# Background/Foreground
jobs                # List background jobs
fg %1               # Bring job to foreground
bg %1               # Send job to background
```

**Interview Context:** Kubernetes sends SIGTERM to containers, giving them grace period to shut down gracefully.

---

### 2. User & Permission Model
```bash
# Linux has 3-tier permission model:
# - Owner (user)
# - Group
# - Others

ls -la /app
# -rwxr-xr-x 1 appuser appgroup
#  ││││││││ └─ others can execute
#  │││││││ └── others can read
#  ││││││ └─── group can execute
#  │││││ └──── group can read
#  ││││ └───── group can write
#  │││ └────── owner can execute
#  ││ └─────── owner can read
#  │ └──────── owner can write
#  └────────── file type (- = regular file)

# Capabilities (Linux 2.2+)
# Processes don't just have "root" or "not root"
# Processes have specific capabilities
# CAP_NET_ADMIN, CAP_SYS_ADMIN, etc.
```

**Interview Context:** Containers run with limited capabilities by default. Docker drops dangerous capabilities:
```bash
docker run \
  --cap-drop=ALL \                    # Drop all
  --cap-add=NET_BIND_SERVICE ubuntu   # Add only needed
```

---

### 3. Filesystem Hierarchy
```bash
/bin            # Essential user commands (ls, cat, bash)
/sbin           # System administration commands (mount, ifconfig)
/etc            # Configuration files
/home           # User home directories
/root           # Root home directory
/lib            # Libraries
/usr            # User programs and data
/var            # Variable data (logs, cache, temp)
/proc           # Virtual filesystem with process info
/sys            # Virtual filesystem with kernel info
/dev            # Device files
/tmp            # Temporary files (cleared on reboot)
```

**Interview Context:** Docker images should be minimal, containing only necessary binaries from /bin, /lib, /usr.

---

### 4. Network Configuration
```bash
# View network interfaces
ip addr show
ifconfig

# View routes
ip route show
route -n

# View open ports
ss -tlnp                # Listening TCP ports with processes
netstat -tlnp          # Alternative

# Network namespaces (used by K8s)
ip netns list          # List network namespaces
ip netns exec ns1 bash # Run bash in namespace ns1
```

**Interview Context:** Kubernetes uses network namespaces and iptables for pod networking and service routing.

---

### 5. Systemd (Service Management)
```bash
# Systemd manages services on modern Linux
systemctl status docker        # Check service status
systemctl start docker         # Start service
systemctl enable docker        # Enable on boot
systemctl restart docker       # Restart

# Service files located in:
/etc/systemd/system/           # System services
~/.config/systemd/user/        # User services

# Example service file:
[Unit]
Description=My App
After=network.target

[Service]
Type=simple
User=appuser
ExecStart=/usr/bin/python /app/main.py
Restart=always

[Install]
WantedBy=multi-user.target
```

**Interview Context:** Kubernetes kubelet runs as systemd service, Docker daemon as systemd service.

---

### 6. Logging
```bash
# Kernel logs
dmesg                          # Kernel ring buffer

# System logs (systemd)
journalctl -u docker          # Logs for docker service
journalctl -f                 # Follow logs (tail -f)

# Application logs
/var/log/                      # Traditional log location

# Container logs (Docker stores them)
docker logs <container-id>     # View container stdout
```

**Interview Context:** K8s kubectl logs retrieves logs from container runtime, which gets them from container's stdout/stderr.

---

## Part 10: Interview Q&A

### Q1: Explain the relationship between Linux namespaces and Docker containers
**Answer:** 
Docker containers are implemented using Linux namespaces for isolation. When you run `docker run`, Docker uses 6 Linux namespaces:
- PID: Process isolation (container's PID 1 ≠ host's PID 1)
- Network: Network isolation (own IP, routing table)
- Mount: Filesystem isolation (own rootfs)
- UTS: Hostname isolation
- IPC: Inter-process communication isolation
- User: User ID remapping for security

Combined with cgroups (resource limits), this creates the container.

---

### Q2: What is the difference between a container and a VM from Linux perspective?
**Answer:**
```
VM (Virtual Machine):
- Runs full Linux kernel (hypervisor emulates hardware)
- Size: 1-2 GB
- Boot time: 30-60 seconds
- Process isolation: Hypervisor level
- Overhead: High (separate OS)

Container:
- Shares host Linux kernel (no emulation)
- Size: 10-100 MB (just libraries + app)
- Boot time: <1 second
- Process isolation: Linux namespace level
- Overhead: Low (no separate OS)
```

---

### Q3: How does Kubernetes networking work at Linux level?
**Answer:**
Kubernetes uses:
1. **Linux namespaces**: Each pod has network namespace
2. **Virtual Ethernet devices (veth)**: Connect pod namespace to bridge
3. **Linux bridges**: Connect containers on same node
4. **iptables rules**: Route traffic between services and pods
5. **Linux routing**: Route traffic between nodes

```
Pod A (10.0.1.5) --veth-- linux bridge (br0) --veth-- Pod B (10.0.1.6)
                                  |
                             iptables rules
                                  |
                      Service VIP (10.1.0.1:80)
```

---

### Q4: What happens when Docker container crashes?
**Answer:**
1. **Container process dies** (Linux process exits)
2. **PID namespace is destroyed**
3. **cgroups are cleaned up**
4. **Mount namespace released**
5. **Network namespace disconnected**
6. **Exit code stored by Docker daemon**
7. **`docker ps -a` shows exited container**
8. **Kubernetes detects pod crash (via health check)**
9. **Restarts container (creates new namespaces)**

---

### Q5: Explain Linux capabilities in containerized environment
**Answer:**
Linux has fine-grained capabilities instead of just "root or not":

```bash
# Docker drops these by default:
CAP_SYS_MODULE          # Load kernel modules
CAP_SYS_ADMIN           # Sysadmin operations
CAP_SYS_TIME            # Set system time
CAP_NET_ADMIN           # Network configuration
CAP_SYS_PTRACE          # Debug other processes

# Docker keeps these:
CAP_NET_BIND_SERVICE    # Bind to <1024 ports
CAP_CHOWN               # Change ownership
CAP_DAC_OVERRIDE        # Ignore permissions

# Security: Even if container is compromised,
# attacker can't load kernel modules or debug other processes
```

---

### Q6: How does Terraform provision Linux instances?
**Answer:**
Terraform acts as a bridge between your code and cloud APIs:

```
Terraform Code (HCL)
    ↓
Terraform reads provider (AWS, GCP, Azure)
    ↓
Calls cloud API to provision VM
    ↓
Cloud provider creates Linux instance with specified image (AMI)
    ↓
Linux kernel boots on VM
    ↓
User data script (bash) runs
    ↓
Software installed (Docker, K8s, etc.)
```

Key point: Terraform doesn't install Linux, it orchestrates cloud APIs that provision Linux instances.

---

### Q7: How does Ansible communicate with Linux systems?
**Answer:**
1. **Agentless**: No software needed on target (unlike Puppet, Chef)
2. **SSH**: Connects to each Linux host via SSH
3. **Python**: Ansible pushes Python scripts (even if target doesn't have Ansible installed)
4. **Remote Execution**: Scripts run with SSH, output returned
5. **Idempotent Modules**: Running twice = same result (Linux state unchanged)

```bash
Ansible Controller
    |
    |-- SSH to host1
    |       |-- Execute module (Python script)
    |       |-- Return result
    |
    |-- SSH to host2
    |       |-- Execute module
    |       |-- Return result
    |
    |-- SSH to host3
            |-- Execute module
            |-- Return result
```

---

### Q8: Explain container orchestration requirement from Linux perspective
**Answer:**
Containers are just Linux processes. Orchestration is needed because:

1. **Process Management**: Keep containers running (restart if crashed)
2. **Scheduling**: Place containers on nodes with resources
3. **Networking**: Connect containers across nodes (not just on same Linux host)
4. **Storage**: Persistent volumes across node reboots
5. **Updates**: Rolling updates without stopping services
6. **Monitoring**: Track container health

Without orchestration, managing 1000 Linux containers across 10 nodes would be impossible.

---

### Q9: What is the Linux memory cgroup and how does K8s use it?
**Answer:**
```bash
# Linux cgroup limits memory per process group:
cgroup.memory.limit_in_bytes = 512M

# Kubernetes pod YAML:
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    resources:
      limits:
        memory: "512Mi"        # Sets cgroup.memory.limit_in_bytes
      requests:
        memory: "256Mi"        # Used by scheduler

# If container exceeds 512Mi:
# 1. Kernel OOM killer triggers
# 2. Sends SIGKILL to container process
# 3. Container terminates
# 4. Kubernetes detects termination
# 5. Restarts container (creates new memory cgroup)
```

---

### Q10: How does pod-to-pod communication work at Linux level?
**Answer:**

```
Pod A (10.0.1.5) -----> Pod B (10.0.1.10) [Same Node]
    |
    |-- Packet in network namespace A
    |-- veth0 sends to linux bridge (br0)
    |-- linux bridge routes to veth1
    |-- veth1 receives in network namespace B
    |-- Pod B receives packet


Pod A (10.0.1.5) -----> Pod C (10.0.2.5) [Different Node]
    |
    |-- Packet in network namespace A
    |-- veth0 sends to linux bridge (br0)
    |-- Linux bridge checks routing table
    |-- Route says: destination 10.0.2.0/24 via 192.168.1.10 (other node)
    |-- Packet sent to other node via physical network
    |-- Other node's linux bridge receives
    |-- Delivered to Pod C's veth

# All routing done via:
# - Linux routing table (ip route)
# - Linux iptables rules
# - Linux bridge forwarding
```

---

## Part 11: Linux Performance Tuning for Containers

### Kernel Parameters for Container Performance

```bash
# File: /etc/sysctl.conf

# Network tuning
net.core.somaxconn=65535              # Max listen backlog
net.ipv4.tcp_max_syn_backlog=65535    # Max SYN backlog
net.ipv4.tcp_tw_reuse=1               # Reuse TIME_WAIT sockets
net.ipv4.ip_local_port_range=1024 65535

# File descriptor limits
fs.file-max=2097152                   # System-wide limit
fs.nr_open=2097152                    # Per-process limit

# Memory
vm.max_map_count=262144               # For Elasticsearch
vm.swappiness=0                       # Prefer page cache

# Apply changes:
sysctl -p
```

**Interview Context:** These kernel parameters are critical for container performance. DevOps engineers tune these when provisioning nodes.

---

## Part 12: Security: Linux Capabilities vs SELinux vs AppArmor

### Linux Security Modules

```bash
# 1. DAC (Discretionary Access Control) - Basic permissions
ls -la /app
-rwxr-xr-x 1 appuser appgroup
# Owner, group, others permissions

# 2. Linux Capabilities - Fine-grained permissions
getcap /usr/bin/ping
/usr/bin/ping = cap_net_raw+ep
# Can send raw packets, no other permissions

# 3. SELinux - Mandatory Access Control (Red Hat-based)
getenforce
# Enforcing / Permissive / Disabled

# 4. AppArmor - Profile-based (Debian-based)
aa-status
# Profiles list
```

**Container Security Layering:**
```
Docker capability dropping (e.g., --cap-drop=ALL)
    ↓
Linux DAC (user:group permissions)
    ↓
SELinux or AppArmor policy
    ↓
Network policies (iptables)
    ↓
= Very restricted container
```

---

## Part 13: Production Checklist - Linux Foundation

### Before running containers in production:

```
☐ Linux Kernel updated (security patches)
☐ Container runtime updated (Docker/containerd)
☐ Kernel parameters tuned (sysctl)
☐ File descriptors limit increased
☐ Memory and CPU cgroups configured
☐ Linux firewall (iptables/firewalld) configured
☐ SELinux or AppArmor enabled (if applicable)
☐ Monitoring for Linux metrics (memory, CPU, disk)
☐ Log rotation configured
☐ Linux user/group permissions correctly set
☐ SSH hardening (keys only, no passwords)
☐ Linux audit logging enabled
☐ Regular Linux security updates scheduled
```

---

## Summary Table: DevOps Tools & Linux Relationship

| Tool | Linux Dependency | How It Uses Linux |
|------|-----------------|-------------------|
| **Docker** | Critical | Namespaces, cgroups, Union FS, containers are Linux processes |
| **Kubernetes** | Critical | Runs on Linux nodes, manages Linux containers, uses iptables/networking |
| **Terraform** | Indirect | Provisions Linux instances, runs on Linux controller |
| **Ansible** | Critical | SSH to Linux hosts, executes Linux commands, manages Linux services |
| **CI/CD** | Critical | Runs on Linux, builds containers, deploys to Linux nodes |

---

## Practice Exercises

### Exercise 1: Understand Namespaces
```bash
# Create your own namespace
sudo ip netns add myns
sudo ip netns exec myns bash
ifconfig              # Different network interfaces!

# Similar to what Docker does internally
```

### Exercise 2: Explore cgroups
```bash
# Find cgroup info for running container
CONTAINER_ID=$(docker ps -q | head -1)
cat /sys/fs/cgroup/memory/docker/$CONTAINER_ID/memory.max_usage_in_bytes
# See actual memory usage
```

### Exercise 3: Track Linux Syscalls
```bash
# See syscalls made by container
strace -p <container_pid>
# Shows clone(), mount(), setns() etc.
```

### Exercise 4: Network Namespace Exercise
```bash
# Create two namespaces and connect them
sudo ip netns add ns1
sudo ip netns add ns2

# Create virtual ethernet pair
sudo ip link add veth1 type veth peer name veth1-peer
sudo ip link set veth1 netns ns1
sudo ip link set veth1-peer netns ns2

# Configure IPs
sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth1
sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth1-peer

# Now ns1 can ping ns2!
sudo ip netns exec ns1 ping 10.0.0.2
# This is how pod-to-pod communication works
```

---

## Final Interview Tip

When answering any DevOps tool question, always relate it back to Linux:

**Question:** "How does Docker isolate containers?"
**Bad Answer:** "Docker uses containers."
**Good Answer:** "Docker uses Linux namespaces (PID, network, mount, IPC, UTS, user) to provide process isolation and Linux cgroups to enforce resource limits. These are kernel features, not Docker inventions. A container is essentially a Linux process with restricted namespaces and cgroups."

**The interviewer knows:** Docker doesn't invent isolation, it orchestrates Linux kernel features.
