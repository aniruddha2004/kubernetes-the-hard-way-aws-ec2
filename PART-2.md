# Kubernetes The Hard Way on AWS EC2 (Part 2)
## Generating Kubernetes Configuration Files for Authentication

## Goal

In this phase we generate **kubeconfig** files for every Kubernetes component that behaves as a client of the Kubernetes API Server. A kubeconfig bundles together:
- API Server endpoint.
- Trusted CA certificate.
- Client certificate.
- Client private key.
- Default context (cluster + user).

This document follows the AWS-specific setup used in this project:
- Debian 12 EC2 instances.
- `admin` user (not `root`).
- AWS PEM key for SSH.
- Worker nodes named `node-1` and `node-2`.

---

## Components Requiring Kubeconfigs

| Component | Identity | Output File |
|-----------|-----------|-------------|
| kubelet (node-1) | system:node:node-1 | node-1.kubeconfig |
| kubelet (node-2) | system:node:node-2 | node-2.kubeconfig |
| kube-proxy | system:kube-proxy | kube-proxy.kubeconfig |
| kube-controller-manager | system:kube-controller-manager | kube-controller-manager.kubeconfig |
| kube-scheduler | system:kube-scheduler | kube-scheduler.kubeconfig |
| Cluster Admin | admin | admin.kubeconfig |

---

## Generate Worker Node Kubeconfigs

```bash
for host in node-1 node-2; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-credentials system:node:${host} \
    --client-certificate=${host}.crt \
    --client-key=${host}.key \
    --embed-certs=true \
    --kubeconfig=${host}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${host} \
    --kubeconfig=${host}.kubeconfig

  kubectl config use-context default \
    --kubeconfig=${host}.kubeconfig
done
```

Brief:
- Creates a kubeconfig for each worker node.
- Embeds the CA certificate and node certificates.
- Configures the API server endpoint.
- Sets the default context.

---

## Generate kube-proxy Kubeconfig

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.crt \
    --client-key=kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-proxy.kubeconfig
}
```

---

## Generate kube-controller-manager Kubeconfig

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.crt \
    --client-key=kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-controller-manager.kubeconfig
}
```

---

## Generate kube-scheduler Kubeconfig

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.crt \
    --client-key=kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default \
    --kubeconfig=kube-scheduler.kubeconfig
}
```

---

## Generate Admin Kubeconfig

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.crt \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.crt \
    --client-key=admin.key \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default \
    --kubeconfig=admin.kubeconfig
}
```

`admin.kubeconfig` uses `127.0.0.1` because it will later be used locally on the control-plane node.

---

## Copy Worker Kubeconfigs (AWS Modified)

```bash
for host in node-1 node-2; do
  ssh ${host} "sudo mkdir -p /var/lib/{kube-proxy,kubelet}"

  scp kube-proxy.kubeconfig ${host}:~/kube-proxy.kubeconfig
  scp ${host}.kubeconfig ${host}:~/kubelet.kubeconfig

  ssh ${host} "
    sudo mv ~/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig &&
    sudo mv ~/kubelet.kubeconfig /var/lib/kubelet/kubeconfig
  "
done
```

Purpose:
- Create required directories.
- Copy kubeconfig files via the `admin` user.
- Move them into privileged directories with `sudo`.

---

## Copy Control Plane Kubeconfigs

```bash
scp admin.kubeconfig \
  kube-controller-manager.kubeconfig \
  kube-scheduler.kubeconfig \
  admin@server:~/
```

These files will later be consumed by:
- kubectl
- kube-controller-manager
- kube-scheduler

---

## Resulting File Placement

### Worker Nodes

```
/var/lib/kubelet/
├── ca.crt
├── kubelet.crt
├── kubelet.key
└── kubeconfig

/var/lib/kube-proxy/
└── kubeconfig
```

### Control Plane Node

```
~/admin.kubeconfig
~/kube-controller-manager.kubeconfig
~/kube-scheduler.kubeconfig
```

---

## Summary

At the end of this stage:
- Every Kubernetes client component has a unique identity.
- Every client knows where the API server is located.
- Every client knows which CA to trust.
- Kubeconfig files have been distributed to their respective machines.
