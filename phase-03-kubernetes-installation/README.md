# Phase 3: Kubernetes Installation

## Overview

This phase installs a production-grade Kubernetes cluster using kubeadm, following CKA exam standards.

**Cluster Architecture:**
- 1x Control Plane node (k8s-cp-01)
- 2x Worker nodes (k8s-worker-01, k8s-worker-02)
- Containerd runtime
- Calico CNI plugin

**Installation Method:** Ansible playbook (automated but educational)

---

## CKA Exam Domains Covered

### ✅ Cluster Architecture, Installation & Configuration (25%)
- Install and configure kubeadm
- Understand container runtime requirements (containerd)
- Bootstrap control plane with kubeadm init
- Join worker nodes to cluster
- Install and configure CNI plugin (Calico)

### ✅ Troubleshooting (30%)
- Debug cluster component logs
- Understand static pod manifests
- Troubleshoot node join issues
- Verify cluster health

---

## Prerequisites

- ✅ Phase 1 completed (Proxmox foundation)
- ✅ Phase 2 completed (VMs created and configured)
- ✅ SSH access to all nodes configured
- ✅ Ansible installed on your local machine

**Verify prerequisites:**
```bash
# Test SSH connectivity
ansible all -i inventory.ini -m ping

# Check VM resources
ansible all -i inventory.ini -m shell -a "free -h && df -h"
```

---

## Configuration

All cluster settings are in `group_vars/all.yml`:

```yaml
Kubernetes version: 1.29.x
Pod network CIDR: 10.244.0.0/16
Service CIDR: 10.96.0.0/12
CNI plugin: Calico
```

**Why these choices?**
- **kubeadm**: CKA exam standard tool
- **containerd**: Industry standard runtime (Docker deprecated)
- **Calico**: Production-grade CNI with NetworkPolicy support
- **1.29.x**: Current stable version, matches CKA exam

---

## Usage

### Step 1: Review Configuration
```bash
cd phase-03-kubernetes-installation
cat group_vars/all.yml
```

### Step 2: Run Playbook
```bash
# Full installation (control plane + workers)
ansible-playbook -i inventory.ini install-kubernetes.yml

# Control plane only (for testing)
ansible-playbook -i inventory.ini install-kubernetes.yml --tags control-plane

# Workers only (if control plane already exists)
ansible-playbook -i inventory.ini install-kubernetes.yml --tags workers
```

### Step 3: Verify Installation
```bash
# Copy kubeconfig to local machine
scp mahmood@192.168.178.34:/home/mahmood/.kube/config ~/.kube/config

# Verify cluster
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

---

## What Happens During Installation

### Phase 1: Pre-flight Checks (All Nodes)
- Disable swap (required by kubelet)
- Load kernel modules (overlay, br_netfilter)
- Configure sysctl parameters (IP forwarding, bridge filtering)
- Install containerd runtime

### Phase 2: Install Kubernetes Components (All Nodes)
- Add Kubernetes apt repository
- Install kubeadm, kubelet, kubectl
- Hold packages (prevent accidental upgrades)

### Phase 3: Bootstrap Control Plane (k8s-cp-01)
```bash
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --apiserver-advertise-address=192.168.178.34
```

**What kubeadm init does:**
1. Generates CA certificates → `/etc/kubernetes/pki/`
2. Creates static pod manifests → `/etc/kubernetes/manifests/`
3. Generates kubeconfig files → `/etc/kubernetes/*.conf`
4. Creates bootstrap token (24h validity)
5. Installs CoreDNS and kube-proxy

### Phase 4: Install Calico CNI (k8s-cp-01)
- Downloads Calico manifests
- Applies to cluster via kubectl
- Pods can now get IP addresses

### Phase 5: Join Worker Nodes (k8s-worker-01, k8s-worker-02)
```bash
kubeadm join 192.168.178.34:6443 \
  --token <bootstrap-token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

**What kubeadm join does:**
1. Connects to API server using bootstrap token
2. Downloads cluster CA certificate
3. Generates kubelet client certificate
4. Registers node with cluster

---

## Post-Installation Verification

### Check Node Status
```bash
kubectl get nodes -o wide
```

**Expected output:**
```
NAME            STATUS   ROLES           AGE   VERSION
k8s-cp-01       Ready    control-plane   5m    v1.29.x
k8s-worker-01   Ready    <none>          3m    v1.29.x
k8s-worker-02   Ready    <none>          3m    v1.29.x
```

### Check System Pods
```bash
kubectl get pods -n kube-system
```

**Expected pods:**
- calico-node (DaemonSet - 1 per node)
- calico-kube-controllers (Deployment)
- coredns (Deployment - 2 replicas)
- kube-proxy (DaemonSet - 1 per node)
- Control plane pods (on k8s-cp-01 only):
  - etcd
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler

### Check Cluster Health
```bash
# Component status
kubectl get componentstatuses

# Cluster info
kubectl cluster-info

# Check API server directly
curl -k https://192.168.178.34:6443/healthz
```

---

## Important Directories (CKA Exam Knowledge)

### Control Plane Node (k8s-cp-01)
```
/etc/kubernetes/
├── admin.conf              # Admin kubeconfig (full cluster access)
├── kubelet.conf            # Kubelet kubeconfig
├── controller-manager.conf # Controller manager kubeconfig
├── scheduler.conf          # Scheduler kubeconfig
├── manifests/              # Static pod manifests
│   ├── etcd.yaml
│   ├── kube-apiserver.yaml
│   ├── kube-controller-manager.yaml
│   └── kube-scheduler.yaml
└── pki/                    # Cluster certificates
    ├── ca.crt
    ├── ca.key
    ├── apiserver.crt
    └── ...

/var/lib/kubelet/           # Kubelet data directory
/var/lib/etcd/              # etcd data directory
```

### Worker Nodes
```
/etc/kubernetes/
├── kubelet.conf            # Kubelet kubeconfig only
└── pki/
    └── ca.crt              # Cluster CA (no private key)

/var/lib/kubelet/           # Kubelet data directory
```

---

## Enterprise vs Homelab Comparison

| Aspect               | Homelab (This Project)      | Enterprise (Real Job)                        |
| -------------------- | --------------------------- | -------------------------------------------- |
| **Installation**     | Manual kubeadm              | Managed (EKS/GKE/AKS) or Terraform + kubeadm |
| **Control Plane**    | Self-hosted on k8s-cp-01    | Cloud provider manages (invisible to you)    |
| **Node Count**       | 3 nodes (1 CP + 2 workers)  | 50-500+ nodes across AZs                     |
| **HA Control Plane** | Single CP (not HA)          | 3+ CP nodes with load balancer               |
| **Storage**          | Will add Longhorn (Phase 5) | Cloud provider storage (EBS, PD, etc.)       |
| **CNI**              | Calico                      | Calico, Cilium, or cloud CNI                 |
| **Upgrades**         | Manual kubeadm upgrade      | Automated with zero downtime                 |
| **Backup**           | Will add Velero (later)     | Automated etcd backups                       |

**Why this matters:**
- You're learning the **foundation** that managed K8s builds on
- CKA tests **kubeadm** knowledge (not managed K8s)
- Troubleshooting skills transfer to any K8s platform

---

## Common CKA Exam Tasks

Practice these after installation:

1. **Generate new bootstrap token**
   ```bash
   kubeadm token create --print-join-command
   ```

2. **Check control plane component logs**
   ```bash
   # Static pod logs
   kubectl logs -n kube-system kube-apiserver-k8s-cp-01
   
   # Via systemd (kubelet)
   journalctl -u kubelet -f
   ```

3. **Backup etcd**
   ```bash
   ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key
   ```

4. **Drain and cordon nodes**
   ```bash
   kubectl drain k8s-worker-01 --ignore-daemonsets
   kubectl cordon k8s-worker-02
   ```

---

## Lessons Learned (To Be Updated)

*Update this section after completing Phase 3 installation*

**What worked well:**
- TBD

**Challenges encountered:**
- TBD

**Key takeaways:**
- TBD

**Skills gained:**
- TBD

---

## Next Steps

After Phase 3 completion:
- **Phase 4**: Observability stack (Prometheus, Grafana, Loki)
- **Phase 5**: Persistent storage (Longhorn)
- **Phase 6**: GitOps (ArgoCD)

---

## References

- [Official kubeadm documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Calico installation guide](https://docs.tigera.io/calico/latest/getting-started/kubernetes/)
- [CKA Exam Curriculum](https://github.com/cncf/curriculum)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) (manual installation for deep learning)

---

**Status:** Ready for execution  
**Last Updated:** 2024-12-31  
**Estimated Duration:** 30-45 minutes
