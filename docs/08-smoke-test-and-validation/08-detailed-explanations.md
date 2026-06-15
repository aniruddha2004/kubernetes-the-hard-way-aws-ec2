# Step 8: Smoke Test and Validation - Detailed Explanations

> Deep-dive explanations for the smoke test: What happens when you create a deployment, how Kubernetes reconciles state, the role of each component, and what all the issues teach us.

---

## Table of Contents

- [What is a Smoke Test?](#what-is-a-smoke-test)
- [Secret Encryption Flow](#secret-encryption-flow)
- [Deployments, ReplicaSets, and Pods](#deployments-replicasets-and-pods)
- [The Reconciliation Loop in Action](#the-reconciliation-loop-in-action)
- [Scheduler Deep Dive](#scheduler-deep-dive)
- [Pod Sandbox and Pause Container](#pod-sandbox-and-pause-container)
- [Pod Creation Sequence](#pod-creation-sequence)
- [Container Runtime Flow](#container-runtime-flow)
- [Port Forwarding Mechanics](#port-forwarding-mechanics)
- [kubectl exec and logs](#kubectl-exec-and-logs)
- [NodePort Services](#nodeport-services)
- [Full Operational Chain](#full-operational-chain)

---

## What is a Smoke Test?

A **smoke test** is a basic set of tests that verify the most critical functionality works. The term comes from electronics: "power it on and see if smoke comes out."

In Kubernetes, a smoke test validates:
- All control plane components are functional
- Worker nodes can run pods
- Networking works (pod-to-pod, NodePort)
- Storage works (secrets encrypted)
- kubectl can manage the cluster remotely

A single `kubectl create deployment` exercises nearly every subsystem.

---

## Secret Encryption Flow

When you run:
```bash
kubectl create secret generic kubernetes-the-hard-way --from-literal="mykey=mydata"
```

This happens:

```
1. kubectl encodes "mydata" as base64: "bXlkYXRh"
     |
     v
2. kubectl sends JSON to API server:
   {
     "kind": "Secret",
     "data": { "mykey": "bXlkYXRh" }
   }
     |
     v
3. API Server authenticates (mTLS with admin cert)
     |
     v
4. API Server authorizes (RBAC: can admin create secrets?)
     |
     v
5. Admission Controllers run
     |
     v
6. API Server detects object type = Secret
     |
     v
7. API Server reads encryption-config.yaml
     |
     v
8. API Server encrypts payload with AES-CBC:
   Original:  {"mykey": "bXlkYXRh"}
   Encrypted: k8s:enc:aescbc:v1:key1:[encrypted bytes]
     |
     v
9. API Server writes to etcd:
   Key:   /registry/secrets/default/kubernetes-the-hard-way
   Value: k8s:enc:aescbc:v1:key1:[encrypted bytes]
     |
     v
10. API Server returns 201 Created to kubectl
```

When reading back:
```
kubectl get secret kubernetes-the-hard-way -o yaml
     |
     v
API Server fetches from etcd (encrypted)
     |
     v
API Server decrypts using same AES key
     |
     v
API Server returns plaintext JSON
     |
     v
kubectl displays base64-encoded data
```

---

## Deployments, ReplicaSets, and Pods

### Hierarchy

```
Deployment (nginx)
     |
     | manages
     v
ReplicaSet (nginx-7bf8c87b5)
     |
     | manages
     v
Pod (nginx-7bf8c87b5-abcde)
```

### Why Three Levels?

| Level | Responsibility |
|-------|---------------|
| **Deployment** | Declarative desired state, rolling updates, rollbacks, strategy |
| **ReplicaSet** | Ensures exact pod count, pod selector, no rolling logic |
| **Pod** | Actual running container(s) |

**Analogy**:
- Deployment = Project manager (decides WHAT and HOW MANY)
- ReplicaSet = Supervisor (ensures correct COUNT)
- Pod = Worker (does the ACTUAL WORK)

### Deployment Controller Flow

```
User: kubectl create deployment nginx --image=nginx:alpine --replicas=1
     |
     v
API Server stores Deployment object in etcd
     |
     v
Deployment Controller (in controller-manager) watches for Deployments
     |
     v
Deployment Controller sees new Deployment
     |
     v
Deployment Controller creates ReplicaSet:
   {
     "kind": "ReplicaSet",
     "spec": {
       "replicas": 1,
       "selector": { "matchLabels": {"app": "nginx"} },
       "template": { ... pod spec ... }
     }
   }
     |
     v
API Server stores ReplicaSet in etcd
     |
     v
ReplicaSet Controller watches for ReplicaSets
     |
     v
ReplicaSet Controller sees ReplicaSet with replicas=1, current=0
     |
     v
ReplicaSet Controller creates Pod object:
   {
     "kind": "Pod",
     "metadata": { "labels": {"app": "nginx"} },
     "spec": { "containers": [{"name": "nginx", "image": "nginx:alpine"}] }
   }
     |
     v
API Server stores Pod in etcd
     |
     v
Scheduler watches for unscheduled Pods
     |
     v
Scheduler assigns node (e.g., node-1)
     |
     v
Kubelet on node-1 sees Pod assigned to it
     |
     v
Kubelet creates pod via containerd
     |
     v
Pod Running!
```

---

## The Reconciliation Loop in Action

### What is Reconciliation?

Kubernetes never "runs commands." Instead, it continuously compares desired state with actual state and takes action to make them match.

```
Desired State          Actual State           Action
-------------          ------------           ------
1 nginx pod            0 nginx pods           Create 1 pod
1 nginx pod            1 nginx pod            Do nothing
1 nginx pod            2 nginx pods           Delete 1 pod
3 nginx pods           1 nginx pod            Create 2 pods
```

### Controllers Run Infinite Loops

```python
while True:
    desired = get_desired_state_from_api_server()
    actual = count_actual_pods_matching_selector()
    
    if actual < desired:
        create_pods(desired - actual)
    elif actual > desired:
        delete_pods(actual - desired)
    
    sleep(30)  # Resync period
```

This is why Kubernetes is **self-healing**:
- If a pod dies, actual < desired → controller creates replacement
- If you manually delete a pod, actual < desired → controller creates replacement
- If you manually create extra pods, actual > desired → controller deletes extras

---

## Scheduler Deep Dive

### What the Scheduler Actually Does

Contrary to popular belief, the scheduler does NOT:
- Create pods
- Start containers
- Allocate IPs

The scheduler ONLY adds `spec.nodeName` to unscheduled pods.

### Scheduling Stages

**Stage 1: Filtering (Predicates)**

Eliminate nodes that cannot run the pod:

| Predicate | Check |
|-----------|-------|
| PodFitsResources | Does node have enough CPU/memory? |
| PodFitsHost | Does pod specify a specific node? |
| PodFitsHostPorts | Are requested ports available? |
| MatchNodeSelector | Do node labels match pod's nodeSelector? |
| NoVolumeZoneConflict | Are pod's volumes available in node's zone? |
| NoDiskConflict | Will volumes conflict? |
| PodToleratesNodeTaints | Does pod tolerate node's taints? |

**Stage 2: Scoring (Priorities)**

Rank remaining nodes:

| Priority | What It Prefers |
|----------|----------------|
| LeastAllocated | Nodes with most free resources |
| BalancedAllocation | Nodes with balanced CPU/memory usage |
| SelectorSpread | Spread pods across zones/nodes |
| NodeAffinity | Nodes matching preferred affinity |
| TaintToleration | Nodes with tolerated taints |
| ImageLocality | Nodes that already have the image |

**Stage 3: Select**

Pick the highest-scoring node.

### Scheduling Example

```
Pod requests: 500m CPU, 256Mi memory

Available nodes:
  node-1: 1000m CPU free, 512Mi memory free
  node-2: 2000m CPU free, 1024Mi memory free
  node-3: 100m CPU free, 128Mi memory free

Filtering:
  node-3 eliminated (insufficient resources)

Scoring:
  node-1: score 70
  node-2: score 90 (more resources, more balanced)

Selection: node-2 wins!

Result:
  Pod.spec.nodeName = "node-2"
```

---

## Pod Sandbox and Pause Container

### The Problem: Linux Namespaces

Linux namespaces (network, IPC, PID) must belong to a running process. If all containers in a pod exit, the namespaces disappear.

### The Solution: Pause Container

Kubernetes creates a **pause container** that:
- Runs the smallest possible process
- Owns the pod's namespaces
- Lives for the entire lifetime of the pod
- Is the parent of all app containers

```
Pod
 ├── pause container (registry.k8s.io/pause:3.10)
 │      ├── Network namespace: netns-12345
 │      ├── IPC namespace: ipc-12345
 │      └── PID namespace (optional): pid-12345
 │
 ├── nginx container
 │      └── Shares netns-12345, ipc-12345
 │
 └── sidecar container (if any)
        └── Shares netns-12345, ipc-12345
```

### The Pause Process

```c
#include <unistd.h>
#include <signal.h>

// Signal handler does nothing
void sig_handler(int sig) {}

int main() {
    // Ignore most signals
    signal(SIGTERM, sig_handler);
    signal(SIGINT, sig_handler);
    
    while (1) {
        pause();  // Sleep until any signal arrives
    }
    
    return 0;
}
```

This binary is ~500KB and uses virtually no CPU or memory.

### Why "pause:3.10" Image Is Critical

containerd needs this image to create the sandbox. Without it:
```
FailedCreatePodSandbox: rpc error: code = Unknown desc = failed to get sandbox image "registry.k8s.io/pause:3.10": not found
```

This is why we pre-loaded it on worker nodes.

---

## Pod Creation Sequence

When you create a Deployment, here's the complete sequence:

```
Phase 1: API Server
1. User runs kubectl create deployment nginx
2. kubectl sends Deployment JSON to API server
3. API server authenticates, authorizes, validates
4. API server writes Deployment to etcd
5. kubectl returns "deployment.apps/nginx created"

Phase 2: Controller Manager
6. Deployment Controller sees new Deployment via watch
7. Deployment Controller creates ReplicaSet
8. API server writes ReplicaSet to etcd

Phase 3: ReplicaSet Controller
9. ReplicaSet Controller sees new ReplicaSet
10. ReplicaSet Controller creates Pod object
11. API server writes Pod to etcd (spec.nodeName is empty)

Phase 4: Scheduler
12. Scheduler sees unscheduled Pod via watch
13. Scheduler filters and scores nodes
14. Scheduler updates Pod with spec.nodeName = "node-1"
15. API server writes updated Pod to etcd

Phase 5: Kubelet
16. Kubelet on node-1 sees Pod assigned to it via watch
17. Kubelet calls containerd via CRI

Phase 6: Containerd
18. Containerd creates network namespace
19. Containerd calls CNI plugin (bridge)
20. CNI creates veth pair, assigns IP
21. Containerd pulls image (or uses cached)
22. Containerd creates pause container (sandbox)

Phase 7: App Container
23. Containerd creates app container (nginx)
24. Containerd starts nginx container

Phase 8: Status
25. Kubelet monitors container health
26. Kubelet reports Pod status to API server
27. API server updates etcd
28. kubectl get pod shows "Running"
```

Total time: 5-30 seconds depending on image pull speed.

---

## Container Runtime Flow

### CRI (Container Runtime Interface)

Kubelet does not start containers directly. It uses CRI, a gRPC API.

```
Kubelet
  |
  | gRPC /run/containerd/containerd.sock
  v
containerd (CRI plugin)
  |
  +---> Image Service
  |       PullImage("nginx:alpine")
  |
  +---> Runtime Service
  |       RunPodSandbox()
  |       CreateContainer()
  |       StartContainer()
  |
  v
containerd-shim-runc-v2
  |
  v
runc (OCI runtime)
  |
  v
Linux kernel (namespaces, cgroups)
  |
  v
Running container
```

### Why So Many Layers?

| Layer | Purpose |
|-------|---------|
| **Kubelet** | Orchestrates pods, monitors health |
| **CRI** | Standardized API, swappable runtimes |
| **containerd** | Image management, container lifecycle |
| **shim** | Allows containerd restart without stopping containers |
| **runc** | Creates actual Linux namespaces and cgroups |
| **kernel** | Provides isolation mechanisms |

The shim is critical: if containerd crashes and restarts, containers keep running because the shim maintains them.

---

## Port Forwarding Mechanics

### What kubectl port-forward Does

```bash
kubectl port-forward deployment/nginx 8080:80
```

Creates a tunnel:
```
Your laptop/jumpbox
     localhost:8080
          |
          v
    +-----------+
    |  kubectl  |  (runs on jumpbox)
    +-----+-----+
          |
          | SPDY/WebSocket tunnel
          | over HTTPS to API server
          v
    +-----------+
    | API Server|  (on server node)
    +-----+-----+
          |
          | HTTP proxy to kubelet
          v
    +-----------+
    |  Kubelet  |  (on pod's node)
    +-----+-----+
          |
          | CRI Exec/Portforward
          v
    +-----------+
    | containerd|
    +-----+-----+
          |
          v
    Pod container (nginx:80)
```

### Why SPDY?

SPDY (and now HTTP/2) supports multiplexing multiple streams over a single connection. This allows:
- Multiple port forwards simultaneously
- Bidirectional data flow
- Separate stdout/stderr/stdin streams for exec

---

## kubectl exec and logs

### kubectl exec

```bash
kubectl exec -it pod-name -- sh
```

Flow:
```
kubectl -> API Server -> Kubelet -> containerd -> runc -> container
```

The API server authenticates to kubelet using the `system:kube-apiserver-to-kubelet` ClusterRole we configured in Step 5.

### kubectl logs

```bash
kubectl logs pod-name
```

Flow:
```
kubectl -> API Server -> Kubelet -> reads container logs
```

Logs are stored on the node's filesystem (usually `/var/log/containers/` or containerd's log directory).

---

## NodePort Services

### How NodePort Works

When you create a NodePort service:

```
kubectl expose deployment nginx --port 80 --type NodePort
```

kube-proxy on EVERY node creates iptables rules:

```
# On node-1:
iptables -t nat -A KUBE-NODEPORTS -p tcp --dport 30080 -j KUBE-SVC-XXX
iptables -t nat -A KUBE-SVC-XXX -j KUBE-SEP-YYY
iptables -t nat -A KUBE-SEP-YYY -p tcp -j DNAT --to-destination 10.200.0.5:80

# On node-2:
iptables -t nat -A KUBE-NODEPORTS -p tcp --dport 30080 -j KUBE-SVC-XXX
iptables -t nat -A KUBE-SVC-XXX -j KUBE-SEP-ZZZ
iptables -t nat -A KUBE-SEP-ZZZ -p tcp -j DNAT --to-destination 10.200.1.8:80
```

This means:
- You can access the service via ANY node's IP on the NodePort
- Traffic is DNAT'd to a backend pod on ANY node
- kube-proxy load balances across backends

### Service Types Comparison

| Type | Accessibility | Use Case |
|------|--------------|----------|
| **ClusterIP** | Internal to cluster only | Internal microservices |
| **NodePort** | External via node IP:port | Development, simple exposure |
| **LoadBalancer** | External via cloud LB | Production, AWS ELB/NLB |
| **ExternalName** | DNS alias | External services |

---

## Full Operational Chain

Here's what happens when you curl a NodePort:

```
User: curl http://node-1:30080
     |
     v
node-1 eth0 (<NODE_1_PRIVATE_IP>)
     |
     v
iptables -t nat -A KUBE-NODEPORTS --dport 30080
     |
     v
iptables -t nat -A KUBE-SVC-XXX
     |
     v
iptables -t nat -A KUBE-SEP-YYY -j DNAT --to-destination 10.200.1.8:80
     |
     v
Kernel routes 10.200.1.8 via node-2 (<NODE_2_PRIVATE_IP>)
     |
     v
AWS VPC delivers packet to node-2
     |
     v
node-2 eth0 (<NODE_2_PRIVATE_IP>)
     |
     v
node-2 routing table: 10.200.1.0/24 -> cni0
     |
     v
cni0 bridge
     |
     v
Pod B veth interface
     |
     v
nginx container (10.200.1.8:80)
     |
     v
HTTP response!
```

And the response travels back along the reverse path.

---

## FAQ

### Q: Why does kubectl get pods show "ContainerCreating" for so long?

A: Usually image pulling. Check `kubectl describe pod` for events. If workers are offline, you need to pre-transfer images.

### Q: Why do I need to pre-load the pause image?

A: containerd requires it for every pod sandbox. Without internet access on workers, it must be pre-loaded.

### Q: Can I use a Deployment without a ReplicaSet?

A: No. Deployments always create/manage ReplicaSets. This separation allows rolling updates without downtime.

### Q: What happens if I delete a ReplicaSet?

A: The Deployment controller immediately creates a new one. The Deployment is the source of truth.

### Q: Why does NodePort use 30000-32767?

A: This range is reserved for NodePort services to avoid conflicts with well-known ports (<1024) and ephemeral ports (>32768).

### Q: How does the scheduler know which nodes are available?

A: It reads Node objects from the API server, which are updated by kubelets.

---

## Troubleshooting

### Issue: "ImagePullBackOff"

**Cause**: Worker can't pull image.
**Fix**: Pre-pull image on jumpbox and transfer, or configure private registry.

### Issue: "CrashLoopBackOff"

**Cause**: Container starts then crashes repeatedly.
**Fix**: Check logs: `kubectl logs pod-name`. Check command/args.

### Issue: "Pending" forever

**Cause**: Scheduler can't place pod (insufficient resources, taints, affinity).
**Fix**: `kubectl describe pod` check events.

### Issue: NodePort not accessible

**Cause**: Security group blocking port, or kube-proxy not running.
**Fix**: Check security group rules. Check `systemctl status kube-proxy`.

---

## Related Concepts

- **All Previous Steps**: The smoke test validates everything built in Steps 1-7
- **Production**: Would add CoreDNS, ingress, monitoring, HA control plane
