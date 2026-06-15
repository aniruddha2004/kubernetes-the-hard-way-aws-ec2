# Step 5: Control Plane Bootstrap - Implementation

> This guide bootstraps the Kubernetes control plane on the server node, installing kube-apiserver, kube-controller-manager, and kube-scheduler, configuring RBAC for kubelet access, and resolving the DNS/certificate hostname mismatch encountered during setup.

---

## Table of Contents

- [Overview](#overview)
- [1. Distribute Control Plane Binaries](#1-distribute-control-plane-binaries)
- [2. Distribute CA Private Key to Server](#2-distribute-ca-private-key-to-server)
- [3. Create kube-apiserver systemd Service](#3-create-kube-apiserver-systemd-service)
- [4. Create kube-controller-manager systemd Service](#4-create-kube-controller-manager-systemd-service)
- [5. Create kube-scheduler Configuration and systemd Service](#5-create-kube-scheduler-configuration-and-systemd-service)
- [6. Start and Enable All Control Plane Services](#6-start-and-enable-all-control-plane-services)
- [7. Verify Control Plane Components](#7-verify-control-plane-components)
- [8. Configure RBAC for API Server to Kubelet](#8-configure-rbac-for-api-server-to-kubelet)
- [9. Fix DNS/Certificate Hostname Mismatch](#9-fix-dnscertificate-hostname-mismatch)
- [10. Summary](#10-summary)

---

## Overview

The **control plane** is the brain of Kubernetes. It makes all cluster-wide decisions, stores the desired state, and exposes the API that everything else talks to. Without a functioning control plane, the cluster cannot schedule pods, manage services, or respond to any API requests.

### Control Plane Components

| Component | Binary | Purpose | Runs On |
|-----------|--------|---------|---------|
| **kube-apiserver** | `kube-apiserver` | Front door to Kubernetes; validates and processes ALL API requests | `server` |
| **kube-controller-manager** | `kube-controller-manager` | Maintains desired state; runs controllers for nodes, replicas, endpoints, etc. | `server` |
| **kube-scheduler** | `kube-scheduler` | Assigns pods to worker nodes based on resource availability and constraints | `server` |

### Control Plane Architecture

```
                    +--------------------+
                    |   kube-apiserver   |
                    |    (port 6443)     |
                    +--+--+----------+---+
                       |               |
                       | HTTPS        | HTTPS
                       v               v
              +-----------+     +---------------------+
              |kube-scheduler|  |kube-controller-manager|
              +-----------+     +---------------------+
                       |               |
                       | Reads/Writes  | Reads/Writes
                       v               v
                    +--------------------+
                    |    etcd (127.0.0.1) |
                    |    (port 2379)     |
                    +--------------------+
```

### Prerequisites

Before starting, ensure:
- Step 4 is complete: etcd is running and healthy on `server`
- Step 3 is complete: All kubeconfigs are distributed
- Step 2 is complete: All certificates are generated and distributed
- You are logged into the **jumpbox** and have SSH access to `server`

### Cluster Details

| Property | Value |
|----------|-------|
| Control Plane IP | `172.31.20.244` |
| Worker Node 1 IP | `172.31.19.88` |
| Worker Node 2 IP | `172.31.25.215` |
| Service CIDR | `10.32.0.0/24` |
| Pod CIDR (node-1) | `10.200.0.0/24` |
| Pod CIDR (node-2) | `10.200.1.0/24` |
| Kubernetes Version | `v1.32.3` |
| etcd Endpoint | `https://127.0.0.1:2379` |
| API Server Port | `6443` |
| Certificates Dir | `/var/lib/kubernetes/` |
| Binaries Path | `/usr/local/bin/` |

---

## 1. Distribute Control Plane Binaries

The control plane binaries (`kube-apiserver`, `kube-controller-manager`, `kube-scheduler`) were downloaded in Step 1 and organized in the `downloads/controller/` directory on the jumpbox.

### Copy Binaries to Server

```bash
scp downloads/controller/kube-apiserver admin@server:~
scp downloads/controller/kube-controller-manager admin@server:~
scp downloads/controller/kube-scheduler admin@server:~
```

Expected output:
```
kube-apiserver                              100%   98MB  12.3MB/s   00:08
kube-controller-manager                     100%   85MB  14.2MB/s   00:06
kube-scheduler                              100%   62MB  15.5MB/s   00:04
```

### Install Binaries

Move the binaries to `/usr/local/bin/` and make them executable:

```bash
ssh server "sudo mv ~/kube-apiserver ~/kube-controller-manager ~/kube-scheduler /usr/local/bin/"
ssh server "sudo chmod +x /usr/local/bin/kube-apiserver /usr/local/bin/kube-controller-manager /usr/local/bin/kube-scheduler"
```

### Verify Installation

```bash
ssh server "kube-apiserver --version"
```

Expected output:
```
Kubernetes v1.32.3
```

Verify all three binaries:
```bash
ssh server "ls -la /usr/local/bin/kube-*"
```

Expected output:
```
-rwxr-xr-x 1 root root 98765432 Jun 14 10:45 /usr/local/bin/kube-apiserver
-rwxr-xr-x 1 root root 85234123 Jun 14 10:45 /usr/local/bin/kube-controller-manager
-rwxr-xr-x 1 root root 62123456 Jun 14 10:45 /usr/local/bin/kube-scheduler
```

---

## 2. Distribute CA Private Key to Server

The **kube-controller-manager** needs the CA private key (`ca.key`) to sign Certificate Signing Requests (CSRs) and to generate service account tokens. By default, `ca.key` was kept only on the jumpbox for security. We must now copy it to the server.

> **Security Warning**: `ca.key` is the most sensitive file in the cluster. Anyone with this key can create valid certificates for any identity. Restrict its permissions strictly.

### Copy CA Private Key to Server

```bash
scp ca/ca.key admin@server:~
ssh server "sudo mv ~/ca.key /var/lib/kubernetes/"
ssh server "sudo chmod 600 /var/lib/kubernetes/ca.key"
```

### Verify CA Key Exists on Server

```bash
ssh server "ls -la /var/lib/kubernetes/ca.key"
```

Expected output:
```
-rw------- 1 root root 3247 Jun 14 10:50 /var/lib/kubernetes/ca.key
```

> **Note**: Ensure `ca.key` has `600` permissions (owner read/write only). Never share or expose this file.

---

## 3. Create kube-apiserver systemd Service

The **kube-apiserver** is the front door to Kubernetes. Every request — whether from `kubectl`, a controller, or a kubelet — goes through the API server. It is the only component that talks directly to etcd.

### Create the systemd Unit File

On the jumpbox, create the service file:

```bash
cat > kube-apiserver.service <<EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
  --advertise-address=172.31.20.244 \
  --allow-privileged=true \
  --apiserver-count=1 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --client-ca-file=/var/lib/kubernetes/ca.crt \
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --etcd-cafile=/var/lib/kubernetes/ca.crt \
  --etcd-certfile=/var/lib/kubernetes/kubernetes.crt \
  --etcd-keyfile=/var/lib/kubernetes/kubernetes.key \
  --etcd-servers=https://127.0.0.1:2379 \
  --event-ttl=1h \
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.crt \
  --kubelet-client-key=/var/lib/kubernetes/kubernetes.key \
  --kubelet-https=true \
  --runtime-config=api/all=true \
  --service-account-key-file=/var/lib/kubernetes/service-account.pub \
  --service-account-signing-key-file=/var/lib/kubernetes/service-account.key \
  --service-account-issuer=https://kubernetes.default.svc.cluster.local \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.crt \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes.key \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Key Flags Explained

| Flag | Purpose |
|------|---------|
| `--advertise-address=172.31.20.244` | The IP other cluster members should use to reach this API server |
| `--allow-privileged=true` | Allow privileged containers (needed for some CNI and system pods) |
| `--authorization-mode=Node,RBAC` | Enable both Node authorizer and RBAC for access control |
| `--bind-address=0.0.0.0` | Listen on all interfaces (needed for remote kubectl access) |
| `--client-ca-file=/var/lib/kubernetes/ca.crt` | CA certificate to validate client certificates (mTLS) |
| `--enable-admission-plugins=...` | Enable admission controllers for policy enforcement |
| `--etcd-servers=https://127.0.0.1:2379` | etcd endpoint (local, since etcd runs on same node) |
| `--encryption-provider-config=...` | Path to encryption config for secrets at rest |
| `--service-cluster-ip-range=10.32.0.0/24` | IP range for Kubernetes Services (ClusterIPs) |
| `--service-node-port-range=30000-32767` | Port range for NodePort Services |
| `--tls-cert-file` / `--tls-private-key-file` | API server's own TLS certificate and key |
| `--service-account-*` | Configuration for service account token signing and validation |
| `--kubelet-*` | Certificates for the API server to authenticate to kubelets |

### Copy Service File to Server

```bash
scp kube-apiserver.service admin@server:~
ssh server "sudo mv ~/kube-apiserver.service /etc/systemd/system/"
```

---

## 4. Create kube-controller-manager systemd Service

The **kube-controller-manager** runs the core control loops that watch the shared state of the cluster through the API server and make changes attempting to move the current state toward the desired state.

### Create the systemd Unit File

```bash
cat > kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --bind-address=0.0.0.0 \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \
  --cluster-signing-key-file=/var/lib/kubernetes/ca.key \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --leader-elect=true \
  --root-ca-file=/var/lib/kubernetes/ca.crt \
  --service-account-private-key-file=/var/lib/kubernetes/service-account.key \
  --service-cluster-ip-range=10.32.0.0/24 \
  --use-service-account-credentials=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Key Flags Explained

| Flag | Purpose |
|------|---------|
| `--bind-address=0.0.0.0` | Listen on all interfaces for metrics and health endpoints |
| `--cluster-cidr=10.200.0.0/16` | The CIDR range used for pod IP addresses (covers both node-1 and node-2 pod CIDRs) |
| `--cluster-name=kubernetes` | Name of the cluster (used in some generated object names) |
| `--cluster-signing-cert-file` / `--cluster-signing-key-file` | CA cert and key for signing CSRs (pod certificates) |
| `--kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig` | Authentication to API server |
| `--leader-elect=true` | Enable leader election (critical for HA; harmless for single-node) |
| `--root-ca-file=/var/lib/kubernetes/ca.crt` | Root CA for validating certificates in the cluster |
| `--service-account-private-key-file=...` | Key used to sign service account tokens |
| `--service-cluster-ip-range=10.32.0.0/24` | Must match API server's service CIDR |
| `--use-service-account-credentials=true` | Use separate service account credentials per controller |

> **Important**: `--cluster-cidr=10.200.0.0/16` is the aggregate of all pod CIDRs. node-1 uses `10.200.0.0/24` and node-2 uses `10.200.1.0/24`, both within this `/16` range.

### Copy Service File to Server

```bash
scp kube-controller-manager.service admin@server:~
ssh server "sudo mv ~/kube-controller-manager.service /etc/systemd/system/"
```

---

## 5. Create kube-scheduler Configuration and systemd Service

The **kube-scheduler** assigns newly created pods to suitable worker nodes. It watches for unscheduled pods and selects the best node based on resource requirements, affinity rules, and other constraints.

### Create kube-scheduler Configuration File

The scheduler uses a YAML configuration file rather than many CLI flags. First create the config directory and file:

```bash
ssh server "sudo mkdir -p /etc/kubernetes/config"
```

Create the scheduler configuration:

```bash
cat > kube-scheduler.yaml <<EOF
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: /var/lib/kubernetes/kube-scheduler.kubeconfig
leaderElection:
  leaderElect: true
EOF
```

Parameters:
- `apiVersion: kubescheduler.config.k8s.io/v1` — API version for scheduler configuration
- `kind: KubeSchedulerConfiguration` — Configuration resource type
- `clientConnection.kubeconfig` — Path to the scheduler's kubeconfig for API server authentication
- `leaderElection.leaderElect: true` — Enable leader election (allows HA; safe for single-node)

### Copy Configuration to Server

```bash
scp kube-scheduler.yaml admin@server:~
ssh server "sudo mv ~/kube-scheduler.yaml /etc/kubernetes/config/"
ssh server "sudo chmod 644 /etc/kubernetes/config/kube-scheduler.yaml"
```

### Create the systemd Unit File

```bash
cat > kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/config/kube-scheduler.yaml \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Copy Service File to Server

```bash
scp kube-scheduler.service admin@server:~
ssh server "sudo mv ~/kube-scheduler.service /etc/systemd/system/"
```

---

## 6. Start and Enable All Control Plane Services

Now that all binaries, certificates, kubeconfigs, and systemd service files are in place, reload systemd and start the control plane.

### Reload systemd Daemon

```bash
ssh server "sudo systemctl daemon-reload"
```

Expected: No output (success).

### Enable Services to Start on Boot

```bash
ssh server "sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler"
```

Expected output:
```
Created symlink /etc/systemd/system/multi-user.target.wants/kube-apiserver.service → /etc/systemd/system/kube-apiserver.service.
Created symlink /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service → /etc/systemd/system/kube-controller-manager.service.
Created symlink /etc/systemd/system/multi-user.target.wants/kube-scheduler.service → /etc/systemd/system/kube-scheduler.service.
```

### Start All Services

```bash
ssh server "sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler"
```

Expected: No output (success). Services start in the background.

> **Note**: The API server may take 10-30 seconds to fully initialize as it connects to etcd and sets up REST storage for all resources. If a service fails immediately, check the logs with `journalctl -u kube-apiserver`.

---

## 7. Verify Control Plane Components

### Check Service Status

Verify each service is active and running:

#### kube-apiserver

```bash
ssh server "sudo systemctl status kube-apiserver"
```

Expected output:
```
● kube-apiserver.service - Kubernetes API Server
     Loaded: loaded (/etc/systemd/system/kube-apiserver.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-06-14 11:00:00 UTC; 30s ago
   Main PID: 5678 (kube-apiserver)
      Tasks: 8 (limit: 2271)
     Memory: 256.3M
        CPU: 1.234s
     CGroup: /system.slice/kube-apiserver.service
             └─5678 /usr/local/bin/kube-apiserver --advertise-address=172.31.20.244 --allow-privileged=true ...
```

#### kube-controller-manager

```bash
ssh server "sudo systemctl status kube-controller-manager"
```

Expected output:
```
● kube-controller-manager.service - Kubernetes Controller Manager
     Loaded: loaded (/etc/systemd/system/kube-controller-manager.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-06-14 11:00:05 UTC; 25s ago
   Main PID: 5689 (kube-controller-manager)
      Tasks: 6 (limit: 2271)
     Memory: 128.5M
        CPU: 876ms
     CGroup: /system.slice/kube-controller-manager.service
             └─5689 /usr/local/bin/kube-controller-manager --bind-address=0.0.0.0 --cluster-cidr=10.200.0.0/16 ...
```

#### kube-scheduler

```bash
ssh server "sudo systemctl status kube-scheduler"
```

Expected output:
```
● kube-scheduler.service - Kubernetes Scheduler
     Loaded: loaded (/etc/systemd/system/kube-scheduler.service; enabled; preset: enabled)
     Active: active (running) since Sat 2026-06-14 11:00:03 UTC; 27s ago
   Main PID: 5685 (kube-scheduler)
      Tasks: 5 (limit: 2271)
     Memory: 64.2M
        CPU: 445ms
     CGroup: /system.slice/kube-scheduler.service
             └─5685 /usr/local/bin/kube-scheduler --config=/etc/kubernetes/config/kube-scheduler.yaml --v=2
```

### Verify API Server is Listening

Check that the API server is listening on port 6443:

```bash
ssh server "sudo ss -tlnp | grep 6443"
```

Expected output:
```
LISTEN 0      2048         0.0.0.0:6443      0.0.0.0:*    users:(("kube-apiserver",pid=5678,fd=7))
```

### Verify API Server Responds (Local on Server)

From the server itself, use `curl` with the local certificate to test:

```bash
ssh server "curl --cacert /var/lib/kubernetes/ca.crt \
  --cert /var/lib/kubernetes/kubernetes.crt \
  --key /var/lib/kubernetes/kubernetes.key \
  https://127.0.0.1:6443/healthz"
```

Expected output:
```
ok
```

### Check API Server Logs for Errors

If any service shows `failed` or `degraded`, inspect the logs:

```bash
ssh server "sudo journalctl -u kube-apiserver --no-pager -n 50"
```

Look for lines containing:
- `"Serving securely on 0.0.0.0:6443"` — Success
- `"etcdclient: no cluster endpoints"` — etcd connection issue
- `"Unable to find eligible endpoint"` — etcd not reachable or unhealthy

### Verify Controller Manager Logs

```bash
ssh server "sudo journalctl -u kube-controller-manager --no-pager -n 30"
```

Look for:
- `"Starting controllers"` — Controllers are initializing
- `"leader election"` — Leader election status
- `"Serving default metrics"` — Metrics endpoint ready

### Verify Scheduler Logs

```bash
ssh server "sudo journalctl -u kube-scheduler --no-pager -n 30"
```

Look for:
- `"Starting Kubernetes Scheduler"` — Scheduler initialized
- `"leader election"` — Leader election active

---

## 8. Configure RBAC for API Server to Kubelet

The API server needs permission to proxy to kubelet endpoints (for `kubectl logs`, `kubectl exec`, `kubectl top`, etc.). By default, the API server authenticates to kubelets as the `kubernetes` user, but this user has no permissions. We must grant them explicitly via RBAC.

### Create the ClusterRole

This ClusterRole grants the API server permission to access kubelet-proxied endpoints:

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

Expected output:
```
clusterrole.rbac.authorization.k8s.io/system:kube-apiserver-to-kubelet created
```

Permissions granted:
- `nodes/proxy` — Access kubelet proxy endpoint (for `kubectl exec` and `kubectl port-forward`)
- `nodes/stats` — Access kubelet stats endpoint (for `kubectl top`)
- `nodes/log` — Access kubelet logs endpoint (for `kubectl logs`)
- `nodes/spec` — Access kubelet node specification
- `nodes/metrics` — Access kubelet metrics (for monitoring)
- `"*"` verbs — Allow all operations (create, get, list, watch, update, patch, delete)

### Create the ClusterRoleBinding

Bind the ClusterRole to the `kubernetes` user (the identity the API server uses when connecting to kubelets):

```bash
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

Expected output:
```
clusterrolebinding.rbac.authorization.k8s.io/system:kube-apiserver created
```

### Verify RBAC Resources

List the created RBAC resources:

```bash
kubectl get clusterrole system:kube-apiserver-to-kubelet --kubeconfig admin.kubeconfig
```

Expected output:
```
NAME                            CREATED AT
system:kube-apiserver-to-kubelet   2026-06-14T11:05:00Z
```

```bash
kubectl get clusterrolebinding system:kube-apiserver --kubeconfig admin.kubeconfig
```

Expected output:
```
NAME                    ROLE                                    AGE
system:kube-apiserver   ClusterRole/system:kube-apiserver-to-kubelet   30s
```

### Describe the ClusterRole for Detail

```bash
kubectl describe clusterrole system:kube-apiserver-to-kubelet --kubeconfig admin.kubeconfig
```

Expected output:
```
Name:         system:kube-apiserver-to-kubelet
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources                    Non-Resource URLs    Resource Names    Verbs
  ---------                    -----------------    --------------    -----
  nodes/log                    []                   []                [*]
  nodes/metrics                []                   []                [*]
  nodes/proxy                  []                   []                [*]
  nodes/spec                   []                   []                [*]
  nodes/stats                  []                   []                [*]
```

---

## 9. Fix DNS/Certificate Hostname Mismatch

During setup, a real error was encountered when trying to connect to the API server. This section documents the problem and the fix so you can verify your setup avoids it.

### The Error Encountered

Initially, the hostname `server.ani-kubernetes.local` was used in `/etc/hosts` and in kubeconfig files. However, the API server certificate was generated with `server.kubernetes.local` as a Subject Alternative Name (SAN), not `server.ani-kubernetes.local`.

When running any `kubectl` command:

```bash
kubectl get nodes --kubeconfig admin.kubeconfig
```

The following error appeared:
```
Unable to connect to the server: x509: certificate is valid for server.kubernetes.local, kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster, kubernetes.default.svc.cluster.local, 10.32.0.1, 172.31.20.244, not server.ani-kubernetes.local
```

### Root Cause

The kubeconfig's `server` field pointed to `https://server.ani-kubernetes.local:6443`, but the API server's TLS certificate did not include `server.ani-kubernetes.local` in its SAN list. TLS hostname verification failed because the name used to connect did not match any name the certificate claimed to represent.

### The Fix

Two options exist:

**Option 1: Regenerate the API Server Certificate (Complex)**
- Regenerate `kubernetes.crt` with `server.ani-kubernetes.local` added to the SAN list
- Redistribute the certificate to all nodes
- This is more work but preserves the desired hostname

**Option 2: Use the SAN That Exists in the Certificate (Simple)**
- Update `/etc/hosts` to map `server.kubernetes.local` to `172.31.20.244`
- Update all kubeconfigs to use `https://server.kubernetes.local:6443`
- This is what was done in this tutorial

### Verify the Fix

Confirm the kubeconfig uses the correct server name:

```bash
grep server admin.kubeconfig
```

Expected output:
```
    server: https://server.kubernetes.local:6443
```

Confirm `/etc/hosts` resolves correctly:

```bash
ssh jumpbox "grep server.kubernetes.local /etc/hosts"
```

Expected:
```
172.31.20.244 server.kubernetes.local
```

Also verify on the server node itself:

```bash
ssh server "grep server.kubernetes.local /etc/hosts"
```

Expected:
```
172.31.20.244 server.kubernetes.local
```

### Confirm API Server Certificate SANs

Verify the certificate actually contains the expected SANs:

```bash
openssl x509 -in kube-api-server/kubernetes.crt -text -noout | grep -A 10 "Subject Alternative Name"
```

Expected output:
```
            X509v3 Subject Alternative Name:
                DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster, DNS:kubernetes.default.svc.cluster.local, DNS:server.kubernetes.local, IP Address:10.32.0.1, IP Address:172.31.20.244
```

> **Lesson**: When generating certificates, include **ALL hostnames and IPs** that will ever be used to connect to the API server in the SAN list. This includes:
> - Human-friendly DNS names (`server.kubernetes.local`)
> - Kubernetes internal service names (`kubernetes.default.svc.cluster.local`)
> - Service cluster IP (`10.32.0.1` — the `kubernetes` service IP)
> - Physical node IP (`172.31.20.244`)

### Test kubectl Connectivity

With the fix in place, test that `kubectl` can now communicate with the API server:

```bash
kubectl version --kubeconfig admin.kubeconfig
```

Expected output:
```
Client Version: v1.32.3
Kustomize Version: v5.5.0
Server Version: v1.32.3
```

The presence of `Server Version:` confirms the API server is responding and the TLS handshake succeeded.

### List Nodes (Should Show No Nodes Yet)

```bash
kubectl get nodes --kubeconfig admin.kubeconfig
```

Expected output:
```
No resources found
```

This is normal! Worker nodes haven't been joined yet. The fact that `kubectl` responds at all means the control plane is operational.

---

## 10. Summary

At the end of Step 5, confirm:

| Check | Command | Expected |
|-------|---------|----------|
| API server binary | `ssh server "ls /usr/local/bin/kube-apiserver"` | File exists |
| Controller manager binary | `ssh server "ls /usr/local/bin/kube-controller-manager"` | File exists |
| Scheduler binary | `ssh server "ls /usr/local/bin/kube-scheduler"` | File exists |
| CA key on server | `ssh server "ls /var/lib/kubernetes/ca.key"` | File exists, mode 600 |
| API server running | `ssh server "sudo systemctl status kube-apiserver"` | `active (running)` |
| Controller manager running | `ssh server "sudo systemctl status kube-controller-manager"` | `active (running)` |
| Scheduler running | `ssh server "sudo systemctl status kube-scheduler"` | `active (running)` |
| API server listens on 6443 | `ssh server "sudo ss -tlnp \| grep 6443"` | `LISTEN` state |
| API server healthz | `curl --cacert ... https://127.0.0.1:6443/healthz` | `ok` |
| RBAC ClusterRole | `kubectl get clusterrole system:kube-apiserver-to-kubelet` | Exists |
| RBAC ClusterRoleBinding | `kubectl get clusterrolebinding system:kube-apiserver` | Exists |
| kubectl server version | `kubectl version --kubeconfig admin.kubeconfig` | Shows `Server Version` |
| kubeconfig server URL | `grep server admin.kubeconfig` | `server.kubernetes.local:6443` |
| Certificate SANs | `openssl x509 -in kubernetes.crt -text -noout \| grep DNS` | Includes `server.kubernetes.local` |

### Control Plane Bootstrap Checklist

- [ ] Control plane binaries copied to `/usr/local/bin/` on server
- [ ] Binaries are executable (`chmod +x`)
- [ ] CA private key (`ca.key`) copied to `/var/lib/kubernetes/` on server with `600` permissions
- [ ] `kube-apiserver.service` created and installed in `/etc/systemd/system/`
- [ ] `kube-controller-manager.service` created and installed in `/etc/systemd/system/`
- [ ] `kube-scheduler.service` created and installed in `/etc/systemd/system/`
- [ ] `kube-scheduler.yaml` config created in `/etc/kubernetes/config/`
- [ ] All services enabled for startup (`systemctl enable`)
- [ ] All services started (`systemctl start`)
- [ ] API server responds to `/healthz` with `ok`
- [ ] RBAC ClusterRole `system:kube-apiserver-to-kubelet` created
- [ ] RBAC ClusterRoleBinding `system:kube-apiserver` created
- [ ] Kubeconfig server URL matches certificate SAN (`server.kubernetes.local`)
- [ ] `kubectl version` returns both client and server versions

### Next Steps

**Next**: Proceed to [Step 6: Worker Nodes Bootstrap](../06-worker-nodes-bootstrap/06-implementation-steps.md) to install kubelet, kube-proxy, containerd, and CNI plugins on `node-1` and `node-2`.

---

## Troubleshooting Quick Reference

### Issue: kube-apiserver fails to start

**Symptoms**: `systemctl status kube-apiserver` shows `failed` or `activating` indefinitely.

**Common Causes & Fixes**:

1. **Cannot connect to etcd**
   - Verify etcd is running: `ssh server "sudo systemctl status etcd"`
   - Check etcd listens on 2379: `ssh server "sudo ss -tlnp | grep 2379"`
   - Verify certificates: `ssh server "ls -la /var/lib/kubernetes/*.crt /var/lib/kubernetes/*.key"`

2. **Certificate file not found**
   - Check all `--*-file` paths in the service file exist on server
   - Verify files were copied in Steps 2 and 4

3. **Port 6443 already in use**
   - Check: `ssh server "sudo ss -tlnp | grep 6443"`
   - Kill any stale process: `ssh server "sudo fuser -k 6443/tcp"`

**View Logs**:
```bash
ssh server "sudo journalctl -u kube-apiserver --no-pager -n 100"
```

### Issue: kube-controller-manager fails to start

**Symptoms**: Service fails shortly after start.

**Common Causes**:
1. **Missing kubeconfig**: Verify `/var/lib/kubernetes/kube-controller-manager.kubeconfig` exists
2. **Missing ca.key**: The controller manager needs `ca.key` for CSR signing. Check `ls /var/lib/kubernetes/ca.key`
3. **API server not ready**: Controller manager needs the API server. Start API server first and wait 30 seconds.

**View Logs**:
```bash
ssh server "sudo journalctl -u kube-controller-manager --no-pager -n 100"
```

### Issue: kube-scheduler fails to start

**Symptoms**: Scheduler exits with config error.

**Common Causes**:
1. **Missing config file**: Verify `/etc/kubernetes/config/kube-scheduler.yaml` exists
2. **Missing kubeconfig**: Verify `/var/lib/kubernetes/kube-scheduler.kubeconfig` exists
3. **Invalid YAML**: Check indentation and fields in the config file

**View Logs**:
```bash
ssh server "sudo journalctl -u kube-scheduler --no-pager -n 100"
```

### Issue: kubectl returns "Unable to connect to the server: x509 certificate error"

**Fix**: See [Section 9: Fix DNS/Certificate Hostname Mismatch](#9-fix-dnscertificate-hostname-mismatch). Ensure the hostname in your kubeconfig matches a SAN in the API server certificate.

### Issue: RBAC resources show "Forbidden" when accessing kubelet

After worker nodes join (Step 6), if `kubectl logs` or `kubectl exec` fails with "Forbidden":
- Verify the ClusterRole exists: `kubectl get clusterrole system:kube-apiserver-to-kubelet`
- Verify the ClusterRoleBinding exists: `kubectl get clusterrolebinding system:kube-apiserver`
- Verify the binding references the correct user (`kubernetes`): `kubectl describe clusterrolebinding system:kube-apiserver`
