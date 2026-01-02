# 🚀 OpenShift Complete Handbook — From Zero to Hero on CRC

> **Author**: Sumon Pal (DevOps & Cloud Engineer)  
> **Platform**: Ubuntu 24.04.1 + CRC v2.57.0 (OpenShift 4.20.5)  
> **Last Updated**: January 1, 2026  
> **GitHub**: [https://github.com/SumonPaul18/openshift](https://github.com/SumonPaul18/openshift)

---

## 📌 Table of Contents

- [1. Introduction to OpenShift](#1-introduction-to-openshift)
  - [1.1 What is OpenShift?](#11-what-is-openshift)
  - [1.2 Why Use OpenShift?](#12-why-use-openshift)
  - [1.3 When to Use OpenShift vs Plain Kubernetes](#13-when-to-use-openshift-vs-plain-kubernetes)
  - [1.4 OpenShift vs AWS EKS / GCP GKE / Azure AKS](#14-openshift-vs-aws-eks--gcp-gke--azure-aks)

- [2. OpenShift Architecture](#2-openshift-architecture)
  - [2.1 Core Components](#21-core-components)
  - [2.2 OpenShift-Exclusive Features](#22-openshift-exclusive-features)
  - [2.3 CRC vs Full OpenShift](#23-crc-vs-full-openshift)

- [3. System Requirements & Prerequisites](#3-system-requirements--prerequisites)

- [4. Step-by-Step CRC Installation on Ubuntu 24.04](#4-step-by-step-crc-installation-on-ubuntu-2404)
  - [4.1 Get Red Hat Pull Secret](#41-get-red-hat-pull-secret)
  - [4.2 Install Dependencies](#42-install-dependencies)
  - [4.3 Create Non-Root User](#43-create-non-root-user)
  - [4.4 Download & Install CRC](#44-download--install-crc)
  - [4.5 Configure RAM & CPU](#45-configure-ram--cpu)
  - [4.6 First-Time Setup & Start](#46-first-time-setup--start)
  - [4.7 Access Web Console (Local & Remote)](#47-access-web-console-local--remote)
  - [4.8 Set Up `oc` CLI](#48-set-up-oc-cli)
  - [4.9 Recover After Reboot/Crash](#49-recover-after-rebootcrash)

- [5. OpenShift as Kubernetes — Full Compatibility Guide](#5-openshift-as-kubernetes----full-compatibility-guide)
  - [5.1 Namespace / Project](#51-namespace--project)
  - [5.2 Pod](#52-pod)
  - [5.3 Deployment](#53-deployment)
  - [5.4 Service & Route (Ingress Replacement)](#54-service--route-ingress-replacement)
  - [5.5 PersistentVolume (PV) & PersistentVolumeClaim (PVC)](#55-persistentvolume-pv--persistentvolumeclaim-pvc)
  - [5.6 RBAC: Role, RoleBinding, ServiceAccount](#56-rbac-role-rolebinding-serviceaccount)
  - [5.7 Node Selector, Affinity & Anti-Affinity](#57-node-selector-affinity--anti-affinity)
  - [5.8 NetworkPolicy (Egress/Ingress Control)](#58-networkpolicy-egressingress-control)

- [6. OpenShift-Exclusive Features](#6-openshift-exclusive-features)
  - [6.1 Route](#61-route)
  - [6.2 ImageStream](#62-imagestream)
  - [6.3 BuildConfig (S2I)]#63-buildconfig-s2i
  - [6.4 Operators & OperatorHub](#64-operators--operatorhub)
  - [6.5 Security Context Constraints (SCC)](#65-security-context-constraints-scc)
  - [6.6 Built-in Monitoring (Prometheus + Grafana)](#66-built-in-monitoring-prometheus--grafana)

- [7. Practical Projects (YAML + CLI)](#7-practical-projects-yaml--cli)
  - [7.1 Nginx Deployment + Route](#71-nginx-deployment--route)
  - [7.2 ConfigMap + Secret Usage](#72-configmap--secret-usage)
  - [7.3 PVC with Nginx](#73-pvc-with-nginx)
  - [7.4 RBAC Demo](#74-rbac-demo)
  - [7.5 NetworkPolicy Demo](#75-networkpolicy-demo)

- [8. Troubleshooting Common Issues](#8-troubleshooting-common-issues)

- [9. Best Practices for Home Lab](#9-best-practices-for-home-lab)

- [10. Learning Roadmap](#10-learning-roadmap)

- [References](#references)

---

## 1. Introduction to OpenShift

### 1.1 What is OpenShift?

**OpenShift** is Red Hat’s enterprise-grade Kubernetes platform that adds developer and operations-centric tooling on top of Kubernetes.

> ✅ **OpenShift = Kubernetes + Enterprise Features**

It includes:
- Web Console
- Built-in CI/CD (S2I, Tekton)
- GitOps (ArgoCD integration)
- Operators for stateful apps
- Advanced security (SCC, OAuth)
- Monitoring & Logging

### 1.2 Why Use OpenShift?

- **For Enterprises**: Compliance, security, audit, support
- **For Developers**: One-command deploy, built-in pipelines
- **For DevOps**: Unified platform for dev, test, prod

> 🌍 **Real-World Users**: bKash, DBBL, Robi, Bangladesh Bank, NASA, BMW

### 1.3 When to Use OpenShift vs Plain Kubernetes

| Use Case | Kubernetes | OpenShift |
|--------|-----------|----------|
| Learning basics | ✅ | ✅ |
| Production enterprise app | ❌ | ✅ |
| Need built-in CI/CD | ❌ | ✅ |
| Government/banking compliance | ❌ | ✅ |
| Cost-sensitive project | ✅ | ❌ (Red Hat subscription) |

### 1.4 OpenShift vs AWS EKS / GCP GKE / Azure AKS

| Feature | OpenShift | EKS/GKE/AKS |
|--------|----------|-------------|
| Self-managed | ✅ (On-prem/cloud) | ❌ (Managed) |
| Built-in Web Console | ✅ | ❌ |
| Operators | ✅ | ❌ |
| S2I Builds | ✅ | ❌ |
| Integrated Monitoring | ✅ | ❌ (Need add-ons) |
| Enterprise Support | ✅ (Red Hat) | ✅ (Cloud provider) |

> 💡 **CRC (CodeReady Containers)** = OpenShift for your laptop

---

## 2. OpenShift Architecture

### 2.1 Core Components (Same as Kubernetes)

- **Control Plane**: API Server, etcd, Scheduler, Controller Manager
- **Worker Node**: kubelet, CRI-O (container runtime), OVN-Kubernetes (CNI)
- **Networking**: OVN-Kubernetes (SDN)
- **Storage**: Dynamic provisioning via CSI

### 2.2 OpenShift-Exclusive Features

| Feature | Purpose | API Group |
|--------|--------|----------|
| Route | External HTTPS URL | `route.openshift.io/v1` |
| ImageStream | Internal image registry ref | `image.openshift.io/v1` |
| BuildConfig | Source-to-Image build | `build.openshift.io/v1` |
| SCC | Pod security policy | `security.openshift.io/v1` |
| OperatorHub | 1-click app install | — |

### 2.3 CRC vs Full OpenShift

| Feature | CRC | Full OpenShift |
|--------|-----|----------------|
| Nodes | 1 (combined master+worker) | 3+ masters, N workers |
| HA | ❌ | ✅ |
| Production Ready | ❌ | ✅ |
| Use Case | Learning, Dev | Enterprise Prod |

---

## 3. System Requirements & Prerequisites

| Component | Requirement |
|---------|-------------|
| OS | Ubuntu 24.04.1 (Desktop/Server) |
| CPU | 4 cores (VT-x/AMD-V enabled) |
| RAM | 16 GB (CRC v2.57+ needs **10.5 GB min**) |
| Disk | 250 GB SSD |
| Network | 1 NIC (e.g., `192.168.0.30`) |
| Internet | Required for first setup |

> ⚠️ **Uninstall Podman**: `sudo apt remove -y podman`  
> (Otherwise CRC runs in podman mode → no remote access)

---

## 4. Step-by-Step CRC Installation on Ubuntu 24.04

### 4.1 Get Red Hat Pull Secret

1. Go to [https://developers.redhat.com](https://developers.redhat.com)
2. Sign up for free account
3. Download pull secret from [https://cloud.redhat.com/openshift/create/local](https://cloud.redhat.com/openshift/create/local)
4. Save as `~/openshift-secret.txt`

### 4.2 Install Dependencies

```bash
sudo apt update
sudo apt install -y curl wget tar jq libvirt-daemon-system libvirt-clients qemu-kvm
sudo usermod -aG libvirt,kvm $(whoami)
```

> 🔁 Reboot or log out/in

### 4.3 Create Non-Root User

```bash
sudo adduser osu
sudo usermod -aG sudo,libvirt,kvm osu
su - osu
```

> ❌ Never run CRC as root!

### 4.4 Download & Install CRC

```bash
cd ~
wget https://mirror.openshift.com/pub/openshift-v4/clients/crc/2.57.0/crc-linux-amd64.tar.xz
tar -xf crc-linux-amd64.tar.xz
cd crc-linux-2.57.0-amd64
sudo cp crc /usr/local/bin/
crc version
```

### 4.5 Configure RAM & CPU

```bash
crc config set memory 10752  # 10.5 GB
crc config set cpus 4
```

> 💡 Enable SWAP on 16GB RAM:
> ```bash
> sudo fallocate -l 4G /swapfile
> sudo chmod 600 /swapfile
> sudo mkswap /swapfile
> sudo swapon /swapfile
> echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
> ```

### 4.6 First-Time Setup & Start

```bash
crc setup
crc start
# Paste full content of ~/openshift-secret.txt when prompted
```

> ✅ Success output includes:
> ```
> Web console URL: https://console-openshift-console.apps-crc.testing
> Username: kubeadmin
> Password: xxxx-xxxx-xxxx-xxxx
> ```

### 4.7 Access Web Console (Local & Remote)

#### On CRC Host:
- Open [https://console-openshift-console.apps-crc.testing](https://console-openshift-console.apps-crc.testing)
- Accept SSL warning

#### From Remote Machine:
1. SSH Tunnel:
   ```bash
   ssh -L 8443:console-openshift-console.apps-crc.testing:443 osu@192.168.0.30
   ```
2. Edit `hosts` file:
   ```text
   127.0.0.1 console-openshift-console.apps-crc.testing
   ```
3. Visit same URL in browser

### 4.8 Set Up `oc` CLI

```bash
chmod +x ~/.crc/bin/oc
echo 'export PATH="$HOME/.crc/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
eval $(crc oc-env)
crc console --credentials  # get password
oc login -u kubeadmin -p 'PASSWORD' https://api.crc.testing:6443
```

### 4.9 Recover After Reboot/Crash

```bash
su - osu
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
nohup ~/.crc/bin/crc daemon > /tmp/crc-daemon.log 2>&1 &
crc start
```

> ✅ All data is restored automatically

---

## 5. OpenShift as Kubernetes — Full Compatibility Guide

> ✅ **All Kubernetes YAML works in OpenShift** — except Ingress

### 5.1 Namespace / Project

```yaml
apiVersion: v1
kind: Namespace
meta
  name: dev-team
```

```bash
oc apply -f ns.yaml
# or
oc new-project dev-team
```

### 5.2 Pod

```yaml
apiVersion: v1
kind: Pod
meta
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx
```

### 5.3 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
meta
  name: web
spec:
  replicas: 2
  selector:
    matchLabels: {app: web}
  template:
    meta
      labels: {app: web}
    spec:
      containers:
      - name: nginx
        image: nginx
```

### 5.4 Service & Route (Ingress Replacement)

```yaml
# Service
apiVersion: v1
kind: Service
meta
  name: web-svc
spec:
  selector: {app: web}
  ports: [{port: 80, targetPort: 80}]
---
# Route (OpenShift Ingress)
apiVersion: route.openshift.io/v1
kind: Route
meta
  name: web-route
spec:
  to: {kind: Service, name: web-svc}
```

### 5.5 PersistentVolume (PV) & PersistentVolumeClaim (PVC)

```yaml
# PVC (PV auto-created in CRC)
apiVersion: v1
kind: PersistentVolumeClaim
meta
  name: web-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources: {requests: {storage: "1Gi"}}
```

Use in pod:
```yaml
volumeMounts:
- name: data
  mountPath: /usr/share/nginx/html
volumes:
- name: data
  persistentVolumeClaim: {claimName: web-pvc}
```

### 5.6 RBAC: Role, RoleBinding, ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
meta
  name: app-reader
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
meta
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
meta
  name: read-pods
subjects:
- kind: ServiceAccount
  name: app-reader
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 5.7 Node Selector, Affinity & Anti-Affinity

```yaml
# In pod/deployment spec.template.spec:
nodeSelector:
  kubernetes.io/hostname: crc
```

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector: {matchExpressions: [{key: app, operator: In, values: ["web"]}]}
      topologyKey: kubernetes.io/hostname
```

### 5.8 NetworkPolicy (Egress/Ingress Control)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
meta
  name: allow-web-egress
spec:
  podSelector: {matchLabels: {app: web}}
  policyTypes: ["Egress"]
  egress:
  - to:
    - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: openshift-monitoring}}
    ports: [{protocol: TCP, port: 443}]
```

---

## 6. OpenShift-Exclusive Features

### 6.1 Route
- Replaces Ingress
- Auto TLS
- HAProxy-based

### 6.2 ImageStream
- Internal image registry reference
- `oc tag nginx:latest my-app:prod`

### 6.3 BuildConfig (S2I)
```bash
oc new-app https://github.com/sclorg/nodejs-ex
```
> Builds from source → creates image → deploys

### 6.4 Operators & OperatorHub
- Install Redis, PostgreSQL, Kafka in 1 click
- Web Console → Operators → OperatorHub

### 6.5 Security Context Constraints (SCC)
- Replaces Kubernetes PSP
- Enforces pod security (no root, etc.)

### 6.6 Built-in Monitoring (Prometheus + Grafana)
- Web Console → Administrator → Observe
- No extra setup needed

---

## 7. Practical Projects (YAML + CLI)

### 7.1 Nginx Deployment + Route

```bash
oc new-project web-demo
oc new-app nginx:latest
oc expose svc/nginx
oc get route
```

### 7.2 ConfigMap + Secret Usage

```bash
oc create configmap app-config --from-literal=ENV=prod
oc create secret generic db-pass --from-literal=password=12345
```

In deployment:
```yaml
envFrom:
- configMapRef: {name: app-config}
- secretRef: {name: db-pass}
```

### 7.3 PVC with Nginx

```bash
oc apply -f pvc.yaml
# In deployment, mount PVC to /usr/share/nginx/html
```

### 7.4 RBAC Demo

```bash
oc apply -f rbac.yaml
oc auth can-i get pods --as=system:serviceaccount:dev-team:app-reader
```

### 7.5 NetworkPolicy Demo

```bash
oc apply -f networkpolicy.yaml
# Test egress access
```

---

## 8. Troubleshooting Common Issues

| Issue | Solution |
|------|--------|
| `crc should not be run as root` | Use non-root user |
| `virtiofsd not found` | `sudo apt install -y qemu-system-x86 virtiofsd` |
| `crc daemon not running` | Run `nohup ~/.crc/bin/crc daemon &` |
| `oc: command not found` | `chmod +x ~/.crc/bin/oc` + add to PATH |
| `Application not available` | Check `hosts` file + SSH tunnel |

---

## 9. Best Practices for Home Lab

- ✅ Always use non-root user
- ✅ Run `crc stop` before shutdown
- ✅ Use HTPasswd users instead of `kubeadmin`
- ✅ Keep `~/.crc/cache/` to avoid re-download
- ✅ Use SSH login (not `su -`)
- ✅ Enable SWAP on 16GB RAM
- ✅ Practice GitOps with ArgoCD

---

## 10. Learning Roadmap

| Day | Topic |
|-----|------|
| 1 | Pods, Deployments, Services |
| 2 | Routes, ConfigMaps, Secrets |
| 3 | PV/PVC, RBAC |
| 4 | Affinity, NetworkPolicy |
| 5 | Operators, Redis/PostgreSQL |
| 6 | Monitoring, Logging |
| 7 | CI/CD with Tekton |
| 8 | GitOps with ArgoCD |

> 🎯 **Goal**: Deploy a full app (Frontend + Backend + DB + CI/CD) in 8 days!

---

## References

- [Red Hat CRC Official Docs](https://access.redhat.com/documentation/en-us/red_hat_codeready_containers)
- [OpenShift 4.20 Release Notes](https://docs.openshift.com/container-platform/4.20/release_notes/ocp-4-20-release-notes.html)
- [GitHub Repository](https://github.com/SumonPaul18/openshift)
- [Red Hat Developer Portal](https://developers.redhat.com)

---

> ✨ **You now have a complete OpenShift home lab — ready for learning, teaching, and career growth!**  
> Save this guide. Share it. And happy DevOps-ing! 🚀