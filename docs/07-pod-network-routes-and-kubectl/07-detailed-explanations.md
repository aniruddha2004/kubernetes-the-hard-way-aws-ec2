# Step 7: Pod Network Routes and kubectl Remote Access - Detailed Explanations

> Deep-dive explanations for pod networking, inter-node routing, and kubectl remote access configuration.

---

## Table of Contents

- [Why Pod Network Routes Are Needed](#why-pod-network-routes-are-needed)
- [Pod IP Allocation](#pod-ip-allocation)
- [The Routing Problem](#the-routing-problem)
- [The Solution: Static Routes](#the-solution-static-routes)
- [Packet Flow: Pod-to-Pod Across Nodes](#packet-flow-pod-to-pod-across-nodes)
- [CNI vs Routes: What's the Difference?](#cni-vs-routes-whats-the-difference)
- [kubectl Remote Access](#kubectl-remote-access)
- [Cluster Networks Summary](#cluster-networks-summary)
- [Making Routes Persistent](#making-routes-persistent)
- [FAQ](#faq)

---

## Why Pod Network Routes Are Needed

### The Kubernetes Networking Model

Kubernetes mandates three fundamental networking requirements:

1. **All pods can communicate with all other pods** (without NAT)
2. **All nodes can communicate with all pods** (and vice versa)
3. **Pods see their own IP as others see it** (no IP masquerading within cluster)

Requirement #1 means a pod on node-1 must directly reach a pod on node-2 using the pod's actual IP.

### Without Routes

```
node-1 knows:
  - 172.31.16.0/20 (VPC network, local)
  - 10.200.0.0/24 (local pod CIDR)
  
node-1 does NOT know:
  - 10.200.1.0/24 (node-2's pod CIDR)
  
Result: Packets to 10.200.1.x sent to default gateway (AWS VPC router)
        VPC router doesn't know about 10.200.1.x
        Packet is DROPPED
```

### With Routes

```
node-1 knows:
  - 172.31.16.0/20 (VPC network)
  - 10.200.0.0/24 (local pod CIDR)
  - 10.200.1.0/24 via <NODE_2_PRIVATE_IP> (node-2)
  
node-2 knows:
  - 172.31.16.0/20 (VPC network)
  - 10.200.1.0/24 (local pod CIDR)
  - 10.200.0.0/24 via <NODE_1_PRIVATE_IP> (node-1)
  
Result: Pod-to-pod communication works!
```

---

## Pod IP Allocation

### How Pods Get IPs

When a pod is created on a node:

```
1. Kubelet asks containerd to create pod sandbox
2. Containerd calls CNI plugin (bridge)
3. CNI plugin calls host-local IPAM
4. host-local allocates next available IP from node's CIDR
5. IP is assigned to pod's network interface
```

Example allocation on node-1 (CIDR: 10.200.0.0/24):
```
Pod nginx-xxx    -> 10.200.0.2
Pod redis-yyy    -> 10.200.0.3
Pod mysql-zzz    -> 10.200.0.4
...              -> ...
Gateway (cni0)   -> 10.200.0.1
Broadcast        -> 10.200.0.255
```

### IPAM (IP Address Management)

The `host-local` IPAM plugin:
- Stores allocations in `/var/lib/cni/networks/<network-name>/`
- Each allocated IP has a lock file
- Reuses IPs when pods are deleted
- Never allocates .0 (network) or .255 (broadcast)

---

## The Routing Problem

### Linux Routing Basics

A Linux routing table is like a map:

```
Destination     Gateway         Iface    Meaning
-----------     -------         -----    -------
10.200.0.0/24   0.0.0.0         cni0     "These IPs are on my bridge"
172.31.16.0/20  0.0.0.0         eth0     "These IPs are on my VPC"
0.0.0.0/0       172.31.16.1     eth0     "Everything else goes to AWS router"
```

When a packet is sent:
1. Kernel checks destination IP
2. Finds the most specific matching route
3. Sends packet accordingly

### The Missing Piece

For cross-node pod traffic:

```
Pod A (10.200.0.5) on node-1 sends to Pod B (10.200.1.8)

Kernel on node-1:
  "Destination is 10.200.1.8"
  "Looking for route..."
  "10.200.0.0/24? No, that's different."
  "172.31.16.0/20? No."
  "0.0.0.0/0? Yes! Default route!"
  "Send to 172.31.16.1 (AWS router)"
  
AWS Router:
  "10.200.1.8? I don't know this private range."
  "DROP"
```

---

## The Solution: Static Routes

### Adding the Route

```bash
ip route add 10.200.1.0/24 via <NODE_2_PRIVATE_IP>
```

Translation: "To reach 10.200.1.0/24, send packets to <NODE_2_PRIVATE_IP> (node-2)."

Now the routing table:
```
Destination     Gateway           Iface
10.200.0.0/24   0.0.0.0           cni0
10.200.1.0/24   <NODE_2_PRIVATE_IP>     eth0   <-- NEW
172.31.16.0/20  0.0.0.0           eth0
0.0.0.0/0       172.31.16.1       eth0
```

### Route Priority

More specific routes win:
- `10.200.1.0/24` (24 bits) beats `0.0.0.0/0` (0 bits)
- So 10.200.1.8 goes to node-2, not the default gateway

---

## Packet Flow: Pod-to-Pod Across Nodes

Complete journey of a packet from Pod A (node-1) to Pod B (node-2):

```
Pod A (10.200.0.5)
    |
    | "Send to 10.200.1.8"
    v
+----------------------------------+
| Pod A's network namespace        |
| Default route: 10.200.0.1 (cni0) |
+----------------------------------+
    |
    v
+----------------------------------+
| cni0 bridge on node-1            |
| Sees destination is not local    |
| Sends to host routing table      |
+----------------------------------+
    |
    v
+----------------------------------+
| node-1 routing table             |
| Matches: 10.200.1.0/24           |
| Gateway: <NODE_2_PRIVATE_IP>           |
| Next hop: node-2's eth0          |
+----------------------------------+
    |
    | Packet travels through AWS VPC
    | Source: 10.200.0.5, Dest: 10.200.1.8
    v
+----------------------------------+
| node-2 eth0 (<NODE_2_PRIVATE_IP>)      |
| Receives packet                  |
+----------------------------------+
    |
    v
+----------------------------------+
| node-2 routing table             |
| Matches: 10.200.1.0/24           |
| Interface: cni0                  |
+----------------------------------+
    |
    v
+----------------------------------+
| cni0 bridge on node-2            |
| Forwards to correct veth         |
+----------------------------------+
    |
    v
+----------------------------------+
| Pod B's network namespace        |
| Pod B (10.200.1.8) receives!     |
+----------------------------------+
```

### TTL (Time To Live)

Each router hop decrements TTL. Initial TTL is usually 64:
- Pod A -> cni0 bridge: TTL 64
- cni0 -> node-1 host: TTL 64
- node-1 -> node-2 via VPC: TTL 63
- node-2 -> cni0: TTL 63
- cni0 -> Pod B: TTL 62

So `ping` shows `ttl=62` for cross-node traffic (2 hops).

---

## CNI vs Routes: What's the Difference?

| Aspect | CNI | Routes |
|--------|-----|--------|
| **Scope** | Within a single node | Between nodes |
| **When** | Pod creation | Node initialization |
| **Who** | CNI plugin (bridge) | Administrator or overlay network |
| **What** | Sets up veth, bridge, IP | Tells kernel where to send packets |
| **Tools** | bridge, host-local, loopback | ip route |

### Analogy

- **CNI** is like wiring the Ethernet cables inside a building
- **Routes** are like road signs between buildings

You need both for complete connectivity.

---

## kubectl Remote Access

### Why Configure kubectl on the Jumpbox?

Until now, we've run kubectl on the server node (via SSH). For convenience and to simulate real-world usage:
- Admin workstation (jumpbox) manages the cluster remotely
- No need to SSH into the control plane for routine operations

### The Kubeconfig Contains Everything

```yaml
clusters:
- cluster:
    certificate-authority-data: ...  # Trust the API server
    server: https://server.kubernetes.local:6443  # Where to connect

users:
- name: admin
  user:
    client-certificate-data: ...  # Prove I'm admin
    client-key-data: ...          # Sign TLS handshake
```

### Updating the Server Address

If the kubeconfig was created with `https://127.0.0.1:6443` (for use on the server itself), we must update it to the server's resolvable hostname for remote access:

```bash
kubectl config set-cluster kubernetes-the-hard-way \
  --server=https://server.kubernetes.local:6443
```

### Verifying kubectl Access

```bash
kubectl cluster-info
```

This tests:
1. Can resolve `server.kubernetes.local`?
2. Is port 6443 reachable?
3. Does TLS handshake succeed?
4. Is the client certificate accepted?
5. Is the user authorized?

---

## Cluster Networks Summary

| Network | CIDR | Purpose | Where Configured |
|---------|------|---------|-----------------|
| **VPC Network** | 172.31.16.0/20 | AWS EC2 instance IPs | AWS VPC |
| **Pod CIDR (node-1)** | 10.200.0.0/24 | Pods on node-1 | CNI config, kubelet |
| **Pod CIDR (node-2)** | 10.200.1.0/24 | Pods on node-2 | CNI config, kubelet |
| **Cluster CIDR** | 10.200.0.0/16 | All pods aggregate | kube-proxy, controller-manager |
| **Service CIDR** | 10.32.0.0/24 | Kubernetes Services | API server, kube-proxy |
| **Service ClusterIP** | 10.32.0.1 | kubernetes.default service | Automatic |
| **DNS Service IP** | 10.32.0.10 | CoreDNS (if installed) | kubelet config |
| **NodePort Range** | 30000-32767 | Exposed service ports | API server flag |

---

## Making Routes Persistent

### Why Routes Disappear

Routes added with `ip route` are stored in memory. They disappear:
- On reboot
- On network interface reset
- Sometimes on DHCP lease renewal

### Persistence Options

**Option 1: /etc/network/interfaces** (Debian/Ubuntu traditional)
```
auto eth0
iface eth0 inet dhcp
    up ip route add 10.200.1.0/24 via <NODE_2_PRIVATE_IP>
```

**Option 2: systemd oneshot service**
```ini
[Unit]
Description=Add Pod CIDR routes
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/sbin/ip route add 10.200.1.0/24 via <NODE_2_PRIVATE_IP>
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

**Option 3: Cloud-init** (for automated provisioning)
```yaml
runcmd:
  - ip route add 10.200.1.0/24 via <NODE_2_PRIVATE_IP>
```

### Production Alternative: Overlay Networks

In production, use CNI plugins that handle routing automatically:

| CNI Plugin | How It Works |
|-----------|--------------|
| **Calico** | BGP peering between nodes, distributes routes |
| **Flannel** | VXLAN encapsulation, creates overlay network |
| **Cilium** | eBPF-based routing, can use BGP or tunnel |
| **Weave Net** | Mesh overlay with encrypted tunnels |

These eliminate the need for manual routes.

---

## FAQ

### Q: Can pods on different nodes reach each other without routes?

A: No. Without routes, packets go to the default gateway which doesn't know about pod CIDRs.

### Q: Why not use NAT for pod-to-pod communication?

A: Kubernetes networking model forbids NAT for pod-to-pod traffic. It would break many applications that rely on direct pod IPs.

### Q: Do I need routes on the control plane node too?

A: Yes, if you want `kubectl exec`, `kubectl logs`, or `kubectl port-forward` to work properly. The API server proxies traffic to pods.

### Q: What about IPv6?

A: This guide uses IPv4. Kubernetes supports IPv6 dual-stack, but requires additional configuration.

### Q: Why does `ip route` show `via`?

A: `via` means "send to this gateway/router." Packets are forwarded to that IP, which must be on a directly connected network.

### Q: What is the difference between `ip route` and `route`?

A: `ip route` is the modern Linux command (iproute2). `route` is the older BSD-style command. Use `ip route`.

---

## Troubleshooting

### Issue: "Network is unreachable" when pinging pod IPs

**Check**:
1. Are routes in place? `ip route`
2. Is br_netfilter loaded? `lsmod | grep br_netfilter`
3. Is ip_forward enabled? `sysctl net.ipv4.ip_forward`
4. Are security groups allowing inter-node traffic?

### Issue: kubectl says "Unable to connect to the server"

**Check**:
1. Is `server.kubernetes.local` in `/etc/hosts` on the jumpbox?
2. Is the API server listening? `ss -tlnp | grep 6443`
3. Is the kubeconfig correct? `cat ~/.kube/config | grep server`

---

## Related Concepts for Other Steps

- **Worker Nodes** (Step 6): kubelet and CNI create pods with IPs from the CIDR
- **Services** (Step 8): kube-proxy uses iptables to load balance to pod IPs
- **Production**: Consider Calico or Flannel for automatic route distribution
