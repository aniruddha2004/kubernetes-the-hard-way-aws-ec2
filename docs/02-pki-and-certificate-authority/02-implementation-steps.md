# Step 2: PKI and Certificate Authority - Implementation

> This guide covers generating the Certificate Authority (CA) and all TLS certificates required for Kubernetes components to authenticate each other using mutual TLS (mTLS).

---

## Table of Contents

- [Overview](#overview)
- [1. Certificate Authority (CA) Setup](#1-certificate-authority-ca-setup)
- [2. Admin Client Certificate](#2-admin-client-certificate)
- [3. Kubelet Client Certificates](#3-kubelet-client-certificates)
- [4. Controller Manager Client Certificate](#4-controller-manager-client-certificate)
- [5. Proxy Client Certificate](#5-proxy-client-certificate)
- [6. Scheduler Client Certificate](#6-scheduler-client-certificate)
- [7. Kubernetes API Server Certificate](#7-kubernetes-api-server-certificate)
- [8. Service Account Key Pair](#8-service-account-key-pair)
- [9. Distribute Certificates](#9-distribute-certificates)
- [10. Verification](#10-verification)
- [11. Summary](#11-summary)

---

## Overview

Kubernetes uses **mutual TLS (mTLS)** for authentication between ALL components. This means:
- Every component has a certificate proving its identity
- When two components connect, they BOTH verify each other's certificates
- The CA certificate is the root of trust that validates all other certificates

### Certificates to Generate

| Certificate | Purpose | Used By |
|-------------|---------|---------|
| `ca.crt` / `ca.key` | Root Certificate Authority | Signs all other certificates |
| `admin.crt` / `admin.key` | Admin user identity | kubectl, human administrators |
| `node-1.crt` / `node-1.key` | Kubelet identity | kubelet on node-1 |
| `node-2.crt` / `node-2.key` | Kubelet identity | kubelet on node-2 |
| `kube-controller-manager.crt` | Controller Manager identity | kube-controller-manager |
| `kube-proxy.crt` / `kube-proxy.key` | Proxy identity | kube-proxy |
| `kube-scheduler.crt` / `kube-scheduler.key` | Scheduler identity | kube-scheduler |
| `kubernetes.crt` / `kubernetes.key` | API Server identity | kube-apiserver |
| `service-account.crt` / `service-account.key` | Service account token signing | kube-apiserver |

### Certificate File Locations

After generation, certificates are organized as:
- `admin.crt`, `admin.key` → `admin-certs/`
- `ca.crt` → `ca/`
- `node-*.crt`, `node-*.key` → `kubelet-certs/`
- `kube-*.crt`, `kube-*.key` → `kube-certs/`
- `kubernetes.crt`, `kubernetes.key` → `kube-api-server/`
- `service-account.crt`, `service-account.key` → `service-account/`

---

## 1. Certificate Authority (CA) Setup

### Generate the CA Private Key

```bash
openssl genrsa -out ca.key 4096
```

Expected output:
```
Generating RSA private key, 4096 bit long modulus (2 primes)
...........................................++++
............................................................................................++++
e is 65537 (0x010001)
```

### Generate the CA Certificate

```bash
openssl req -new -x509 -sha256 -days 365 -key ca.key -out ca.crt \
  -subj "/CN=KUBERNETES-CA/O=Kubernetes"
```

Parameters:
- `-new -x509` → Create a self-signed X.509 certificate
- `-sha256` → Use SHA-256 for the signature (secure hash)
- `-days 365` → Valid for 1 year
- `-subj "/CN=KUBERNETES-CA/O=Kubernetes"` → Subject name and organization
  - `CN=KUBERNETES-CA` → Common Name = KUBERNETES-CA
  - `O=Kubernetes` → Organization = Kubernetes

> **Important**: Keep `ca.key` SECRET. Anyone with the CA private key can create valid certificates for your cluster.

---

## 2. Admin Client Certificate

The admin certificate is used by kubectl and human administrators to authenticate to the API server.

### Generate Admin Private Key

```bash
openssl genrsa -out admin.key 4096
```

### Generate Certificate Signing Request (CSR)

```bash
openssl req -new -sha256 -key admin.key -subj "/CN=admin/O=system:masters" -out admin.csr
```

Parameters:
- `CN=admin` → Username is "admin"
- `O=system:masters` → Group is "system:masters" (built-in superuser group in Kubernetes)

### Sign the CSR with the CA

```bash
openssl x509 -req -sha256 -days 365 -in admin.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out admin.crt
```

Parameters:
- `-CA ca.crt` → Use our CA certificate
- `-CAkey ca.key` → Use our CA private key
- `-CAcreateserial` → Auto-generate serial number file

Expected output:
```
Certificate request self-signature ok
subject=CN = admin, O = system:masters
```

---

## 3. Kubelet Client Certificates

Generate a certificate for each worker node's kubelet.

### For node-1

```bash
# Generate private key
openssl genrsa -out node-1.key 4096

# Generate CSR with node-specific identity
openssl req -new -sha256 -key node-1.key -subj "/CN=system:node:node-1/O=system:nodes" -out node-1.csr

# Sign with CA
openssl x509 -req -sha256 -days 365 -in node-1.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out node-1.crt
```

Parameters:
- `CN=system:node:node-1` → Kubernetes username format for nodes: `system:node:<hostname>`
- `O=system:nodes` → Built-in Kubernetes group for all nodes

### For node-2

```bash
# Generate private key
openssl genrsa -out node-2.key 4096

# Generate CSR with node-specific identity
openssl req -new -sha256 -key node-2.key -subj "/CN=system:node:node-2/O=system:nodes" -out node-2.csr

# Sign with CA
openssl x509 -req -sha256 -days 365 -in node-2.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out node-2.crt
```

---

## 4. Controller Manager Client Certificate

```bash
# Generate private key
openssl genrsa -out kube-controller-manager.key 4096

# Generate CSR
openssl req -new -sha256 -key kube-controller-manager.key \
  -subj "/CN=system:kube-controller-manager/O=system:kube-controller-manager" \
  -out kube-controller-manager.csr

# Sign with CA
openssl x509 -req -sha256 -days 365 -in kube-controller-manager.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-controller-manager.crt
```

Parameters:
- `CN=system:kube-controller-manager` → Identifies as the controller manager
- `O=system:kube-controller-manager` → Dedicated group for RBAC

---

## 5. Proxy Client Certificate

```bash
# Generate private key
openssl genrsa -out kube-proxy.key 4096

# Generate CSR
openssl req -new -sha256 -key kube-proxy.key \
  -subj "/CN=system:kube-proxy/O=system:node-proxier" -out kube-proxy.csr

# Sign with CA
openssl x509 -req -sha256 -days 365 -in kube-proxy.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out kube-proxy.crt
```

Parameters:
- `CN=system:kube-proxy` → Identifies as kube-proxy
- `O=system:node-proxier` → Built-in group for proxy components

---

## 6. Scheduler Client Certificate

```bash
# Generate private key
openssl genrsa -out kube-scheduler.key 4096

# Generate CSR
openssl req -new -sha256 -key kube-scheduler.key \
  -subj "/CN=system:kube-scheduler/O=system:kube-scheduler" \
  -out kube-scheduler.csr

# Sign with CA
openssl x509 -req -sha256 -days 365 -in kube-scheduler.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial -out kube-scheduler.crt
```

---

## 7. Kubernetes API Server Certificate

The API server certificate is special because it must include multiple subject alternative names (SANs) — all the names and IPs that clients use to connect to it.

### Generate API Server Private Key

```bash
openssl genrsa -out kubernetes.key 4096
```

### Generate CSR with SANs

Create an OpenSSL config file with SAN extensions:

```bash
cat > kubernetes.cnf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster
DNS.5 = kubernetes.default.svc.cluster.local
DNS.6 = server.kubernetes.local
IP.1 = 10.32.0.1
IP.2 = <SERVER_PRIVATE_IP>
EOF
```

Generate the CSR using the config:
```bash
openssl req -new -sha256 -key kubernetes.key \
  -subj "/CN=kube-apiserver/O=Kubernetes" \
  -config kubernetes.cnf -out kubernetes.csr
```

### Sign the CSR

```bash
openssl x509 -req -sha256 -days 365 -in kubernetes.csr \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -extensions v3_req -extfile kubernetes.cnf \
  -out kubernetes.crt
```

Expected output:
```
Certificate request self-signature ok
subject=CN = kube-apiserver, O = Kubernetes
```

**SANs included:**
- `kubernetes` — Short name for in-cluster access
- `kubernetes.default` — Service DNS name
- `kubernetes.default.svc` — Service DNS name with namespace
- `kubernetes.default.svc.cluster` — Extended service DNS
- `kubernetes.default.svc.cluster.local` — Full cluster-local DNS
- `server.kubernetes.local` — The actual hostname of our API server
- `10.32.0.1` — The first IP in the service CIDR (reserved for kubernetes.default service)
- `<SERVER_PRIVATE_IP>` — Private IP of the control plane node

---

## 8. Service Account Key Pair

Service accounts use a different mechanism — a **key pair** (not a certificate) to sign JWT tokens.

```bash
openssl genrsa -out service-account.key 4096
openssl rsa -in service-account.key -pubout -out service-account.pub
```

The private key (`service-account.key`) is used by the API server to sign tokens. The public key (`service-account.pub`) can be distributed to components that need to verify tokens.

---

## 9. Distribute Certificates

### Create Directory Structure

```bash
mkdir -p {ca,admin-certs,kubelet-certs,kube-certs,kube-api-server,service-account}
```

### Move Certificates

```bash
mv ca.crt ca/
mv admin.crt admin.certs/
mv admin.key admin-certs/
mv node-1.crt node-1.key kubelet-certs/
mv node-2.crt node-2.key kubelet-certs/
mv kube-controller-manager.crt kube-controller-manager.key kube-certs/
mv kube-proxy.crt kube-proxy.key kube-certs/
mv kube-scheduler.crt kube-scheduler.key kube-certs/
mv kubernetes.crt kubernetes.key kube-api-server/
mv service-account.key service-account.pub service-account/
```

### Distribute to Control Plane (server)

```bash
scp ca/ca.crt admin@server:~
scp kube-api-server/kubernetes.crt admin@server:~
scp kube-api-server/kubernetes.key admin@server:~
scp service-account/service-account.key admin@server:~
scp service-account/service-account.pub admin@server:~
```

Move into place on server:
```bash
ssh server "sudo mkdir -p /var/lib/kubernetes"
ssh server "sudo mv ca.crt /var/lib/kubernetes/"
ssh server "sudo mv kubernetes.crt /var/lib/kubernetes/"
ssh server "sudo mv kubernetes.key /var/lib/kubernetes/"
ssh server "sudo mv service-account.key /var/lib/kubernetes/"
ssh server "sudo mv service-account.pub /var/lib/kubernetes/"
ssh server "sudo chmod 600 /var/lib/kubernetes/*.key"
```

### Distribute to Worker Nodes

**For node-1:**
```bash
scp ca/ca.crt admin@node-1:~
scp kubelet-certs/node-1.crt admin@node-1:~
scp kubelet-certs/node-1.key admin@node-1:~
scp kube-certs/kube-proxy.crt admin@node-1:~
scp kube-certs/kube-proxy.key admin@node-1:~
```

Move into place:
```bash
ssh node-1 "sudo mkdir -p /var/lib/kubernetes"
ssh node-1 "sudo mv ca.crt /var/lib/kubernetes/"
ssh node-1 "sudo mv node-1.crt /var/lib/kubernetes/kubelet.crt"
ssh node-1 "sudo mv node-1.key /var/lib/kubernetes/kubelet.key"
ssh node-1 "sudo mv kube-proxy.crt /var/lib/kubernetes/"
ssh node-1 "sudo mv kube-proxy.key /var/lib/kubernetes/"
ssh node-1 "sudo chmod 600 /var/lib/kubernetes/*.key"
```

**For node-2:**
```bash
scp ca/ca.crt admin@node-2:~
scp kubelet-certs/node-2.crt admin@node-2:~
scp kubelet-certs/node-2.key admin@node-2:~
scp kube-certs/kube-proxy.crt admin@node-2:~
scp kube-certs/kube-proxy.key admin@node-2:~
```

Move into place:
```bash
ssh node-2 "sudo mkdir -p /var/lib/kubernetes"
ssh node-2 "sudo mv ca.crt /var/lib/kubernetes/"
ssh node-2 "sudo mv node-2.crt /var/lib/kubernetes/kubelet.crt"
ssh node-2 "sudo mv node-2.key /var/lib/kubernetes/kubelet.key"
ssh node-2 "sudo mv kube-proxy.crt /var/lib/kubernetes/"
ssh node-2 "sudo mv kube-proxy.key /var/lib/kubernetes/"
ssh node-2 "sudo chmod 600 /var/lib/kubernetes/*.key"
```

### Distribute Admin Certificate (for kubectl)

Keep admin certs on the jumpbox for kubectl:
```bash
mkdir -p ~/admin-certs
scp ca/ca.crt ~/admin-certs/
scp admin-certs/admin.crt ~/admin-certs/
scp admin-certs/admin.key ~/admin-certs/
```

---

## 10. Verification

### Verify CA Certificate

```bash
openssl x509 -in ca/ca.crt -text -noout | head -20
```

Expected (check Subject and Issuer):
```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            5b:72:34:...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = KUBERNETES-CA, O = Kubernetes
        Validity
            Not Before: Jun 14 10:00:00 2026 GMT
            Not After : Jun 14 10:00:00 2027 GMT
        Subject: CN = KUBERNETES-CA, O = Kubernetes
```

### Verify API Server Certificate SANs

```bash
openssl x509 -in kube-api-server/kubernetes.crt -text -noout | grep -A 20 "Subject Alternative Name"
```

Expected:
```
            X509v3 Subject Alternative Name:
                DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster, DNS:kubernetes.default.svc.cluster.local, DNS:server.kubernetes.local, IP Address:10.32.0.1, IP Address:<SERVER_PRIVATE_IP>
```

### Verify Admin Certificate

```bash
openssl x509 -in admin-certs/admin.crt -text -noout | grep "Subject:"
```

Expected:
```
        Subject: CN = admin, O = system:masters
```

---

## 11. Summary

At the end of Step 2, confirm:

| Check | Command | Expected |
|-------|---------|----------|
| CA exists | `ls ca/` | `ca.crt` |
| Admin cert | `ls admin-certs/` | `admin.crt`, `admin.key` |
| Kubelet certs | `ls kubelet-certs/` | `node-1.crt`, `node-1.key`, `node-2.crt`, `node-2.key` |
| API server cert | `ls kube-api-server/` | `kubernetes.crt`, `kubernetes.key` |
| Service account keys | `ls service-account/` | `service-account.key`, `service-account.pub` |
| Certs on server | `ssh server "ls /var/lib/kubernetes"` | All server-side certs |
| Certs on node-1 | `ssh node-1 "ls /var/lib/kubernetes"` | `ca.crt`, `kubelet.crt`, `kubelet.key`, `kube-proxy.crt`, `kube-proxy.key` |
| Certs on node-2 | `ssh node-2 "ls /var/lib/kubernetes"` | Same as node-1 |
| Cert validity | `openssl x509 -in ca/ca.crt -dates -noout` | Not expired |

**Next**: Proceed to [Step 3: Kubeconfigs and Authentication](../03-kubeconfigs-and-authentication/03-implementation-steps.md) to create the authentication configuration files.
