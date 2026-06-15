# Step 4: Encryption Config and etcd - Implementation

> This guide configures encryption for Kubernetes secrets at rest and bootstraps the etcd distributed key-value store that serves as the cluster's database.

---

## Table of Contents

- [Overview](#overview)
- [1. Generate Encryption Config](#1-generate-encryption-config)
- [2. Distribute Encryption Config](#2-distribute-encryption-config)
- [3. Install etcd Binaries](#3-install-etcd-binaries)
- [4. Create etcd systemd Service](#4-create-etcd-systemd-service)
- [5. Start etcd Service](#5-start-etcd-service)
- [6. Verify etcd](#6-verify-etcd)
- [7. Summary](#7-summary)

---

## Overview

**etcd** is a distributed, consistent key-value store that holds ALL Kubernetes cluster data:
- Pod definitions and statuses
- Service configurations
- Secret data
- ConfigMaps
- Node information
- Events
- And everything else

### Why Encryption at Rest?

By default, etcd stores data as plain text. Anyone with access to the etcd data directory can read secrets, including passwords, API keys, and tokens. The **encryption config** ensures secrets are encrypted before being written to disk.

### Architecture

```
Kubernetes API Server
        |
        | HTTPS (mTLS)
        v
      etcd
   /var/lib/etcd/
        |
        | Encrypted on disk
        v
   Physical Disk
```

### etcd Deployment Model

For this lab, we run a **single-node etcd** on the control plane. In production, etcd should run as a **3 or 5 node cluster** for high availability.

---

## 1. Generate Encryption Config

### Generate Encryption Key

Create a 32-byte random key for AES-GCM encryption:

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

### Create Encryption Config File

```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

Configuration explained:
- `kind: EncryptionConfig` — Kubernetes resource type for encryption settings
- `resources: [secrets]` — Only encrypt Secret resources (can include ConfigMaps too)
- `providers:` — List of encryption providers in priority order
  - `aescbc` — AES encryption in CBC mode with the generated key
  - `identity: {}` — Fallback to no encryption (for reading unencrypted legacy data)

> **Important**: Save `ENCRYPTION_KEY` securely! If you lose it, you cannot decrypt secrets.

---

## 2. Distribute Encryption Config

Copy the encryption config to the control plane:

```bash
scp encryption-config.yaml admin@server:~
ssh server "sudo mkdir -p /var/lib/kubernetes"
ssh server "sudo mv encryption-config.yaml /var/lib/kubernetes/"
ssh server "sudo chmod 600 /var/lib/kubernetes/encryption-config.yaml"
```

---

## 3. Install etcd Binaries

### Copy etcd Binaries to Control Plane

```bash
scp downloads/controller/etcd admin@server:~
scp downloads/controller/etcdctl admin@server:~
```

### Install etcd and etcdctl

```bash
ssh server "sudo mv etcd etcdctl /usr/local/bin/"
ssh server "sudo chmod +x /usr/local/bin/etcd /usr/local/bin/etcdctl"
```

Verify installation:
```bash
ssh server "etcd --version"
```

Expected:
```
etcd Version: 3.6.0-rc.3
Git SHA: 12345678
Go Version: go1.23.4
Go OS/Arch: linux/amd64
```

---

## 4. Create etcd systemd Service

Create the systemd unit file for etcd:

```bash
cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name server \
  --cert-file=/var/lib/kubernetes/kubernetes.crt \
  --key-file=/var/lib/kubernetes/kubernetes.key \
  --peer-cert-file=/var/lib/kubernetes/kubernetes.crt \
  --peer-key-file=/var/lib/kubernetes/kubernetes.key \
  --trusted-ca-file=/var/lib/kubernetes/ca.crt \
  --peer-trusted-ca-file=/var/lib/kubernetes/ca.crt \
  --peer-client-cert-auth \
  --client-cert-auth \
  --initial-advertise-peer-urls https://<SERVER_PRIVATE_IP>:2380 \
  --listen-peer-urls https://<SERVER_PRIVATE_IP>:2380 \
  --listen-client-urls https://<SERVER_PRIVATE_IP>:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://<SERVER_PRIVATE_IP>:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster server=https://<SERVER_PRIVATE_IP>:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Flags explained:
- `--name server` — Human-readable name for this etcd member
- `--cert-file` / `--key-file` — Server certificate and key for client connections (mTLS)
- `--peer-cert-file` / `--peer-key-file` — Certificate and key for peer-to-peer communication
- `--trusted-ca-file` / `--peer-trusted-ca-file` — CA certificate to validate client/peer certificates
- `--peer-client-cert-auth` / `--client-cert-auth` — Require mTLS for all connections
- `--initial-advertise-peer-urls` — URL other etcd members use to reach this node
- `--listen-peer-urls` — URLs to listen on for peer traffic
- `--listen-client-urls` — URLs to listen on for client traffic (includes localhost for local tools)
- `--advertise-client-urls` — URL clients should use to connect
- `--initial-cluster-token` — Unique token for cluster initialization
- `--initial-cluster` — List of all etcd members in the cluster
- `--initial-cluster-state new` — Initialize a new cluster (vs joining existing)
- `--data-dir` — Directory for etcd data files

> **Replace `<SERVER_PRIVATE_IP>` with your control plane's actual private IP!**

---

## 5. Start etcd Service

### Copy and Enable Service

```bash
scp etcd.service admin@server:~
ssh server "sudo mv etcd.service /etc/systemd/system/"
ssh server "sudo systemctl daemon-reload"
ssh server "sudo systemctl enable etcd"
ssh server "sudo systemctl start etcd"
```

### Verify Service Status

```bash
ssh server "sudo systemctl status etcd"
```

Expected output:
```
● etcd.service - etcd
   Loaded: loaded (/etc/systemd/system/etcd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2026-06-14 10:45:00 UTC; 1min ago
 Main PID: 1234 (etcd)
    Tasks: 7 (limit: 2271)
   Memory: 45.2M
   CGroup: /system.slice/etcd.service
           └─1234 /usr/local/bin/etcd --name server --cert-file=/var/lib/kubernetes/kubernetes.crt ...
```

---

## 6. Verify etcd

### List etcd Members

```bash
ssh server "ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/kubernetes/ca.crt \
  --cert=/var/lib/kubernetes/kubernetes.crt \
  --key=/var/lib/kubernetes/kubernetes.key"
```

Expected output:
```
8e9e05c52164694d, started, server, https://<SERVER_PRIVATE_IP>:2380, https://<SERVER_PRIVATE_IP>:2379
```

Fields:
- `8e9e05c52164694d` — Member ID (hexadecimal)
- `started` — Member status
- `server` — Member name
- `https://<SERVER_PRIVATE_IP>:2380` — Peer URL
- `https://<SERVER_PRIVATE_IP>:2379` — Client URL

### Check etcd Health

```bash
ssh server "ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/kubernetes/ca.crt \
  --cert=/var/lib/kubernetes/kubernetes.crt \
  --key=/var/lib/kubernetes/kubernetes.key"
```

Expected:
```
https://127.0.0.1:2379 is healthy: successfully committed proposal: took = 2.3341ms
```

### Write and Read a Test Key

```bash
ssh server "ETCDCTL_API=3 etcdctl put /test/key \"hello-world\" \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/kubernetes/ca.crt \
  --cert=/var/lib/kubernetes/kubernetes.crt \
  --key=/var/lib/kubernetes/kubernetes.key"
```

Read it back:
```bash
ssh server "ETCDCTL_API=3 etcdctl get /test/key \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/kubernetes/ca.crt \
  --cert=/var/lib/kubernetes/kubernetes.crt \
  --key=/var/lib/kubernetes/kubernetes.key"
```

Expected:
```
/test/key
hello-world
```

Delete the test key:
```bash
ssh server "ETCDCTL_API=3 etcdctl del /test/key \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/kubernetes/ca.crt \
  --cert=/var/lib/kubernetes/kubernetes.crt \
  --key=/var/lib/kubernetes/kubernetes.key"
```

---

## 7. Summary

At the end of Step 4, confirm:

| Check | Command | Expected |
|-------|---------|----------|
| etcd installed | `ssh server "etcd --version"` | Version 3.6.0-rc.3 |
| etcd running | `ssh server "sudo systemctl status etcd"` | `active (running)` |
| etcd listens on 2379 | `ssh server "sudo ss -tlnp \| grep 2379"` | `LISTEN` state |
| etcd listens on 2380 | `ssh server "sudo ss -tlnp \| grep 2380"` | `LISTEN` state |
| Member listed | `ETCDCTL_API=3 etcdctl member list` | Shows 1 member |
| Healthy | `ETCDCTL_API=3 etcdctl endpoint health` | `is healthy` |
| Encryption config | `ssh server "ls /var/lib/kubernetes/encryption-config.yaml"` | File exists |
| Data directory | `ssh server "ls /var/lib/etcd/"` | Contains `member/` |

**Next**: Proceed to [Step 5: Control Plane Bootstrap](../05-control-plane-bootstrap/05-implementation-steps.md) to start the Kubernetes brain components.
