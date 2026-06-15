# Step 5: Control Plane Bootstrap - Detailed Explanations

> Deep-dive explanations for the Kubernetes control plane components: kube-apiserver, kube-controller-manager, kube-scheduler, the authorization pipeline, admission controllers, RBAC, leader election, and the real-world DNS/certificate mismatch error encountered during this setup.

---

## Table of Contents

- [What is the Control Plane?](#what-is-the-control-plane)
- [Control Plane Architecture](#control-plane-architecture)
- [kube-apiserver Deep Dive](#kube-apiserver-deep-dive)
- [API Server Startup Sequence](#api-server-startup-sequence)
- [The Authorization Pipeline](#the-authorization-pipeline)
- [Admission Controllers](#admission-controllers)
- [kube-controller-manager Deep Dive](#kube-controller-manager-deep-dive)
- [Controller Reconciliation Loop](#controller-reconciliation-loop)
- [Cluster Signing and CSR Flow](#cluster-signing-and-csr-flow)
- [kube-scheduler Deep Dive](#kube-scheduler-deep-dive)
- [Scheduling Algorithm](#scheduling-algorithm)
- [Leader Election](#leader-election)
- [RBAC Deep Dive](#rbac-deep-dive)
- [The system:kube-apiserver-to-kubelet Role](#the-systemkube-apiserver-to-kubelet-role)
- [DNS/Certificate Validation and the Hostname Mismatch Error](#dnscertificate-validation-and-the-hostname-mismatch-error)
- [etcd as the Control Plane's Database](#etcd-as-the-control-planes-database)
- [Service Cluster IP Range and Kubernetes Service](#service-cluster-ip-range-and-kubernetes-service)
- [Audit Logging](#audit-logging)
- [FAQ](#faq)
- [Troubleshooting](#troubleshooting)

---

## What is the Control Plane?

The **control plane** is the decision-making layer of Kubernetes. It is responsible for:

1. **Storing cluster state** — What pods exist? Where do they run? What services expose them?
2. **Making scheduling decisions** — Which node should run a new pod?
3. **Maintaining desired state** — If a pod crashes, create a replacement. If a node fails, reschedule its pods.
4. **Enforcing policies** — Who can do what? Are resource limits respected?
5. **Exposing the API** — The unified interface for ALL cluster operations.

Without the control plane, worker nodes cannot be managed, new workloads cannot be deployed, and the cluster cannot heal itself.

### Control Plane vs Worker Plane

| Aspect | Control Plane | Worker Plane |
|--------|---------------|--------------|
| **Purpose** | Decision making | Workload execution |
| **Components** | API server, controller manager, scheduler, etcd | kubelet, kube-proxy, container runtime, CNI |
| **Location** | `server` node (this tutorial) | `node-1`, `node-2` |
| **Can it run pods?** | Technically yes (with taints removed), but not recommended | Yes, primary purpose |
| **Critical for operation?** | Yes — cluster stops without it | No — existing pods keep running |

---

## Control Plane Architecture

### High-Level Architecture

```
                    +--------------------+
                    |    kubectl / CI     |
                    |   (Human/External)  |
                    +---------+----------+
                              |
                              | HTTPS + mTLS
                              v
                    +--------------------+
                    |   kube-apiserver   |
                    |     (Port 6443)    |
                    |  The Front Door    |
                    +--+--+----------+---+
                       |               |
           +-----------+               +-----------+
           | HTTPS                     | HTTPS
           v                           v
    +-------------+            +---------------------+
    |  etcd       |            |kube-controller-manager|
    | (Database)  |            |   (State Manager)   |
    +-------------+            +---------------------+
                                       |
                                       | Watches/Updates
                                       v
                              +--------------------+
                              |   kube-scheduler   |
                              | (Placement Engine) |
                              +--------------------+
```

### Request Flow Through the Control Plane

When you run `kubectl apply -f pod.yaml`, here's what happens:

```
1. kubectl reads ~/.kube/config (admin.kubeconfig)
   - Finds cluster: https://server.kubernetes.local:6443
   - Finds client cert: admin.crt
   - Establishes mTLS connection

2. kube-apiserver receives the request
   - TLS termination (verifies client cert against ca.crt)
   - Authentication: "Who is this?" -> CN=admin, O=system:masters
   - Authorization: "What can admin do?" -> system:masters can do everything
   - Admission: "Should we allow this?" -> NamespaceLifecycle, LimitRanger, etc.
   - Validation: Does the Pod spec conform to the API schema?
   - Storage: Writes to etcd at /registry/pods/default/my-pod

3. kube-scheduler detects the new unscheduled pod
   - Watches /registry/pods for pods with spec.nodeName empty
   - Scores all nodes based on resources, affinities, taints
   - Selects best node (e.g., node-1)
   - Updates pod with spec.nodeName = "node-1"
   - API server writes update to etcd

4. kubelet on node-1 sees the assignment
   - Watches for pods assigned to its node
   - Starts the container via containerd
   - Reports status back to API server

5. kube-controller-manager ensures desired state
   - ReplicaSet controller: "ReplicaSet wants 3 pods, I see 3 pods. OK."
   - If a pod dies: "Only 2 pods. Create 1 more."
```

---

## kube-apiserver Deep Dive

### What kube-apiserver Does

The **kube-apiserver** (often called just "API server") is the **central management entity** and the **only component that talks directly to etcd**. Every other component — controllers, schedulers, kubelets, kubectl — communicates exclusively with the API server.

### Responsibilities

| Responsibility | Description |
|----------------|-------------|
| **API Exposure** | Serves the Kubernetes REST API on port 6443 |
| **Authentication** | Verifies client identity via TLS certificates |
| **Authorization** | Determines if the authenticated identity can perform the requested action |
| **Admission Control** | Mutates and/or validates requests before they are persisted |
| **etcd Proxy** | Reads and writes all cluster state to/from etcd |
| **Watch Broadcasting** | Broadcasts change events to watchers (controllers, kubelets) |
| **API Versioning** | Handles multiple API versions (v1, apps/v1, etc.) |

### Why Only the API Server Talks to etcd

```
Wrong (components talking directly to etcd):
  +--------+      +--------+      +--------+
  |kubectl |      |controller|    |kubelet |
  +---+----+      +----+---+      +---+----+
      |                |               |
      | etcd?          | etcd?         | etcd?
      v                v               v
  +----------------------------------------+
  |               etcd                     |
  +----------------------------------------+

Correct (all through API server):
  +--------+      +--------+      +--------+
  |kubectl |      |controller|    |kubelet |
  +---+----+      +----+---+      +---+----+
      |                |               |
      | HTTPS          | HTTPS         | HTTPS
      v                v               v
  +----------------------------------------+
  |         kube-apiserver                 |
  |    (auth, authz, admission)            |
  +----------------------------------------+
      |
      | etcd protocol
      v
  +----------------------------------------+
  |               etcd                     |
  +----------------------------------------+
```

Centralizing etcd access through the API server ensures:
- **Uniform policy enforcement**: Auth, authz, and admission apply to EVERY request
- **Schema validation**: All data conforms to API object schemas
- **Consistency**: No component can bypass the API to modify state
- **Watch efficiency**: The API server multiplexes watches efficiently

### API Server Is Stateless

The API server does NOT store any persistent state itself. All state is in etcd. This means:
- You can run multiple API servers behind a load balancer
- API servers are horizontally scalable
- Losing an API server does not lose data
- API servers can be restarted safely at any time

---

## API Server Startup Sequence

When `systemctl start kube-apiserver` runs, here's what happens internally:

### Phase 1: Parse Configuration (0-2 seconds)

```
1. Parse command-line flags
   - Read all --flag=value arguments
   - Validate required flags (etcd-servers, tls-cert-file, etc.)

2. Load configuration files
   - Read encryption-config.yaml
   - Load TLS certificates (kubernetes.crt, kubernetes.key, ca.crt)
   - Verify certificates are valid and not expired

3. Validate network configuration
   - Check service-cluster-ip-range is valid CIDR
   - Verify service-node-port-range (30000-32767)
```

### Phase 2: Initialize REST Storage (2-5 seconds)

```
4. Register API groups
   - core/v1 (pods, services, nodes, etc.)
   - apps/v1 (deployments, replica sets, daemon sets)
   - rbac.authorization.k8s.io/v1 (roles, bindings)
   - ... and many more

5. Initialize storage backends
   - For each resource type, create an etcd-backed storage
   - Set up encoding/decoding (JSON <-> protobuf)
   - Configure watches for each resource type
```

### Phase 3: Connect to etcd (5-10 seconds)

```
6. Establish etcd connection
   - Dial https://127.0.0.1:2379
   - Perform TLS handshake using etcd-cafile, etcd-certfile, etcd-keyfile
   - Verify etcd is healthy

7. Check etcd data version
   - Ensure etcd data is compatible with this API server version
   - Run any required data migrations
```

### Phase 4: Initialize Subsystems (10-20 seconds)

```
8. Start admission controllers
   - Initialize NamespaceLifecycle
   - Initialize NodeRestriction
   - Initialize LimitRanger
   - Initialize ServiceAccount
   - Initialize DefaultStorageClass
   - Initialize ResourceQuota
   - ... and any others

9. Start informers
   - Internal caches for frequently accessed resources
   - Pre-populate caches from etcd

10. Start aggregation layer (if configured)
    - Allows extending the API with custom resources (CRDs)
```

### Phase 5: Start Serving (20-30 seconds)

```
11. Bind to port 6443
    - Open TLS listener on 0.0.0.0:6443
    - Log: "Serving securely on 0.0.0.0:6443"

12. Start health endpoints
    - /healthz — Overall health
    - /livez — Liveness probe
    - /readyz — Readiness probe
    - /metrics — Prometheus metrics

13. Notify systemd (Type=notify)
    - Signal that the service is ready
    - systemd marks it as active
```

### Startup Timeline

```
Time:  0s    5s    10s   15s   20s   25s   30s
       |     |     |     |     |     |     |
       [Parse][REST ][etcd ][Subsys][Serve]
       [Flags][Store][Conn][Init  ]

Logs you might see:
  I0614 11:00:00.12345    5678 flags.go:59] FLAG: --advertise-address="172.31.20.244"
  I0614 11:00:02.23456    5678 client.go:360] parsed scheme: "endpoint"
  I0614 11:00:05.34567    5678 storage_factory.go:285] storing {pods ...} in v1, reading as __internal from etcd3
  I0614 11:00:15.45678    5678 plugins.go:158] Loaded 6 mutating admission controller(s)
  I0614 11:00:25.56789    5678 secure_serving.go:210] Serving securely on 0.0.0.0:6443
```

---

## The Authorization Pipeline

Every request to the API server passes through a pipeline of checks. This is the security backbone of Kubernetes.

### The Three Gates

```
        Request arrives at API server
                    |
                    v
        +-----------------------+
   1    |     Authentication    |  <- "Who are you?"
        |  (mTLS, token, etc.)  |
        +-----------+-----------+
                    |
                    | Identity: User="admin", Groups=["system:masters"]
                    v
        +-----------------------+
   2    |     Authorization     |  <- "What can you do?"
        |  (Node, RBAC, ABAC)   |
        +-----------+-----------+
                    |
                    | Decision: Allowed
                    v
        +-----------------------+
   3    |   Admission Control   |  <- "Should we allow this?"
        | (Mutating + Validating) |
        +-----------+-----------+
                    |
                    | Object is modified/validated
                    v
               Persist to etcd
```

### Gate 1: Authentication

Authentication answers: **"Who is making this request?"**

Methods supported by our API server:

| Method | How It Works | Used By |
|--------|-------------|---------|
| **X.509 Client Certificates** | mTLS — the client's certificate CN becomes the username, O fields become groups | kubelets, controllers, kubectl |
| **Service Account Tokens** | JWT tokens signed by the service account key, presented in Authorization header | Pods running in the cluster |
| **Webhook Token** | Delegates authentication to an external service | Enterprise setups |

In our setup, we use **mTLS exclusively** for component-to-component communication. Each component's certificate identifies it:

| Certificate | CN (Username) | O (Group) |
|-------------|---------------|-----------|
| admin.crt | `admin` | `system:masters` |
| kubernetes.crt | `kubernetes` | (none) |
| node-1.crt | `system:node:node-1` | `system:nodes` |
| node-2.crt | `system:node:node-2` | `system:nodes` |
| kube-controller-manager.crt | `system:kube-controller-manager` | (none) |
| kube-scheduler.crt | `system:kube-scheduler` | (none) |

The `--client-ca-file=/var/lib/kubernetes/ca.crt` flag tells the API server which CA to use for validating client certificates.

### Gate 2: Authorization

Authorization answers: **"Is this identity allowed to perform this action on this resource?"**

Modes enabled by `--authorization-mode=Node,RBAC`:

#### Node Authorizer

Special-purpose authorizer for **kubelets only**.

```
Node Authorizer Logic:
  IF user.name starts with "system:node:" THEN
    ALLOW only operations on:
      - The node object with matching name
      - Pods bound to that node
      - Secrets/configmaps referenced by those pods
    DENY everything else
  ELSE
    Pass to next authorizer
```

This prevents `node-1`'s kubelet from reading `node-2`'s pods or modifying other nodes.

#### RBAC (Role-Based Access Control)

Flexible, policy-based authorization using Kubernetes resources.

```
RBAC Logic:
  1. Look up all Roles/ClusterRoles that match the request
  2. Look up all RoleBindings/ClusterRoleBindings referencing those roles
  3. Check if the requesting user (or group) appears in any binding
  4. If yes, check if the role grants the specific verb on the specific resource
  5. ALLOW if match found; DENY otherwise
```

### Gate 3: Admission Control

Admission control answers: **"Should this request be allowed, modified, or rejected?"**

See [Admission Controllers](#admission-controllers) for full details.

---

## Admission Controllers

Admission controllers are plugins that intercept requests to the API server **after** authentication and authorization but **before** the object is persisted to etcd.

### Types of Admission Controllers

```
Request passes authn/authz
          |
          v
   +----------------+
   |  Mutating      |  <- Can MODIFY the object
   |  Admisson      |
   +--------+-------+
            |
            | Object may be changed
            v
   +----------------+
   |  Validating    |  <- Can only ACCEPT or REJECT
   |  Admission     |
   +--------+-------+
            |
            | Object is finalized
            v
       Persist to etcd
```

| Type | Can Modify Object? | Can Reject? | Examples |
|------|-------------------|-------------|----------|
| **Mutating** | Yes | Yes | ServiceAccount, DefaultStorageClass |
| **Validating** | No | Yes | ResourceQuota, LimitRanger |

### Our Enabled Admission Controllers

Enabled by `--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota`:

#### NamespaceLifecycle

```
What it does:
  - Prevents creation of objects in non-existent namespaces
  - Prevents deletion of namespaces that have active resources
  - Prevents updates to objects in namespaces that are being terminated

Example:
  User tries: kubectl create pod -n deleted-ns
  Rejected with: "namespace deleted-ns is being terminated"
```

#### NodeRestriction

```
What it does:
  - Limits kubelet to only modify its OWN node object
  - Limits kubelet to only modify pods bound to its node
  - Prevents kubelets from seeing secrets they don't need

Example:
  node-1's kubelet tries to modify node-2's status
  Rejected: Forbidden
```

#### LimitRanger

```
What it does:
  - Enforces default resource limits per namespace
  - Validates that resource requests don't exceed limits
  - Can assign default CPU/memory requests to pods without them

Example:
  Namespace has default memory limit of 512Mi
  User creates pod without memory limit
  Admission controller adds: resources.limits.memory = 512Mi
```

#### ServiceAccount

```
What it does:
  - Auto-creates service accounts if they don't exist
  - Auto-mounts service account tokens into pods
  - Ensures every pod has a service account identity

Example:
  User creates pod without specifying serviceAccountName
  Admission controller adds: serviceAccountName = "default"
  And mounts the default token at /var/run/secrets/kubernetes.io/serviceaccount
```

#### DefaultStorageClass

```
What it does:
  - Adds a default storage class to PersistentVolumeClaims that don't specify one
  - Enables dynamic provisioning without explicit class selection

Example:
  User creates PVC without storageClassName
  Admission controller adds: storageClassName = "standard" (or whatever is default)
```

#### ResourceQuota

```
What it does:
  - Tracks cumulative resource usage per namespace
  - Rejects requests that would exceed namespace quotas
  - Counts pods, services, PVCs, CPU, memory, etc.

Example:
  Namespace quota: 10 pods
  User tries to create 11th pod
  Rejected: "exceeded quota: pod-quota, requested: pods=1, used: pods=10, limited: pods=10"
```

### Why These Six?

Together, these six admission controllers provide:
- **Safety** (NamespaceLifecycle, NodeRestriction)
- **Resource governance** (LimitRanger, ResourceQuota)
- **Operational convenience** (ServiceAccount, DefaultStorageClass)

They are the **default recommended set** for a secure, well-behaved cluster.

---

## kube-controller-manager Deep Dive

### What kube-controller-manager Does

The **kube-controller-manager** is a single binary that bundles many **controllers** — background processes that watch the cluster state and take action to move it toward the desired state.

Think of controllers as **thermostats**: they continuously measure the current temperature and adjust the heater/cooler to reach the desired temperature.

### Controller Pattern

```
Desired State (from user/YAML):
  "I want 3 replicas of nginx"

Current State (from cluster):
  "I see 2 nginx pods running"

Controller observes:
  Desired != Current (3 != 2)

Controller takes action:
  "Create 1 more nginx pod"

Loop repeats...
```

### Controllers Bundled in kube-controller-manager

| Controller | What It Watches | What It Does |
|-----------|----------------|--------------|
| **ReplicationController** | ReplicaSets, ReplicationControllers | Ensures correct number of pod replicas |
| **DeploymentController** | Deployments | Manages ReplicaSets for rolling updates |
| **NodeController** | Node objects | Marks nodes unhealthy, evicts pods from failed nodes |
| **EndpointSliceController** | Services, Pods | Maintains endpoint lists for services |
| **ServiceAccountController** | ServiceAccounts | Creates tokens, manages secrets |
| **PersistentVolumeController** | PVs, PVCs | Binds claims to volumes |
| **RouteController** | Nodes | Configures cloud provider routes for pod networking |
| **JobController** | Jobs | Manages job execution and completion |
| **CronJobController** | CronJobs | Schedules jobs based on cron expressions |
| **NamespaceController** | Namespaces | Cleans up resources when namespace is deleted |
| **CertificateSigningRequest** | CSRs | Approves/denies certificate requests |
| **TokenCleaner** | Bootstrap tokens | Deletes expired tokens |

### Why Bundle Them?

Each controller could run as a separate process. Bundling them simplifies:
- **Deployment**: One binary, one service
- **Shared caches**: All controllers share an in-memory cache of API objects
- **Resource efficiency**: Less overhead than N separate processes

In production, some organizations split controllers across multiple controller managers for isolation.

---

## Controller Reconciliation Loop

Every controller follows the same fundamental pattern:

```
+------------------+
|  Watch for       |<-- Long-polling connection to API server
|  changes         |    Watches specific resource types
+--------+---------+
         |
         v
+------------------+
|  Retrieve        |<-- Get current state from API server/cache
|  current state   |
+--------+---------+
         |
         v
+------------------+
|  Compare         |<-- Desired vs Actual
|  desired vs      |    "Do I need to act?"
|  actual state    |
+--------+---------+
         |
         +-------- No change needed --------+
         |                                  |
         v                                  v
+------------------+              +------------------+
|  Take action     |              |  Sleep/wait for  |
|  (create/update/ |              |  next event      |
|  delete)         |              |                  |
+--------+---------+              +------------------+
         |
         v
+------------------+
|  Report status   |->-- Update object status via API server
|  back to API     |
+------------------+
         |
         +-- Go back to Watch
```

### Example: NodeController in Action

```
1. Administrator terminates node-1's EC2 instance

2. node-1's kubelet stops sending node status updates

3. NodeController detects:
   - node-1's Ready condition hasn't been updated in 40 seconds
   - Default grace period: 40s (NodeMonitorGracePeriod)

4. NodeController marks node-1:
   status.conditions["Ready"] = False
   status.conditions["Reason"] = "NodeStatusUnknown"

5. Pod eviction begins (after PodEvictionTimeout, default 5m):
   - All pods on node-1 are deleted
   - ReplicaSet controller notices: "Only 2 of 3 nginx pods"
   - ReplicaSet controller creates replacement pod
   - Scheduler assigns replacement to node-2

6. Cluster recovers automatically
```

---

## Cluster Signing and CSR Flow

The controller manager signs certificates for pods and nodes using the CA key.

### Certificate Signing Request (CSR) Flow

```
+--------+         +-------------+         +------------------+
| Pod or |         | API Server  |         | ControllerManager |
| Node   |         |             |         |                   |
+---+----+         +------+------+         +--------+----------+
    |                     |                         |
    | 1. Create CSR       |                         |
    |-------------------->|                         |
    |    "Please sign     |                         |
    |     my cert"        |                         |
    |                     |                         |
    |                     | 2. Store CSR in etcd    |
    |                     |                         |
    |                     | 3. Broadcast watch event|
    |<--------------------|                         |
    |                     |                         |
    |                     | 4. Watch triggers       |
    |                     |    Controller           |
    |                     |------------------------>|
    |                     |                         |
    |                     |                         | 5. Validate CSR
    |                     |                         |    (check permissions,
    |                     |                         |     approvers)
    |                     |                         |
    |                     |                         | 6. Sign with CA key
    |                     |                         |    (--cluster-signing-*)
    |                     |                         |
    |                     | 7. Update CSR status    |
    |                     |    with signed cert     |
    |                     |<------------------------|
    |                     |                         |
    | 8. Read signed cert |                         |
    |<--------------------|                         |
```

### Why the Controller Manager Needs ca.key

The controller manager uses `ca.key` to sign:
1. **Kubelet client certificates** — When nodes bootstrap TLS
2. **Service account tokens** — JWTs signed with the service account key
3. **User-submitted CSRs** — Custom certificate requests

This is why we distribute `ca.key` to the server node in Step 5. Without it:
- Nodes cannot get client certificates automatically
- Service account tokens cannot be issued
- `kubectl certificate approve` would have no effect

---

## kube-scheduler Deep Dive

### What kube-scheduler Does

The **kube-scheduler** assigns newly created pods to nodes. It is a **placement engine** that solves an optimization problem: given a pod's requirements and all available nodes, which node is the best fit?

### Scheduling is NOT Stateful

The scheduler is **completely stateless**:
- It does not remember past decisions
- It does not track which nodes have which pods
- Every scheduling decision is made fresh, based on current cluster state
- This makes it safe to restart or replace schedulers

### The Scheduling Decision

```
Input:
  Pod (with resource requests, affinity rules, tolerations)
  List of all Nodes

Output:
  Selected Node (or failure reason if no node fits)
```

---

## Scheduling Algorithm

Kubernetes scheduling happens in two phases:

### Phase 1: Filtering (Feasibility Check)

```
For each node, check if the pod CAN run there:

  - NodeResourcesFit: Does node have enough CPU/memory?
  - NodeSelector: Does node match pod's nodeSelector labels?
  - NodeAffinity: Does node satisfy affinity rules?
  - TaintToleration: Can pod tolerate node's taints?
  - VolumeBinding: Are required volumes available on this node?
  - PodTopologySpread: Does placement violate spread constraints?

Result: A subset of nodes that CAN run the pod (or empty set)
```

### Phase 2: Scoring (Optimization)

```
For each feasible node, calculate a score:

  - NodeResourcesFit: Prefer nodes with balanced resource usage
  - ImageLocality: Prefer nodes that already have the container image
  - InterPodAffinity: Prefer nodes near related pods
  - NodeAffinity: Weight nodes matching preferred affinity

Result: Ranked list of nodes (highest score wins)
```

### Scheduling Diagram

```
+-----------+     +------------------+     +----------------+
| Unscheduled|     |  Filter Phase    |     |  Score Phase   |
|    Pod    | --> |  (Predicates)    | --> |  (Priorities)  |
+-----------+     +--------+---------+     +--------+-------+
                           |                        |
                           v                        v
                  +----------------+        +----------------+
                  | Feasible Nodes |        | Ranked Nodes   |
                  | [node-1, node-2]|       | [node-2=95,    |
                  | (both OK)      |        |  node-1=80]    |
                  +----------------+        +--------+-------+
                                                     |
                                                     v
                                           +----------------+
                                           | Assign to      |
                                           | node-2         |
                                           +----------------+
```

### Example Scheduling Decision

Given:
- Pod requests: 500m CPU, 512Mi memory
- node-1: 1500m CPU available, 2Gi memory, has nginx image, no taints
- node-2: 800m CPU available, 1Gi memory, no nginx image, has taint special=true

Filtering:
- Both nodes have sufficient resources OK
- No nodeSelector or affinity specified OK
- Pod has no tolerations for special=true, so node-2 is filtered out X

Scoring (only node-1 remains):
- node-1 wins by default (only feasible node)

Result: Pod scheduled to node-1

---

## Leader Election

Both `kube-controller-manager` and `kube-scheduler` support **leader election**. This ensures that in a highly available (HA) setup, only ONE instance of each component is actively making decisions.

### Why Leader Election?

Without leader election, running multiple controller managers would cause:
- **Duplicate actions** — Two controllers might create the same pod twice
- **Race conditions** — Simultaneous updates could corrupt state
- **Thundering herd** — All controllers reacting to the same event

### How Leader Election Works

```
Instance A                            Instance B
(control plane 1)                    (control plane 2)
    |                                      |
    | 1. Try to acquire lease              | 1. Try to acquire lease
    |    in etcd                           |    in etcd
    |                                      |
    | 2. Lease acquired!                   | 2. Lease held by Instance A
    |    "I am the leader"                 |    "I am standby"
    |                                      |
    | 3. Renew lease every 15s             | 3. Watch lease
    |    (lease duration: 30s)             |
    |                                      |
    | 4. CRASH!                            | 4. Lease expires (no renewal)
    |                                      | 5. Acquire lease
    |                                      | 6. "I am now the leader"
```

### Lease Object

Leader election uses a special object stored in etcd:

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: server.kubernetes.local
  leaseDurationSeconds: 15
  acquireTime: "2026-06-14T11:00:00Z"
  renewTime: "2026-06-14T11:00:14Z"
```

- **holderIdentity**: Name of the current leader
- **leaseDurationSeconds**: How long the lease lasts without renewal
- **renewTime**: When the leader last renewed (must be < leaseDurationSeconds ago)

### Leader Election Flags

In our setup:
- `--leader-elect=true` on controller manager
- `leaderElection.leaderElect: true` in scheduler config

Even with a single control plane node, enabling leader election is **harmless and future-proof**. If we later add a second control plane node, leader election will automatically prevent conflicts.

### Single-Node Behavior

With only one instance:
```
Controller Manager starts
  -> Attempts to acquire lease
  -> No competing instances
  -> Acquires lease immediately
  -> Begins normal operation
  -> Renews lease every 10 seconds
  -> If it crashes, systemd restarts it
  -> New process re-acquires lease
```

---

## RBAC Deep Dive

### RBAC Fundamentals

**RBAC** (Role-Based Access Control) is the primary authorization mechanism in Kubernetes. It maps **WHO** can do **WHAT** to **WHICH RESOURCES**.

### Four RBAC Resources

```
+---------------+         +-------------------+
| ClusterRole   |         | Role              |
| (Global scope)|         | (Namespace scope) |
|               |         |                   |
| Defines WHAT  |         | Defines WHAT      |
| permissions   |         | permissions       |
| (verbs on     |         | (verbs on         |
| resources)    |         | resources)        |
+-------+-------+         +---------+---------+
        |                           |
        | RoleBinding references    | RoleBinding references
        v                           v
+---------------+         +-------------------+
|ClusterRoleBinding|      | RoleBinding       |
| (Global scope)|         | (Namespace scope) |
|               |         |                   |
| Defines WHO   |         | Defines WHO       |
| gets the      |         | gets the          |
| permissions   |         | permissions       |
+---------------+         +-------------------+
```

| Resource | Scope | Defines |
|----------|-------|---------|
| **Role** | Namespace | Permissions within ONE namespace |
| **ClusterRole** | Cluster-wide | Permissions across ALL namespaces |
| **RoleBinding** | Namespace | WHO gets a Role within ONE namespace |
| **ClusterRoleBinding** | Cluster-wide | WHO gets a ClusterRole globally |

### Verbs

RBAC verbs map to HTTP methods:

| Verb | HTTP Method | Example |
|------|------------|---------|
| `create` | POST | `kubectl create pod` |
| `get` | GET | `kubectl get pod` |
| `list` | GET (collection) | `kubectl get pods` |
| `watch` | GET + watch | `kubectl get pods --watch` |
| `update` | PUT | `kubectl replace` |
| `patch` | PATCH | `kubectl patch` |
| `delete` | DELETE | `kubectl delete pod` |
| `deletecollection` | DELETE (collection) | `kubectl delete pods --all` |

The `"*"` verb means ALL of the above.

### Built-In ClusterRoles

Kubernetes ships with many default ClusterRoles:

| ClusterRole | Purpose |
|-------------|---------|
| `cluster-admin` | Superuser (bound to system:masters) |
| `admin` | Admin within a namespace |
| `edit` | Can modify most objects |
| `view` | Read-only access |
| `system:node` | Permissions for kubelets |
| `system:dns` | Permissions for CoreDNS |
| `system:kube-scheduler` | Permissions for the scheduler |
| `system:kube-controller-manager` | Permissions for controllers |

### Default Bindings

```
ClusterRoleBinding: cluster-admin
  ClusterRole: cluster-admin
  Subjects:
    - Group: system:masters

This means: ANY user in the "system:masters" group
            has FULL CLUSTER ADMIN access.

Our admin user (CN=admin, O=system:masters) gets
cluster-admin privileges through this binding.
```

---

## The system:kube-apiserver-to-kubelet Role

### Why This Role Is Needed

When you run `kubectl logs my-pod` or `kubectl exec -it my-pod -- /bin/sh`, the traffic flows like this:

```
kubectl -> API Server -> Kubelet
                         (on worker node)
```

The API server opens a connection to the kubelet on the worker node. It authenticates to the kubelet as the `kubernetes` user (from its TLS certificate). The kubelet then checks: "Does the `kubernetes` user have permission to read logs/exec into this pod?"

Without the `system:kube-apiserver-to-kubelet` ClusterRole and binding:
```
kubectl logs my-pod
  -> API server connects to kubelet
  -> Kubelet checks RBAC for user "kubernetes"
  -> No permissions found
  -> Response: "Forbidden"
```

### The Fix

We create:
1. **ClusterRole** `system:kube-apiserver-to-kubelet` — Grants permissions on `nodes/proxy`, `nodes/stats`, `nodes/log`, `nodes/spec`, `nodes/metrics`
2. **ClusterRoleBinding** `system:kube-apiserver` — Binds that role to the `kubernetes` user

Now:
```
kubectl logs my-pod
  -> API server connects to kubelet as "kubernetes"
  -> Kubelet checks RBAC
  -> ClusterRoleBinding says: "kubernetes" has system:kube-apiserver-to-kubelet
  -> ClusterRole says: can access nodes/log, nodes/proxy
  -> Response: Logs streamed back
```

### Permission Details

| Resource | What It Enables |
|----------|----------------|
| `nodes/proxy` | `kubectl exec`, `kubectl port-forward`, `kubectl cp` |
| `nodes/stats` | `kubectl top node`, resource metrics |
| `nodes/log` | `kubectl logs`, `kubectl logs --previous` |
| `nodes/spec` | Node capacity and allocatable information |
| `nodes/metrics` | Prometheus-style node metrics collection |

### Why Not Just Give "kubernetes" Cluster-Admin?

You COULD bind the `kubernetes` user to `cluster-admin`, but that violates the **principle of least privilege**. The API server only needs kubelet proxy access, not the ability to delete namespaces or modify RBAC. Giving minimal permissions reduces blast radius if the API server is compromised.

---

## DNS/Certificate Validation and the Hostname Mismatch Error

### TLS Certificate Validation Basics

When a TLS client (like `kubectl`) connects to a TLS server (like `kube-apiserver`), the client verifies two things:

1. **Chain of trust**: The server's certificate must be signed by a trusted CA
2. **Hostname match**: The server's certificate must contain the hostname used to connect

```
Client (kubectl)                    Server (API Server)
     |                                     |
     | 1. TCP connect to server.kubernetes.local:6443
     |------------------------------------>|
     |                                     |
     | 2. Server presents certificate
     |<------------------------------------|
     |   "I am kubernetes.default.svc.cluster.local"
     |   Signed by KUBERNETES-CA           |
     |                                     |
     | 3. Client verifies chain
     |   "Is this CA in my trusted store?"
     |   YES (ca.crt is embedded in kubeconfig)
     |                                     |
     | 4. Client verifies hostname
     |   "Does cert contain 'server.kubernetes.local'?"
     |   YES (it's in the SAN list)
     |                                     |
     | 5. Handshake succeeds!
```

### Subject Alternative Names (SANs)

Modern TLS uses **SANs** instead of the older Common Name (CN) field for hostname validation. A certificate can list multiple identities:

```
Certificate SANs for kubernetes.crt:
  DNS: kubernetes
  DNS: kubernetes.default
  DNS: kubernetes.default.svc
  DNS: kubernetes.default.svc.cluster
  DNS: kubernetes.default.svc.cluster.local
  DNS: server.kubernetes.local
  IP: 10.32.0.1
  IP: 172.31.20.244
```

Each of these names is valid for connecting to the API server:
- From inside a pod: `https://kubernetes.default.svc:443`
- From a node: `https://172.31.20.244:6443`
- From kubectl: `https://server.kubernetes.local:6443`

### The Error We Encountered

#### Setup

Initially, the `/etc/hosts` file contained:
```
172.31.20.244 server.ani-kubernetes.local server
```

And the kubeconfig pointed to:
```yaml
clusters:
- cluster:
    server: https://server.ani-kubernetes.local:6443
```

But the API server certificate was generated with:
```
DNS: server.kubernetes.local  <- YES
DNS: server.ani-kubernetes.local  <- NO! Not present.
```

#### The Error Message

```
$ kubectl get nodes --kubeconfig admin.kubeconfig

Unable to connect to the server: x509: certificate is valid for
  server.kubernetes.local,
  kubernetes,
  kubernetes.default,
  kubernetes.default.svc,
  kubernetes.default.svc.cluster,
  kubernetes.default.svc.cluster.local,
  10.32.0.1,
  172.31.20.244,
not server.ani-kubernetes.local
```

#### What TLS Is Telling Us

The Go TLS library (used by kubectl) performs hostname verification:
1. It extracts the hostname from the URL: `server.ani-kubernetes.local`
2. It checks the certificate's SAN list
3. It finds 8 valid names, none of which match `server.ani-kubernetes.local`
4. It aborts the connection with the x509 error

This is **not a bug** — it is correct security behavior. TLS is doing exactly what it should.

### The Fix Applied

**Option 1 (Regenerate Certificate)**:
```bash
# Regenerate kubernetes.crt with the additional SAN
# Then redistribute to all nodes
# More work, preserves desired hostname
```

**Option 2 (Match Existing Certificate)** — What we did:
```bash
# Update /etc/hosts to use the name in the certificate
sudo sed -i 's/server.ani-kubernetes.local/server.kubernetes.local/' /etc/hosts

# Update kubeconfigs
kubectl config set-cluster kubernetes-the-hard-way \
  --server=https://server.kubernetes.local:6443 \
  --kubeconfig=admin.kubeconfig
```

Now the connection succeeds because `server.kubernetes.local` IS in the SAN list.

### Lesson: Plan Your SANs Before Generating Certificates

When generating the API server certificate in Step 2, include EVERY possible way the API server might be addressed:

| Connection Source | Hostname/IP to Include |
|-------------------|------------------------|
| kubectl from jumpbox | `server.kubernetes.local` |
| kubectl from nodes | `server.kubernetes.local` or `172.31.20.244` |
| Pods via Service | `kubernetes.default.svc.cluster.local` |
| Service ClusterIP | `10.32.0.1` |
| Direct IP | `172.31.20.244` |
| Internal DNS | `kubernetes`, `kubernetes.default` |

### Disabling Verification (DANGEROUS — Never Do This)

You might be tempted to bypass the check:
```bash
kubectl --insecure-skip-tls-verify get nodes  # DON'T DO THIS
```

This disables ALL certificate validation, making you vulnerable to man-in-the-middle attacks. **Never use this flag in production.**

---

## etcd as the Control Plane's Database

### Relationship Between API Server and etcd

```
+-------------+      Watch/GET/PUT/DELETE      +-------------+
| API Server  |<----------------------------->|    etcd     |
|             |    HTTPS over localhost:2379   |             |
| - REST API  |                                | - Key-Value |
| - AuthN/AuthZ|                               | - Consistent|
| - Admission |                                | - Watchable |
+-------------+                                +-------------+
```

The API server is essentially a **smart proxy** to etcd:
- It translates REST API calls into etcd key-value operations
- It adds authentication, authorization, and validation
- It multiplexes watch connections efficiently
- It caches frequently accessed data

### etcd Keys for Control Plane State

| Resource | etcd Key Prefix |
|----------|----------------|
| Pods | `/registry/pods/<namespace>/<name>` |
| Nodes | `/registry/minions/<name>` |
| Services | `/registry/services/<namespace>/<name>` |
| Endpoints | `/registry/services/endpoints/<namespace>/<name>` |
| ConfigMaps | `/registry/configmaps/<namespace>/<name>` |
| Secrets | `/registry/secrets/<namespace>/<name>` |
| ServiceAccounts | `/registry/serviceaccounts/<namespace>/<name>` |
| Roles | `/registry/roles/<namespace>/<name>` |
| ClusterRoles | `/registry/clusterroles/<name>` |
| RoleBindings | `/registry/rolebindings/<namespace>/<name>` |
| ClusterRoleBindings | `/registry/clusterrolebindings/<name>` |
| Leases | `/registry/leases/<namespace>/<name>` |
| Events | `/registry/events/<namespace>/<name>` |

### Reading etcd Directly

You can inspect what the control plane has stored:

```bash
ssh server "ETCDCTL_API=3 etcdctl get /registry/namespaces/default \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/kubernetes/ca.crt \
  --cert=/var/lib/kubernetes/kubernetes.crt \
  --key=/var/lib/kubernetes/kubernetes.key"
```

The output is binary protobuf, but you can see the structure.

---

## Service Cluster IP Range and Kubernetes Service

### What Is `--service-cluster-ip-range`?

The API server flag `--service-cluster-ip-range=10.32.0.0/24` defines the IP range from which Kubernetes assigns virtual IPs to Services.

### How Service IPs Work

```
Pod A (10.200.0.5) wants to talk to Service "nginx" (10.32.0.10)

                    +-----------------+
                    |   Service       |
                    |   nginx         |
                    |   ClusterIP:    |
                    |   10.32.0.10    |
                    +--------+--------+
                             |
              +--------------+--------------+
              |                             |
        +-----+-----+               +-----+-----+
        |  Pod nginx|               |  Pod nginx|
        |  10.200.0.5|              |  10.200.1.5|
        |  on node-1 |              |  on node-2 |
        +-----------+               +-----------+
```

- The Service IP (`10.32.0.10`) is NOT attached to any physical interface
- kube-proxy on each node sets up iptables/ipvs rules to DNAT traffic to Service IPs
- The controller manager's endpoint controller maintains the list of backend pods

### The Internal Kubernetes Service

Kubernetes automatically creates a Service named `kubernetes` in the `default` namespace:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes
  namespace: default
spec:
  type: ClusterIP
  clusterIP: 10.32.0.1
  ports:
  - port: 443
    targetPort: 6443
  selector: {}  # Endpoints are manually managed by API server
```

This is why `kubernetes.default.svc.cluster.local` must be in the API server certificate's SANs. Pods inside the cluster use this DNS name to discover the API server.

### Service IP Allocation

```
Range: 10.32.0.0/24
Usable: 10.32.0.1 - 10.32.0.254 (254 addresses)

Reserved:
  10.32.0.1  -> kubernetes service (API server)
  10.32.0.10 -> first user-created service
  ...
```

For a small cluster, `/24` (254 services) is plenty. In production, use `/12` or larger.

---

## Audit Logging

### What Is Audit Logging?

The API server can log every request it receives, providing a security trail of who did what and when.

### Our Audit Configuration

Enabled by these flags:
```
--audit-log-path=/var/log/audit.log
--audit-log-maxage=30
--audit-log-maxbackup=3
--audit-log-maxsize=100
```

| Flag | Meaning |
|------|---------|
| `--audit-log-path` | Where to write audit logs |
| `--audit-log-maxage=30` | Retain logs for 30 days |
| `--audit-log-maxbackup=3` | Keep up to 3 rotated log files |
| `--audit-log-maxsize=100` | Rotate when file reaches 100 MB |

### Audit Log Entry Example

```json
{
  "kind": "Event",
  "apiVersion": "audit.k8s.io/v1",
  "level": "Metadata",
  "auditID": "550e8400-e29b-41d4-a716-446655440000",
  "stage": "ResponseComplete",
  "requestURI": "/api/v1/namespaces/default/pods",
  "verb": "create",
  "user": {
    "username": "admin",
    "groups": ["system:masters", "system:authenticated"]
  },
  "sourceIPs": ["172.31.16.36"],
  "objectRef": {
    "resource": "pods",
    "namespace": "default",
    "name": "nginx"
  },
  "responseStatus": {"code": 201},
  "requestReceivedTimestamp": "2026-06-14T11:30:00.000000Z",
  "stageTimestamp": "2026-06-14T11:30:00.500000Z"
}
```

This tells us:
- User `admin` (in group `system:masters`)
- From IP `172.31.16.36` (the jumpbox)
- Created a pod named `nginx` in namespace `default`
- At 2026-06-14 11:30:00 UTC
- The request succeeded (HTTP 201)

### Why Audit Logs Matter

- **Security forensics**: Who deleted that critical deployment?
- **Compliance**: Many regulations require access logs
- **Debugging**: Trace what changed and when
- **Intrusion detection**: Spot unusual API activity

---

## FAQ

### Q: Can I run multiple API servers?

Yes! API servers are stateless. Run multiple instances behind a load balancer:
```
         +----------+
         |  LB:443  |
         +--+----+--+
            |    |
     +------+    +------+
     | API Server 1     |
     | (172.31.20.244)  |
     +------------------+
            |
            | All connect
            v
     +------------------+
     |    etcd cluster  |
     +------------------+
```

Ensure all API servers use the same `--etcd-servers` and certificates.

### Q: What happens if etcd goes down?

- API server cannot read or write data
- New resources cannot be created
- Existing pods continue running
- Controllers cannot make changes
- The cluster is "frozen" but not broken
- When etcd comes back, everything resumes

### Q: Why does the scheduler need leader election if there's only one?

For consistency and future-proofing. If you add a second scheduler later, leader election prevents conflicts automatically. With only one instance, it acquires the lease instantly and operates normally.

### Q: Can I disable RBAC?

You could set `--authorization-mode=AlwaysAllow`, but **never do this in production**. It removes ALL access control — any authenticated user can do anything. RBAC is essential for multi-tenant and production clusters.

### Q: What does `--apiserver-count=1` do?

It tells the API server "there is only one of me." This affects how endpoints for the `kubernetes` service are maintained. With multiple API servers, set this to the actual count so endpoint IPs are correctly managed.

### Q: Why `--allow-privileged=true`?

Some system pods (CNI plugins, storage drivers, monitoring) need to run in privileged mode to access host resources. For a learning cluster, enabling this is fine. In production, use Pod Security Standards or admission webhooks to restrict privileged pods.

### Q: What's the difference between `--client-ca-file` and `--tls-cert-file`?

| Flag | Purpose |
|------|---------|
| `--client-ca-file` | CA certificate used to VALIDATE incoming client certificates ("Who are they?") |
| `--tls-cert-file` / `--tls-private-key-file` | The API server's OWN certificate and key presented to clients ("Who am I?") |

### Q: What is `--runtime-config=api/all=true`?

It enables ALL API versions and resources, including alpha/beta features. This ensures backward compatibility and access to all API endpoints. In production, you might selectively enable only stable APIs.

### Q: Why does the controller manager need `--cluster-cidr`?

The route controller uses this to configure pod network routes. It needs to know the overall pod IP space to set up routing between nodes (especially important in cloud environments where routes may need to be configured).

### Q: Can I run the control plane on worker nodes?

Technically yes (by removing taints), but it's strongly discouraged for production:
- **Security**: Workers run untrusted user workloads
- **Resource contention**: User pods compete with control plane for CPU/memory
- **Stability**: Worker failures shouldn't bring down the control plane

For learning/small clusters, it can work.

---

## Troubleshooting

### Issue: API server starts but controller manager keeps crashing

**Cause**: The controller manager needs the API server to be fully up. If it starts too fast, it fails.

**Solution**: Ensure API server is running before starting controller manager:
```bash
ssh server "sudo systemctl start kube-apiserver"
sleep 30
ssh server "sudo systemctl start kube-controller-manager kube-scheduler"
```

### Issue: "failed to create listener: failed to listen on 0.0.0.0:6443"

**Cause**: Another process is already using port 6443.

**Solution**:
```bash
ssh server "sudo ss -tlnp | grep 6443"
ssh server "sudo fuser -k 6443/tcp"
ssh server "sudo systemctl restart kube-apiserver"
```

### Issue: Scheduler logs show "leaderelection.go: Failed to update lock"

**Cause**: API server connectivity issue or etcd latency.

**Solution**: Check API server and etcd health. May be transient; scheduler will retry automatically.

### Issue: RBAC resources applied but kubectl exec still forbidden

**Cause**: Wrong user in the ClusterRoleBinding.

**Solution**: Verify the binding references the exact user the API server presents:
```bash
kubectl describe clusterrolebinding system:kube-apiserver --kubeconfig admin.kubeconfig
```

Ensure the subject is:
```yaml
subjects:
- kind: User
  name: kubernetes
```

Not `admin` or another name.

### Issue: Certificate error persists after fixing hostname

**Cause**: Old connection caching or wrong kubeconfig.

**Solution**:
```bash
# Clear any cached connections
rm -rf ~/.kube/cache

# Verify the correct kubeconfig
cat admin.kubeconfig | grep server

# Test with verbose output
kubectl get nodes --kubeconfig admin.kubeconfig -v=6
```

### Issue: High memory usage by API server

**Cause**: Large number of objects or excessive watches.

**Solution**: Normal for API server to use 200-500MB. If higher:
- Check for runaway controllers creating many objects
- Enable profiling: `--enable-profiling=true`
- Inspect heap: `curl https://server:6443/debug/pprof/heap`

---

## Related Concepts for Other Steps

- **PKI** (Step 2): Certificates used for mTLS between all control plane components
- **Kubeconfigs** (Step 3): Authentication files consumed by scheduler and controller manager
- **etcd** (Step 4): The database backing the entire control plane
- **Worker Nodes** (Step 6): Where the scheduler places pods; what controllers manage
- **Pod Networking** (Step 7): Route controller configures inter-node routes using `--cluster-cidr`
- **Smoke Test** (Step 8): Validates the control plane by creating real workloads

---

## Summary of Key Architectural Principles

1. **Separation of concerns**: API server handles API, controllers handle state, scheduler handles placement
2. **Single source of truth**: etcd is the only persistent store; all components read from it via the API server
3. **Statelessness**: API server and scheduler are stateless; they can be restarted or replicated freely
4. **Defense in depth**: Authentication -> Authorization -> Admission -> Validation -> Persistence
5. **Least privilege**: RBAC grants minimal permissions; the API server-to-kubelet role is narrowly scoped
6. **Watch-based architecture**: All components react to state changes via watches, not polling
7. **Leader election**: Prevents split-brain in HA scenarios; gracefully handles failover
8. **Certificate discipline**: Every name used to connect must be in the certificate SANs
