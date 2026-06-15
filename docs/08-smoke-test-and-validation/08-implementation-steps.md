# Step 8: Smoke Test and Validation - Implementation

> This guide validates the entire Kubernetes cluster by running end-to-end tests: secret encryption, deploying an application, port forwarding, logs, exec, and NodePort services.

---

## Table of Contents

- [Overview](#overview)
- [1. Test Secret Encryption](#1-test-secret-encryption)
- [2. Deploy a Test Application](#2-deploy-a-test-application)
- [3. Verify Pod Status](#3-verify-pod-status)
- [4. Port Forwarding](#4-port-forwarding)
- [5. Container Logs](#5-container-logs)
- [6. Execute Commands in Container](#6-execute-commands-in-container)
- [7. Create a NodePort Service](#7-create-a-nodeport-service)
- [8. Scale the Deployment](#8-scale-the-deployment)
- [9. Clean Up](#9-clean-up)
- [10. Validation Checklist](#10-validation-checklist)
- [11. Issues Encountered](#11-issues-encountered)
- [12. Summary](#12-summary)

---

## Overview

A **smoke test** is a quick validation that all major subsystems work together. A single `kubectl create deployment` exercises nearly every component:

```
kubectl create deployment nginx --image=nginx:alpine
     |
     v
kubectl (with kubeconfig) -> mTLS -> API Server -> etcd
     |
     v
Controller Manager -> ReplicaSet Controller -> Pod object
     |
     v
Scheduler -> assigns node
     |
     v
Kubelet -> containerd -> CNI -> Pod Running
     |
     v
kube-proxy -> iptables rules -> Service accessible
```

---

## 1. Test Secret Encryption

### Create a Secret

```bash
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

Expected:
```
secret/kubernetes-the-hard-way created
```

### Verify Encryption at Rest

Read the secret directly from etcd to confirm it's encrypted:

```bash
ssh server "ETCDCTL_API=3 etcdctl get /registry/secrets/default/kubernetes-the-hard-way \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/var/lib/kubernetes/ca.crt \
  --cert=/var/lib/kubernetes/kube-api-server.crt \
  --key=/var/lib/kubernetes/kube-api-server.key"
```

Expected output (binary, but shows encrypted prefix):
```
/registry/secrets/default/kubernetes-the-hard-way
k8s:enc:aescbc:v1:key1:[encrypted binary data...]
```

The `k8s:enc:aescbc:v1:key1` prefix confirms encryption is working.

### Read Secret via kubectl

```bash
kubectl get secret kubernetes-the-hard-way -o yaml
```

Expected (data is base64-encoded):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kubernetes-the-hard-way
  namespace: default
data:
  mykey: bXlkYXRh
```

Decode the value:
```bash
echo "bXlkYXRh" | base64 -d
```

Expected:
```
mydata
```

---

## 2. Deploy a Test Application

### Create Deployment

```bash
kubectl create deployment nginx --image=nginx:alpine
```

Expected:
```
deployment.apps/nginx created
```

### Check Deployment Status

```bash
kubectl get deployment nginx
```

Expected:
```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           30s
```

### Check ReplicaSet

```bash
kubectl get replicaset -l app=nginx
```

Expected:
```
NAME              DESIRED   CURRENT   READY   AGE
nginx-7bf8c87b5   1         1         1       30s
```

### Check Pod

```bash
kubectl get pods -l app=nginx -o wide
```

Expected:
```
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-7bf8c87b5-abcde  1/1     Running   0          45s   10.200.0.x   node-1
```

> **Note**: If pod stays in `ContainerCreating` or `Pending`, see the [Issues Encountered](#11-issues-encountered) section.

---

## 3. Verify Pod Status

### Describe the Pod

```bash
kubectl describe pod -l app=nginx
```

Check for:
- `Status: Running`
- No error events in the Events section
- Correct node assignment
- IP assigned from pod CIDR

### Check Pod Logs

```bash
kubectl logs -l app=nginx
```

Expected (nginx startup logs):
```
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
...
```

---

## 4. Port Forwarding

Forward a local port to the nginx pod:

```bash
kubectl port-forward deployment/nginx 8080:80
```

Expected:
```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

In another terminal on the jumpbox, test:

```bash
curl -s http://localhost:8080 | head -5
```

Expected:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

Press `Ctrl+C` to stop port-forward.

---

## 5. Container Logs

Stream logs from the nginx pod:

```bash
kubectl logs -f deployment/nginx
```

Generate some traffic to see new log entries:

```bash
kubectl port-forward deployment/nginx 8080:80 &
curl -s http://localhost:8080 > /dev/null
```

Logs should show:
```
127.0.0.1 - - [14/Jun/2026:16:30:00 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.88.1"
```

---

## 6. Execute Commands in Container

Run a command inside the nginx container:

```bash
kubectl exec -it deployment/nginx -- nginx -v
```

Expected:
```
nginx version: nginx/1.27.0
```

Open an interactive shell:

```bash
kubectl exec -it deployment/nginx -- sh
```

Inside the container, verify networking:
```bash
# From inside the container:
hostname
# Output: nginx-7bf8c87b5-abcde

# Check network interfaces
ip addr
# Should show eth0 with pod IP (10.200.x.x)

exit
```

---

## 7. Create a NodePort Service

Expose the nginx deployment via NodePort:

```bash
kubectl expose deployment nginx --port 80 --type NodePort
```

Expected:
```
service/nginx exposed
```

### Get Service Details

```bash
kubectl get svc nginx
```

Expected:
```
NAME    TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.32.0.xxx   <none>        80:3xxxx/TCP   15s
```

Note the NodePort (e.g., `30080` or another port in the 30000-32767 range).

### Test NodePort from Jumpbox

```bash
NODE_PORT=$(kubectl get svc nginx --output=jsonpath='{.spec.ports[0].nodePort}')
curl -I http://node-1:${NODE_PORT}
```

Expected:
```
HTTP/1.1 200 OK
Server: nginx/1.27.0
```

Also test from node-2:
```bash
curl -I http://node-2:${NODE_PORT}
```

Both nodes should respond because kube-proxy creates iptables rules on ALL nodes.

---

## 8. Scale the Deployment

Scale to 3 replicas:

```bash
kubectl scale deployment nginx --replicas=3
```

Verify:
```bash
kubectl get pods -l app=nginx -o wide
```

Expected:
```
NAME                    READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-7bf8c87b5-abcde  1/1     Running   0          5m    10.200.0.x   node-1
nginx-7bf8c87b5-fghij  1/1     Running   0          10s   10.200.1.x   node-2
nginx-7bf8c87b5-klmno  1/1     Running   0          10s   10.200.0.x   node-1
```

Pods are distributed across nodes by the scheduler.

Test load balancing via NodePort (refresh multiple times):
```bash
for i in {1..5}; do
  curl -s http://node-1:${NODE_PORT} | grep "Welcome to nginx"
done
```

---

## 9. Clean Up

Delete the test resources:

```bash
kubectl delete deployment nginx
kubectl delete service nginx
kubectl delete secret kubernetes-the-hard-way
```

Verify cleanup:
```bash
kubectl get all
```

Expected:
```
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.32.0.1    <none>        443/TCP   2h
```

Only the default `kubernetes` service should remain.

---

## 10. Validation Checklist

| Component | Test | Command | Expected |
|-----------|------|---------|----------|
| Secret Encryption | Create secret | `kubectl create secret generic ...` | Secret created |
| Secret Encryption | Verify encryption | `etcdctl get /registry/secrets/...` | `k8s:enc:aescbc:v1:key1` prefix |
| API Server | Deployment | `kubectl create deployment nginx ...` | Deployment created |
| Controller Manager | ReplicaSet | `kubectl get rs -l app=nginx` | RS with DESIRED=1 |
| Scheduler | Pod placement | `kubectl get pod -o wide` | Pod assigned to a node |
| Kubelet | Pod running | `kubectl get pod` | STATUS=Running |
| containerd | Container creation | Pod reaches Running | No ImagePullBackoff |
| CNI | IP allocation | `kubectl get pod -o wide` | IP from pod CIDR |
| Pod Network | Pod-to-pod | `ping` from one pod to another | 0% packet loss |
| kube-proxy | NodePort | `curl node-IP:NodePort` | HTTP 200 |
| kubectl | Port forward | `kubectl port-forward` + `curl localhost:8080` | nginx welcome page |
| kubectl | Logs | `kubectl logs` | nginx access logs |
| kubectl | Exec | `kubectl exec -- nginx -v` | nginx version string |
| kube-proxy | Load balancing | Multiple curls to NodePort | All backends respond |

---

## 11. Issues Encountered

### Issue 1: Worker Nodes Without Internet Access

**Symptom**: Pods stuck in `ImagePullBackOff`

**Root Cause**: Worker nodes have no public IP and can't reach Docker Hub or `registry.k8s.io`.

**Solution**:
```bash
# On jumpbox (which has internet):
sudo ctr image pull registry.k8s.io/pause:3.10
sudo ctr image export pause.tar registry.k8s.io/pause:3.10

# Transfer to workers:
scp pause.tar node-1:~/
scp pause.tar node-2:~/

# On each worker:
sudo ctr image import pause.tar
```

For application images (like nginx), either:
- Pre-pull and transfer the same way
- Set up a private registry accessible within the VPC
- Use containerd registry mirrors pointing to internal locations

### Issue 2: Missing Pause Image

**Symptom**: `FailedCreatePodSandbox: failed to pull registry.k8s.io/pause:3.10`

**Root Cause**: containerd requires the pause image to create pod sandboxes. Without it, no pods can start.

**Solution**: Same as Issue 1 — pre-load the pause image on all worker nodes.

### Issue 3: Controller Manager Can't Resolve API Server

**Symptom**: Deployment created, but no ReplicaSet or Pod appears.

**Root Cause**: Controller manager cannot resolve `server.kubernetes.local` to connect to the API server.

**Solution**:
```bash
ssh server "echo '<SERVER_PRIVATE_IP> server.kubernetes.local server' | sudo tee -a /etc/hosts"
ssh server "sudo systemctl restart kube-controller-manager kube-scheduler"
```

### Issue 4: Kubelet Can't Resolve API Server

**Symptom**: Nodes not registering or stuck `NotReady`.

**Root Cause**: Worker nodes can't resolve `server.kubernetes.local`.

**Solution**:
```bash
ssh node-1 "echo '<SERVER_PRIVATE_IP> server.kubernetes.local server' | sudo tee -a /etc/hosts"
ssh node-2 "echo '<SERVER_PRIVATE_IP> server.kubernetes.local server' | sudo tee -a /etc/hosts"
ssh node-1 "sudo systemctl restart kubelet"
ssh node-2 "sudo systemctl restart kubelet"
```

### Issue 5: Certificate Hostname Mismatch

**Symptom**: `curl` fails with `SSL: no alternative certificate subject name matches`

**Root Cause**: Hostname in URL doesn't match SANs in API server certificate.

**Solution**: Use the exact hostname in the certificate (`server.kubernetes.local`), not variations.

---

## 12. Summary

Congratulations! You have successfully built a Kubernetes cluster from scratch. At the end of Step 8:

| Check | Status |
|-------|--------|
| Secret Encryption | Working |
| API Server | Running |
| etcd | Running |
| Controller Manager | Running |
| Scheduler | Running |
| Kubelet (node-1) | Running, Ready |
| Kubelet (node-2) | Running, Ready |
| containerd | Running |
| CNI / Pod Networking | Working |
| kube-proxy | Running |
| kubectl Remote Access | Working |
| Pod-to-Pod Communication | Working |
| NodePort Services | Working |

### What You've Built

```
+----------------------------------------------------------+
|                   Kubernetes Cluster                      |
|----------------------------------------------------------|
|                                                           |
|  Control Plane (server)                                   |
|  - kube-apiserver    :6443                               |
|  - kube-controller-manager                                |
|  - kube-scheduler                                         |
|  - etcd              :2379, :2380                        |
|                                                           |
|  +---------------------+    +---------------------+       |
|  | Worker: node-1      |    | Worker: node-2      |       |
|  | - kubelet           |    | - kubelet           |       |
|  | - kube-proxy        |    | - kube-proxy        |       |
|  | - containerd        |    | - containerd        |       |
|  | - CNI plugins       |    | - CNI plugins       |       |
|  | Pods: 10.200.0.0/24 |    | Pods: 10.200.1.0/24 |       |
|  +---------------------+    +---------------------+       |
|                                                           |
|  Jumpbox (admin access)                                   |
|  - kubectl configured                                      |
|  - SSH access to all nodes                                 |
|                                                           |
+----------------------------------------------------------+
```

### Next Steps

- Deploy a real application
- Set up CoreDNS for cluster DNS resolution
- Install an ingress controller
- Configure monitoring (Prometheus/Grafana)
- Set up cluster autoscaling
- Harden security (PodSecurityPolicies, NetworkPolicies)

### Important Notes

- **Not for Production**: This cluster lacks HA, automated backups, monitoring, and many production hardening measures.
- **Clean Up**: Remember to terminate EC2 instances when done to avoid AWS charges.
- **Learning**: You've now seen every component that managed Kubernetes abstracts away. This understanding is invaluable for troubleshooting production issues.

---

**Thank you for following Kubernetes The Hard Way on AWS EC2!**
