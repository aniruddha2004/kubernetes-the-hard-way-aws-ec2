# Step 3: Kubeconfigs and Authentication - Detailed Explanations

> Deep-dive explanations for kubeconfigs, Kubernetes authentication mechanisms, mTLS in practice, and how components use these configuration files to establish identity and connect to the API server.

---

## Table of Contents

- [What is a Kubeconfig?](#what-is-a-kubeconfig)
- [Kubeconfig Structure](#kubeconfig-structure)
- [How kubectl Uses Kubeconfig](#how-kubectl-uses-kubeconfig)
- [TLS vs mTLS Recap](#tls-vs-mtls-recap)
- [The Full mTLS Handshake in Kubernetes](#the-full-mtls-handshake-in-kubernetes)
- [Why Does Every Component Have Its Own Kubeconfig?](#why-does-every-component-have-its-own-kubeconfig)
- [End-to-End Authentication Flows](#end-to-end-authentication-flows)
- [Service Account Token Signing](#service-account-token-signing)

---

## What is a Kubeconfig?

A **kubeconfig** is a YAML configuration file that tells Kubernetes clients:

1. **WHERE** is the API server? (cluster.server)
2. **WHO** am I? (users[*].user.client-certificate-data)
3. **HOW** do I trust the server? (clusters[*].cluster.certificate-authority-data)
4. **WHICH** cluster/user combination should I use? (contexts, current-context)

Think of it as a **driver's license + address book**. The license proves your identity, and the address book tells you where to go.

### Without a Kubeconfig

If you run `kubectl get pods` without a kubeconfig:
```
The connection to the server localhost:8080 was refused
```

kubectl defaults to `localhost:8080` — the API server isn't running there!

### With a Kubeconfig

kubectl reads `~/.kube/config` (or `$KUBECONFIG`), finds the current context, extracts the server address and credentials, and establishes a connection.

---

## Kubeconfig Structure

```yaml
apiVersion: v1
kind: Config
clusters:                          # Section 1: WHERE
- cluster:
    certificate-authority-data: <base64-encoded-ca-crt>
    server: https://server.kubernetes.local:6443
  name: kubernetes-the-hard-way

users:                             # Section 2: WHO
- name: admin
  user:
    client-certificate-data: <base64-encoded-client-crt>
    client-key-data: <base64-encoded-client-key>

contexts:                          # Section 3: COMBINATION
- context:
    cluster: kubernetes-the-hard-way
    user: admin
    namespace: default             # Optional: default namespace
  name: default

current-context: default           # Section 4: DEFAULT
```

### Sections Explained

**Clusters Section**
- `certificate-authority-data` — Base64-encoded CA certificate. Used to verify the API server's certificate during TLS handshake.
- `server` — Full URL of the API server endpoint
- `name` — Arbitrary label to reference this cluster

**Users Section**
- `client-certificate-data` — Base64-encoded client certificate. Presented during mTLS to prove identity.
- `client-key-data` — Base64-encoded private key. Signs the TLS handshake to prove possession.
- `name` — Arbitrary label to reference these credentials

**Contexts Section**
- Combines one cluster + one user + optional namespace
- Allows switching between environments (dev/staging/prod) easily

**current-context**
- Specifies which context to use when no `--context` flag is provided

### Base64 Encoding

Certificates in kubeconfig are **base64-encoded** to fit in YAML:

```bash
# Encode a certificate
cat ca.crt | base64 -w 0

# Decode from kubeconfig
echo "LS0tLS1CRUdJTi..." | base64 -d
```

Using `--embed-certs=true` with `kubectl config` handles this automatically.

---

## How kubectl Uses Kubeconfig

### Resolution Order

kubectl looks for configuration in this order:

```
1. --kubeconfig flag (highest priority)
2. $KUBECONFIG environment variable
3. ~/.kube/config (default)
```

### Connection Flow

```
kubectl get pods
     |
     v
+--------------------------------+
| Read ~/.kube/config            |
| current-context = default      |
|                                |
| cluster = kubernetes-the-hard- |
|   way                          |
| user = admin                   |
+--------------------------------+
     |
     v
+--------------------------------+
| Resolve server address         |
| server.kubernetes.local        |
|   -> 172.31.20.244             |
+--------------------------------+
     |
     v
+--------------------------------+
| TLS Handshake                  |
| 1. Connect to 172.31.20.244:6443|
| 2. Verify server cert with CA  |
| 3. Present client cert (admin) |
| 4. Server verifies with CA     |
+--------------------------------+
     |
     v
+--------------------------------+
| Send HTTP GET /api/v1/pods     |
| Authorization: N/A (mTLS       |
|   already proved identity)     |
+--------------------------------+
     |
     v
+--------------------------------+
| API Server                     |
| 1. mTLS successful             |
| 2. Extract CN=admin,           |
|    O=system:masters            |
| 3. RBAC check: ALLOW (masters  |
|    bypass RBAC)                |
| 4. Query etcd for pods         |
| 5. Return JSON response        |
+--------------------------------+
     |
     v
+--------------------------------+
| kubectl renders table output   |
+--------------------------------+
```

---

## TLS vs mTLS Recap

### Standard TLS (One-Way)

```
CLIENT                                    SERVER
  |                                          |
  |-------- 1. "I want to connect" --------->|
  |                                          |
  |<------- 2. "Here's my certificate" ------|
  |       [Signed by trusted CA]             |
  |                                          |
  |    [Client verifies cert]                |
  |    "Yes, this server is legitimate"      |
  |                                          |
  |======== ENCRYPTED CHANNEL ==============|
  |                                          |
  |-------- 3. "Here are my credentials" --->|
  |    [Username/password/API key]           |
```

**Used by**: HTTPS websites, most APIs

### Mutual TLS (Two-Way)

```
CLIENT                                    SERVER
  |                                          |
  |-------- 1. "I want to connect" --------->|
  |    [Presents client certificate]         |
  |                                          |
  |<------- 2. "Here's my certificate" ------|
  |       [Signed by trusted CA]             |
  |                                          |
  |    [BOTH verify each other's certs]      |
  |    "We both trust the same CA"           |
  |                                          |
  |======== ENCRYPTED CHANNEL ==============|
  |    [No additional credentials needed]    |
```

**Used by**: Kubernetes, service meshes (Istio, Linkerd), internal microservices

---

## The Full mTLS Handshake in Kubernetes

### kubectl → API Server (Detailed)

```
PHASE 1: TCP Connection
-----------------------
kubectl opens TCP connection to server.kubernetes.local:6443
  -> Resolves to 172.31.20.244:6443
  -> SYN, SYN-ACK, ACK

PHASE 2: TLS Handshake
----------------------
1. CLIENT HELLO
   kubectl: "I support TLS 1.3, cipher suites: [...], 
            here's a random nonce: 0xabc123..."

2. SERVER HELLO + CERTIFICATE
   API Server: "Let's use TLS 1.3, cipher: TLS_AES_256_GCM_SHA384,
               here's my certificate: CN=kube-apiserver, SANs=[...],
               signed by KUBERNETES-CA"

3. CLIENT VERIFICATION
   kubectl: "I have ca.crt, I can verify your signature...
             Valid! Your CN is kube-apiserver, SANs include
             server.kubernetes.local. You are who you claim to be."

4. CLIENT CERTIFICATE
   kubectl: "Here's MY certificate: CN=admin, O=system:masters,
             signed by KUBERNETES-CA"

5. SERVER VERIFICATION
   API Server: "I have ca.crt, verifying your signature...
                Valid! CN=admin, O=system:masters. Welcome, admin."

6. KEY EXCHANGE
   Both sides generate session keys using ephemeral Diffie-Hellman
   (Forward secrecy — even if private keys are later compromised,
    past sessions remain secure)

7. FINISHED
   Both sides send "Finished" message encrypted with session key
   If either can decrypt, handshake succeeded

PHASE 3: Encrypted Application Data
-----------------------------------
All subsequent HTTP requests are encrypted:
- GET /api/v1/pods
- Headers, body, everything encrypted with AES-256-GCM

PHASE 4: Connection Close
-------------------------
Either side sends TLS close_notify
TCP FIN
```

---

## Why Does Every Component Have Its Own Kubeconfig?

You might think: "Why not use the same kubeconfig everywhere?"

### Reason 1: Different Identities

Each component has a different Kubernetes identity:

| Component | Identity | Permissions |
|-----------|----------|-------------|
| kubelet | `system:node:node-1` | Read pods assigned to this node, write node status |
| kube-proxy | `system:kube-proxy` | Read services, endpoints, nodes |
| controller-manager | `system:kube-controller-manager` | Full read access, write to most resources |
| scheduler | `system:kube-scheduler` | Read pods, write bindings |
| admin | `admin` (system:masters) | God mode — everything |

### Reason 2: Least Privilege

If one component is compromised, the attacker only gets that component's permissions:
- Stolen kube-proxy kubeconfig → Can read services, can't delete pods
- Stolen admin kubeconfig → Full cluster access

### Reason 3: Audit and Debugging

When the API server logs a request, it logs the identity:
```
"user":"system:node:node-1","verb":"get","resource":"pods"
```

This helps trace which component made which request.

### Reason 4: Certificate Lifecycle

Different components may have different certificate renewal schedules. Separate kubeconfigs allow independent rotation.

---

## End-to-End Authentication Flows

### Flow 1: Kubelet Registers Node

```
+--------+     mTLS      +-----------------+     mTLS      +--------+
| kubelet|=============>| kube-apiserver  |=============>|  etcd  |
| node-1 |               |                 |               |        |
|        | CN=system:    | validates       | CN=kube-      |        |
|        | node:node-1   | node identity   | apiserver     |        |
+--------+               +-----------------+               +--------+
     |                         |
     | POST /api/v1/nodes      |
     | "I'm node-1, here's    |
     |  my status"             |
     |                         |
     |<----- 201 Created ------|
     | "Welcome to the cluster"|
```

**RBAC Check**: Node Authorizer asks "Is this node trying to modify ITS OWN node object?" Yes → Allow.

### Flow 2: Controller Manager Watches Pods

```
+-------------------+     mTLS      +-----------------+
| kube-controller   |=============>| kube-apiserver  |
| -manager          |               |                 |
|                   | CN=system:    | validates       |
|                   | kube-controller| controller     |
|                   | -manager      | identity        |
+-------------------+               +-----------------+
     |                                     |
     | GET /api/v1/pods?watch=true         |
     | "Tell me about ALL pods"            |
     |                                     |
     |<----- Streaming response -----------|
     | "Pod nginx created in default"      |
     | "Pod nginx status updated"          |
     | ...                                 |
```

**RBAC Check**: Controller manager has read access to all resources. It watches for changes and reacts.

### Flow 3: Scheduler Assigns Pod

```
+-------------------+     mTLS      +-----------------+
| kube-scheduler    |=============>| kube-apiserver  |
|                   |               |                 |
|                   | CN=system:    | validates       |
|                   | kube-scheduler| scheduler       |
+-------------------+               +-----------------+
     |                                     |
     | LIST /api/v1/pods?fieldSelector=    |
     |   spec.nodeName=                    |
     | "Find unscheduled pods"             |
     |                                     |
     |<----- [pod-nginx, unassigned] ------|
     |                                     |
     | POST /api/v1/namespaces/default/    |
     |   pods/nginx/binding                |
     | "Assign nginx to node-1"            |
     |                                     |
     |<----- 201 Created ------------------|
```

**RBAC Check**: Scheduler can create bindings but not arbitrary pods.

---

## Service Account Token Signing

While mTLS is for system components, **service accounts** use JWT tokens for in-cluster authentication.

### How It Works

```
+--------+                          +-----------------+
|  Pod   |                          | kube-apiserver  |
| nginx  |                          |                 |
+--------+                          +-----------------+
     |                                     |
     | 1. Mount service account token      |
     |    /var/run/secrets/kubernetes.io/  |
     |    serviceaccount/token             |
     |                                     |
     | 2. API request with token           |
     |    Authorization: Bearer eyJhbG...  |
     |---------------------------->|
     |                                     |
     |                                     | 3. Verify JWT signature
     |                                     |    using service-account.pub
     |                                     |
     |                                     | 4. Extract identity:
     |                                     |    "serviceaccount:default:nginx"
     |                                     |
     |                                     | 5. RBAC check
     |                                     |
     |<----- Response --------------------|
```

### Token Structure (JWT)

A JWT has three parts separated by dots:
```
eyJhbGciOiJSUzI1NiIs...  <- Header (algorithm, key ID)
.eyJpc3MiOiJrdWJlcm5ldGVz...  <- Payload (claims: issuer, subject, audience, expiry)
.SflKxwRJSMeKKF2QT4f...  <- Signature (signed with service-account.key)
```

**Claims in a Kubernetes service account token:**
- `iss` (issuer) — `https://kubernetes.default.svc.cluster.local`
- `sub` (subject) — `system:serviceaccount:default:nginx`
- `aud` (audience) — `https://kubernetes.default.svc.cluster.local`
- `exp` (expiry) — Unix timestamp when token expires
- `iat` (issued at) — When token was created

The API server verifies the signature using `service-account.pub`. If valid, the token is trusted.

---

## FAQ

### Q: Can I have multiple contexts in one kubeconfig?

A: Yes! This is common for managing multiple clusters:

```yaml
contexts:
- context:
    cluster: prod
    user: admin
  name: prod
- context:
    cluster: staging
    user: admin
  name: staging
current-context: prod
```

Switch with: `kubectl config use-context staging`

### Q: What if I use `--embed-certs=false`?

A: The kubeconfig stores file paths instead of base64 data:
```yaml
certificate-authority: /path/to/ca.crt
client-certificate: /path/to/admin.crt
```

This makes the kubeconfig smaller but less portable — it won't work if the files move.

### Q: Why port 6443?

A: It's the IANA-registered port for Kubernetes API server. Port 443 is commonly used too, but 6443 avoids conflicts with other HTTPS services.

### Q: What is `insecure-skip-tls-verify: true`?

A: A dangerous flag that disables CA verification. Never use in production! It makes you vulnerable to man-in-the-middle attacks.

### Q: Can I use token-based auth instead of certificates?

A: Yes, but only for humans. System components should use mTLS for stronger security. Tokens can be leaked, copied, and don't prove possession of a private key.

---

## Troubleshooting

### Issue: "Unable to connect to the server: x509: certificate signed by unknown authority"

**Cause**: The CA certificate in kubeconfig doesn't match the CA that signed the API server certificate.

**Solution**: Regenerate kubeconfig with correct `ca.crt`.

### Issue: "error: You must be logged in to the server (Unauthorized)"

**Cause**: Client certificate is invalid, expired, or RBAC denies access.

**Solution**: Check certificate dates, verify CN/O fields, check RBAC roles/bindings.

### Issue: "The connection to the server server.kubernetes.local:6443 was refused"

**Cause**: API server is not running (Step 5 not yet completed) or DNS doesn't resolve.

**Solution**: Verify API server is running: `ssh server "sudo systemctl status kube-apiserver"`. Check DNS: `nslookup server.kubernetes.local`.

### Issue: kubectl hangs indefinitely

**Cause**: Firewall blocking port 6443, or API server overloaded.

**Solution**: Check security groups, verify API server process is responsive.

---

## Related Concepts for Other Steps

- **PKI** (Step 2): The certificates referenced by kubeconfigs were generated here
- **etcd** (Step 4): The API server uses its kubeconfig (mTLS) to connect to etcd
- **Control Plane** (Step 5): Components use their kubeconfigs to start and connect
- **Worker Nodes** (Step 6): Kubelet and kube-proxy read kubeconfigs on startup
- **RBAC** (Step 5): After authentication, RBAC determines authorization
