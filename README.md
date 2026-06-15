# Kubernetes The Hard Way on AWS EC2

> A comprehensive, step-by-step guide to bootstrapping a Kubernetes cluster from scratch on AWS EC2, adapted from [Kelsey Hightower's Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way).

## Overview

This repository documents the complete journey of building a Kubernetes cluster "the hard way" — manually configuring every component without automation tools. The goal is to deeply understand how Kubernetes works under the hood.

### Cluster Architecture

```
                        +------------------+
                        |     kubectl      |
                        +--------+---------+
                                 |
                                 | HTTPS + mTLS
                                 v
                       +--------------------+
                       |  kube-apiserver    |
                       +----+----------+----+
                            |          |
                            |          |
                            v          v
                   kube-scheduler   kube-controller-manager
                            \          /
                             \        /
                              \      /
                               v    v
                                etcd
                                 |
          +----------------------+----------------------+
          |                                             |
  +-------+---------+                           +-------+---------+
  |    node-1       |                           |    node-2       |
  |-----------------|                           |-----------------|
  | kubelet         |                           | kubelet         |
  | kube-proxy      |                           | kube-proxy      |
  | containerd      |                           | containerd      |
  | CNI plugins     |                           | CNI plugins     |
  +-----------------+                           +-----------------+
```

### Component Versions

| Component | Version |
|-----------|---------|
| Kubernetes | v1.32.3 |
| containerd | v2.1.0-beta.0 |
| CNI plugins | v1.6.2 |
| etcd | v3.6.0-rc.3 |
| crictl | v1.32.0 |
| runc | v1.3.0-rc.1 |

### Key Adaptations from Original Guide

This AWS EC2 adaptation differs from the original tutorial in the following ways:

| Aspect | Original | This AWS Version |
|--------|----------|------------------|
| Platform | Google Cloud | AWS EC2 |
| OS user | `root` | `admin` (default Debian user) |
| SSH keys | Generated via `ssh-keygen` | Pre-existing AWS `.pem` keypair |
| Worker nodes | `node-0`, `node-1` | `node-1`, `node-2` |
| Network access | All nodes have internet | Workers are private (offline packages) |

## Cluster Nodes

| Name | Purpose | Instance Type | Private IP |
|------|---------|---------------|------------|
| `jumpbox` | Administration / Jump host | t3.micro | <JUMPBOX_PRIVATE_IP> |
| `server` | Kubernetes control plane | t3.small | <SERVER_PRIVATE_IP> |
| `node-1` | Worker node | t3.small | <NODE_1_PRIVATE_IP> |
| `node-2` | Worker node | t3.small | <NODE_2_PRIVATE_IP> |

## Guide Structure

This repository is organized into sequential steps. Each step has its own folder containing:
- **Implementation Steps** (`*-implementation-steps.md`): Exact commands to run
- **Deep Dive Explanations** (`*-detailed-explanations.md`): How and why things work, architecture diagrams, troubleshooting

This separation lets you follow the commands when setting up the cluster, and read the deep explanations when you want to understand the internals.

### Quick Navigation

| Part | Topic | Implementation | Explanations |
|------|-------|----------------|--------------|
| Part 1 | Prerequisites & Infrastructure Setup | [`01-implementation-steps.md`](docs/01-prerequisites-and-infrastructure/01-implementation-steps.md) | [`02-detailed-explanations.md`](docs/01-prerequisites-and-infrastructure/02-detailed-explanations.md) |
| Part 2 | PKI and Certificate Authority | [`02-implementation-steps.md`](docs/02-pki-and-certificate-authority/02-implementation-steps.md) | [`02-detailed-explanations.md`](docs/02-pki-and-certificate-authority/02-detailed-explanations.md) |
| Part 3 | Kubeconfigs and Authentication | [`03-implementation-steps.md`](docs/03-kubeconfigs-and-authentication/03-implementation-steps.md) | [`03-detailed-explanations.md`](docs/03-kubeconfigs-and-authentication/03-detailed-explanations.md) |
| Part 4 | Encryption Config and etcd Bootstrap | [`04-implementation-steps.md`](docs/04-encryption-config-and-etcd/04-implementation-steps.md) | [`04-detailed-explanations.md`](docs/04-encryption-config-and-etcd/04-detailed-explanations.md) |
| Part 5 | Control Plane Bootstrap | [`05-implementation-steps.md`](docs/05-control-plane-bootstrap/05-implementation-steps.md) | [`05-detailed-explanations.md`](docs/05-control-plane-bootstrap/05-detailed-explanations.md) |
| Part 6 | Worker Nodes Bootstrap | [`06-implementation-steps.md`](docs/06-worker-node-bootstrap/06-implementation-steps.md) | [`06-detailed-explanations.md`](docs/06-worker-node-bootstrap/06-detailed-explanations.md) |
| Part 7 | Pod Network Routes and kubectl Remote Access | [`07-implementation-steps.md`](docs/07-pod-network-routes-and-kubectl/07-implementation-steps.md) | [`07-detailed-explanations.md`](docs/07-pod-network-routes-and-kubectl/07-detailed-explanations.md) |
| Part 8 | Smoke Test and Cluster Validation | [`08-implementation-steps.md`](docs/08-smoke-test-and-validation/08-implementation-steps.md) | [`08-detailed-explanations.md`](docs/08-smoke-test-and-validation/08-detailed-explanations.md) |
| Appendix | AWS Infrastructure Specifications | [`aws-infrastructure-specs.md`](docs/appendix-aws-infrastructure/aws-infrastructure-specs.md) | [`aws-infrastructure-explanations.md`](docs/appendix-aws-infrastructure/aws-infrastructure-explanations.md) |

## What You Will Learn

By following this guide, you will gain hands-on experience with:

1. **Infrastructure Provisioning**: Setting up EC2 instances with proper security groups
2. **Networking**: Configuring hostnames, DNS resolution, and SSH access between nodes
3. **Public Key Infrastructure (PKI)**: Creating a Certificate Authority and generating TLS certificates
4. **Authentication**: Understanding mTLS, kubeconfigs, and Kubernetes identities
5. **Data Encryption**: Configuring encryption-at-rest for Kubernetes Secrets
6. **etcd**: Deploying and managing the Kubernetes backing store
7. **Control Plane**: Installing kube-apiserver, kube-controller-manager, and kube-scheduler
8. **Worker Nodes**: Installing kubelet, kube-proxy, containerd, and CNI plugins
9. **Networking Routes**: Configuring inter-node Pod communication
10. **Validation**: Testing the cluster with real workloads

## Target Audience

This guide is designed for anyone who wants to understand Kubernetes internals, including:
- DevOps engineers preparing for CKA/CKAD certifications
- System administrators transitioning to cloud-native infrastructure
- Developers who want to understand how Kubernetes works under the hood
- Anyone interested in learning infrastructure-as-code principles manually first

## Prerequisites

Before starting, you should have:
- An AWS account with permissions to create EC2 instances
- A basic understanding of Linux command line
- Familiarity with SSH and SCP
- Basic networking concepts (IP addresses, subnets, routes)
- Optional: Basic understanding of TLS/SSL certificates

## Important Notes

- **Zero Information Loss**: This guide captures every command, error, and fix encountered during the actual deployment. Nothing has been sanitized or simplified.
- **Production Disclaimer**: This setup is for educational purposes. Do not use this configuration in production without additional hardening.
- **Offline Workers**: Worker nodes in this setup do not have internet access. All required packages and container images are transferred manually from the jumpbox.
- **Cost Considerations**: Running 4 EC2 instances will incur AWS charges. Remember to clean up resources when finished.

## License

This documentation is provided for educational purposes. The original [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) by Kelsey Hightower is licensed under Apache 2.0.

## Contributing

This is a personal learning journey documented in detail. If you spot errors or have suggestions, feel free to open an issue.

---

**Ready to start?** Begin with [Part 1: Prerequisites & Infrastructure Setup](docs/01-prerequisites-and-infrastructure.md).
