# Step 3: Kubeconfigs and Authentication - Implementation

> This guide creates the kubeconfig files that tell Kubernetes components (and kubectl) how to authenticate and connect to the API server using the certificates generated in Step 2.

---

## Table of Contents

- [Overview](#overview)
- [1. Set Cluster Configuration](#1-set-cluster-configuration)
- [2. Generate Kubelet Kubeconfigs](#2-generate-kubelet-kubeconfigs)
- [3. Generate Kube-Proxy Kubeconfig](#3-generate-kube-proxy-kubeconfig)
- [4. Generate Kube-Controller-Manager Kubeconfig](#4-generate-kube-controller-manager-kubeconfig)
- [5. Generate Kube-Scheduler Kubeconfig](#5-generate-kube-scheduler-kubeconfig)
- [6. Generate Admin Kubeconfig](#6-generate-admin-kubeconfig)
- [7. Distribute Kubeconfigs](#7-distribute-kubeconfigs)
- [8. Verification](#8-verification)
- [9. Summary](#9-summary)

---

## Overview

A **kubeconfig** is a YAML file that answers four questions:
1. **Which cluster?** — API server address and CA certificate
2. **Who are you?** — Client certificate and key for authentication
3. **Which context?** — Combines cluster + user for a specific environment
4. **What is current?** — Which context to use by default

### Components Requiring Kubeconfigs

| Component | Kubeconfig Name | Identity (from Step 2) |
|-----------|----------------|------------------------|
| kubelet (node-1) | node-1.kubeconfig | node-1.crt, node-1.key |
| kubelet (node-2) | node-2.kubeconfig | node-2.crt, node-2.key |
| kube-proxy | kube-proxy.kubeconfig | kube-proxy.crt, kube-proxy.key |
| kube-controller-manager | kube-controller-manager.kubeconfig | kube-controller-manager.crt |
| kube-scheduler | kube-scheduler.kubeconfig | kube-scheduler.crt |
| admin (kubectl) | admin.kubeconfig | admin.crt, admin.key |

### Cluster Details

- **API Server**: `https://server.kubernetes.local:6443`
- **CA Certificate**: `ca/ca.crt`
- **Server Name**: Must match SAN in API server certificate (`server.kubernetes.local`)

---

## 1. Set Cluster Configuration

Each kubeconfig starts with the cluster definition:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca/ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=<component>.kubeconfig
```

Parameters:
- `set-cluster kubernetes-the-hard-way` — Names the cluster entry
- `--certificate-authority=ca/ca.crt` — CA certificate to validate API server's cert
- `--embed-certs=true` — Embed the CA cert directly into the kubeconfig (self-contained file)
- `--server=https://server.kubernetes.local:6443` — API server endpoint
- `--kubeconfig=<component>.kubeconfig` — Output file name

> **Repeat this for each kubeconfig** or combine with the user configuration below.

---

## 2. Generate Kubelet Kubeconfigs

### For node-1

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca/ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=node-1.kubeconfig

kubectl config set-credentials system:node:node-1 \
  --client-certificate=kubelet-certs/node-1.crt \
  --client-key=kubelet-certs/node-1.key \
  --embed-certs=true \
  --kubeconfig=node-1.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:node-1 \
  --kubeconfig=node-1.kubeconfig

kubectl config use-context default --kubeconfig=node-1.kubeconfig
```

### For node-2

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca/ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=node-2.kubeconfig

kubectl config set-credentials system:node:node-2 \
  --client-certificate=kubelet-certs/node-2.crt \
  --client-key=kubelet-certs/node-2.key \
  --embed-certs=true \
  --kubeconfig=node-2.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:node:node-2 \
  --kubeconfig=node-2.kubeconfig

kubectl config use-context default --kubeconfig=node-2.kubeconfig
```

---

## 3. Generate Kube-Proxy Kubeconfig

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca/ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
  --client-certificate=kube-certs/kube-proxy.crt \
  --client-key=kube-certs/kube-proxy.key \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

---

## 4. Generate Kube-Controller-Manager Kubeconfig

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca/ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-certs/kube-controller-manager.crt \
  --client-key=kube-certs/kube-controller-manager.key \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

---

## 5. Generate Kube-Scheduler Kubeconfig

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca/ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-certs/kube-scheduler.crt \
  --client-key=kube-certs/kube-scheduler.key \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

---

## 6. Generate Admin Kubeconfig

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca/ca.crt \
  --embed-certs=true \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=admin.kubeconfig

kubectl config set-credentials admin \
  --client-certificate=admin-certs/admin.crt \
  --client-key=admin-certs/admin.key \
  --embed-certs=true \
  --kubeconfig=admin.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=admin \
  --kubeconfig=admin.kubeconfig

kubectl config use-context default --kubeconfig=admin.kubeconfig
```

---

## 7. Distribute Kubeconfigs

### Distribute to Worker Nodes

**For node-1:**
```bash
scp node-1.kubeconfig admin@node-1:~
scp kube-proxy.kubeconfig admin@node-1:~
```

Move into place:
```bash
ssh node-1 "sudo mkdir -p /var/lib/kubelet /var/lib/kube-proxy"
ssh node-1 "sudo mv node-1.kubeconfig /var/lib/kubelet/kubeconfig"
ssh node-1 "sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig"
```

**For node-2:**
```bash
scp node-2.kubeconfig admin@node-2:~
scp kube-proxy.kubeconfig admin@node-2:~
```

Move into place:
```bash
ssh node-2 "sudo mkdir -p /var/lib/kubelet /var/lib/kube-proxy"
ssh node-2 "sudo mv node-2.kubeconfig /var/lib/kubelet/kubeconfig"
ssh node-2 "sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig"
```

### Distribute to Control Plane

```bash
scp admin.kubeconfig admin@server:~
scp kube-controller-manager.kubeconfig admin@server:~
scp kube-scheduler.kubeconfig admin@server:~
```

Move into place:
```bash
ssh server "sudo mkdir -p /var/lib/kubernetes"
ssh server "sudo mv admin.kubeconfig /var/lib/kubernetes/"
ssh server "sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/"
ssh server "sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/"
```

### Keep Admin Kubeconfig on Jumpbox

```bash
mkdir -p ~/.kube
scp admin.kubeconfig ~/.kube/config
chmod 600 ~/.kube/config
```

---

## 8. Verification

### Inspect a Kubeconfig File

```bash
cat node-1.kubeconfig
```

Expected structure:
```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTi...  # Base64-encoded CA cert
    server: https://server.kubernetes.local:6443
  name: kubernetes-the-hard-way
contexts:
- context:
    cluster: kubernetes-the-hard-way
    user: system:node:node-1
  name: default
current-context: default
users:
- name: system:node:node-1
  user:
    client-certificate-data: LS0tLS1CRUdJTi...  # Base64-encoded client cert
    client-key-data: LS0tLS1CRUdJTi...          # Base64-encoded client key
```

### Verify All Files Exist

```bash
ls -la *.kubeconfig
```

Expected:
```
-rw------- 1 admin admin 5.4K Jun 14 10:30 admin.kubeconfig
-rw------- 1 admin admin 5.4K Jun 14 10:30 kube-controller-manager.kubeconfig
-rw------- 1 admin admin 5.4K Jun 14 10:30 kube-proxy.kubeconfig
-rw------- 1 admin admin 5.4K Jun 14 10:30 kube-scheduler.kubeconfig
-rw------- 1 admin admin 5.4K Jun 14 10:30 node-1.kubeconfig
-rw------- 1 admin admin 5.4K Jun 14 10:30 node-2.kubeconfig
```

### Verify Distribution

```bash
ssh node-1 "ls -la /var/lib/kubelet/kubeconfig /var/lib/kube-proxy/kubeconfig"
ssh node-2 "ls -la /var/lib/kubelet/kubeconfig /var/lib/kube-proxy/kubeconfig"
ssh server "ls -la /var/lib/kubernetes/*.kubeconfig"
```

---

## 9. Summary

At the end of Step 3, confirm:

| Check | Command | Expected |
|-------|---------|----------|
| All kubeconfigs | `ls *.kubeconfig` | 6 files |
| Embedded certs | `cat admin.kubeconfig \| grep certificate-authority-data` | Base64 string (not file path) |
| Correct server | `cat admin.kubeconfig \| grep server` | `https://server.kubernetes.local:6443` |
| Node-1 has kubeconfig | `ssh node-1 "ls /var/lib/kubelet/kubeconfig"` | File exists |
| Node-2 has kubeconfig | `ssh node-2 "ls /var/lib/kubelet/kubeconfig"` | File exists |
| Server has configs | `ssh server "ls /var/lib/kubernetes/*.kubeconfig"` | 3 files |
| Jumpbox has admin | `ls ~/.kube/config` | File exists |

**Next**: Proceed to [Step 4: Encryption Config and etcd](../04-encryption-config-and-etcd/04-implementation-steps.md) to secure data at rest and bootstrap the cluster datastore.
