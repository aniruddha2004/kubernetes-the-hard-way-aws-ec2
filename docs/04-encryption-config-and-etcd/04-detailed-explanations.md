# Step 4: Encryption Config and etcd - Detailed Explanations

> Deep-dive explanations for etcd, encryption at rest, the Raft consensus algorithm, how Kubernetes stores data, and why every configuration choice matters.

---

## Table of Contents

- [What is etcd?](#what-is-etcd)
- [Why Does Kubernetes Need etcd?](#why-does-kubernetes-need-etcd)
- [How Kubernetes Stores Data in etcd](#how-kubernetes-stores-data-in-etcd)
- [The Watch/Event Model](#the-watchevent-model)
- [Single Node vs Cluster](#single-node-vs-cluster)
- [Why Encryption at Rest?](#why-encryption-at-rest)
- [Encryption Config Deep Dive](#encryption-config-deep-dive)
- [etcd Architecture](#etcd-architecture)
- [etcd Ports Explained](#etcd-ports-explained)
- [Why Copy the API Server Certificate to etcd?](#why-copy-the-api-server-certificate-to-etcd)
- [Understanding systemd](#understanding-systemd)
- [etcd Flags Explained](#etcd-flags-explained)
- [The Write Ahead Log (WAL)](#the-write-ahead-log-wal)
- [etcdctl Member List Network Flow](#etcdctl-member-list-network-flow)

---

## What is etcd?

**etcd** (pronounced "et-see-dee") is a distributed, reliable key-value store that uses the **Raft consensus algorithm** to maintain consistency across replicas.

### Key Characteristics

| Property | Description |
|----------|-------------|
| **Distributed** | Runs as a cluster of 3, 5, or 7 nodes |
| **Consistent** | Uses Raft for strong consistency (CP in CAP theorem) |
| **Key-Value** | Simple key-value pairs, like a distributed hash map |
| **Watchable** | Clients can watch keys for changes in real-time |
| **Secure** | Supports TLS for transport and optionally encryption at rest |

### History

- Created by **CoreOS** in 2013
- Donated to **CNCF** (Cloud Native Computing Foundation) in 2018
- Written in **Go**
- Uses **Protocol Buffers** for efficient serialization
- Default datastore for Kubernetes since v1.0

---

## Why Does Kubernetes Need etcd?

### The Stateless Nature of Kubernetes

Kubernetes control plane components are **stateless** — they don't store persistent data themselves:

| Component | Stores State? | Data Stored |
|-----------|---------------|-------------|
| kube-apiserver | No | Reads/writes all data to etcd |
| kube-controller-manager | No | Reconciles desired state from etcd |
| kube-scheduler | No | Makes scheduling decisions based on etcd data |
| kubelet | Minimal | Only local pod state, rest comes from API server |
| kube-proxy | No | Reads service endpoints from API server |

### What Happens Without etcd?

If etcd stops:
- API server cannot read or write any data
- New pods cannot be scheduled
- Existing pods continue running (kubelet operates independently)
- No configuration changes can be made
- The cluster is effectively **frozen**

### etcd is the Source of Truth

```
User/API/Controller          etcd
     |                          |
     | POST /api/v1/pods        |
     |------------------------->|
     |                          | Store as key-value
     |                          | /registry/pods/default/nginx
     |                          |
     |<-------------------------|
     | 201 Created              |
     |                          |
     |                          |
Controller Watcher           etcd
     |                          |
     | WATCH /registry/pods     |
     |<-------------------------|
     | "New pod nginx created!" |
     |                          |
```

All cluster state flows through etcd. It's the **single source of truth**.

---

## How Kubernetes Stores Data in etcd

### etcd Key Hierarchy

Kubernetes organizes data in etcd using a hierarchical key structure:

```
/registry/
├── pods/
│   └── <namespace>/
│       └── <pod-name>
├── services/
│   └── <namespace>/
│       └── <service-name>
├── configmaps/
│   └── <namespace>/
│       └── <configmap-name>
├── secrets/
│   └── <namespace>/
│       └── <secret-name>
├── deployments/
│   └── <namespace>/
│       └── <deployment-name>
├── nodes/
│   └── <node-name>
├── events/
│   └── <namespace>/
│       └── <event-uuid>
├── serviceaccounts/
│   └── <namespace>/
│       └── <sa-name>
├── replicasets/
│   └── <namespace>/
│       └── <rs-name>
└── ... (many more resource types)
```

### Data Serialization

When you create a pod:

```yaml
# Your YAML
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:alpine
```

It becomes:

```
Key: /registry/pods/default/nginx
Value: <protobuf-encoded JSON representation>
```

The value is NOT plain YAML — it's:
1. **JSON representation** of the object
2. **Encoded as Protocol Buffers** for efficiency
3. **Stored as a byte string** in etcd

### Reading etcd Data Directly

```bash
ETCDCTL_API=3 etcdctl get /registry/pods/default/nginx \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/kubernetes/ca.crt \
  --cert=/var/lib/kubernetes/kubernetes.crt \
  --key=/var/lib/kubernetes/kubernetes.key
```

Output is binary protobuf — not human-readable. That's why we use kubectl instead.

---

## The Watch/Event Model

etcd supports **watches** — long-polling connections that notify clients when keys change.

### How Kubernetes Uses Watches

```
+-----------+         +-------------+         +-----------+
| Controller|         | API Server  |         |   etcd    |
| Manager   |         |             |         |           |
+-----------+         +-------------+         +-----------+
     |                      |                      |
     | GET /api/v1/pods     |                      |
     | ?watch=true          |                      |
     |--------------------->|                      |
     |                      | WATCH /registry/pods |
     |                      |--------------------->|
     |                      |                      |
     |                      |<---------------------|
     |                      | Event: MODIFIED      |
     |                      | /registry/pods/...   |
     |                      |                      |
     |<---------------------|                      |
     | "Pod status changed" |                      |
     |                      |                      |
```

This event-driven architecture is the foundation of Kubernetes' **reconciliation model**:
1. Controllers watch resources
2. When something changes, they react
3. They take action to move the cluster toward desired state

Without watches, controllers would have to constantly poll (wasteful and slow).

---

## Single Node vs Cluster

### Single Node (This Tutorial)

```
+---------+
|  etcd   |
| (alone) |
+---------+
```

- Simple to set up
- No consensus overhead
- Single point of failure
- Suitable for learning/labs

### Production Cluster (3 Nodes)

```
+---------+     +---------+     +---------+
| etcd-1  |<--->| etcd-2  |<--->| etcd-3  |
| Leader  |     | Follower|     | Follower|
+---------+     +---------+     +---------+
```

- Requires majority (2 of 3) for writes
- Survives 1 node failure
- Leader election if leader dies
- Network partition tolerance

### Production Cluster (5 Nodes)

```
+---------+     +---------+     +---------+     +---------+     +---------+
| etcd-1  |<--->| etcd-2  |<--->| etcd-3  |<--->| etcd-4  |<--->| etcd-5  |
| Leader  |     | Follower|     | Follower|     | Follower|     | Follower|
+---------+     +---------+     +---------+     +---------+     +---------+
```

- Requires majority (3 of 5) for writes
- Survives 2 node failures
- Higher write latency due to consensus

### Why Odd Numbers?

Raft requires a **majority quorum**:
- 3 nodes: majority = 2 (tolerates 1 failure)
- 4 nodes: majority = 3 (tolerates 1 failure)
- 5 nodes: majority = 3 (tolerates 2 failures)

Adding a 4th node doesn't improve fault tolerance over 3 nodes! Always use odd numbers.

---

## Why Encryption at Rest?

### The Threat Model

Without encryption at rest:

```
Attacker gains access to etcd data directory (/var/lib/etcd/)
     |
     v
Reads raw files
     |
     v
Extracts secret data in plaintext:
   {
     "kind": "Secret",
     "data": {
       "password": "my-super-secret-password",  <-- PLAINTEXT!
       "api-key": "sk-live-1234567890"         <-- PLAINTEXT!
     }
   }
```

With encryption at rest:

```
Attacker gains access to etcd data directory
     |
     v
Reads raw files
     |
     v
Extracts encrypted blob:
   {
     "kind": "Secret",
     "data": {
       "password": "ENC:AESCBC:v1:key1:abc123...xyz==",  <-- ENCRYPTED!
     }
   }
     |
     v
Cannot decrypt without the encryption key!
```

### What Gets Encrypted?

The encryption config specifies which resources to encrypt:

```yaml
resources:
  - resources:
      - secrets
```

You can add more:
```yaml
resources:
  - resources:
      - secrets
      - configmaps
      - ingresses.tls
```

**Best Practice**: Encrypt ALL sensitive resources. At minimum, always encrypt `secrets`.

### Performance Impact

- Read: Decrypt on-the-fly (~1-5% overhead)
- Write: Encrypt before storing (~2-10% overhead)
- Minimal impact for most workloads

---

## Encryption Config Deep Dive

### Provider Types

| Provider | Algorithm | Security Level |
|----------|-----------|----------------|
| `aescbc` | AES-CBC with PKCS#7 padding | Good (legacy) |
| `aesgcm` | AES-GCM with random nonce | Better |
| `secretbox` | XSalsa20 + Poly1305 | Good |
| `kms` | External KMS (AWS KMS, etc.) | Best for enterprise |
| `identity` | No encryption (pass-through) | None |

### Key Rotation

When you need to rotate keys:

```yaml
providers:
  - aescbc:
      keys:
        - name: key2        # NEW key (first = active for writes)
          secret: <NEW_KEY>
        - name: key1        # OLD key (still used for reads)
          secret: <OLD_KEY>
  - identity: {}
```

1. Add new key at the TOP (becomes active for writes)
2. Restart API server
3. All NEW secrets encrypted with new key
4. Old secrets still readable with old key
5. Rewriting secrets re-encrypts them with new key

### Base64 Encoding

The key is base64-encoded raw bytes:

```bash
# Generate 32 random bytes
head -c 32 /dev/urandom
# Binary output (unprintable)

# Encode to base64 for YAML
head -c 32 /dev/urandom | base64
# Printable: YWJjZGVmZ2hpamtsbW5vcHFyc3R1dnd4eXoxMjM0NTY=
```

32 bytes = 256 bits → AES-256 (industry standard)

---

## etcd Architecture

### Inside an etcd Node

```
+-------------------+
|   HTTP/gRPC API   |  <- Clients connect here (2379)
|   (TLS/mTLS)      |
+-------------------+
         |
         v
+-------------------+
|   Raft Module     |  <- Consensus and log replication
|                   |
|   - Leader Election
|   - Log Append    |
|   - Commit        |
+-------------------+
         |
         v
+-------------------+
|   WAL + Snapshots |  <- Persistent storage
|                   |
|   WAL: Incremental
|   changes         |
|                   |
|   Snapshot: Full  |
|   state at point  |
|   in time         |
+-------------------+
         |
         v
+-------------------+
|   BoltDB (bbolt)  |  <- Embedded key-value store
|   /var/lib/etcd/  |
+-------------------+
```

### Components

**HTTP/gRPC API**
- Accepts client requests on port 2379
- Supports PUT, GET, DELETE, WATCH operations
- TLS-encrypted by default

**Raft Module**
- Ensures all nodes agree on the state
- Leader accepts writes, followers replicate
- Uses heartbeat timeouts and election timeouts

**WAL (Write Ahead Log)**
- Append-only log of all changes
- Durability guarantee: written to disk before acknowledged
- Replayed on startup to restore state

**Snapshots**
- Periodic full-state saves
- Prevents WAL from growing infinitely
- Used to bring new nodes up to speed

**BoltDB (bbolt)**
- Embedded B+tree database
- Stores the actual key-value pairs
- Pure Go implementation

---

## etcd Ports Explained

| Port | Purpose | Who Connects |
|------|---------|--------------|
| **2379** | Client API | API server, etcdctl, backup tools |
| **2380** | Peer (Raft) | Other etcd members |

### Why Two Ports?

Separation of concerns:
- **2379** handles client traffic (reads/writes/watches)
- **2380** handles Raft consensus (heartbeat, log replication, snapshot transfer)

Separating them:
- Allows different security policies
- Prevents client traffic from interfering with consensus
- Enables independent scaling/monitoring

### URL Types

| Flag | URL Type | Purpose |
|------|----------|---------|
| `--listen-client-urls` | Bind address | What interfaces to listen on for clients |
| `--advertise-client-urls` | Public address | What URL to tell clients to use |
| `--listen-peer-urls` | Bind address | What interfaces to listen on for peers |
| `--initial-advertise-peer-urls` | Public address | What URL to tell peers to use |

**Listen** = "I'm listening HERE" (bind)
**Advertise** = "Connect to me THERE" (public)

In cloud environments, these may differ (e.g., bind to 0.0.0.0 but advertise the private IP).

---

## Why Copy the API Server Certificate to etcd?

In Step 2, we generated `kubernetes.crt` for the API server. Now we copy it to etcd. Why?

### etcd's Dual Role

etcd acts as BOTH:
1. **Server** — Accepts connections from API server (client role)
2. **Peer** — Communicates with other etcd members (peer role)

Since our etcd is single-node, the peer certificate isn't actively used for inter-node communication, but it's still configured for consistency.

### Using the Same Certificate

We use `kubernetes.crt` because:
- etcd needs a certificate with `CN=kube-apiserver` (its identity)
- The API server validates that the etcd server presents a certificate signed by the trusted CA
- etcd validates that connecting clients (like the API server) present valid certificates

In production, you might use separate certificates:
- `etcd-server.crt` — For client-facing connections
- `etcd-peer.crt` — For peer-to-peer connections

But for simplicity in this lab, we reuse the API server certificate.

---

## Understanding systemd

systemd is the Linux system and service manager. It replaces the old SysV init system.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Unit** | A service, mount, device, socket, etc. |
| **Service** | A process systemd manages (like etcd) |
| **Target** | A group of units (like runlevel) |
| **Daemon** | Background process |

### Service Lifecycle

```
systemctl start etcd     -> Start the service
systemctl stop etcd      -> Stop the service
systemctl restart etcd   -> Stop then start
systemctl enable etcd    -> Start on boot
systemctl disable etcd   -> Don't start on boot
systemctl status etcd    -> Show current status
```

### Our etcd.service Explained

```ini
[Unit]
Description=etcd                     # Human-readable name
Documentation=https://github.com/coreos  # Help URL

[Service]
Type=notify                          # Tell systemd when ready
ExecStart=/usr/local/bin/etcd ...    # Command to run
Restart=on-failure                   # Restart if process exits with error
RestartSec=5                         # Wait 5 seconds before restarting

[Install]
WantedBy=multi-user.target           # Start in multi-user mode
```

**Type=notify**
- etcd sends a notification to systemd when it's fully started
- systemd knows the service is "ready" and can start dependent services
- Prevents race conditions (starting API server before etcd is ready)

**Restart=on-failure**
- If etcd crashes (non-zero exit code), systemd automatically restarts it
- Doesn't restart if stopped cleanly (systemctl stop)

### systemd vs Kubernetes Analogy

| systemd | Kubernetes |
|---------|------------|
| `systemctl start etcd` | `kubectl apply -f pod.yaml` |
| `systemctl status etcd` | `kubectl get pod etcd` |
| `systemctl stop etcd` | `kubectl delete pod etcd` |
| `Restart=on-failure` | `restartPolicy: OnFailure` |
| `ExecStart=` | `containers[].command` |
| `After=network.target` | `initContainers` / pod readiness |

---

## etcd Flags Explained

### Identity Flags

```
--name server
```
- Human-readable name for this member
- Used in cluster status output and logging
- Must be unique within the cluster

### TLS Flags (Client)

```
--cert-file=/var/lib/kubernetes/kubernetes.crt
--key-file=/var/lib/kubernetes/kubernetes.key
--trusted-ca-file=/var/lib/kubernetes/ca.crt
--client-cert-auth
```

- `--cert-file` / `--key-file` — etcd's identity certificate (presented to clients)
- `--trusted-ca-file` — CA to validate client certificates
- `--client-cert-auth` — Require clients to present valid certificates

### TLS Flags (Peer)

```
--peer-cert-file=/var/lib/kubernetes/kubernetes.crt
--peer-key-file=/var/lib/kubernetes/kubernetes.key
--peer-trusted-ca-file=/var/lib/kubernetes/ca.crt
--peer-client-cert-auth
```

Same as client flags but for peer-to-peer communication between etcd members.

### Network Flags

```
--listen-client-urls https://<SERVER_PRIVATE_IP>:2379,https://127.0.0.1:2379
--advertise-client-urls https://<SERVER_PRIVATE_IP>:2379
```

- `--listen-client-urls` — Interfaces and ports to bind
  - Includes `127.0.0.1` for local tools (etcdctl on the same machine)
  - Includes private IP for remote connections (API server)
- `--advertise-client-urls` — What URL to publish in the cluster membership
  - Other clients use this to connect

```
--listen-peer-urls https://<SERVER_PRIVATE_IP>:2380
--initial-advertise-peer-urls https://<SERVER_PRIVATE_IP>:2380
```

Same pattern for peer communication.

### Cluster Initialization Flags

```
--initial-cluster-token etcd-cluster-0
--initial-cluster server=https://<SERVER_PRIVATE_IP>:2380
--initial-cluster-state new
```

- `--initial-cluster-token` — Unique identifier for this cluster
  - Prevents accidental joining of wrong clusters
- `--initial-cluster` — Static bootstrap configuration
  - Lists ALL members with their peer URLs
  - Used only during initial startup
- `--initial-cluster-state new` — Create a new cluster
  - Alternative: `existing` (for joining a running cluster)

### Data Flag

```
--data-dir=/var/lib/etcd
```

Where etcd stores:
- WAL files (`member/wal/`)
- Snapshots (`member/snap/`)
- BoltDB database (`member/snap/db`)

**Backup this directory regularly!** It contains all cluster state.

---

## The Write Ahead Log (WAL)

### What is WAL?

WAL is an append-only log that records all changes before they're applied to the database:

```
+--------+     +--------+     +---------+
| Client |     |  WAL   |     | BoltDB  |
| Request| --> | Append | --> | Apply   |
+--------+     +--------+     +---------+
     |              |              |
     |              |              |
     v              v              v
  "PUT foo=bar"  fsync to disk   B+tree update
```

### Why WAL?

1. **Durability**: Data is safely on disk before acknowledging the client
2. **Crash Recovery**: Replay WAL on startup to recover state
3. **Replication**: Followers receive WAL entries from the leader

### WAL Format

Each entry:
```
[Header][Data][CRC]
```

- **Header**: Entry type, term, index
- **Data**: The actual key-value operation
- **CRC**: Checksum for integrity verification

### Snapshotting

WAL grows forever if unchecked:

```
WAL after 1M operations: 500 MB
     |
     v
Snapshot at index 1,000,000
     |
     v
Compact WAL to index 1,000,000
     |
     v
WAL size resets to near zero
```

etcd automatically snapshots and compacts based on:
- `--snapshot-count` (default: 100,000 operations)
- `--quota-backend-bytes` (size limit)

---

## etcdctl Member List Network Flow

When you run `etcdctl member list`, here's what happens:

```
+--------+      HTTPS       +--------+      Raft       +--------+
|etcdctl | ---------------> | etcd   | ---------------> | etcd   |
| (local)|  Connect to     | (local)|  Query internal | (leader|
|        |  127.0.0.1:2379 | node)  |  state          |  if not|
+--------+                  +--------+                 | local) |
     |                          |                      +--------+
     | 1. TLS handshake         |                           |
     |    - Verify server cert  |                           |
     |    - Present client cert |                           |
     |                          |                           |
     | 2. gRPC request          |                           |
     |    MemberList RPC        |                           |
     |                          |                           |
     | 3. Response              |                           |
     |    Member ID, Name,      |                           |
     |    Peer URLs,            |                           |
     |    Client URLs           |                           |
     |                          |                           |
     v                          v                           v
+--------+                  +--------+                 +--------+
| Output  |                 | Done   |                 |        |
| ID=abc..|                 |        |                 |        |
| Name=srv|                 |        |                 |        |
+--------+                  +--------+                 +--------+
```

### Member List Output Format

```
<MEMBER_ID>, <STATUS>, <NAME>, <PEER_URLS>, <CLIENT_URLS>
```

Example:
```
8e9e05c52164694d, started, server, https://<SERVER_PRIVATE_IP>:2380, https://<SERVER_PRIVATE_IP>:2379
```

- `8e9e05c52164694d` — Unique 64-bit hex member ID (hashed from peer URLs)
- `started` — Member is active
- `server` — Name from `--name` flag
- `https://<SERVER_PRIVATE_IP>:2380` — Where peers connect
- `https://<SERVER_PRIVATE_IP>:2379` — Where clients connect

---

## FAQ

### Q: Can I run etcd on worker nodes?

A: Technically yes, but it's strongly discouraged. etcd should run on dedicated control plane nodes for:
- Resource isolation (etcd is I/O intensive)
- Security (workers run untrusted workloads)
- Stability (worker failures shouldn't affect cluster state)

### Q: What if etcd runs out of disk space?

A: etcd enters read-only mode and refuses writes. Monitor disk usage and set `--quota-backend-bytes`.

### Q: How do I backup etcd?

A: Use `etcdctl snapshot save`:
```bash
ETCDCTL_API=3 etcdctl snapshot save backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/kubernetes/ca.crt \
  --cert=/var/lib/kubernetes/kubernetes.crt \
  --key=/var/lib/kubernetes/kubernetes.key
```

### Q: What's the difference between etcd v2 and v3?

A: Completely different APIs! Always use `ETCDCTL_API=3`. v3 uses gRPC, supports leases, transactions, and watches. Kubernetes requires etcd v3.

### Q: Why not use a database like PostgreSQL or MySQL?

A: etcd is designed for:
- Small key-value pairs (not large blobs)
- High consistency (not eventual consistency)
- Watch semantics (not polling)
- Simple operations (not complex queries)

PostgreSQL/MySQL are great but don't provide the watch API Kubernetes needs.

### Q: What happens during a leader election?

A:
1. Leader stops sending heartbeats (crashed/network partitioned)
2. Followers time out (election timeout ~1s)
3. Followers increment term and start election
4. Candidate requests votes
5. Majority votes → new leader elected
6. Cluster resumes (typically < 2 seconds downtime)

---

## Troubleshooting

### Issue: etcd fails to start with "bind: address already in use"

**Cause**: Another process is using port 2379 or 2380.

**Solution**: Check with `sudo ss -tlnp | grep 2379` and kill the conflicting process.

### Issue: "transport: authentication handshake failed"

**Cause**: TLS certificate mismatch — wrong CA, expired cert, or wrong SANs.

**Solution**: Verify certificates with `openssl x509 -in kubernetes.crt -text -noout`. Ensure CA matches.

### Issue: etcd starts but API server can't connect

**Cause**: API server certificate doesn't have etcd's expected identity, or firewall blocking 2379.

**Solution**: Check API server logs and security group rules.

### Issue: "mvcc: database space exceeded"

**Cause**: etcd reached its quota limit (default 2GB or `--quota-backend-bytes`).

**Solution**: Compact and defragment:
```bash
ETCDCTL_API=3 etcdctl compact $(ETCDCTL_API=3 etcdctl endpoint status --write-out=json | grep revision)
ETCDCTL_API=3 etcdctl defrag
```

---

## Related Concepts for Other Steps

- **PKI** (Step 2): Certificates used for etcd mTLS
- **Control Plane** (Step 5): API server is the primary etcd client
- **Smoke Test** (Step 8): Creating secrets tests encryption at rest
- **Production**: Consider etcd backup strategies, monitoring, and clustering
