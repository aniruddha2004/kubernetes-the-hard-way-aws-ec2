# Appendix: AWS Infrastructure Specifications - Detailed Explanations

> Deep-dive explanations for AWS infrastructure decisions, networking concepts, security group logic, and cost optimization strategies.

---

## Table of Contents

- [Why AWS EC2?](#why-aws-ec2)
- [VPC Networking Basics](#vpc-networking-basics)
- [Security Groups Explained](#security-groups-explained)
- [Instance Types and Sizing](#instance-types-and-sizing)
- [Public vs Private IPs](#public-vs-private-ips)
- [Why Debian?](#why-debian)
- [NAT and Internet Access](#nat-and-internet-access)
- [Cost Optimization Strategies](#cost-optimization-strategies)
- [Security Best Practices](#security-best-practices)
- [Multi-AZ Considerations](#multi-az-considerations)

---

## Why AWS EC2?

### Cloud vs Bare Metal

| Aspect | Cloud (AWS) | Bare Metal |
|--------|-------------|------------|
| **Setup time** | Minutes | Days/weeks |
| **Cost model** | Pay per hour | Capital expense |
| **Scalability** | Instant | Physical limits |
| **Network** | Software-defined | Physical cabling |
| **Isolation** | Security groups | VLANs/firewalls |
| **Learning** | Easy to reset/recreate | Harder to experiment |

For learning Kubernetes internals, cloud VMs provide the perfect balance of realism and convenience.

### Why Not EKS?

Amazon EKS is a managed Kubernetes service. While excellent for production, it hides all the components we're trying to learn:
- Control plane is managed (you don't see API server, scheduler, etc.)
- Nodes are managed (you don't configure kubelet, CNI, etc.)
- Certificates are managed
- etcd is invisible

Kubernetes The Hard Way is about understanding what EKS abstracts away.

---

## VPC Networking Basics

### What is a VPC?

A **VPC (Virtual Private Cloud)** is your isolated network in AWS. It's like having your own datacenter network in the cloud.

```
AWS Cloud
|
+-- VPC (172.31.0.0/16)  <-- Your private network
    |
    +-- Subnet (172.31.16.0/20)  <-- Availability zone subset
    |   |
    |   +-- jumpbox (172.31.16.36)
    |   +-- server (172.31.20.244)
    |   +-- node-1 (172.31.19.88)
    |   +-- node-2 (172.31.25.215)
    |
    +-- Internet Gateway  <-- Connects VPC to internet
    |
    +-- Route Table
        172.31.0.0/16 -> local
        0.0.0.0/0 -> Internet Gateway
```

### CIDR Blocks

| CIDR | Usable IPs | Description |
|------|-----------|-------------|
| 172.31.0.0/16 | 65,531 | VPC range |
| 172.31.16.0/20 | 4,091 | Subnet range |
| 172.31.16.0/28 | 14 | Very small subnet |
| 10.0.0.0/8 | 16,777,214 | Common private range |
| 192.168.0.0/16 | 65,534 | Common private range |

The `/16` means the first 16 bits are fixed. `172.31.x.x` is your network, and the last two octets are for hosts.

### Subnets and AZs

A subnet lives in exactly one Availability Zone. For this tutorial:
- All instances are in one subnet
- That subnet is in one AZ
- This simplifies networking but is not highly available

For production, you'd span multiple AZs:
```
VPC
|-- Subnet A (us-east-1a) 172.31.16.0/20
|-- Subnet B (us-east-1b) 172.32.16.0/20
|-- Subnet C (us-east-1c) 172.33.16.0/20
```

---

## Security Groups Explained

### Stateful Firewall

Security Groups are **stateful** — if you allow inbound, the return traffic is automatically allowed.

```
Inbound rule: Allow TCP 22 from 0.0.0.0/0
    |
    v
SSH SYN packet arrives
    |
    v
Security Group allows it
    |
    v
Server responds with SYN-ACK
    |
    v
Security Group AUTOMATICALLY allows return traffic
(No explicit outbound rule needed for the response!)
```

### Why All Traffic Between CP and Workers?

We allowed "All traffic" between control plane and workers for simplicity. In production, restrict to specific ports:

| Source | Destination | Port | Purpose |
|--------|-------------|------|---------|
| Workers | Control Plane | 6443 | Kubernetes API |
| Workers | Control Plane | 2379 | etcd client |
| Workers | Control Plane | 2380 | etcd peer |
| Control Plane | Workers | 10250 | Kubelet API |
| Control Plane | Workers | 10256 | Kube-proxy health |
| Workers | Workers | All | Pod-to-pod (CNI-dependent) |

### Security Group vs NACL

| Feature | Security Group | NACL |
|---------|---------------|------|
| Level | Instance (ENI) | Subnet |
| Stateful | Yes | No |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules | Ordered rules |
| Default | Deny all | Allow all |

Security Groups are generally preferred for instance-level control.

---

## Instance Types and Sizing

### t3 Burstable Instances

**t3** instances use CPU credits:
- Baseline CPU performance with ability to burst
- Credits accumulate when under baseline
- Credits consumed when over baseline
- Unlimited mode: can burst beyond credits (charges extra)

For our cluster:
- **t3.micro**: 2 vCPU, 1 GB RAM (sufficient for jumpbox)
- **t3.small**: 2 vCPU, 2 GB RAM (minimum for control plane and workers)

### Why Not Smaller?

| Component | Minimum RAM | Why |
|-----------|-------------|-----|
| kube-apiserver | ~512 MB | Loads all objects into memory |
| etcd | ~256 MB | Memory-mapped database |
| kubelet | ~256 MB | Watches many resources |
| containerd | ~256 MB | Image storage, runtime |
| OS overhead | ~512 MB | Kernel, systemd, etc. |
| **Total** | **~1.8 GB** | Per node minimum |

2 GB RAM leaves little headroom. For production workloads, use t3.medium (4 GB) or larger.

### Why Not Larger?

Larger instances cost more. For learning:
- t3.small is the sweet spot for cost/performance
- You can always resize if needed (stop, change type, start)

---

## Public vs Private IPs

### Public IP Assignment

| Instance | Public IP | Why |
|----------|-----------|-----|
| jumpbox | Yes | Admin access from internet |
| server | No | Control plane shouldn't be exposed |
| node-1 | No | Workers shouldn't be exposed |
| node-2 | No | Workers shouldn't be exposed |

### How Traffic Flows

```
Internet
    |
    | SSH to 32.199.185.61
    v
+---------------+
|   jumpbox     |
| 172.31.16.36  |
+---------------+
    |
    | SSH to 172.31.20.244 (private)
    v
+---------------+
|    server     |
| 172.31.20.244 |
+---------------+
```

The jumpbox is the **only** entry point. All cluster nodes are private.

### Elastic IPs vs Dynamic Public IPs

| Type | Behavior | Cost |
|------|----------|------|
| **Dynamic** | Changes on stop/start | Free |
| **Elastic** | Persistent, can remap | Charged when unattached |

For learning, dynamic is fine. For production jumpbox, use Elastic IP.

---

## Why Debian?

### Distribution Comparison

| Distro | User | Package Manager | Kubernetes Support |
|--------|------|----------------|-------------------|
| Debian | `admin` | apt | Excellent |
| Ubuntu | `ubuntu` | apt | Excellent |
| Amazon Linux | `ec2-user` | yum/dnf | Good |
| RHEL | `ec2-user` | yum/dnf | Good |
| Fedora | `fedora` | dnf | Good |

### Why Debian 12 Specifically?

1. **Stable base**: Conservative package versions, well-tested
2. **Small footprint**: Minimal pre-installed software
3. **Long support**: 5-year LTS support cycle
4. **Kubernetes docs**: Many upstream guides use Debian/Ubuntu
5. **Kernel**: Modern enough for all Kubernetes features

### Could I Use Ubuntu?

Absolutely. Ubuntu is Debian-based and uses the same package manager. Key differences:
- Default user: `ubuntu` instead of `admin`
- Some paths differ slightly
- Snap is installed (though we don't use it)

---

## NAT and Internet Access

### The NAT Problem

Our worker nodes have no public IPs. How do they reach the internet?

**Short answer**: They don't, by design.

```
Worker node (no public IP)
    |
    | "I need to apt update"
    v
Routing table: 0.0.0.0/0 -> ?
    |
    | No internet gateway route for private IPs
    v
Connection fails
```

### Solutions

**Option 1: NAT Gateway** (Production)
```
Private Subnet -> NAT Gateway -> Internet Gateway -> Internet
```
- Costs ~$0.045/hour
- Provides outbound internet for private instances
- Highly available (per AZ)

**Option 2: Jumpbox as Proxy** (This tutorial)
```
Worker -> apt download on jumpbox -> scp to worker -> dpkg -i
```
- Free (no extra AWS resources)
- Manual but educational
- Shows offline/air-gapped scenarios

**Option 3: VPC Endpoints** (AWS Services)
```
Private Subnet -> VPC Endpoint -> S3/ECR/etc.
```
- For AWS services only
- Doesn't help with general internet

---

## Cost Optimization Strategies

### Spot Instances

Spot instances use unused EC2 capacity at up to 90% discount:

```
On-demand price: $0.0208/hour (t3.small)
Spot price:      $0.0062/hour (70% savings)
```

**Trade-offs**:
- Can be interrupted with 2-minute warning
- Not suitable for control plane (needs stability)
- Great for stateless worker nodes

### Savings Plans

Commit to 1- or 3-year usage for discounts:

| Plan | Discount | Flexibility |
|------|----------|-------------|
| Compute Savings | ~30-40% | Any instance, any region |
| EC2 Instance Savings | ~40-50% | Specific instance family |
| Reserved Instances | ~40-60% | Specific instance type |

For a learning cluster, don't buy savings plans. Use them when you have steady production workloads.

### Stopping vs Terminating

| Action | EBS Volume | Cost | Data |
|--------|-----------|------|------|
| **Stop** | Kept | Only EBS storage ($0.10/GB/month) | Preserved |
| **Terminate** | Deleted (by default) | None | Lost |

For learning: **Stop** instances when not in use. **Terminate** when completely done.

---

## Security Best Practices

### Defense in Depth

```
Layer 1: Network ACLs (subnet level)
    |
Layer 2: Security Groups (instance level)
    |
Layer 3: Host firewall (iptables)
    |
Layer 4: Kubernetes RBAC
    |
Layer 5: Pod Security Standards
    |
Layer 6: Application authentication
```

### SSH Hardening

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

### Instance Metadata Protection

AWS provides instance metadata at `169.254.169.254`. Protect it:

```bash
# Block pod access to instance metadata
iptables -A OUTPUT -d 169.254.169.254 -m owner ! --uid-owner root -j DROP
```

Modern AMIs do this automatically with IMDSv2.

---

## Multi-AZ Considerations

### Why Multiple AZs?

Availability Zones are physically separate datacenters:
- Power independence
- Network independence
- Flood/fire independence

If one AZ fails, others keep running.

### For Kubernetes

| Component | Single AZ | Multi-AZ |
|-----------|-----------|----------|
| etcd | Single point of failure | 3 nodes across 3 AZs |
| API server | One instance | Load balanced across AZs |
| Workers | All in one AZ | Spread across AZs |
| Pod disruption | Entire cluster | Only pods in failed AZ |

### Network Complexity

Multi-AZ adds complexity:
- Cross-AZ traffic costs $0.01/GB
- Need subnet per AZ
- Pod CIDRs must not overlap across AZs
- Routes need to span AZs

For learning, single AZ is sufficient. For production, multi-AZ is mandatory.

---

## FAQ

### Q: Can I use my existing AWS account?

A: Yes, but create a separate VPC or use resource tags to avoid conflicts.

### Q: Will this incur charges?

A: Yes. Approximately $0.07/hour while running. Remember to terminate when done.

### Q: Can I use AWS Free Tier?

A: Partially. Free tier includes 750 hours of t2/t3.micro per month. Our jumpbox qualifies. Control plane and workers do not (they're t3.small).

### Q: Why not use Terraform from the start?

A: This tutorial focuses on Kubernetes internals, not IaC. Manual provisioning ensures you understand each component.

### Q: Can I run this locally with VirtualBox/Vagrant?

A: Yes! Replace EC2 instances with VMs. All Kubernetes steps remain the same. Networking is simpler (no security groups).

---

## Related Concepts

- **Step 1**: Uses the infrastructure described here
- **Security Groups**: Affect pod-to-pod communication if misconfigured
- **VPC Routes**: Critical for inter-node pod networking
- **Costs**: Remember to clean up resources when done learning
