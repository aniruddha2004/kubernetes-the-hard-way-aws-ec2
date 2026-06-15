# Step 1: Prerequisites & Infrastructure Setup - Implementation

> This guide covers the initial setup of a Kubernetes cluster following the philosophy of Kubernetes The Hard Way, adapted for AWS EC2 instances. Unlike the original guide, this setup uses AWS EC2 instances, the default `admin` user instead of `root`, a pre-existing AWS `.pem` keypair instead of generating and distributing SSH keys, and worker nodes named `node-1` and `node-2`.

---

## Table of Contents

- [Overview](#overview)
- [1. Infrastructure Setup](#1-infrastructure-setup)
- [2. Jumpbox Setup](#2-jumpbox-setup)
- [3. Download Kubernetes Binaries](#3-download-kubernetes-binaries)
- [4. Extract and Organize Binaries](#4-extract-and-organize-binaries)
- [5. Install kubectl](#5-install-kubectl)
- [6. Create Machine Database](#6-create-machine-database)
- [7. Configure SSH Access](#7-configure-ssh-access)
- [8. Configure Hostnames](#8-configure-hostnames)
- [9. Configure Hostname Resolution](#9-configure-hostname-resolution)
- [10. Summary](#10-summary)

---

## Overview

Before bootstrapping Kubernetes, we must:
1. Provision and verify all EC2 instances are running and accessible
2. Prepare the jumpbox as the central administration host
3. Download all Kubernetes component binaries for offline distribution
4. Establish passwordless SSH access between all nodes
5. Configure hostnames and DNS resolution for certificate validation
6. Organize binaries into categorized directories for later use

### Cluster Nodes

| Name | Purpose | Instance Type | Private IP | Public IP |
|------|---------|---------------|------------|-----------|
| `jumpbox` | Administration host | t3.micro | <JUMPBOX_PRIVATE_IP> | <JUMPBOX_PUBLIC_IP> |
| `server` | Kubernetes control plane | t3.small | <SERVER_PRIVATE_IP> | None |
| `node-1` | Worker node 1 | t3.small | <NODE_1_PRIVATE_IP> | None |
| `node-2` | Worker node 2 | t3.small | <NODE_2_PRIVATE_IP> | None |

> **Important**: All instances must be in the same VPC and subnet with private connectivity between all nodes. Worker nodes do not have public IPs.

### Component Versions

| Component | Version |
|-----------|---------|
| Kubernetes | v1.32.3 |
| containerd | v2.1.0-beta.0 |
| CNI plugins | v1.6.2 |
| etcd | v3.6.0-rc.3 |
| crictl | v1.32.0 |
| runc | v1.3.0-rc.1 |

---

## 1. Infrastructure Setup

Provision four Debian-based EC2 instances with the following specifications:

| Specification | Jumpbox | Control Plane | Worker Nodes |
|--------------|---------|---------------|--------------|
| **Instance Type** | t3.micro | t3.small | t3.small |
| **vCPU** | 2 | 2 | 2 |
| **RAM** | 1 GB | 2 GB | 2 GB |
| **OS** | Debian 12 (Bookworm) | Debian 12 | Debian 12 |
| **Architecture** | x86_64 | x86_64 | x86_64 |
| **AMI** | ami-0b75f821522bcff85 | ami-0b75f821522bcff85 | ami-0b75f821522bcff85 |
| **Key Pair** | <YOUR_KEY_PAIR> | <YOUR_KEY_PAIR> | <YOUR_KEY_PAIR> |
| **Public IP** | Yes | No | No |

### Security Group Requirements

**Jump Server SG** (`sg-0053e3f4589ec3ad5`):
- Ingress: TCP 22 from 0.0.0.0/0 (SSH from laptop)
- Egress: All traffic to 0.0.0.0/0

**Control Plane SG** (`sg-0945b3734714ba7e2`):
- Ingress: TCP 22 from Jump Server SG (SSH from jumpbox)
- Ingress: TCP 6443 from Jump Server SG (Kubernetes API)
- Ingress: ICMP All from Jump Server SG (Ping)
- Ingress: All from Worker Node SG (All traffic from workers)
- Egress: All to Worker Node SG (All traffic to workers)

**Worker Node SG** (`sg-009e0dcf3f5a70fa9`):
- Ingress: TCP 22 from Jump Server SG (SSH from jumpbox)
- Ingress: TCP 80 from Jump Server SG (HTTP from jumpbox)
- Ingress: ICMP All from Jump Server SG (Ping)
- Ingress: All from Control Plane SG (All from control plane)
- Egress: All to Control Plane SG (All to control plane)

Refer to `../appendix-aws-infrastructure/` for exact AWS resource specifications.

---

## 2. Jumpbox Setup

### SSH into the Jumpbox

From your local machine:

```bash
ssh -i <YOUR_KEY_PAIR>.pem admin@<JUMPBOX_PUBLIC_IP>
```

Expected output:
```
Linux ip-172-31-16-36 6.12.74+deb13+1-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.74-2 (2026-03-08) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jun 14 08:40:58 2026 from 103.214.63.187
admin@ip-172-31-16-36:~$
```

### Install Required Utilities

Update package lists and install essential tools:

```bash
sudo apt-get update
sudo apt-get -y install wget curl vim openssl git
```

Purpose of each tool:
- `wget` / `curl` — Downloading component binaries
- `openssl` — Generating TLS certificates and Certificate Authority
- `vim` — Editing configuration files
- `git` — Cloning the upstream Kubernetes The Hard Way repository

---

## 3. Download Kubernetes Binaries

### Clone the Repository

```bash
git clone --depth 1 \
  https://github.com/kelseyhightower/kubernetes-the-hard-way.git
cd kubernetes-the-hard-way
```

Expected output:
```
Cloning into 'kubernetes-the-hard-way'...
remote: Enumerating objects: 41, done.
remote: Counting objects: 100% (41/41), done.
remote: Compressing objects: 100% (40/40), done.
remote: Total 41 (delta 2), reused 18 (delta 1), pack-reused 0 (from 0)
Receiving objects: 100% (41/41), 29.37 KiB | 14.69 MiB/s, done.
Resolving deltas: 100% (2/2), done.
```

Verify repository contents:
```bash
ls
```

Output:
```
CONTRIBUTING.md  COPYRIGHT.md  LICENSE  README.md  ca.conf  configs  docs
 downloads-amd64.txt  downloads-arm64.txt  units
```

### View Binaries to Download

```bash
cat downloads-$(dpkg --print-architecture).txt
```

Output:
```
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kubectl
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-apiserver
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-controller-manager
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-scheduler
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-proxy
https://dl.k8s.io/v1.32.3/bin/linux/amd64/kubelet
https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-linux-amd64.tar.gz
https://github.com/opencontainers/runc/releases/download/v1.3.0-rc.1/runc.amd64
https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz
https://github.com/containerd/containerd/releases/download/v2.1.0-beta.0/containerd-2.1.0-beta.0-linux-amd64.tar.gz
https://github.com/etcd-io/etcd/releases/download/v3.6.0-rc.3/etcd-v3.6.0-rc.3-linux-amd64.tar.gz
```

### Download All Binaries

```bash
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-$(dpkg --print-architecture).txt
```

Expected output (all binaries downloaded):
```
kubectl                                        100%[========================================>]  54.67M   298MB/s    in 0.2s
kube-apiserver                                 100%[========================================>]  88.94M  6.96MB/s    in 14s
kube-controller-manager                        100%[========================================>]  82.00M  76.4MB/s    in 1.1s
kube-scheduler                                 100%[========================================>]  62.79M   154MB/s    in 0.4s
kube-proxy                                     100%[========================================>]  63.75M   106MB/s    in 0.6s
kubelet                                        100%[========================================>]  73.82M  23.5MB/s    in 3.1s
crictl-v1.32.0-linux-amd64.tar.gz              100%[========================================>]  18.21M  --.-KB/s    in 0.05s
runc.amd64                                     100%[========================================>]  11.30M  --.-KB/s    in 0.03s
cni-plugins-linux-amd64-v1.6.2.tgz             100%[========================================>]  50.35M   291MB/s    in 0.2s
containerd-2.1.0-beta.0-linux-amd64.tar.gz     100%[========================================>]  37.01M   169MB/s    in 0.2s
etcd-v3.6.0-rc.3-linux-amd64.tar.gz            100%[========================================>]  22.48M  --.-KB/s    in 0.07s
```

---

## 4. Extract and Organize Binaries

Create categorized directories and extract archives:

```bash
{
  ARCH=$(dpkg --print-architecture)
  mkdir -p downloads/{client,cni-plugins,controller,worker}

  tar -xvf downloads/crictl-v1.32.0-linux-${ARCH}.tar.gz \
    -C downloads/worker/

  tar -xvf downloads/containerd-2.1.0-beta.0-linux-${ARCH}.tar.gz \
    --strip-components 1 \
    -C downloads/worker/

  tar -xvf downloads/cni-plugins-linux-amd64-v1.6.2.tgz \
    -C downloads/cni-plugins/

  tar -xvf downloads/etcd-v3.6.0-rc.3-linux-${ARCH}.tar.gz \
    -C downloads/ \
    --strip-components 1 \
    etcd-v3.6.0-rc.3-linux-${ARCH}/etcdctl \
    etcd-v3.6.0-rc.3-linux-${ARCH}/etcd

  mv downloads/{etcdctl,kubectl} downloads/client/

  mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \
    downloads/controller/

  mv downloads/{kubelet,kube-proxy} downloads/worker/

  mv downloads/runc.${ARCH} downloads/worker/runc
}
```

Clean up archive files:
```bash
rm -rf downloads/*gz
```

Make all binaries executable:
```bash
chmod +x downloads/{client,cni-plugins,controller,worker}/*
```

Verify the organized structure:
```bash
ls -oh downloads/
```

Output:
```
total 566M
drwxrwxr-x 2 admin admin 4.0K Jun 14 10:00 client
drwxrwxr-x 2 admin admin 4.0K Jun 14 10:00 cni-plugins
drwxrwxr-x 2 admin admin 4.0K Jun 14 10:00 controller
drwxrwxr-x 2 admin admin 4.0K Jun 14 10:00 worker
```

Binary organization:
- `downloads/client/` — `kubectl`, `etcdctl`
- `downloads/controller/` — `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd`
- `downloads/worker/` — `kubelet`, `kube-proxy`, `containerd`, `runc`, `crictl`, `ctr`
- `downloads/cni-plugins/` — `bridge`, `host-local`, `loopback`, `portmap`, etc.

---

## 5. Install kubectl

Install kubectl on the jumpbox for cluster management:

```bash
sudo cp downloads/client/kubectl /usr/local/bin/
```

> **Note**: `sudo` is required because `/usr/local/bin/` requires root privileges on Debian.

Verify installation:
```bash
kubectl version --client
```

Expected output:
```
Client Version: v1.32.3
Kustomize Version: v5.5.0
```

> **Note**: `kubectl version` (without `--client`) shows `The connection to the server localhost:8080 was refused` — this is normal. We have not configured a cluster yet.

---

## 6. Create Machine Database

Create `machines.txt` as the single source of truth for all cluster nodes:

```bash
cat > machines.txt <<EOF
<SERVER_PRIVATE_IP> server.kubernetes.local server
<NODE_1_PRIVATE_IP> node-1.kubernetes.local node-1 10.200.0.0/24
<NODE_2_PRIVATE_IP> node-2.kubernetes.local node-2 10.200.1.0/24
EOF
```

Format:
```
<private-ip>  <fqdn>                    <hostname> <pod-subnet>
```

The **fourth column** (POD_SUBNET) is used later when assigning Pod CIDR ranges to worker nodes via CNI.

> **Replace these IP addresses with your actual instance private IPs.** Do not copy these exact IPs unless your instances have the same addresses.

---

## 7. Configure SSH Access

### Copy AWS Key to Jumpbox

From your local machine, copy the `.pem` key:

```bash
scp -i <YOUR_KEY_PAIR>.pem <YOUR_KEY_PAIR>.pem admin@<JUMPBOX_PUBLIC_IP>:~/kubernetes-the-hard-way/
```

On the jumpbox, set restrictive permissions:
```bash
chmod 400 ~/kubernetes-the-hard-way/<YOUR_KEY_PAIR>.pem
```

### Create SSH Config File

Create `~/.ssh/config` for easy passwordless SSH:

```bash
cat > ~/.ssh/config <<EOF
Host server
    HostName <SERVER_PRIVATE_IP>
    User admin
    IdentityFile ~/.ssh/<YOUR_KEY_PAIR>.pem

Host node-1
    HostName <NODE_1_PRIVATE_IP>
    User admin
    IdentityFile ~/.ssh/<YOUR_KEY_PAIR>.pem

Host node-2
    HostName <NODE_2_PRIVATE_IP>
    User admin
    IdentityFile ~/.ssh/<YOUR_KEY_PAIR>.pem
EOF
```

Set correct permissions:
```bash
chmod 600 ~/.ssh/config
```

### Verify Connectivity

Test SSH to each node:

```bash
ssh server
# exit

ssh node-1
# exit

ssh node-2
# exit
```

On first connection, accept the host key:
```
The authenticity of host '<SERVER_PRIVATE_IP> (<SERVER_PRIVATE_IP>)' can't be established.
ED25519 key fingerprint is SHA256:9sc/csAnZT3VAQd+fMDbmf4H2stfkTU9oslQZNBUNh4.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
```

---

## 8. Configure Hostnames

Set the hostname of each machine using a scripted loop:

```bash
while read IP FQDN HOST SUBNET; do
    CMD="sudo sed -i 's/^127.0.1.1.*/127.0.1.1\\t${FQDN} ${HOST}/' /etc/hosts"
    ssh -n ${HOST} "$CMD"
    ssh -n ${HOST} "sudo hostnamectl set-hostname ${HOST}"
    ssh -n ${HOST} "sudo systemctl restart systemd-hostnamed"
done < machines.txt
```

Verify hostnames:

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n ${HOST} "hostname"
done < machines.txt
```

Expected output:
```
server
node-1
node-2
```

Also verify fully-qualified domain names:
```bash
while read IP FQDN HOST SUBNET; do
  ssh -n ${HOST} "hostname --fqdn"
done < machines.txt
```

---

## 9. Configure Hostname Resolution

### Generate Common Hosts File

Create a hosts file with all cluster entries:

```bash
echo "" > hosts
echo "# Kubernetes The Hard Way" >> hosts

while read IP FQDN HOST SUBNET; do
    ENTRY="${IP} ${FQDN} ${HOST}"
    echo "$ENTRY" >> hosts
done < machines.txt
```

Verify:
```bash
cat hosts
```

Output:
```

# Kubernetes The Hard Way
<SERVER_PRIVATE_IP> server.kubernetes.local server
<NODE_1_PRIVATE_IP> node-1.kubernetes.local node-1
<NODE_2_PRIVATE_IP> node-2.kubernetes.local node-2
```

### Distribute to Jumpbox

```bash
sudo bash -c "cat hosts >> /etc/hosts"
```

Verify:
```bash
tail -n 5 /etc/hosts
```

### Distribute to All Cluster Nodes

Copy to each node then append to `/etc/hosts`:

```bash
while read IP FQDN HOST SUBNET; do
  scp hosts ${HOST}:~/
done < machines.txt
```

```bash
while read IP FQDN HOST SUBNET; do
  ssh -n ${HOST} "cat /home/admin/hosts | sudo tee -a /etc/hosts > /dev/null"
done < machines.txt
```

### Verify Resolution

Test connectivity between nodes:

```bash
ping -c 1 server
ping -c 1 node-1
ping -c 1 node-2
```

Expected:
```
64 bytes from server.kubernetes.local (<SERVER_PRIVATE_IP>): icmp_seq=1 ttl=64 time=0.239 ms
```

Test from server to workers:
```bash
ssh server "ping -c 1 node-1"
```

Expected:
```
64 bytes from node-1.kubernetes.local (<NODE_1_PRIVATE_IP>): icmp_seq=1 ttl=64 time=0.202 ms
```

---

## 10. Summary

At the end of Step 1, confirm all checks pass:

| Check | Command | Expected Result |
|-------|---------|----------------|
| kubectl installed | `kubectl version --client` | `v1.32.3` |
| Binaries downloaded | `ls downloads/` | 4 subdirectories with binaries |
| SSH to server | `ssh server 'hostname'` | `server` |
| SSH to node-1 | `ssh node-1 'hostname'` | `node-1` |
| SSH to node-2 | `ssh node-2 'hostname'` | `node-2` |
| Hosts distributed | `ssh node-1 'cat /etc/hosts'` | Contains cluster entries |
| DNS works | `ping -c 1 server` | ICMP reply |

**Next**: Proceed to [Step 2: PKI and Certificate Authority](../02-pki-and-certificate-authority/02-implementation-steps.md) to create the cryptographic identity infrastructure.
