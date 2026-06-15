# Step 6: Worker Node Bootstrap - Implementation

> This guide joins worker nodes to the Kubernetes cluster by installing and configuring containerd, CNI plugins, kubelet, and kube-proxy. Includes workarounds for offline worker nodes.

---

## Table of Contents

- [Overview](#overview)
- [1. Install Worker Binaries](#1-install-worker-binaries)
- [2. Install Required Packages (Offline Workaround)](#2-install-required-packages-offline-workaround)
- [3. Configure CNI Networking](#3-configure-cni-networking)
- [4. Configure Kernel Modules](#4-configure-kernel-modules)
- [5. Configure containerd](#5-configure-containerd)
- [6. Configure Kubelet](#6-configure-kubelet)
- [7. Configure Kube-Proxy](#7-configure-kube-proxy)
- [8. Start Worker Services](#8-start-worker-services)
- [9. Verify Node Registration](#9-verify-node-registration)
- [10. Troubleshooting DNS Resolution](#10-troubleshooting-dns-resolution)
- [11. Summary](#11-summary)

---

## Overview

Worker nodes run the actual workloads. Each worker needs:

| Component | Purpose |
|-----------|---------|
| **containerd** | Container runtime — pulls images, creates/destroys containers |
| **CNI plugins** | Sets up pod networking (bridge, host-local, loopback) |
| **kubelet** | Kubernetes agent — registers node, manages pods on this node |
| **kube-proxy** | Network proxy — implements Kubernetes Services via iptables |

### Prerequisites

- Control plane is running (Step 5 completed)
- Worker node certificates are in `/var/lib/kubelet/` (from Step 2)
- Worker node kubeconfigs are in `/var/lib/kubelet/kubeconfig` and `/var/lib/kube-proxy/kubeconfig` (from Step 3)

---

## 1. Install Worker Binaries

From the jumpbox, copy binaries to both worker nodes:

```bash
for HOST in node-1 node-2; do
  scp \
    downloads/worker/containerd \
    downloads/worker/containerd-shim-runc-v2 \
    downloads/worker/containerd-stress \
    downloads/worker/crictl \
    downloads/worker/ctr \
    downloads/worker/kube-proxy \
    downloads/worker/kubelet \
    downloads/worker/runc \
    downloads/client/kubectl \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    admin@${HOST}:~/
done
```

Copy CNI plugins:

```bash
for HOST in node-1 node-2; do
  scp downloads/cni-plugins/* admin@${HOST}:~/cni-plugins/
done
```

### On Each Worker Node

Create directories and install binaries:

```bash
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes

sudo mv containerd containerd-shim-runc-v2 containerd-stress /bin/
sudo mv crictl kube-proxy kubelet runc /usr/local/bin/
sudo mv cni-plugins/* /opt/cni/bin/
```

---

## 2. Install Required Packages (Offline Workaround)

Worker nodes don't have internet access. Required packages (`socat`, `conntrack`, `ipset`) must be transferred manually.

### On Jumpbox

```bash
mkdir ~/worker-debs && cd ~/worker-debs
apt download socat conntrack ipset kmod

# Check dependencies for ipset
apt-cache depends ipset
# ipset depends on libipset13t64

apt download libipset13t64
```

### Transfer to Workers

```bash
scp ~/worker-debs/*.deb node-1:~/
scp ~/worker-debs/*.deb node-2:~/
```

### On Each Worker Node

```bash
sudo dpkg -i *.deb
```

Expected output:
```
Selecting previously unselected package conntrack.
Unpacking conntrack (1:1.4.8-2) ...
Selecting previously unselected package libipset13t64:amd64.
Unpacking libipset13t64:amd64 (7.22-1+b1) ...
Selecting previously unselected package ipset.
Unpacking ipset (7.22-1+b1) ...
Setting up conntrack (1:1.4.8-2) ...
Setting up libipset13t64:amd64 (7.22-1+b1) ...
Setting up ipset (7.22-1+b1) ...
Setting up socat (1.8.0.3-1) ...
Setting up kmod (34.2-2) ...
```

---

## 3. Configure CNI Networking

### Create CNI Config Directory

```bash
sudo mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
```

The `10-bridge.conf` file (generated per-node in earlier steps) should contain:

**For node-1**:
```json
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "10.200.0.0/24"}]
    ],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
```

**For node-2**:
```json
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "10.200.1.0/24"}]
    ],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
```

The `99-loopback.conf` file:
```json
{
  "cniVersion": "1.1.0",
  "name": "lo",
  "type": "loopback"
}
```

---

## 4. Configure Kernel Modules

Load required kernel modules and configure sysctl:

```bash
sudo modprobe br_netfilter
echo "br_netfilter" | sudo tee /etc/modules-load.d/modules.conf
```

Configure sysctl for Kubernetes networking:

```bash
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
sudo sysctl -p /etc/sysctl.d/kubernetes.conf
```

Verify:
```bash
sudo lsmod | grep br_netfilter
sudo sysctl net.bridge.bridge-nf-call-iptables
sudo sysctl net.bridge.bridge-nf-call-ip6tables
```

Expected:
```
br_netfilter           36864  0
bridge                372736  1 br_netfilter
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

---

## 5. Configure containerd

Create containerd config directory and move config:

```bash
sudo mkdir -p /etc/containerd/
sudo mv containerd-config.toml /etc/containerd/config.toml
sudo mv containerd.service /etc/systemd/system/
```

> **Note**: If your containerd config doesn't specify a sandbox image, add it:
> ```toml
> [plugins."io.containerd.grpc.v1.cri"]
>   sandbox_image = "registry.k8s.io/pause:3.10"
> ```

---

## 6. Configure Kubelet

Move kubelet config and service:

```bash
sudo mv kubelet-config.yaml /var/lib/kubelet/
sudo mv kubelet.service /etc/systemd/system/
```

The `kubelet-config.yaml` should contain:

```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: "0.0.0.0"
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubelet/ca.crt"
authorization:
  mode: Webhook
cgroupDriver: systemd
containerRuntimeEndpoint: "unix:///var/run/containerd/containerd.sock"
enableServer: true
failSwapOn: false
maxPods: 16
memorySwap:
  swapBehavior: NoSwap
port: 10250
resolvConf: "/etc/resolv.conf"
registerNode: true
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/kubelet.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/kubelet.key"
```

The `kubelet.service` file:

```ini
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 7. Configure Kube-Proxy

Move kube-proxy config and service:

```bash
sudo mv kube-proxy-config.yaml /var/lib/kube-proxy/
sudo mv kube-proxy.service /etc/systemd/system/
```

The `kube-proxy-config.yaml` should contain:

```yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
```

The `kube-proxy.service` file:

```ini
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 8. Start Worker Services

Reload systemd and start all services:

```bash
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

Verify services are active:

```bash
systemctl is-active containerd
systemctl is-active kubelet
systemctl is-active kube-proxy
```

Expected output for each:
```
active
```

---

## 9. Verify Node Registration

From the jumpbox or server, check if nodes registered:

```bash
ssh admin@server "kubectl get nodes --kubeconfig admin.kubeconfig"
```

Initially, you may see:
```
No resources found
```

If nodes are registering, you should eventually see:
```
NAME     STATUS     ROLES    AGE   VERSION
node-1   NotReady   <none>   30s   v1.32.3
node-2   NotReady   <none>   30s   v1.32.3
```

Nodes show `NotReady` until the pod network (CNI + routes) is fully configured (Steps 6 and 7 combined).

Check kubelet logs if nodes aren't registering:

```bash
ssh node-1 "sudo journalctl -u kubelet -f"
```

---

## 10. Troubleshooting DNS Resolution

### Issue: "Temporary failure in name resolution"

If kubelet logs show:
```
Unable to register node with API server" err="Post \"https://server.kubernetes.local:6443/api/v1/nodes\": dial tcp: lookup server.kubernetes.local: Temporary failure in name resolution"
```

**Cause**: Worker nodes cannot resolve `server.kubernetes.local` to the API server's IP.

**Fix**: Add the API server's hostname to `/etc/hosts` on **each worker node**:

```bash
ssh node-1 "echo '172.31.20.244 server.kubernetes.local server' | sudo tee -a /etc/hosts"
ssh node-2 "echo '172.31.20.244 server.kubernetes.local server' | sudo tee -a /etc/hosts"
```

Then restart kubelet on each node:

```bash
ssh node-1 "sudo systemctl restart kubelet"
ssh node-2 "sudo systemctl restart kubelet"
```

### Issue: Missing pause Image

When creating pods later, if you see:
```
FailedCreatePodSandbox: failed to pull registry.k8s.io/pause:3.10
```

**Cause**: Worker nodes can't pull the pause image from the internet.

**Fix**: Pre-pull the image on the jumpbox and transfer it:

On jumpbox:
```bash
sudo ctr image pull registry.k8s.io/pause:3.10
sudo ctr image export pause.tar registry.k8s.io/pause:3.10
scp pause.tar node-1:~/
scp pause.tar node-2:~/
```

On each worker:
```bash
sudo ctr image import pause.tar
```

Alternatively, configure containerd to use a private registry or mirror.

---

## 11. Summary

At the end of Step 6, confirm:

| Check | Command | Expected |
|-------|---------|----------|
| containerd running | `systemctl is-active containerd` | `active` |
| kubelet running | `systemctl is-active kubelet` | `active` |
| kube-proxy running | `systemctl is-active kube-proxy` | `active` |
| CNI config | `cat /etc/cni/net.d/10-bridge.conf` | JSON with correct subnet |
| br_netfilter loaded | `lsmod \| grep br_netfilter` | Module present |
| sysctl set | `sysctl net.bridge.bridge-nf-call-iptables` | `= 1` |
| kubelet config | `cat /var/lib/kubelet/kubelet-config.yaml` | Valid YAML |
| Nodes registered | `kubectl get nodes --kubeconfig admin.kubeconfig` | Shows node-1, node-2 |

**Next**: Proceed to [Step 7: Pod Network Routes and kubectl Remote Access](../07-pod-network-routes-and-kubectl/07-implementation-steps.md).
