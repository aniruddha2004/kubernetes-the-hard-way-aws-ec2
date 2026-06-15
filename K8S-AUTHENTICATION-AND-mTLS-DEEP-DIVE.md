# Kubernetes Authentication, Kubeconfig and mTLS Deep Dive

## 1. Core Mental Model

A Kubernetes cluster revolves around one central component:

```
Clients <----> kube-apiserver <----> etcd
```

Almost every Kubernetes component behaves as a client of the API server.

---

## 2. PKI Basics

### Private Key
A secret cryptographic key that must never leave the machine.

### Public Key
Can be shared freely. It is embedded inside certificates.

### Certificate Authority (CA)

A CA has:
- `ca.key` (private signing key)
- `ca.crt` (public verification certificate)

The CA signs certificates. Everyone verifies them using `ca.crt`.

### Analogy

- `ca.key` = passport office stamping machine.
- `ca.crt` = public sample of the official stamp.

---

## 3. Certificate Signing Request (CSR)

Flow:

```
Generate private key
        ↓
Create CSR (contains identity + public key)
        ↓
CA signs CSR using ca.key
        ↓
Certificate (.crt) produced
```

A CSR says:
> "I claim to be CN=system:node:node-1. Please certify me."

---

## 4. What is Inside a Certificate?

A certificate contains:
- Subject (identity).
- Issuer (who signed it).
- Public key.
- Validity period.
- Extensions (clientAuth/serverAuth).
- CA digital signature.

Examples:
- `CN=admin`
- `CN=system:node:node-1`
- `O=system:masters`
- `O=system:nodes`

---

## 5. Kubernetes Identities

| Component | Certificate Identity |
|------------|--------------------|
| Admin | CN=admin, O=system:masters |
| Node-1 | CN=system:node:node-1 |
| Node-2 | CN=system:node:node-2 |
| kube-proxy | CN=system:kube-proxy |
| Scheduler | CN=system:kube-scheduler |

The API server authenticates clients by extracting these values from certificates.

---

## 6. What is a Kubeconfig?

A kubeconfig answers four questions:

1. Which API server do I contact?
2. Which CA do I trust?
3. Who am I?
4. How do I prove my identity?

A kubeconfig contains:
- Cluster.
- User.
- Context.
- Current Context.

### Cluster

Stores:
- API server URL.
- Trusted CA certificate.

### User

Stores:
- Client certificate.
- Client private key.

### Context

Binds one user to one cluster.

### Current Context

Selects which context is active.

---

## 7. Why Does Every Component Have Its Own Kubeconfig?

Each component is an independent API client.

| Component | Uses |
|------------|------|
| kubelet | node-X.kubeconfig |
| kube-proxy | kube-proxy.kubeconfig |
| scheduler | kube-scheduler.kubeconfig |
| controller-manager | kube-controller-manager.kubeconfig |
| kubectl | admin.kubeconfig |

The kubeconfig acts as an "identity package".

---

## 8. API Server as the Hub

Normal components never communicate directly with etcd.

```
kubectl ----\
kubelet -----\
scheduler ----> kube-apiserver ----> etcd
proxy -------/
```

The API server is the single source of truth interface.

---

## 9. API Server and etcd

The API server acts as an etcd client.

- Client: kube-apiserver.
- Server: etcd.

The API server establishes an mTLS connection to etcd before reading or writing cluster state.

---

## 10. TLS vs mTLS

### TLS

Only the server proves its identity.

```
Browser ----TLS----> Website
```

### mTLS

Both sides prove their identity.

```
kubelet <--mTLS--> kube-apiserver
apiserver <--mTLS--> etcd
```

---

## 11. TCP vs TLS

TLS runs on top of TCP.

### TCP Handshake

```
Client ---- SYN ----> Server
Client <-- SYN/ACK -- Server
Client ---- ACK ----> Server
```

Only after TCP is established does TLS begin.

---

## 12. Full mTLS Handshake

### Step 1
TCP connection established.

### Step 2
Client sends `ClientHello`.

### Step 3
Server replies with:
- ServerHello.
- Server certificate (`etcd.crt`).

### Step 4
Client verifies server certificate using `ca.crt`.

### Step 5
Server requests a client certificate (`CertificateRequest`).

### Step 6
Client sends:
- Client certificate.
- CertificateVerify message.

### Step 7
Server verifies:
- Certificate signature using `ca.crt`.
- Client owns the private key.

### Step 8
Both exchange `Finished` messages.

Secure encrypted channel established.

---

## 13. How Does etcd Verify the API Server?

Suppose etcd has:

```
--trusted-ca-file=ca.crt
--client-cert-auth=true
```

The API server presents `kube-api-server.crt`.

etcd performs:
1. Check certificate validity.
2. Verify CA signature using `ca.crt`.
3. Check certificate extensions.
4. Verify proof of private key ownership.

The same CA certificate is used by both sides.

---

## 14. Proof of Possession

The private key is never transmitted.

Conceptually:

```
etcd: "Sign this random challenge."

API Server:
    signature = SIGN(challenge, kube-api-server.key)

etcd:
    VERIFY(signature,
           public_key_inside(kube-api-server.crt))
```

If verification succeeds, the API server owns the corresponding private key.

---

## 15. Why is ca.crt Public?

| File | Purpose |
|------|----------|
| ca.key | Creates signatures |
| ca.crt | Verifies signatures |

You can distribute `ca.crt` everywhere.
If `ca.key` leaks, an attacker can issue fake certificates.

---

## 16. Service Account Tokens vs mTLS

Control-plane components:
- Authenticate using client certificates (mTLS).

Pods:
- Usually authenticate using Service Account JWT tokens.

The `service-accounts.key` pair is used by the controller-manager to sign those JWT tokens.

---

## 17. End-to-End Flow: kubectl get pods

```
kubectl
   ↓
Read admin.kubeconfig
   ↓
Connect to API Server
   ↓
Verify API Server certificate using ca.crt
   ↓
Present admin.crt
   ↓
Prove ownership using admin.key
   ↓
API Server authenticates:
CN=admin
O=system:masters
   ↓
API Server queries etcd
   ↓
API Server returns Pod list
```

---

## 18. End-to-End Flow: API Server to etcd

```
TCP: SYN -> SYN/ACK -> ACK
       ↓
TLS ClientHello
       ↓
TLS ServerHello
       ↓
etcd.crt sent
       ↓
API server verifies using ca.crt
       ↓
API server sends kube-api-server.crt
       ↓
CertificateVerify
       ↓
etcd verifies using same ca.crt
       ↓
Finished messages
       ↓
Encrypted mTLS session established
```

---

## 19. Mental Models

### Office Building Analogy

A kubeconfig contains:
- Office address.
- Company root certificate.
- Employee ID card.
- Employee cryptographic token.
- Which identity to use.

### Passport Office Analogy

CA = Passport office.

Certificate = Passport.

Private key = Secret biometric chip proving the passport belongs to you.

### Web Application Analogy

| Web World | Kubernetes |
|------------|------------|
| Mobile App | kubectl / kubelet |
| REST API | kube-apiserver |
| Database | etcd |
| Login Credentials | Certificates |

---

## 20. Final Summary

A Kubernetes authentication flow is built from four layers:

1. TCP establishes connectivity.
2. TLS encrypts communication.
3. mTLS authenticates both sides.
4. Kubeconfig tells each client how to perform the above steps.

A kubeconfig does not replace certificates; it simply binds together:
- API server endpoint,
- trusted CA,
- client certificate,
- client private key,
- and the selected context.

The API server is the center of the cluster, and almost every Kubernetes component—including the control plane itself when talking to etcd—acts as a secure client using this model.
