---
title: Troubleshooting
description: Common issues and solutions for NITA
---

# :material-lifebuoy: Troubleshooting

This page covers common issues that may arise when using NITA, along with their solutions.

---

## Quick Diagnostics

Run these commands first to assess the state of your NITA installation:

```bash
# Check pod status
nita-cmd kube pods

# Check for pod errors
kubectl describe pods -n nita

# View pod logs
kubectl logs <pod-name> -n nita

# Check Kubernetes cluster
kubectl cluster-info
kubectl get nodes
```

---

## Common Issues

### Pods Not Starting

**Symptoms:** Pods are in `Pending`, `CrashLoopBackOff`, or `ImagePullBackOff` state.

**Diagnosis:**

```bash
kubectl describe pod <pod-name> -n nita
kubectl get events -n nita --sort-by='.lastTimestamp'
```

**Common Causes:**

| State | Cause | Solution |
|---|---|---|
| `ImagePullBackOff` | Cannot download container image | Check internet, DNS, and certificates |
| `Pending` | PVC not bound or insufficient resources | Check `kubectl get pvc -n nita` |
| `CrashLoopBackOff` | Container keeps crashing | Check logs with `kubectl logs` |
| `ContainerCreating` | ConfigMap or volume missing | Check ConfigMaps exist |

---

### Jenkins Pod Won't Start

**Symptoms:** Jenkins pod is in `CrashLoopBackOff`.

**Possible Causes:**

1. **Missing ConfigMaps** — Jenkins requires `jenkins-crt` and `jenkins-keystore`:

    ```bash
    kubectl get cm -n nita | grep jenkins
    ```

    If missing, recreate them (see [Certificates](certificates.md)).

2. **Corrupted PVC** — The `jenkins-home` PVC may be corrupted:

    ```bash
    kubectl delete pvc jenkins-home -n nita
    kubectl apply -f /opt/nita/k8s/jenkins-home-persistentvolumeclaim.yaml
    kubectl rollout restart deployment/jenkins -n nita
    ```

---

### NITA Webapp Returns HTTP Errors

**Symptoms:** 502 Bad Gateway or connection refused.

**Possible Causes:**

1. **Webapp pod not running:**

    ```bash
    kubectl get pods -n nita | grep webapp
    ```

2. **Database connection failed:**

    ```bash
    kubectl logs <webapp-pod> -n nita | grep -i error
    kubectl get pods -n nita | grep db
    ```

3. **Proxy misconfigured:**

    ```bash
    kubectl logs <proxy-pod> -n nita
    nita-cmd proxy restart
    ```

---

### Build Job Fails / Times Out

**Symptoms:** Build action triggered from Webapp fails with timeout.

```
error: timed out waiting for the condition on jobs/dump
Build step 'Execute shell' marked build as failure
```

**Most Common Cause:** Windows CRLF line endings in project files.

**Solution:**

```bash
# Configure git to not convert line endings
git config --global core.autocrlf false

# Re-clone the NITA repository
rm -rf /opt/nita
git clone https://github.com/Juniper/nita.git /opt/nita
```

**Alternative:** Install Git for Windows from [git-scm.com](https://git-scm.com/install/windows) and choose **"Checkout as-is, commit as-is"** during installation.

---

### KUBECONFIG Not Set

**Symptoms:** `kubectl` commands fail with connection errors.

**Solution:**

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
# Or for non-root users:
export KUBECONFIG=$HOME/.kube/config
```

Add to `~/.bashrc` for persistence:

```bash
echo "export KUBECONFIG=$HOME/.kube/config" >> ~/.bashrc
source ~/.bashrc
```

---

### Certificate Errors During Installation

**Symptoms:** `kubeadm init` fails with certificate verification error.

```
[ERROR ImagePull]: failed to pull image...
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

**Cause:** Zero-trust security solution (e.g., Zscaler) intercepting traffic.

**Solution:** See the [Certificate Management — Zscaler](certificates.md#zscaler--zero-trust-environments) section.

---

### Kubernetes Certificates Expired

**Symptoms:** `kubectl` commands fail, API server unreachable.

**Solution:**

```bash
# Check expiration
sudo kubeadm certs check-expiration

# Renew all
sudo kubeadm certs renew all
sudo systemctl restart kubelet
```

---

### Ansible/Robot Container Already Running

**Symptoms:** Build or test fails because an ephemeral container is already running.

**Solution:** Wait for the existing job to complete, or delete it manually:

```bash
kubectl get jobs -n nita
kubectl delete job <job-name> -n nita
```

---

### Cannot Access NITA Webapp

**Symptoms:** Browser shows connection refused on `https://<host>:443`.

**Checklist:**

1. ✅ Are all pods running? (`nita-cmd kube pods`)
2. ✅ Is the proxy pod running?
3. ✅ Are the proxy ConfigMaps created?
4. ✅ Are the certificates valid?
5. ✅ Is the firewall allowing port 443?

**AlmaLinux firewall:**

```bash
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
```

**Ubuntu firewall:**

```bash
sudo ufw allow 443/tcp
```

---

## Resetting NITA

### Restart All Pods

```bash
kubectl rollout restart deployment --all -n nita
```

### Full Reset

```bash
# Delete all NITA resources
kubectl delete namespace nita

# Reset Kubernetes
sudo kubeadm reset
sudo systemctl restart containerd.service

# Re-initialize
sudo kubeadm init --control-plane-endpoint="localhost" --ignore-preflight-errors=NumCPU
export KUBECONFIG=/etc/kubernetes/admin.conf

# Remove taints and deploy Calico
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

# Re-deploy NITA
cd /opt/nita/k8s && bash apply-k8s.sh

# Re-create ConfigMaps (certificates, proxy config, Jenkins keystore)
```

---

## Getting Help

- :material-github: [Open an Issue on GitHub](https://github.com/Juniper/nita/issues)
- :material-file-document: Review the [NITA README](https://github.com/Juniper/nita)
- :material-youtube: Watch the [NITA Introduction Video](https://www.youtube.com/watch?v=6edtVe8Ueis)
