# Step 2: PKI and Certificate Authority - Detailed Explanations

> Deep-dive explanations for every concept in Step 2: How PKI works, what certificates are, how mTLS enables Kubernetes security, and why each certificate is configured the way it is.

---

## Table of Contents

- [What is PKI?](#what-is-pki)
- [What is a Certificate?](#what-is-a-certificate)
- [The Certificate Authority (CA)](#the-certificate-authority-ca)
- [Mutual TLS (mTLS) in Kubernetes](#mutual-tls-mtls-in-kubernetes)
- [Certificate Identity Mapping](#certificate-identity-mapping)
- [Subject Alternative Names (SANs)](#subject-alternative-names-sans)
- [The mTLS Handshake](#the-mtls-handshake)
- [Proof of Possession](#proof-of-possession)
- [Service Account Tokens vs mTLS](#service-account-tokens-vs-mtls)
- [End-to-End Authentication Flow](#end-to-end-authentication-flow)
- [Mental Models](#mental-models)

---

## What is PKI?

**PKI (Public Key Infrastructure)** is a system for creating, managing, distributing, using, storing, and revoking digital certificates. It's the foundation of trust on the internet and in Kubernetes.

### Core Components of PKI

1. **Private Key** — Kept secret, used to decrypt data or sign certificates
2. **Public Key** — Shared openly, used to encrypt data or verify signatures
3. **Certificate** — A digital document that binds a public key to an identity
4. **Certificate Authority (CA)** — Trusted entity that signs certificates
5. **Chain of Trust** — Root CA → Intermediate CA → End-entity certificates

### How It Works (Simple Analogy)

Imagine a government (CA) that issues passports (certificates):
- The government has a master seal (CA private key)
- When you apply for a passport, you prove your identity (generate CSR)
- The government verifies you and stamps your passport (signs certificate)
- Anyone who trusts the government can verify your passport is real (validate certificate)
- The passport contains your photo and details (public key and identity)

---

## What is a Certificate?

A TLS certificate is an X.509 digital document containing:

### Certificate Anatomy

```
+--------------------------------------------------+
|                  CERTIFICATE                       |
+--------------------------------------------------+
| Subject (Who am I?)                              |
|   CN = admin                                     |
|   O = system:masters                             |
+--------------------------------------------------+
| Issuer (Who signed me?)                          |
|   CN = KUBERNETES-CA                             |
|   O = Kubernetes                                 |
+--------------------------------------------------+
| Public Key (Encrypt to me)                       |
|   (2048-4096 bit RSA key)                        |
+--------------------------------------------------+
| Validity Period                                  |
|   Not Before: 2026-06-14                         |
|   Not After:  2027-06-14                         |
+--------------------------------------------------+
| Extensions                                       |
|   Subject Alternative Names (SANs)               |
|   Key Usage (digitalSignature, keyEncipherment)  |
|   Extended Key Usage (clientAuth, serverAuth)    |
+--------------------------------------------------+
| CA Signature (Proves I'm legitimate)             |
|   (Encrypted hash by CA private key)             |
+--------------------------------------------------+
```

### Key Fields Explained

**Subject (Who)**
- `CN` (Common Name) — The primary identity. In Kubernetes, this becomes the username.
- `O` (Organization) — The group membership. In Kubernetes, this maps to RBAC groups.

**Issuer (Trusted By)**
- Who signed this certificate. Clients must trust this issuer.

**Public Key**
- Mathematically linked to the private key. Used to encrypt data that only the private key holder can decrypt.

**Validity**
- Certificates expire! This limits the window of compromise if a key is stolen.

**Extensions**
- Additional metadata. SANs are critical for Kubernetes API server certificates.

**Signature**
- The CA encrypted a hash of the certificate contents. Anyone can verify this with the CA public key.

---

## The Certificate Authority (CA)

The CA is the **root of trust** in your Kubernetes cluster. All components trust certificates signed by this CA.

### Why Self-Signed?

In this tutorial, we create a **self-signed CA** — the CA signs its own certificate. This is common for internal/private PKI:
- No need to pay a commercial CA (like DigiCert or Let's Encrypt)
- Complete control over certificate policies
- Suitable for internal infrastructure

In production, you might use:
- An internal corporate CA
- HashiCorp Vault
- AWS Private CA
- cert-manager with Let's Encrypt (for public-facing endpoints)

### CA Certificate vs CA Key

| File | Purpose | Security |
|------|---------|----------|
| `ca.crt` | Public — distributed to ALL nodes | Can be shared freely |
| `ca.key` | Private — kept ONLY on jumpbox | Must be protected! |

> **⚠️ CRITICAL**: If `ca.key` is compromised, an attacker can create valid certificates for ANY component in your cluster. Store it securely, ideally encrypted at rest.

---

## Mutual TLS (mTLS) in Kubernetes

Standard TLS (like HTTPS) authenticates the **server** to the **client**:
- Browser connects to bank.com
- Bank presents certificate
- Browser verifies certificate is signed by trusted CA
- Browser trusts bank.com

**mTLS goes further** — BOTH sides authenticate each other:
- Client presents certificate to server
- Server verifies client certificate
- Server presents certificate to client
- Client verifies server certificate
- Only if BOTH succeed is the connection established

### Why mTLS for Kubernetes?

Kubernetes components are distributed across multiple machines. Every connection must be mutually authenticated:
- kubectl → API Server (prove you're admin)
- API Server → etcd (prove you're the API server)
- Kubelet → API Server (prove you're node-1)
- Controller Manager → API Server (prove you're the controller)

Without mTLS, any machine on the network could impersonate a Kubernetes component.

---

## Certificate Identity Mapping

Kubernetes maps certificate subjects to usernames and groups for RBAC:

| Certificate | CN (Username) | O (Group) | Used By |
|-------------|---------------|-----------|---------|
| admin | `admin` | `system:masters` | kubectl |
| node-1 | `system:node:node-1` | `system:nodes` | kubelet on node-1 |
| node-2 | `system:node:node-2` | `system:nodes` | kubelet on node-2 |
| kube-controller-manager | `system:kube-controller-manager` | `system:kube-controller-manager` | controller manager |
| kube-proxy | `system:kube-proxy` | `system:node-proxier` | kube-proxy |
| kube-scheduler | `system:kube-scheduler` | `system:kube-scheduler` | scheduler |
| kubernetes | `kube-apiserver` | `Kubernetes` | API server |

### Built-in Kubernetes Groups

| Group | Purpose |
|-------|---------|
| `system:masters` | Superuser group — bypasses all RBAC checks |
| `system:nodes` | All kubelets belong here |
| `system:node-proxier` | All kube-proxy instances |
| `system:kube-controller-manager` | Controller manager |
| `system:kube-scheduler` | Scheduler |

---

## Subject Alternative Names (SANs)

SANs allow a single certificate to be valid for **multiple identities** — multiple hostnames AND multiple IP addresses.

### Why SANs Matter for Kubernetes API Server

The API server is accessed by many different names:

1. **Inside pods** — `https://kubernetes` (short name)
2. **Inside pods with namespace** — `https://kubernetes.default`
3. **Full service DNS** — `https://kubernetes.default.svc.cluster.local`
4. **Direct hostname** — `https://server.kubernetes.local`
5. **Direct IP** — `https://<SERVER_PRIVATE_IP>`
6. **Service IP** — `https://10.32.0.1` (the kubernetes service)

If the API server certificate lacks ANY of these SANs, connections using that name will fail with:
```
x509: certificate is valid for kubernetes, kubernetes.default, ...
not server.custom-kubernetes.local
```

### The `10.32.0.1` IP

This is the **first IP in the service CIDR** (`10.32.0.0/24`). Kubernetes automatically creates a Service named `kubernetes` in the `default` namespace with this cluster IP. All in-cluster API access uses this IP.

---

## The mTLS Handshake

Here's the full sequence when kubectl connects to the API server:

```
CLIENT (kubectl)                           SERVER (API Server)
     |                                             |
     |-------- 1. Client Hello ------------------->|
     |    "I support TLS 1.3, here's a random"     |
     |                                             |
     |<------- 2. Server Hello --------------------|
     |    "Let's use TLS 1.3, here's my random"    |
     |                                             |
     |<------- 3. Server Certificate --------------|
     |    "Here's my cert, signed by KUBERNETES-CA"|
     |                                             |
     |    [Client verifies: CN=kube-apiserver,     |
     |     SANs include server.kubernetes.local,   |
     |     signed by trusted CA]                   |
     |                                             |
     |-------- 4. Client Certificate ------------->|
     |    "Here's MY cert, signed by KUBERNETES-CA"|
     |                                             |
     |    [Server verifies: CN=admin,              |
     |     O=system:masters, signed by trusted CA] |
     |                                             |
     |<------- 5. Server Key Exchange -------------|
     |-------- 6. Client Key Exchange ------------>|
     |    [Exchange session keys]                  |
     |                                             |
     |<------- 7. Finished ------------------------|
     |-------- 8. Finished ----------------------->|
     |                                             |
     |======== ENCRYPTED CONNECTION ESTABLISHED ===|
     |                                             |
```

### What Makes It "Mutual"

In standard TLS (HTTPS), only steps 1-3 happen. The client verifies the server.

In mTLS, steps 4-8 also happen. The server verifies the client.

Both sides must present valid certificates signed by the same CA.

---

## Proof of Possession

When generating a certificate, how does the CA know the requester actually owns the private key?

Answer: **The CSR is signed with the private key.**

```
1. You generate a key pair (private + public)
2. You create a CSR containing your public key and identity
3. You SIGN the CSR with your PRIVATE KEY
4. You send CSR to CA
5. CA verifies the signature using the PUBLIC KEY in the CSR
6. If verification succeeds, CA knows you possess the private key
7. CA signs and returns the certificate
```

This prevents an attacker from requesting a certificate for someone else's identity.

---

## Service Account Tokens vs mTLS

These are two different authentication mechanisms in Kubernetes:

### mTLS (Certificate-Based)
- Used by: System components (kubelet, controller manager, etc.)
- Mechanism: Present X.509 certificate during TLS handshake
- Pros: Strong cryptographic identity, no tokens to manage
- Cons: Requires certificate distribution and renewal

### Service Account Tokens (JWT-Based)
- Used by: Pods/applications running in the cluster
- Mechanism: Present a JWT token in the HTTP Authorization header
- Pros: Easy to rotate, can be bound to specific pods
- Cons: Tokens can be leaked, require API server to validate

### The Service Account Key Pair

The `service-account.key` is used to **sign** JWT tokens. When a pod requests a token:

```
1. API Server generates JWT with pod identity
2. API Server signs JWT with service-account.key
3. Pod receives token and uses it in API requests
4. API Server verifies token signature with service-account.pub
```

This is separate from mTLS because pods don't have X.509 identities — they have tokens.

---

## End-to-End Authentication Flow

### kubectl get pods (Admin → API Server)

```
admin@jumpbox$ kubectl get pods
     |
     v
+---------+     mTLS      +-----------------+
| kubectl |<=============>| kube-apiserver  |
|         |               |                 |
| admin   |  proves       | validates       |
| cert    |  identity     | CN=admin        |
|         |               | O=system:masters|
+---------+               +-----------------+
                                   |
                                   v
                            [RBAC Check]
                            "system:masters
                             can do ANYTHING"
                                   |
                                   v
                            [Return pod list]
```

### API Server → etcd

```
+-----------------+     mTLS      +--------+
| kube-apiserver  |<=============>|  etcd  |
|                 |               |        |
| kubernetes.crt  |  proves       | validates|
|                 |  identity     | CN=kube-|
|                 |               | apiserver|
+-----------------+               +--------+
```

### Kubelet → API Server (Node Registration)

```
+---------+     mTLS      +-----------------+
| kubelet |<=============>| kube-apiserver  |
| node-1  |               |                 |
| cert    |  CN=system:   | validates       |
|         |  node:node-1  | CN=system:node: |
|         |               | node-1          |
+---------+               +-----------------+
                                   |
                                   v
                            [Node Authorizer]
                            "Can this node
                             access its own
                             resources?"
```

---

## Mental Models

### Model 1: The Office Building

Imagine an office building (Kubernetes cluster) with security:

- **CA (Certificate Authority)** = The company that makes employee badges
- **Certificate** = Your employee badge with photo, name, department
- **Private Key** = Your fingerprint (only you have it)
- **mTLS Handshake** = Both people show badges and scan fingerprints at every door
- **RBAC** = Badge color determines which floors/rooms you can access
  - Red badge (`system:masters`) = All-access pass
  - Blue badge (`system:nodes`) = Server room only
  - Green badge (`system:kube-proxy`) = Network closet only

### Model 2: The Passport Office

- **Root CA** = Your country's passport office
- **Intermediate CA** = Embassy abroad
- **Certificate** = Your passport
- **Private Key** = Your biometric data (fingerprint, iris scan)
- **Validation** = Border guard scans passport chip and verifies with passport office database

### Model 3: The Web Application

Compare Kubernetes mTLS to a typical web app:

| Web App | Kubernetes mTLS |
|---------|-----------------|
| User registers with username/password | Component gets certificate from CA |
| User logs in with username/password | Component presents certificate |
| Server checks password hash | Server verifies certificate signature |
| Session cookie issued | TLS session established |
| Cookie sent with every request | Certificate presented on every connection |
| Server validates cookie | Server validates certificate |

---

## FAQ

### Q: Why 4096-bit RSA keys?

A: 2048-bit is still secure, but 4096-bit provides greater security margin against future quantum computing advances. The trade-off is slightly slower operations.

### Q: Why 365 days validity?

A: Shorter validity reduces the window of exposure if a certificate is compromised. In production, you might use 90 days with automated rotation (like Let's Encrypt).

### Q: What happens when a certificate expires?

A: The component will fail to authenticate. Kubernetes components will crash-loop with TLS errors. You must regenerate and redistribute certificates before expiry.

### Q: Can I use ECDSA instead of RSA?

A: Yes. ECDSA provides equivalent security with smaller key sizes and faster operations:
```bash
openssl ecparam -genkey -name prime256v1 -out ca.key
```

### Q: What's the difference between `.crt` and `.pem`?

A: They're both PEM-encoded (Base64) certificates. `.crt` is convention for certificates, `.pem` is generic. The content format is identical.

### Q: Why do we use `system:` prefix for Kubernetes identities?

A: The `system:` prefix is reserved for Kubernetes internal users and groups. It prevents conflicts with regular user accounts.

---

## Troubleshooting

### Issue: "x509: certificate signed by unknown authority"

**Cause**: The CA certificate is not in the client's trust store.

**Solution**: Ensure `ca.crt` is distributed to all nodes and referenced in kubeconfigs.

### Issue: "x509: certificate is valid for X, not Y"

**Cause**: The certificate's SANs don't include the hostname/IP being connected to.

**Solution**: Regenerate the certificate with the correct SANs. Double-check the `kubernetes.cnf` file.

### Issue: "remote error: tls: bad certificate"

**Cause**: The client certificate doesn't match what the server expects, or the certificate has expired.

**Solution**: Verify certificate dates, CN, and O fields match the expected identity.

### Issue: Permission denied after successful TLS

**Cause**: TLS succeeded, but RBAC denies the action.

**Solution**: Check that the certificate's `O` field maps to a group with the necessary permissions.

---

## Related Concepts for Other Steps

- **Kubeconfigs** (Step 3): Reference these certificates to define how to connect
- **Encryption Config** (Step 4): Uses CA identity for etcd encryption
- **RBAC** (Step 5): Maps certificate groups to permissions
- **Service Accounts** (Step 8): Use service-account.key for token signing
