# Step 6: Worker Node Bootstrap - Detailed Explanations

> Deep-dive explanations for worker node components, networking, the reconciliation loop, and troubleshooting common issues.

---

## Table of Contents

- [Worker Node Architecture](#worker-node-architecture)
- [containerd: The Container Runtime](#containerd-the-container-runtime)
- [Kubelet: The Node Agent](#kubelet-the-node-agent)
- [Kube-Proxy: Service Networking](#kube-proxy-service-networking)
- [CNI Plugins: Pod Networking](#cni-plugins-pod-networking)
- [br_netfilter and sysctl](#br_netfilter-and-sysctl)
- [Kubelet Registration Flow](#kubelet-registration-flow)
- [Understanding Kubelet Logs](#understanding-kubelet-logs)
- [The Pause Container](#the-pause-container)
- [Worker vs Control Plane](#worker-vs-control-plane)

---

## Worker Node Architecture

```
+--------------------------+
|       Worker Node        |
|--------------------------|
|                          |
|  +------------------+    |
|  |     kubelet      |    |
|  |  - Node agent    |    |
|  |  - Pod manager   |    |
|  +--------+---------+    |
|           |              |
|           v              |
|  +------------------+    |
|  |    containerd    |    |
|  |  - Image pulls   |    |
|  |  - Container mgmt|    |
|  +--------+---------+    |
|           |              |
|           v              |
|  +------------------+    |
|  |      runc        |    |
|  |  - Linux ns/cgroups    |
|  +------------------+    |
|                          |
|  +------------------+    |
|  |   kube-proxy     |    |
|  |  - iptables rules|    |
|  +------------------+    |
|                          |
|  +------------------+    |
|  |  CNI plugins     |    |
|  |  - bridge        |    |
|  |  - host-local    |    |
|  |  - loopback      |    |
|  +------------------+    |
|                          |
+--------------------------+
```

---

## containerd: The Container Runtime

### What is containerd?

**containerd** is a high-level container runtime that manages the complete container lifecycle:

1. **Image Management**: Pulls, stores, and manages container images
2. **Container Execution**: Creates, starts, stops, and deletes containers
3. **Storage**: Manages container filesystem layers
4. **Network**: Prepares network namespaces (delegates actual networking to CNI)
5. **Runtime**: Calls low-level runtime (runc) to create Linux namespaces

### containerd Architecture

```
+-----------+
|  kubelet  |
+-----+-----+
      |
      | CRI (gRPC)
      v
+-----------+
| containerd|
+-----+-----+
      |
      +---> Image Service (pull, store, manage)
      |
      +---> Runtime Service (create, start, stop)
      |
      +---> Snapshots (layered filesystems)
      |
      +---> CNI (network setup)
      |
      v
+-----------+
|   runc    |
+-----------+
```

### CRI (Container Runtime Interface)

The **CRI** is a gRPC API that Kubernetes uses to talk to container runtimes. It decouples Kubernetes from specific runtime implementations.

Key CRI operations:
- `RunPodSandbox` → Creates pod network namespace and pause container
- `CreateContainer` → Creates a container in a pod
- `StartContainer` → Starts a created container
- `StopContainer` → Gracefully stops a container
- `RemovePodSandbox` → Cleans up a pod

### Why Not Docker?

Docker includes extra layers (dockerd, containerd-shim) that add complexity. containerd is:
- Simpler and lighter
- The default in Kubernetes since v1.24
- Directly implements CRI without shims

---

## Kubelet: The Node Agent

### What Does Kubelet Do?

The **kubelet** is the primary agent on each worker node:

1. **Registers the node** with the API server
2. **Watches for pods** assigned to this node
3. **Manages pod lifecycle** via containerd
4. **Monitors container health** (liveness/readiness probes)
5. **Reports node and pod status** back to the API server

### The Reconciliation Loop

```
+---------+
| Kubelet |
+----+----+
     |
     v
Register node with API server
     |
     v
Watch for Pod objects with spec.nodeName = this node
     |
     v
For each assigned pod:
  |
  +--> Call CRI to create PodSandbox (pause container)
  |
  +--> Call CNI to setup networking
  |
  +--> Call CRI to create app containers
  |
  +--> Monitor health
  |
     v
Report status to API server
     |
     v
Repeat forever
```

### Key Kubelet Configuration

| Field | Purpose |
|-------|---------|
| `authentication.webhook.enabled` | Verify API requests via webhook |
| `authorization.mode: Webhook` | Ask API server for authorization |
| `containerRuntimeEndpoint` | Where to find containerd socket |
| `registerNode: true` | Register this node automatically |
| `tlsCertFile` / `tlsPrivateKeyFile` | mTLS credentials |
| `cgroupDriver: systemd` | Use systemd cgroups (matches containerd) |

---

## Kube-Proxy: Service Networking

### What is a Kubernetes Service?

A **Service** provides a stable network endpoint for a set of pods:

```
Service: nginx (ClusterIP: 10.32.0.10)
     |
     +----> Pod nginx-xxx (10.200.0.5)
     +----> Pod nginx-yyy (10.200.1.8)
     +----> Pod nginx-zzz (10.200.0.12)
```

Pods can come and go, but the Service IP remains constant.

### How Kube-Proxy Implements Services

Kube-proxy watches Services and Endpoints in the API server, then creates iptables rules:

**ClusterIP Services**:
```
iptables -t nat -A KUBE-SERVICES \
  -d 10.32.0.10/32 -p tcp --dport 80 \
  -j KUBE-SVC-ABC123
```

This DNATs traffic to backend pods:
```
iptables -t nat -A KUBE-SVC-ABC123 \
  -j KUBE-SEP-XYZ789
```

**NodePort Services**:
```
iptables -t nat -A KUBE-NODEPORTS \
  -p tcp --dport 30080 \
  -j KUBE-SVC-ABC123
```

### iptables vs IPVS

| Mode | How It Works | Scale |
|------|--------------|-------|
| **iptables** | Creates iptables rules | Up to ~1,000 services |
| **IPVS** | Uses Linux kernel IPVS | Up to ~10,000+ services |

For this cluster, iptables is sufficient.

---

## CNI Plugins: Pod Networking

### What is CNI?

**CNI (Container Network Interface)** is a specification and library for configuring network interfaces in Linux containers.

When a pod is created, the CNI plugin:
1. Creates a network namespace for the pod
2. Creates a virtual Ethernet pair (veth)
3. Attaches one end to a bridge on the host
4. Assigns an IP address from the pod CIDR
5. Sets up routes

### The Bridge CNI Plugin

```
Host Network Namespace           Pod Network Namespace
+--------------------+          +--------------------+
| eth0 (AWS)         |          | eth0 (in pod)      |
| 172.31.19.88       |          | 10.200.0.5/24      |
+---------+----------+          +---------+----------+
          |                               |
          |                               |
    +-----+-----+                   +-----+-----+
    |   cni0    |<=================>| veth-xxxxx|
    |  bridge   |   veth pair       | (pod end) |
    |10.200.0.1 |                   |10.200.0.5 |
    +-----------+                   +-----------+
          |
          v
    iptables rules
    (MASQUERADE for outbound)
```

### CNI Config Fields

| Field | Purpose |
|-------|---------|
| `bridge: cni0` | Name of the Linux bridge on the host |
| `isGateway: true` | Bridge acts as default gateway |
| `ipMasq: true` | SNAT pod traffic leaving the node |
| `ipam.type: host-local` | Allocate IPs from local range |
| `ranges.subnet` | Pod CIDR for this node |

### Why Each Node Gets Different Subnet

- **node-1**: `10.200.0.0/24` (IPs: 10.200.0.2 - 10.200.0.254)
- **node-2**: `10.200.1.0/24` (IPs: 10.200.1.2 - 10.200.1.254)

This ensures pods across the cluster have unique IPs. In Step 7, we'll add routes so nodes know which subnet belongs to which node.

---

## br_netfilter and sysctl

### Why br_netfilter?

The **br_netfilter** kernel module enables iptables to see traffic that passes through Linux bridges.

Without br_netfilter:
```
Pod A (on node-1) -> Bridge -> Pod B (on node-1)
                    [iptables CANNOT see this traffic!]
```

With br_netfilter:
```
Pod A (on node-1) -> Bridge -> Pod B (on node-1)
                    [iptables CAN see and filter this traffic]
```

This is required for:
- Kubernetes NetworkPolicies
- Service load balancing
- kube-proxy iptables rules

### sysctl Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| `net.bridge.bridge-nf-call-iptables` | 1 | Enable iptables on bridged IPv4 traffic |
| `net.bridge.bridge-nf-call-ip6tables` | 1 | Enable iptables on bridged IPv6 traffic |

### Why ip_forward?

Pods need to reach services and external destinations. This requires IP forwarding (routing) on the host:

```
Pod (10.200.0.5) -> cni0 bridge -> host routing -> eth0 -> internet
```

If `ip_forward=0`, packets from pods are dropped at the host.

---

## Kubelet Registration Flow

When kubelet starts with `--register-node=true`:

```
1. Kubelet starts
      |
      v
2. Read kubeconfig from /var/lib/kubelet/kubeconfig
      |
      v
3. mTLS handshake with API server
      |  - Present kubelet certificate (CN=system:node:node-1)
      |  - Verify API server certificate
      |
      v
4. POST /api/v1/nodes
      |  {
      |    "kind": "Node",
      |    "metadata": { "name": "node-1" },
      |    "spec": {},
      |    "status": { ...capacity... }
      |  }
      |
      v
5. API server writes Node to etcd
      |
      v
6. Node object exists in cluster
      |
      v
7. Kubelet maintains heartbeat
      |  - Updates node status every few seconds
      |  - Renews lease in kube-node-lease namespace
      |
      v
8. Node appears in `kubectl get nodes`
```

### Node Status Conditions

| Condition | Meaning |
|-----------|---------|
| **Ready** | Kubelet is healthy and node can run pods |
| **MemoryPressure** | Node is running low on memory |
| **DiskPressure** | Node is running low on disk |
| **PIDPressure** | Node is running low on PIDs |
| **NetworkUnavailable** | Node network is not configured |

A node shows `NotReady` if:
- Kubelet stops responding
- CNI is not configured
- Out of disk/memory
- Network is down

---

## Understanding Kubelet Logs

### Common Log Messages

**"Attempting to register node"**
```
I0614 15:35:34.731646 kubelet_node_status.go:75] "Attempting to register node" node="node-1"
```
Good — kubelet is trying to register.

**"Unable to register node with API server"**
```
E0614 15:35:34.732325 kubelet_node_status.go:107] "Unable to register node with API server"
err="Post \"https://server.kubernetes.local:6443/api/v1/nodes\": dial tcp: lookup server.kubernetes.local: Temporary failure in name resolution"
```
Bad — kubelet can't resolve the API server hostname. Fix `/etc/hosts`.

**"Failed to contact API server"**
```
I0614 15:35:35.807378 csi_plugin.go:887] Failed to contact API server when waiting for CSINode publishing
```
Usually the same DNS issue. Kubelet tries to contact API server for multiple reasons.

**"NodeHasSufficientMemory / NodeHasNoDiskPressure / NodeHasSufficientPID"**
```
I0614 15:35:34.731619 kubelet_node_status.go:687] "Recording event message for node" node="node-1" event="NodeHasSufficientPID"
```
Informational — node status events.

**"Failed to ensure lease exists"**
```
E0614 15:35:41.492342 controller.go:145] "Failed to ensure lease exists, will retry"
err="Get \"https://server.kubernetes.local:6443/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/node-1\"
```
Still a connectivity/DNS issue. Kubelet can't renew its heartbeat lease.

---

## The Pause Container

### What is the Pause Container?

Every pod has a hidden **pause container** that exists solely to hold the pod's Linux namespaces.

```
Pod "nginx"
  ├── pause (registry.k8s.io/pause:3.10)
  │      ├── Network namespace
  │      ├── IPC namespace
  │      └── (Optional) PID namespace
  │
  └── nginx (your application)
```

### Why Do We Need It?

Linux namespaces must belong to a running process. If all app containers in a pod die, the namespaces would be destroyed. The pause container:
- Runs forever (does nothing but sleep)
- Owns the namespaces
- Acts as the parent of all pod containers

### The Pause Process

It's essentially this C code:

```c
#include <unistd.h>

int main() {
    while (1)
        pause();  // Sleep until signal arrives
    return 0;
}
```

The image is extremely small (~500KB) because it contains almost nothing.

### Why Did We Need to Pre-load It?

The pause image must be available before any pod is created. If workers can't reach the internet, they can't pull it. Hence we:
1. Pulled it on the jumpbox (which has internet)
2. Exported as a tarball
3. Copied to workers
4. Imported into containerd

---

## Worker vs Control Plane

| Aspect | Control Plane (server) | Worker (node-1, node-2) |
|--------|------------------------|------------------------|
| **Runs** | API server, scheduler, controller manager, etcd | kubelet, kube-proxy, containerd |
| **Stores state** | Yes (in etcd) | No (reads from API server) |
| **Schedules pods** | Yes (scheduler) | No (executes assigned pods) |
| **Needs certs** | API server cert, CA key | Kubelet cert, CA cert |
| **Needs kubeconfig** | admin, controller-manager, scheduler | kubelet, kube-proxy |
| **Public IP** | No | No |
| **Internet access** | No | No |

---

## FAQ

### Q: Can I use Docker instead of containerd?

A: Yes, but you need cri-dockerd shim. containerd is recommended as it's simpler and the Kubernetes default.

### Q: What is the difference between CRI and CNI?

A: **CRI** manages containers (containerd ↔ kubelet). **CNI** manages networking (bridge ↔ pod).

### Q: Why does kubelet need a kubeconfig?

A: To authenticate to the API server. The kubeconfig contains the kubelet's client certificate.

### Q: What happens if kubelet stops?

A: Pods keep running, but no new pods are started. Node status stops updating. After timeout, node is marked NotReady.

### Q: What is the kube-node-lease namespace?

A: Contains Lease objects for node heartbeats. Faster than updating Node status for liveness detection.

---

## Troubleshooting

### Issue: "mkdir: cannot create directory '/var/lib/kubelet/': Permission denied"

**Fix**: Use `sudo` for all directory creation and file moves.

### Issue: "Container runtime is down"

**Fix**: Check containerd status: `sudo systemctl status containerd`. Start if needed.

### Issue: Pods stuck in ContainerCreating

**Check**:
1. Is CNI config correct? `cat /etc/cni/net.d/10-bridge.conf`
2. Is br_netfilter loaded? `lsmod | grep br_netfilter`
3. Are sysctl settings correct? `sysctl net.bridge.bridge-nf-call-iptables`
4. Is pause image available? `sudo crictl images | grep pause`

---

## Related Concepts for Other Steps

- **Control Plane** (Step 5): Assigns pods to nodes, which kubelet then executes
- **Pod Routes** (Step 7): Complete pod-to-pod networking across nodes
- **kubectl** (Step 7): Remote access to the API server from jumpbox
- **Smoke Test** (Step 8): Validates that worker nodes actually run pods
