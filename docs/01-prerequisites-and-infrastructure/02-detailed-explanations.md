# Step 1: Prerequisites & Infrastructure Setup - Detailed Explanations

> This document provides deep-dive explanations for every concept, decision, and command used in Step 1. Read this alongside the implementation steps to understand WHY we do what we do, not just HOW.

---

## Table of Contents

- [Why Do We Need a Jumpbox?](#why-do-we-need-a-jumpbox)
- [Why Download Binaries Instead of Using Package Managers?](#why-download-binaries-instead-of-using-package-managers)
- [Understanding Binary Organization](#understanding-binary-organization)
- [Why Do We Need SSH Access Between All Nodes?](#why-do-we-need-ssh-access-between-all-nodes)
- [What is a Machine Database?](#what-is-a-machine-database)
- [Why Are Hostnames and DNS Resolution Critical?](#why-are-hostnames-and-dns-resolution-critical)
- [Security Group Rules Explained](#security-group-rules-explained)
- [Component Overview](#component-overview)

---

## Why Do We Need a Jumpbox?

A **jumpbox** (also called a bastion host) is a dedicated machine that acts as the single point of administrative access to all other machines in a private network. Think of it as the "front door" to your cluster.

### The Problem: Private Networks

In our setup, the Kubernetes control plane (`server`) and worker nodes (`node-1`, `node-2`) do **not** have public IP addresses. They exist entirely within a private subnet. This is a security best practice — you don't want your cluster nodes directly exposed to the internet.

### The Solution: Jumpbox as Gateway

The jumpbox is the **only** machine with a public IP address. All administrative tasks flow through it:

```
Your Laptop (Internet)
       |
       | SSH (Port 22)
       v
  Jumpbox (Public IP: 32.199.185.61)
       |
       | Internal Network
       +-----> server (172.31.20.244)
       +-----> node-1 (172.31.19.88)
       +-----> node-2 (172.31.25.215)
```

### Benefits of Using a Jumpbox

1. **Security**: Only one entry point to monitor and secure
2. **Consistency**: All administrative tools live in one place
3. **Offline Capability**: We download all binaries to the jumpbox, then distribute them to nodes that may not have internet access
4. **Audit Trail**: All admin activity originates from a single host

### Real-World Analogy

Imagine a corporate office building. Employees (cluster nodes) work inside. Visitors (administrators) must first go through the security desk (jumpbox) before accessing any office. The security desk checks credentials and logs all entries.

---

## Why Download Binaries Instead of Using Package Managers?

You might wonder: "Why not just run `apt-get install kubelet` or `snap install kubectl`?"

### Reason 1: Version Control

Package managers typically install the **latest stable version**, which may not match the version used in this tutorial. Kubernetes evolves rapidly, and commands/configurations change between versions. By downloading specific binary versions, we ensure exact reproducibility.

### Reason 2: Offline Installation

Worker nodes in our setup may not have internet access. By downloading all binaries to the jumpbox first, we can distribute them via SCP even if nodes are completely air-gapped.

### Reason 3: Learning the Components

When you install via package manager, binaries are placed in standard locations (`/usr/bin/`, `/usr/local/bin/`), configs in `/etc/`, and services are auto-configured. This hides the complexity. By manually placing binaries, you learn:
- What each component actually is
- Where it needs to run (control plane vs worker)
- How to configure it from scratch

### Reason 4: No Hidden Dependencies

Package managers may install additional dependencies, init scripts, or configuration files that interfere with manual setup. Raw binaries give us complete control.

---

## Understanding Binary Organization

After downloading and extracting, we organize binaries into four categories. Here's why:

### Client Binaries (`downloads/client/`)

| Binary | Purpose | Used By |
|--------|---------|---------|
| `kubectl` | CLI to interact with Kubernetes API | Administrators, CI/CD |
| `etcdctl` | CLI to interact with etcd datastore | Administrators, backups |

These are **client-side tools** — they don't run as services. They're used by humans or automation to manage the cluster.

### Control Plane Binaries (`downloads/controller/`)

| Binary | Purpose | Runs On |
|--------|---------|---------|
| `kube-apiserver` | Front door to Kubernetes; validates and processes all API requests | server |
| `kube-controller-manager` | Runs controllers that maintain desired cluster state | server |
| `kube-scheduler` | Assigns pods to nodes based on resource requirements | server |
| `etcd` | Distributed key-value store for all cluster data | server |

These are the **brain** of Kubernetes. They make decisions, store state, and coordinate the cluster. They run exclusively on the control plane node.

### Worker Binaries (`downloads/worker/`)

| Binary | Purpose | Runs On |
|--------|---------|---------|
| `kubelet` | Agent that manages containers on a node | node-1, node-2 |
| `kube-proxy` | Manages network rules for Services | node-1, node-2 |
| `containerd` | Container runtime (runs containers) | node-1, node-2 |
| `runc` | Low-level container runtime (OCI standard) | node-1, node-2 |
| `crictl` | CLI for debugging container runtime | node-1, node-2 |
| `ctr` | containerd's native CLI | node-1, node-2 |

These are the **muscle** of Kubernetes. They actually run workloads, manage networking, and report status back to the control plane.

### CNI Plugins (`downloads/cni-plugins/`)

| Plugin | Purpose |
|--------|---------|
| `bridge` | Creates Linux bridges for pod networking |
| `host-local` | Manages IP allocation from local ranges |
| `loopback` | Configures loopback interface in pods |
| `portmap` | Maps host ports to pod ports |
| `firewall` | Applies firewall rules |

CNI (Container Network Interface) plugins are **pluggable networking components** that Kubernetes calls to set up pod networking. They're not Kubernetes-specific — any container orchestrator can use them.

---

## Why Do We Need SSH Access Between All Nodes?

Throughout this guide, we use SSH to:
1. Copy files (binaries, certificates, configs) to nodes
2. Execute commands remotely
3. Verify node status and troubleshoot issues

### The SSH Config File

Instead of typing long commands like:
```bash
ssh -i ~/.ssh/ani-key.pem admin@172.31.20.244
```

We create a config file that maps short names to full connection details:
```bash
ssh server   # Expands to: ssh admin@172.31.20.244 -i ~/.ssh/ani-key.pem
ssh node-1   # Expands to: ssh admin@172.31.19.88 -i ~/.ssh/ani-key.pem
```

### Why chmod 600?

SSH requires strict permissions on config files:
- `~/.ssh/config` must be `600` (owner read/write only)
- `~/.ssh/ani-key.pem` must be `400` (owner read only)

This prevents other users on the system from reading your private key or connection details.

---

## What is a Machine Database?

`machines.txt` is a simple text file that serves as the **single source of truth** for all cluster nodes. It uses a space-delimited format:

```
<private-ip>  <fully-qualified-domain-name>  <short-hostname>  <pod-cidr>
```

### Why This Format?

1. **Machine-parseable**: Easy to read with `while read` loops in bash
2. **Human-readable**: You can glance at it and understand the topology
3. **Single source of truth**: Change an IP here, and all scripts that source this file get the update
4. **Extensible**: The fourth column (POD_CIDR) is used later by CNI configuration

### Why FQDN (Fully Qualified Domain Name)?

A Fully Qualified Domain Name includes the hostname AND domain:
- `server.kubernetes.local` — FQDN
- `server` — short hostname

We use FQDNs because Kubernetes components validate certificates using FQDNs. If we only used short hostnames, certificate validation would fail.

---

## Why Are Hostnames and DNS Resolution Critical?

This is one of the most important concepts in the entire setup. Getting this wrong will cause cryptic certificate errors later.

### The Problem: Certificate Validation

When Kubernetes components connect to each other, they use **TLS certificates** to prove identity. These certificates contain the hostname of the machine. During the TLS handshake:

1. Client connects to server
2. Server presents its certificate
3. Client checks: "Does the hostname I'm connecting to match the hostname in the certificate?"
4. If NO → Connection refused with error like: `x509: certificate is valid for server.kubernetes.local, not server.ani-kubernetes.local`

### What We Configure

1. **`/etc/hostname`** — The system's own hostname (set via `hostnamectl`)
2. **`/etc/hosts`** — Local DNS override (maps IPs to hostnames without needing a DNS server)
3. **Hostnames match certificates** — When we generate certificates in Step 2, we'll use these exact hostnames

### Why `/etc/hosts` and Not a Real DNS Server?

For a small lab cluster, setting up BIND or CoreDNS for internal resolution is overkill. `/etc/hosts` is:
- Simple to understand
- Zero dependency on external services
- Instant updates (no caching issues)
- Sufficient for a 4-node cluster

In production, you'd use an internal DNS server or cloud provider DNS.

### Hostname Resolution Flow

When a process tries to resolve `server.kubernetes.local`:

```
Application calls gethostbyname("server.kubernetes.local")
                     |
                     v
           Check /etc/hosts first
                     |
         Found? ----+----> Return IP immediately
            |                (Fast, no network)
            |
            v
    Query DNS server
    (if configured in /etc/resolv.conf)
            |
            v
    Return IP from DNS
```

By putting entries in `/etc/hosts`, we ensure fast, reliable resolution without depending on external DNS.

---

## Security Group Rules Explained

AWS Security Groups are **stateful firewalls** that control traffic at the instance level. Understanding the rules is critical for troubleshooting connectivity issues.

### Stateful vs Stateless

- **Stateful**: If you allow inbound traffic, the return traffic is automatically allowed
- **Stateless**: You must explicitly allow both inbound and outbound

AWS Security Groups are **stateful**, so you only need to define the initiating direction.

### Why These Specific Rules?

**Jump Server (Ingress: 22 from anywhere)**
- This is the entry point for administrators
- Port 22 = SSH
- From `0.0.0.0/0` = Any IP on the internet
- In production, you'd restrict this to your office IP

**Control Plane (Ingress: 6443 from Jump Server)**
- Port 6443 = Kubernetes API Server (HTTPS)
- Only the jumpbox should access the API directly
- Later, worker nodes will also need this (via separate rule)

**Worker Node (Ingress: 80 from Jump Server)**
- For testing deployed applications (NGINX smoke test in Step 8)

**Bidirectional "All Traffic" Between Control Plane and Workers**
- etcd cluster communication
- Kubelet reports to API Server
- Kubernetes proxy connections
- This simplifies things; in production you'd restrict to specific ports

### ICMP (Ping) Rules

ICMP is not TCP or UDP — it's a separate protocol used for:
- Ping (echo request/reply)
- Path MTU discovery
- Error reporting

Allowing ICMP helps with basic connectivity testing and troubleshooting.

---

## Component Overview

Here's what each downloaded component does in the broader Kubernetes ecosystem:

### Kubernetes Core (k8s.io)

| Component | Type | Role |
|-----------|------|------|
| `kube-apiserver` | Control Plane | Exposes the Kubernetes API; the front door to the cluster |
| `kube-controller-manager` | Control Plane | Runs background controllers (node lifecycle, replication, endpoints) |
| `kube-scheduler` | Control Plane | Watches for unscheduled pods and assigns them to nodes |
| `kubelet` | Worker | Agent on each node that manages pod lifecycle |
| `kube-proxy` | Worker | Maintains network rules for pod-to-service communication |
| `kubectl` | Client | CLI tool for interacting with the cluster |

### Container Runtime (OCI - Open Container Initiative)

| Component | Type | Role |
|-----------|------|------|
| `containerd` | Runtime | High-level container runtime; manages images, containers, storage |
| `runc` | Runtime | Low-level container runtime; creates/isolates Linux namespaces and cgroups |
| `crictl` | Client | CRI-compatible CLI for debugging container runtime |
| `ctr` | Client | containerd's native CLI (less Kubernetes-aware than crictl) |

### Networking (CNI - Container Network Interface)

| Component | Type | Role |
|-----------|------|------|
| `bridge` | CNI Plugin | Creates Linux bridges to connect pod network interfaces |
| `host-local` | CNI Plugin | Allocates IPs from a local range defined in CNI config |
| `loopback` | CNI Plugin | Brings up the loopback (lo) interface in each pod |

### Data Store

| Component | Type | Role |
|-----------|------|------|
| `etcd` | Data Store | Distributed, consistent key-value store; the "database" of Kubernetes |
| `etcdctl` | Client | CLI for backing up, restoring, and inspecting etcd |

---

## FAQ

### Q: Why Debian instead of Ubuntu or Amazon Linux?

A: Debian is the reference distribution for Kubernetes The Hard Way. The commands and paths in this guide assume a Debian-based system. Ubuntu would work similarly, but Amazon Linux uses different package names and paths.

### Q: Why t3.micro for the jumpbox but t3.small for others?

A: The jumpbox only runs SSH and distributes files — it needs minimal resources. Control plane and worker nodes run actual Kubernetes components and containers, requiring more CPU and RAM.

### Q: What if my worker nodes have internet access?

A: This guide assumes they might not. If they do, you could theoretically install packages directly, but downloading to the jumpbox and distributing is still a best practice for:
- Version consistency
- Faster reinstalls (no re-downloading)
- Air-gapped environment preparation

### Q: Can I use a different container runtime instead of containerd?

A: Yes. Kubernetes supports any CRI-compliant runtime. Other options include CRI-O and Docker (with cri-dockerd shim). However, containerd is the default and recommended runtime for Kubernetes.

### Q: What is CRI?

A: **Container Runtime Interface** — a standardized API that Kubernetes uses to communicate with container runtimes. It allows Kubernetes to work with any compliant runtime without code changes.

---

## Troubleshooting

### Issue: "Permission denied (publickey)" when SSHing

**Cause**: The `.pem` file doesn't have correct permissions, or the `admin` user doesn't exist.

**Solution**:
```bash
chmod 400 ani-key.pem
# Verify user exists:
ssh -i ani-key.pem admin@<ip> "whoami"
```

### Issue: "Could not resolve hostname server"

**Cause**: `/etc/hosts` hasn't been updated on the jumpbox.

**Solution**:
```bash
cat hosts | sudo tee -a /etc/hosts
```

### Issue: Downloads fail with 404

**Cause**: The URL in the downloads file may be outdated.

**Solution**: Check the official Kubernetes release page for updated URLs, or use `gsutil` to list available versions in the Google Cloud Storage bucket.

### Issue: "kubectl version" shows connection refused

**Cause**: kubectl without `--client` tries to connect to a Kubernetes API server. Since we haven't started one yet, it fails.

**Solution**: Use `kubectl version --client` until the control plane is running.

---

## Related Concepts for Future Steps

- **TLS Certificates**: Generated in Step 2, used everywhere for mutual authentication
- **Kubeconfigs**: Created in Step 3, define how clients connect to the API server
- **etcd**: Bootstrapped in Step 4, stores all cluster state
- **Pod CIDR**: Assigned in Step 6, defines IP ranges for pods on each worker
