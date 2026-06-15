# Step 7: Pod Network Routes and kubectl Remote Access - Implementation

> This guide configures inter-node routing for pod-to-pod communication and sets up kubectl on the jumpbox for remote cluster management.

---

## Table of Contents

- [Overview](#overview)
- [1. Why Pod Network Routes Are Needed](#1-why-pod-network-routes-are-needed)
- [2. Add Pod CIDR Routes](#2-add-pod-cidr-routes)
- [3. Verify Pod Connectivity](#3-verify-pod-connectivity)
- [4. Configure kubectl on Jumpbox](#4-configure-kubectl-on-jumpbox)
- [5. Verify Cluster State](#5-verify-cluster-state)
- [6. Summary](#6-summary)

---

## Overview

After worker nodes are running, we must:
1. **Configure routes** so pods on different nodes can communicate
2. **Set up kubectl** on the jumpbox to manage the cluster remotely
3. **Verify** nodes become Ready and pods can communicate

### Network Topology

```
         +------------------+
         |    jumpbox       |
         |  (admin host)    |
         +--------+---------+
                  |
                  | SSH / kubectl
                  v
         +------------------+
         |     server       |
         |  (control plane) |
         +--------+---------+
                  |
      +-----------+-----------+
      |                       |
      v                       v
+-------------+         +-------------+
|   node-1    |         |   node-2    |
| 10.200.0.0/24|         | 10.200.1.0/24|
+-------------+         +-------------+
```

Each worker node's pods get IPs from their assigned CIDR:
- **node-1**: Pod CIDR `10.200.0.0/24` (e.g., pod IPs: 10.200.0.2, 10.200.0.3, ...)
- **node-2**: Pod CIDR `10.200.1.0/24` (e.g., pod IPs: 10.200.1.2, 10.200.1.3, ...)

For a pod on node-1 to reach a pod on node-2, node-1 needs a route saying "send 10.200.1.0/24 traffic to node-2."

---

## 1. Why Pod Network Routes Are Needed

By default, Linux nodes only know about their own directly connected networks:

```
node-1 routing table (before routes):
Destination     Gateway         Genmask         Flags   Iface
0.0.0.0         172.31.16.1     0.0.0.0         UG      eth0
172.31.16.0     0.0.0.0         255.255.240.0   U       eth0
10.200.0.0      0.0.0.0         255.255.255.0   U       cni0

# Problem: node-1 doesn't know how to reach 10.200.1.0/24!
# Packets to 10.200.1.x are dropped or sent to default gateway
```

Without routes, pod-to-pod communication across nodes fails:
```
Pod A on node-1 (10.200.0.5) -> Pod B on node-2 (10.200.1.8)
     |
     | "I need to send to 10.200.1.8"
     v
node-1 routing table
     |
     | "10.200.1.8? I don't know that network."
     | "Sending to default gateway (AWS VPC router)..."
     v
AWS VPC Router
     |
     | "10.200.1.8? This is a private pod IP, not routable in VPC."
     v
Packet DROPPED
```

The fix: Add explicit routes on each node for other nodes' pod CIDRs.

---

## 2. Add Pod CIDR Routes

On each worker node, add routes pointing to the other worker nodes.

### On node-1

Add a route to node-2's pod CIDR through node-2's private IP:

```bash
ssh node-1 "sudo ip route add 10.200.1.0/24 via <NODE_2_PRIVATE_IP>"
```

Verify:
```bash
ssh node-1 "ip route"
```

Expected output:
```
default via 172.31.16.1 dev eth0
172.31.16.0/20 dev eth0 proto kernel scope link src <NODE_1_PRIVATE_IP>
10.200.0.0/24 dev cni0 proto kernel scope link src 10.200.0.1
10.200.1.0/24 via <NODE_2_PRIVATE_IP> dev eth0
```

### On node-2

Add a route to node-1's pod CIDR through node-1's private IP:

```bash
ssh node-2 "sudo ip route add 10.200.0.0/24 via <NODE_1_PRIVATE_IP>"
```

Verify:
```bash
ssh node-2 "ip route"
```

Expected output:
```
default via 172.31.16.1 dev eth0
172.31.16.0/20 dev eth0 proto kernel scope link src <NODE_2_PRIVATE_IP>
10.200.1.0/24 dev cni0 proto kernel scope link src 10.200.1.1
10.200.0.0/24 via <NODE_1_PRIVATE_IP> dev eth0
```

### On the Control Plane (server)

For kubectl exec/logs/port-forward to work properly, the control plane also needs routes:

```bash
ssh server "sudo ip route add 10.200.0.0/24 via <NODE_1_PRIVATE_IP>"
ssh server "sudo ip route add 10.200.1.0/24 via <NODE_2_PRIVATE_IP>"
```

### Make Routes Persistent (Optional)

Routes added with `ip route` disappear on reboot. To persist them:

Create a systemd service or add to network configuration. For Debian:

```bash
ssh node-1 "echo 'up ip route add 10.200.1.0/24 via <NODE_2_PRIVATE_IP>' | sudo tee -a /etc/network/interfaces"
ssh node-2 "echo 'up ip route add 10.200.0.0/24 via <NODE_1_PRIVATE_IP>' | sudo tee -a /etc/network/interfaces"
```

Or use a simple systemd oneshot service:

```bash
cat > pod-routes.service <<EOF
[Unit]
Description=Add Pod CIDR routes
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/ip route add 10.200.1.0/24 via <NODE_2_PRIVATE_IP>
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

> **Note**: For production, use a proper CNI plugin (like Calico, Flannel, or Cilium) that handles routing automatically. Manual routes are sufficient for learning.

---

## 3. Verify Pod Connectivity

### Create Test Pods

Create a simple test pod on each node to verify connectivity:

```bash
kubectl run test-node-1 --image=busybox --restart=Never --overrides='{"spec":{"nodeName":"node-1"}}' -- sleep 3600 --kubeconfig admin.kubeconfig
kubectl run test-node-2 --image=busybox --restart=Never --overrides='{"spec":{"nodeName":"node-2"}}' -- sleep 3600 --kubeconfig admin.kubeconfig
```

Wait for pods to be Running:
```bash
kubectl get pods -o wide --kubeconfig admin.kubeconfig
```

Expected:
```
NAME          READY   STATUS    RESTARTS   AGE   IP           NODE
test-node-1   1/1     Running   0          30s   10.200.0.2   node-1
test-node-2   1/1     Running   0          30s   10.200.1.2   node-2
```

### Test Pod-to-Pod Communication

Ping from test-node-1 to test-node-2:

```bash
kubectl exec test-node-1 -- ping -c 3 10.200.1.2 --kubeconfig admin.kubeconfig
```

Expected:
```
PING 10.200.1.2 (10.200.1.2): 56 data bytes
64 bytes from 10.200.1.2: seq=0 ttl=62 time=1.234 ms
64 bytes from 10.200.1.2: seq=1 ttl=62 time=0.987 ms
64 bytes from 10.200.1.2: seq=2 ttl=62 time=1.056 ms

--- 10.200.1.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
```

Also test the reverse direction:
```bash
kubectl exec test-node-2 -- ping -c 3 10.200.0.2 --kubeconfig admin.kubeconfig
```

Clean up test pods:
```bash
kubectl delete pod test-node-1 test-node-2 --kubeconfig admin.kubeconfig
```

---

## 4. Configure kubectl on Jumpbox

The jumpbox should be able to manage the cluster using kubectl.

### Copy Admin Kubeconfig

From the server (or from jumpbox if you still have it):

```bash
scp admin@server:~/admin.kubeconfig ~/.kube/config
chmod 600 ~/.kube/config
```

Or if you generated it on the jumpbox in Step 3:
```bash
mkdir -p ~/.kube
cp admin.kubeconfig ~/.kube/config
chmod 600 ~/.kube/config
```

### Update Server Address

If the kubeconfig points to `https://127.0.0.1:6443`, update it to use the server's actual hostname:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig ~/.kube/config
```

### Test kubectl

```bash
kubectl cluster-info
```

Expected:
```
Kubernetes control plane is running at https://server.kubernetes.local:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```bash
kubectl get nodes
```

Expected:
```
NAME     STATUS   ROLES    AGE     VERSION
node-1   Ready    <none>   5m      v1.32.3
node-2   Ready    <none>   5m      v1.32.3
```

Nodes should now show `Ready` because:
- Kubelet is communicating with API server
- CNI is configured
- Routes are set up
- Pod networking is functional

---

## 5. Verify Cluster State

### Check All Nodes

```bash
kubectl get nodes -o wide
```

Expected:
```
NAME     STATUS   ROLES    AGE   VERSION   INTERNAL-IP    OS-IMAGE
node-1   Ready    <none>   10m   v1.32.3   <NODE_1_PRIVATE_IP>   Debian GNU/Linux 12
node-2   Ready    <none>   10m   v1.32.3   <NODE_2_PRIVATE_IP>  Debian GNU/Linux 12
```

### Check System Pods

```bash
kubectl get pods -n kube-system
```

Initially, there may be no pods in kube-system (we haven't deployed DNS or other addons).

### Check Node Conditions

```bash
kubectl describe node node-1 | grep -A 10 "Conditions"
```

Expected:
```
Conditions:
  Type             Status  LastHeartbeatTime                 Reason                       Message
  ----             ------  -----------------                 ------                       -------
  MemoryPressure   False   Sun, 14 Jun 2026 16:00:00 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sun, 14 Jun 2026 16:00:00 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sun, 14 Jun 2026 16:00:00 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Sun, 14 Jun 2026 16:00:00 +0000   KubeletReady                 kubelet is posting ready status
```

---

## 6. Summary

At the end of Step 7, confirm:

| Check | Command | Expected |
|-------|---------|----------|
| Route on node-1 | `ssh node-1 "ip route \| grep 10.200.1"` | `10.200.1.0/24 via <NODE_2_PRIVATE_IP>` |
| Route on node-2 | `ssh node-2 "ip route \| grep 10.200.0"` | `10.200.0.0/24 via <NODE_1_PRIVATE_IP>` |
| Pod-to-pod ping | `kubectl exec pod-a -- ping pod-b-ip` | 0% packet loss |
| kubectl works | `kubectl get nodes` | Shows both nodes |
| Nodes Ready | `kubectl get nodes` | STATUS = Ready |
| Node conditions | `kubectl describe node node-1` | Ready = True |

**Next**: Proceed to [Step 8: Smoke Test and Validation](../08-smoke-test-and-validation/08-implementation-steps.md) to fully validate the cluster with real workloads.
