# OpenShift – The Ultimate Practical Guide for DevOps Engineers

> **Author**: Sumon  
> **Last Updated**: January 2026  
> **Intended Audience**: DevOps Engineers, SREs, Platform Engineers, and Technologists with Linux, Networking & Cloud Experience  
> **Language**: Simple, Clear English — Built for Real-World Use  

---

## Table of Contents

1. [What is OpenShift?](#1-what-is-openshift)
2. [Why Use OpenShift?](#2-why-use-openshift)
3. [Key Features of OpenShift](#3-key-features-of-openshift)
4. [Pros and Cons of OpenShift](#4-pros-and-cons-of-openshift)
5. [OpenShift vs Other Platforms (Kubernetes, Rancher, etc.)](#5-openshift-vs-other-platforms)
6. [OpenShift Architecture Overview](#6-openshift-architecture-overview)
7. [Core Components & Services](#7-core-components--services)
8. [Installation Methods – From Laptop to Enterprise](#8-installation-methods--from-laptop-to-enterprise)
9. [Real-World Enterprise Usage – With Official Examples](#9-real-world-enterprise-usage--with-official-examples)
10. [Getting Started: Your First OpenShift Project](#10-getting-started-your-first-openshift-project)
11. [Hands-On Labs: Deployments, Services, Ingress, RBAC & More](#11-hands-on-labs-deployments-services-ingress-rbac--more)
12. [Advanced Scenarios: Affinity, Node Selectors, Autoscaling](#12-advanced-scenarios-affinity-node-selectors-autoscaling)
13. [Operational Best Practices](#13-operational-best-practices)
14. [Troubleshooting Common Issues](#14-troubleshooting-common-issues)
15. [Further Learning & Resources](#15-further-learning--resources)

---

## 1. What is OpenShift?

Red Hat OpenShift is a **container application platform** built on top of **Kubernetes**. It adds enterprise-grade features like built-in CI/CD, security policies, developer tooling, and multi-tenant management—making it ideal for organizations that want to run Kubernetes **at scale, securely, and with operational simplicity**.

> Think of it as **Kubernetes++**: All the power of Kubernetes, plus Red Hat’s hardening, support, and developer experience.

Key facts:
- Developed and maintained by **Red Hat** (now part of IBM).
- Certified Kubernetes distribution (CNCF compliant).
- Available in multiple flavors: **OpenShift Container Platform (OCP)**, **OpenShift Dedicated (OSD)**, **Azure Red Hat OpenShift (ARO)**, and **CodeReady Containers (CRC)** for local development.

---

## 2. Why Use OpenShift?

### Pain Points OpenShift Solves:
| Problem | OpenShift Solution |
|--------|--------------------|
| Raw Kubernetes is complex to secure & operate | Built-in **SCC (Security Context Constraints)**, automatic CVE patching, integrated auth (OAuth, LDAP, SAML) |
| Developers struggle with YAML & CLI | **Developer Console (GUI)**, **Source-to-Image (S2I)**, **Tekton Pipelines** |
| No native CI/CD | **OpenShift Pipelines**, **GitOps via Argo CD**, **ImageStreams** |
| Multi-team isolation is hard | **Projects = Namespaces + RBAC + Quotas** |
| Day-2 operations are manual | **Cluster Operators**, **OperatorHub**, **Monitoring Stack (Prometheus, Grafana)** |

OpenShift is designed for **teams that need production-ready Kubernetes without building everything from scratch**.

---

## 3. Key Features of OpenShift

| Category | Feature | Description |
|--------|--------|-----------|
| **Core** | Kubernetes + Enhancements | Full K8s API + Red Hat additions |
| **Security** | SCC, Network Policies, Image Scanning | Enforces pod security, scans container images |
| **Developer UX** | Web Console, Dev Spaces, CLI (`oc`) | GUI + powerful CLI with shortcuts |
| **CI/CD** | Tekton Pipelines, BuildConfigs | Git-triggered builds & deployments |
| **Automation** | Operators | Manage apps/services declaratively (e.g., databases, Kafka) |
| **Observability** | Logging, Monitoring, Tracing | Integrated stack out-of-the-box |
| **Networking** | Ingress (Routes), Service Mesh | Expose apps securely with TLS |
| **Storage** | CSI Drivers, Dynamic Provisioning | Supports AWS EBS, Azure Disk, Ceph, etc. |

---

## 4. Pros and Pros of OpenShift

### ✅ Pros
- **Enterprise-Grade Security**: SCC, FIPS compliance, SELinux integration.
- **Developer-Friendly**: No need to learn raw K8s to deploy apps.
- **Fully Managed Options**: ARO, OSD reduce operational burden.
- **Red Hat Support**: 24/7 SLA-backed enterprise support.
- **Integrated Toolchain**: No need to bolt on Prometheus, Grafana, etc.

### ❌ Cons
- **Cost**: Licensing can be expensive (though CRC is free).
- **Complexity**: Learning curve for pure K8s users (extra layers).
- **Vendor Lock-in Risk**: Some features (e.g., Routes, ImageStreams) are OpenShift-specific.
- **Resource Heavy**: Minimum 4 vCPUs + 16GB RAM for CRC.

> **Verdict**: If you need **production Kubernetes with security, compliance, and ease-of-use**, OpenShift is worth it. For learning K8s basics, use `minikube` or `kind`.

---

## 5. OpenShift vs Other Platforms

| Platform | Based On | Best For | Key Difference |
|--------|--------|--------|---------------|
| **OpenShift** | Kubernetes | Enterprises, regulated industries | Built-in security, CI/CD, support |
| **Vanilla Kubernetes** | — | DIY teams, cloud-native purists | Maximum flexibility, no opinionated layers |
| **Rancher** | Kubernetes | Multi-cluster management | Lightweight, UI-focused, less opinionated |
| **EKS/AKS/GKE** | Managed K8s | Cloud-native apps on AWS/Azure/GCP | Simpler setup, but you add security/tooling |
| **K3s** | Lightweight K8s | Edge, IoT, labs | Minimal footprint, not for enterprise |

> OpenShift = **"Batteries included, enterprise grade, Red Hat backed."**

---

## 6. OpenShift Architecture Overview

OpenShift follows a **control plane + worker node** model (like Kubernetes), but with Red Hat enhancements:

```
+--------------------------------------------------+
|                Control Plane (Master)            |
|--------------------------------------------------|
| - API Server (with OAuth)                        |
| - etcd                                           |
| - Controller Manager                             |
| - Scheduler                                      |
| - OpenShift-Specific:                            |
|   • OAuth Server                                 |
|   • Image Registry Operator                      |
|   • Cluster Version Operator (CVO)               |
|   • Machine Config Operator (MCO)                |
+--------------------------------------------------+

+--------------------------------------------------+
|                Worker Nodes                      |
|--------------------------------------------------|
| - kubelet, container runtime (CRI-O)             |
| - OpenShift SDN (OVN-Kubernetes or OpenShift-SDN)|
| - Monitoring agents (Node Exporter, etc.)        |
+--------------------------------------------------+

+--------------------------------------------------+
|                External Services                 |
|--------------------------------------------------|
| - Identity Provider (LDAP, GitHub, etc.)         |
| - Storage (NFS, Ceph, Cloud Volumes)             |
| - Load Balancer (for Ingress)                    |
+--------------------------------------------------+
```

> **Note**: OpenShift uses **CRI-O** (not Docker) as its default container runtime.

---

## 7. Core Components & Services

| Component | Purpose | OpenShift-Specific? |
|---------|--------|-------------------|
| **Projects** | Namespaces with extra RBAC & quotas | ✅ Yes |
| **Routes** | Ingress with TLS termination | ✅ Yes |
| **ImageStreams** | Track container image versions | ✅ Yes |
| **BuildConfigs** | Define build strategies (Docker, S2I) | ✅ Yes |
| **DeploymentConfigs** | Legacy deployment controller (now use `Deployment`) | ⚠️ Deprecated |
| **Operators** | Automate app lifecycle (e.g., PostgreSQL Operator) | ✅ Yes |
| **Security Context Constraints (SCC)** | Pod security policies | ✅ Yes |
| **Machine Config Operator** | OS-level node management | ✅ Yes |

---

## 8. Installation Methods – From Laptop to Enterprise

### 🧪 For Learning & Development
1. **CodeReady Containers (CRC)**
   - Single-node OpenShift on your laptop.
   - Uses **Podman** and **libvirt** (Linux) or **Hyper-V** (Windows).
   - Free for developers.
   - **Requirements**: 4+ vCPU, 16GB+ RAM, 30GB disk.
   - **Lab Guides:** **[Install-OpenShift-CRC-Ubuntu](https://github.com/SumonPaul18/openshift/blob/main/install-oshift-crc.md)**

2. **OpenShift Local (successor to CRC)**
   - Newer, smoother experience (uses `crc` CLI).

### 🏢 For Production
| Method | Description | Use Case |
|------|------------|--------|
| **Installer-Provisioned Infrastructure (IPI)** | Automated cluster on AWS/Azure/GCP/vSphere | Cloud or virtualized environments |
| **User-Provisioned Infrastructure (UPI)** | You manage VMs/networking; OpenShift installs on them | On-prem, air-gapped, custom networking |
| **Assisted Installer** | GUI-driven install for bare metal | Edge, data centers |
| **OpenShift Dedicated (OSD)** | Fully managed by Red Hat on AWS/GCP | No ops team needed |
| **Azure Red Hat OpenShift (ARO)** | Jointly managed on Azure | Azure-centric orgs |

> **Tip**: Start with **CRC** → move to **IPI on AWS** → then **OSD/ARO** for production.

---

## 9. Real-World Enterprise Usage – With Official Examples

### ✅ Case Study 1: **ING Bank**
- **Challenge**: Needed secure, scalable platform for 100+ microservices.
- **Solution**: Migrated to **OpenShift Container Platform** on-prem.
- **Result**: 70% faster deployment, unified security policy across teams.
- **Source**: [Red Hat Customer Success – ING](https://www.redhat.com/en/resources/ing-case-study)

### ✅ Case Study 2: **U.S. Department of Defense**
- **Challenge**: Comply with FedRAMP, run containers securely.
- **Solution**: **OpenShift with FIPS 140-2**, SCC, and air-gapped registry.
- **Result**: Achieved **IL5** compliance.
- **Source**: [DoD DevSecOps Reference Architecture](https://public.cyber.mil/devsecops/)

### ✅ Common Patterns:
- **GitOps**: Argo CD + OpenShift for declarative deployments.
- **Service Mesh**: OpenShift Service Mesh (based on Istio) for observability.
- **Hybrid Cloud**: Run workloads across on-prem + AWS using **Red Hat Advanced Cluster Management (RHACM)**.

---

## 10. Getting Started: Your First OpenShift Project

> Assume you’ve installed **CRC** and logged in via `oc login`.

### Step 1: Create a Project
```bash
oc new-project my-first-app
```

### Step 2: Deploy a Simple App
```bash
# Deploy NGINX from Docker Hub
oc new-app nginx:latest --name=my-nginx

# Check pods
oc get pods

# Expose as Route (OpenShift Ingress)
oc expose svc/my-nginx
oc get route
```

You’ll get a URL like `my-nginx-my-first-app.apps-crc.testing` — open it in browser!

---

## 11. Hands-On Labs: Deployments, Services, Ingress, RBAC & More

All examples use **standard Kubernetes YAML** + OpenShift-specific features.

### 📄 1. Deployment + Service + Route
```yaml
# nginx-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nginx-route
spec:
  to:
    kind: Service
    name: nginx-service
  tls:
    termination: edge
```

Apply:
```bash
oc apply -f nginx-app.yaml
oc get route nginx-route
```

### 📄 2. RBAC: Restrict User Access
```yaml
# developer-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: my-first-app
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: my-first-app
subjects:
- kind: User
  name: developer@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 📄 3. Build from Source (S2I)
```bash
# Build Ruby app from GitHub
oc new-app https://github.com/sclorg/ruby-ex.git
```

OpenShift auto-detects language and builds image!

---

## 12. Advanced Scenarios

### 🔧 Node Selector
```yaml
spec:
  template:
    spec:
      nodeSelector:
        disktype: ssd
```

### 🔧 Pod Affinity
```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: ["cache"]
      topologyKey: "kubernetes.io/hostname"
```

### 🔧 Horizontal Pod Autoscaler (HPA)
```bash
oc autoscale deployment/nginx-deployment --cpu-percent=50 --min=2 --max=10
```

---

## 13. Operational Best Practices

- **Never run as root**: Use SCC to restrict privileges.
- **Use ImageStreams** for version control of container images.
- **Enable Network Policies** to isolate namespaces.
- **Monitor with OpenShift Monitoring Stack** (do not install your own Prometheus).
- **Rotate credentials** via OAuth identity providers (not local users).

---

## 14. Troubleshooting Common Issues

| Symptom | Likely Cause | Fix |
|-------|-------------|----|
| `Pod stuck in Pending` | Insufficient resources / no matching node | Check `oc describe pod`, node selectors |
| `Route not resolving` | Ingress controller down | Check `oc -n openshift-ingress get pods` |
| `Image pull error` | Private registry auth missing | Create `docker-registry` secret |
| `Permission denied` | SCC too restrictive | Assign `anyuid` SCC or custom SCC |
| `CRC fails to start` | Hypervisor not enabled | Enable VT-x/AMD-V in BIOS |

Use:
```bash
oc logs <pod>
oc describe <resource>
oc get events --sort-by=.metadata.creationTimestamp
```

---

## 15. Further Learning & Resources

- 📘 [Official OpenShift Documentation](https://docs.openshift.com/)
- 🎥 [Red Hat OpenShift YouTube Channel](https://www.youtube.com/c/RedHatOpenShift)
- 🧪 [OpenShift Interactive Learning Portal](https://learn.openshift.com/)
- 💬 [OpenShift Commons Community](https://commons.openshift.org/)
- 🐙 [GitHub: OpenShift Examples](https://github.com/openshift)

---

> **Final Note**: OpenShift isn’t just Kubernetes—it’s a **platform for building, deploying, and running secure applications at scale**. Start small (CRC), experiment, then scale to production with confidence.

---

✅ **Ready to contribute?**  
This guide is designed as a living document. Fork it, extend labs, add your real-world examples, and share with your team!

> *“OpenShift turns Kubernetes from a toolbox into a factory.”* — Sumon, DevOps Practitioner

---
