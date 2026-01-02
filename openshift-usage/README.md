## 🔹 অংশ ১: OpenShift-এ Kubernetes-সমতুল্য সব কম্পোনেন্টের **ধারাবাহিক প্র্যাকটিক্যাল গাইড**  
(সব YAML, CLI, verify, delete-সহ)

---

### 1. **Namespace** (K8s) = **Project** (OpenShift)

```bash
# Kubernetes style
oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: dev-team
EOF

# OpenShift shortcut
oc new-project dev-team
```

> ✅ দুটোই কাজ করে। `oc new-project` অটোমেটিক RBAC সেট করে।

---

### 2. **Pod**

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: dev-team
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

```bash
oc apply -f pod.yaml
oc get pods -n dev-team
oc delete -f pod.yaml
```

---

### 3. **Deployment**

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: dev-team
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

```bash
oc apply -f deployment.yaml
oc get deploy,rs,pods -n dev-team
oc delete -f deployment.yaml
```

---

### 4. **Service (All Types)**

#### (a) ClusterIP (default)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: dev-team
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

#### (b) NodePort (CRC-এ কাজ করে না — single-node)
> ❌ CRC-এ NodePort external থেকে অ্যাক্সেসযোগ্য নয়

#### (c) LoadBalancer (CRC-এ কাজ করে না)
> ❌ CRC-এ কোনো cloud LB নেই

#### (d) **OpenShift Route = Ingress**
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: web-route
  namespace: dev-team
spec:
  to:
    kind: Service
    name: web-svc
  port:
    targetPort: 80
```

```bash
oc apply -f route.yaml
oc get route -n dev-team
# Output: web-route-dev-team.apps-crc.testing
```

> ✅ **Route = OpenShift-এর Ingress** — সবচেয়ে গুরুত্বপূর্ণ!

---

### 5. **PersistentVolume (PV) & PersistentVolumeClaim (PVC)**

CRC-এ **dynamic storage** অটো সেটআপ থাকে (`hostpath-provisioner`)।

#### PVC তৈরি:
```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-pvc
  namespace: dev-team
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

#### Deployment-এ ব্যবহার:
```yaml
# In deployment spec.template.spec.containers:
volumeMounts:
- name: data
  mountPath: /usr/share/nginx/html

# spec.template.spec:
volumes:
- name: data
  persistentVolumeClaim:
    claimName: web-pvc
```

```bash
oc apply -f pvc.yaml -f deployment.yaml
oc get pvc -n dev-team
oc get pv
```

> ✅ CRC-এ PV অটো তৈরি হয় (StorageClass: `hostpath-provisioner`)

---

### 6. **RBAC: Role, RoleBinding, ServiceAccount**

#### (a) ServiceAccount
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-reader
  namespace: dev-team
```

#### (b) Role (namespace-scoped)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev-team
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

#### (c) RoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev-team
subjects:
- kind: ServiceAccount
  name: app-reader
  namespace: dev-team
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
oc apply -f rbac.yaml
oc auth can-i get pods --as=system:serviceaccount:dev-team:app-reader
# Output: yes
```

---

### 7. **Node Selector & Affinity**

#### (a) Node Selector
```yaml
# In pod/deployment spec.template.spec:
nodeSelector:
  kubernetes.io/hostname: crc
```

> ✅ CRC-এ একমাত্র node-এর নাম `crc` (`oc get nodes` দেখুন)

#### (b) Node Affinity
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - crc
```

#### (c) Pod Anti-Affinity (একই node-এ দুটি pod না চলুক)
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - web
      topologyKey: kubernetes.io/hostname
```

> ✅ CRC-এ একটি node থাকায় এটি প্রভাব ফেলবে না — কিন্তু multi-node-এ কাজ করে।

---

### 8. **NetworkPolicy (Egress/Ingress Control)**

> ✅ CRC-এ OVN-Kubernetes CNI চলে — NetworkPolicy সাপোর্ট করে

#### উদাহরণ: শুধু `web` পড থেকে বাইরে যাওয়া যাবে
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-egress
  namespace: dev-team
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata:data.name: openshift-monitoring
    ports:
    - protocol: TCP
      port: 443
```

> 🔒 ডিফল্টে CRC-এ **সব ট্র্যাফিক allow** — NetworkPolicy দিয়ে restrict করুন।

---

## 🔹 অংশ ২: **OpenShift = Kubernetes + Enterprise Features**

| Kubernetes Resource | OpenShift Support | OpenShift-Only Equivalent |
|---------------------|------------------|----------------------------|
| Namespace | ✅ | Project (same) |
| Pod | ✅ | + SCC (Security Context) |
| Deployment | ✅ | + DeploymentConfig (legacy) |
| Service | ✅ | — |
| Ingress | ❌ | ✅ **Route** (`route.openshift.io/v1`) |
| StorageClass | ✅ | + **Storage** UI in Console |
| RBAC | ✅ | + **RoleBindings in Console** |
| NetworkPolicy | ✅ | + **Egress Firewall** (advanced) |
| ConfigMap/Secret | ✅ | + **Secrets in Console**, Vault integration |
| HPA | ✅ | + **Autoscaling UI** |
| CRD | ✅ | + **Operators** (managed CRDs) |

> ✅ **সব Kubernetes YAML CRC-এ কাজ করবে** — শুধু **Ingress-এর বদলে Route** ব্যবহার করুন।

---

## 🔹 অংশ ৩: **OpenShift-এর সম্পূর্ণ ফিচার লিস্ট (Kubernetes + Extra)**

### 🧱 **Core Kubernetes Components (সব কাজ করে)**
- Pods, ReplicaSets, Deployments, StatefulSets, DaemonSets
- Services (ClusterIP only in CRC)
- ConfigMaps, Secrets
- PersistentVolumes, PersistentVolumeClaims
- Namespaces, RBAC (Roles, RoleBindings, ClusterRoles)
- ServiceAccounts
- NetworkPolicies
- Resource Quotas, LimitRanges
- Taints & Tolerations, Node Affinity, Pod Affinity/Anti-Affinity
- Jobs, CronJobs

### ⚡ **OpenShift-Exclusive Features**
| ফিচার | ব্যবহার | API Group |
|-------|--------|----------|
| **Route** | External URL (HTTPS) | `route.openshift.io/v1` |
| **ImageStream** | Internal image registry reference | `image.openshift.io/v1` |
| **BuildConfig** | Source-to-Image (S2I) builds | `build.openshift.io/v1` |
| **DeploymentConfig** | Legacy deployment with triggers | `apps.openshift.io/v1` |
| **SecurityContextConstraints (SCC)** | Pod security policy (PSP replacement) | `security.openshift.io/v1` |
| **OperatorHub** | 1-click app install (Redis, PostgreSQL, etc.) | — |
| **Monitoring Stack** | Prometheus, Grafana, Alertmanager | — |
| **OAuth Identity Providers** | HTPasswd, LDAP, GitHub login | `config.openshift.io/v1` |
| **Egress Firewall** | Restrict outbound traffic (enterprise) | `k8s.ovn.org/v1` |

### 🛠️ **CLI & Tooling**
- `oc` = `kubectl` + extra commands (`oc new-app`, `oc expose`, `oc set`)
- Web Console (Developer + Administrator views)
- **Topology View** (real-time app diagram)
- **Observe** (Metrics, Logs, Alerts)

---

## ✅ সারসংক্ষেপ

- ✅ **আপনি Kubernetes-এ যা করেন, সব CRC/OpenShift-এ করতে পারবেন**  
- ✅ **শুধু Ingress → Route** পরিবর্তন করুন  
- ✅ **PV/PVC, RBAC, Affinity, NetworkPolicy — সবই কাজ করে**  
- ✅ **+ OpenShift-এর enterprise features (Routes, Operators, S2I, SCC) অতিরিক্ত সুবিধা**

> 📚 **প্র্যাকটিসের জন্য**:  
> আপনার পুরানো Kubernetes YAML গুলো CRC-এ `oc apply` করুন — 95% ক্ষেত্রে সরাসরি কাজ করবে!

আপনি যদি চান, আমি আপনাকে **একটি ZIP ফাইলের মতো** সব YAML একসাথে দিতে পারি — যেখানে প্রতিটি ফিচারের জন্য আলাদা ফাইল থাকবে।  
শুধু বলুন! 😊

---
# 🧪 OpenShift CRC – ধাপে ধাপে ব্যবহার ও ম্যানেজমেন্ট গাইড (প্র্যাকটিক্যাল + CLI)

> **লক্ষ্য**: CRC ইনস্টলেশনের পর থেকে শুরু করে বাস্তব প্রজেক্ট ডিপ্লয়মেন্ট পর্যন্ত সম্পূর্ণ গাইড  
> **প্ল্যাটফর্ম**: Ubuntu 24.04.1  
> **CRC ভার্সন**: v2.57.0 (OpenShift 4.20.5)  
> **ব্যবহারকারী**: `osu` (non-root)

---

## 1️⃣ CRC ভেরিফাই, ফাইল লোকেশন, এবং বেসিক অপারেশন

### ✅ CRC সফলভাবে ইনস্টল হয়েছে কিনা চেক করুন

```bash
# CLI ভার্সন চেক
crc version

# Output হওয়া উচিত:
# CRC version: 2.57.0+...
# OpenShift version: 4.20.5
```

### 📁 গুরুত্বপূর্ণ ফাইল/ফোল্ডার লোকেশন

| আইটেম | লোকেশন |
|-------|--------|
| CRC Config | `~/.crc/crc.json` |
| CRC Cache (Bundle) | `~/.crc/cache/` |
| CRC VM Data | `~/.crc/machines/crc/` |
| `oc` CLI | `~/.crc/bin/oc` |
| CRC Daemon | `~/.crc/bin/crc-daemon` |
| Pull Secret | যেখানে আপনি রেখেছেন (যেমন: `~/openshift-secret.txt`) |

> 💡 **বান্ডেল মুছবেন না** — নতুন ইনস্টলে 7GB ডাউনলোড লাগবে!

### ⏯️ CRC স্টার্ট / স্টপ / রিস্টার্ট

```bash
# Start
crc start

# Stop (সব ডেটা সেভ থাকবে)
crc stop

# Restart
crc stop && crc start

# Delete (সব ডেটা মুছে ফেলবে)
crc delete
```

> ✅ **Best Practice**: মেশিন শাটডাউনের আগে সবসময় `crc stop` চালান।

### 🔁 মেশিন রিবুট পরে CRC রিকভার

```bash
su - osu
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
nohup ~/.crc/bin/crc daemon > /tmp/crc-daemon.log 2>&1 &
crc start
```

---

## 2️⃣ OpenShift CRC ম্যানেজমেন্ট: CLI (`oc`) সেটআপ ও ব্যবহার

### 🛠️ `oc` CLI সেটআপ

```bash
# 1. Execute permission দিন
chmod +x ~/.crc/bin/oc

# 2. PATH-এ যোগ করুন
echo 'export PATH="$HOME/.crc/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# 3. CLI কনফিগার করুন
eval $(crc oc-env)

# 4. লগইন করুন (প্রথমবার)
crc console --credentials  # password কপি করুন
oc login -u kubeadmin -p 'YOUR_PASSWORD' https://api.crc.testing:6443
```

### 🔍 বেসিক কমান্ডস

```bash
# নোড চেক
oc get nodes

# সব পড (সব namespace)
oc get pods -A

# কারেন্ট প্রজেক্ট
oc project

# নতুন প্রজেক্ট
oc new-project my-app

# কনফিগ চেক
oc config view
```

---

## 3️⃣ OpenShift CRC-এর ধারাবাহিক ব্যবহার: প্র্যাকটিক্যাল ওয়ার্কফ্লো

### 📌 স্টেপ 1: প্রথম প্রজেক্ট তৈরি

```bash
oc new-project web-app
```

### 📌 স্টেপ 2: Nginx ডিপ্লয়

```bash
oc new-app nginx:latest
```

> ✅ এটি অটোমেটিক করবে:
> - Deployment
> - ReplicaSet
> - Pod
> - Service

### 📌 স্টেপ 3: রুট (URL) তৈরি

```bash
oc expose svc/nginx
oc get route
# Output: nginx-web-app.apps-crc.testing
```

> 🔓 এখন আপনি ব্রাউজারে যান:  
> 👉 [https://nginx-web-app.apps-crc.testing](https://nginx-web-app.apps-crc.testing)

### 📌 স্টেপ 4: কনফিগম্যাপ ও সিক্রেট যোগ করুন

```bash
# ConfigMap
oc create configmap app-config --from-literal=ENV=production

# Secret
oc create secret generic db-pass --from-literal=password=super123

# Deployment-এ যোগ করুন (যদি কাস্টম হয়)
oc set env deployment/nginx --from=configmap/app-config
```

### 📌 স্টেপ 5: মনিটরিং চেক করুন

- Web Console → **Administrator** → **Observe**  
  → Metrics, Logs, Alerts

অথবা CLI দিয়ে:

```bash
oc get pods -n openshift-monitoring
```

---

## 4️⃣ বাস্তব প্রজেক্টের প্রাক্টিক্যাল গাইড

### 🧩 প্রজেক্ট ১: স্ট্যাটিক ওয়েবসাইট (HTML)

```bash
# 1. প্রজেক্ট তৈরি
oc new-project static-site

# 2. GitHub থেকে ডিপ্লয় (S2I)
oc new-app https://github.com/yourname/simple-html-site

# 3. Route তৈরি
oc expose svc/simple-html-site
oc get route
```

> ✅ OpenShift অটো বিল্ড করবে, ইমেজ তৈরি করবে, আর ডিপ্লয় করবে!

---

### 🧩 প্রজেক্ট ২: PHP + MySQL অ্যাপ

```bash
# 1. প্রজেক্ট
oc new-project php-mysql

# 2. MySQL (OperatorHub থেকে)
# Web Console → Developer → +Add → Developer Catalog → MySQL

# 3. PHP App
oc new-app php:8.0~https://github.com/yourname/php-app

# 4. কনফিগারেশন .env হিসেবে Secret দিন
oc create secret generic app-db --from-literal=DB_HOST=mysql --from-literal=DB_PASS=12345
oc set env deployment/php-app --from=secret/app-db
```

---

### 🧩 প্রজেক্ট ৩: CI/CD পাইপলাইন (Tekton)

> 📌 প্রয়োজন: OpenShift Pipelines Operator

```bash
# 1. OperatorHub থেকে "OpenShift Pipelines" ইনস্টল করুন

# 2. Pipeline তৈরি
oc apply -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/main/01_pipeline/01_apply_manifest_task.yaml

# 3. ট্রিগার সেটআপ → GitHub push-এ অটো ডিপ্লয়!
```

> 🌐 এটি আপনাকে **real-world DevOps workflow** দেখাবে।

---

## 5️⃣ Command Line দিয়ে OpenShift ম্যানেজমেন্ট

### 🔧 কমন CLI রেফারেন্স

| কাজ | কমান্ড |
|-----|--------|
| পড লগ | `oc logs -f <pod-name>` |
| পড ডেস্ক্রাইব | `oc describe pod <pod-name>` |
| শেল এন্ট্রি | `oc rsh <pod-name>` |
| পর্ট ফরওয়ার্ডিং | `oc port-forward <pod> 8080:80` |
| ইভেন্টস | `oc get events -n <namespace>` |
| অপারেটর লিস্ট | `oc get clusteroperators` |
| রিসোর্স ইউজেজ | `oc top pods` |

### 🔐 ইউজার ম্যানেজমেন্ট

```bash
# HTPasswd user তৈরি
htpasswd -b -B users.htpasswd dev1 pass123
oc create secret generic htpasswd-secret -n openshift-config --from-file=htpasswd=users.htpasswd

# রোল অ্যাসাইন
oc policy add-role-to-user admin dev1 -n web-app
```

---

## 6️⃣ ট্রাবলশুটিং: কমন ইস্যু ও সমাধান

### ❌ Issue 1: `crc start` → `virtiofsd not found`

```bash
sudo apt install -y qemu-system-x86 virtiofsd
```

### ❌ Issue 2: `crc daemon not running`

```bash
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
nohup ~/.crc/bin/crc daemon > /tmp/crc.log 2>&1 &
crc start
```

### ❌ Issue 3: ওয়েব কনসোলে "Application not available"

- **কারণ**: `hosts` ফাইলে এন্ট্রি নেই বা SSH টানেল নেই  
- **সমাধান**:
  ```text
  # Remote machine-এ /etc/hosts-এ যোগ করুন:
  127.0.0.1 console-openshift-console.apps-crc.testing
  ```
  + SSH tunnel চালু রাখুন

### ❌ Issue 4: `oc: command not found`

```bash
chmod +x ~/.crc/bin/oc
export PATH="$HOME/.crc/bin:$PATH"
```

### ❌ Issue 5: CRC VM crash হয়েছে

```bash
virsh destroy crc    # force stop
virsh start crc      # start VM
crc start            # resume cluster
```

---

## ✅ সারসংক্ষেপ: আপনার ওয়ার্কফ্লো চেকলিস্ট

- [ ] `crc start` → Web Console অ্যাক্সেস
- [ ] `oc` CLI সেটআপ
- [ ] নতুন প্রজেক্ট তৈরি
- [ ] Nginx/HTML অ্যাপ ডিপ্লয়
- [ ] Route তৈরি → ব্রাউজারে অ্যাক্সেস
- [ ] ConfigMap/Secret ব্যবহার
- [ ] HTPasswd user তৈরি
- [ ] মনিটরিং চেক
- [ ] ব্যাকআপ নিন (`oc get all -o yaml`)
- [ ] `crc stop` before shutdown

---

> 🎯 **লক্ষ্য**: ৭ দিনে আপনি একটি **সম্পূর্ণ অ্যাপ** (Frontend + Backend + DB + CI/CD) OpenShift-এ ডিপ্লয় করতে পারবেন!

> 📚 **গাইড লিঙ্ক**: [https://github.com/SumonPaul18/openshift](https://github.com/SumonPaul18/openshift)

Happy OpenShifting! 🚀

---

অবশ্যই! নিচে আমি আপনাকে **সম্পূর্ণ প্র্যাকটিক্যাল, ধাপে-ধাপে, YAML-সহ** দুটি অংশে গাইড দিচ্ছি:

1. **5টি বেসিক থেকে ইন্টারমিডিয়েট OpenShift প্রজেক্ট** (YAML + CLI commands)  
2. **OpenShift-এর প্রতিটি Kubernetes-সমতুল্য কম্পোনেন্ট ও এন্টারপ্রাইজ ফিচারের ব্যবহার**

সবকিছু **আপনার CRC (v2.57.0, OpenShift 4.20.5)**-এ চলবে।

---

## 🧪 অংশ ১: ৫টি প্র্যাকটিক্যাল OpenShift প্রজেক্ট (YAML + CLI)

> ✅ সব প্রজেক্ট আপনার CRC-এ সরাসরি চালানো যাবে  
> ✅ প্রতিটির জন্য `apply`, `verify`, `delete` গাইড আছে

---

### 📁 প্রজেক্ট ১: **Namespace + Pod (বেসিক)**

#### `pod.yaml`
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-ns
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: demo-ns
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
```

#### চালানো ও ভেরিফাই:
```bash
oc apply -f pod.yaml
oc get pods -n demo-ns
oc describe pod nginx-pod -n demo-ns
```

#### ডিলিট:
```bash
oc delete -f pod.yaml
# বা
oc delete ns demo-ns  # সব রিসোর্স একসাথে মুছবে
```

---

### 📁 প্রজেক্ট ২: **Deployment + Service**

#### `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: demo-app
spec:
  replicas: 3
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
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: demo-app
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

#### চালানো:
```bash
oc create namespace demo-app
oc apply -f deployment.yaml
oc get pods -n demo-app
oc get svc -n demo-app
```

#### ইন্টারনাল টেস্ট:
```bash
# অন্য পড থেকে curl করুন
oc run tester --image=busybox --rm -it --restart=Never -n demo-app -- sh
# তারপর:
wget -qO- http://nginx-svc:80
```

#### ডিলিট:
```bash
oc delete -f deployment.yaml
oc delete ns demo-app
```

---

### 📁 প্রজেক্ট ৩: **OpenShift Route (Ingress-এর বদলে)**

> ⚠️ OpenShift-এ **Ingress নয়, Route ব্যবহার করা হয়**

#### `route.yaml`
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nginx-route
  namespace: demo-app
spec:
  to:
    kind: Service
    name: nginx-svc
  port:
    targetPort: 80
```

#### চালানো:
```bash
# প্রথমে deployment.yaml চালান (উপরের)
oc apply -f route.yaml
oc get route -n demo-app
```

> ✅ আউটপুট: `nginx-route-demo-app.apps-crc.testing`  
> 👉 ব্রাউজারে যান: [https://nginx-route-demo-app.apps-crc.testing](https://nginx-route-demo-app.apps-crc.testing)

#### ডিলিট:
```bash
oc delete -f route.yaml
```

---

### 📁 প্রজেক্ট ৪: **ConfigMap + Secret**

#### `config-secret.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo-app

  LOG_LEVEL: "info"
  ENV: "prod"
---
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: demo-app
type: Opaque
data:
  password: c3VwZXJzZWNyZXQ=  # base64 of "supersecret"
```

> 💡 Base64 encode: `echo -n "supersecret" | base64`

#### ডেপ্লয়মেন্টে ব্যবহার (`app-with-config.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-config
  namespace: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: configured-app
  template:
    metadata:
      labels:
        app: configured-app
    spec:
      containers:
      - name: app
        image: nginx
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: db-secret
```

#### চালানো:
```bash
oc create ns demo-app
oc apply -f config-secret.yaml
oc apply -f app-with-config.yaml
oc exec -it <pod-name> -n demo-app -- env | grep -E 'LOG_LEVEL|password'
```

#### ডিলিট:
```bash
oc delete -f app-with-config.yaml
oc delete -f config-secret.yaml
```

---

### 📁 প্রজেক্ট ৫: **Build from Source (S2I - OpenShift Exclusive)**

> 🌟 এটি Kubernetes-এ নেই — OpenShift-এর ইউনিক ফিচার!

#### `s2i.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
  namespace: s2i-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs
  template:
    metadata:
      labels:
        app: nodejs
    spec:
      containers:
      - name: nodejs
        image: ' '  # S2I will fill this
        ports:
        - containerPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nodejs-route
  namespace: s2i-demo
spec:
  to:
    kind: Service
    name: nodejs-app
```

#### চালানো (CLI দিয়ে S2I):
```bash
oc new-project s2i-demo
oc new-app https://github.com/sclorg/nodejs-ex#master --name=nodejs-app
oc expose svc/nodejs-app
oc get route
```

> ✅ OpenShift অটো:
> - GitHub clone করবে  
> - Node.js build করবে  
> - ইমেজ বানাবে  
> - ডিপ্লয় করবে

#### ডিলিট:
```bash
oc delete project s2i-demo
```

---

## 🧩 অংশ ২: OpenShift কম্পোনেন্টস vs Kubernetes + এন্টারপ্রাইজ ফিচারস

| Kubernetes Component | OpenShift Equivalent | OpenShift-Only Features |
|----------------------|----------------------|--------------------------|
| **Ingress** | **Route** (`route.openshift.io/v1`) | - Auto TLS<br>- HAProxy-based<br>- Path-based routing built-in |
| **Deployment** | Same (`apps/v1`) | + **DeploymentConfig** (legacy, with triggers) |
| **Service** | Same (`v1`) | + **Service Mesh** (Istio-based) |
| **Namespace** | **Project** (same, but `oc new-project`) | - Built-in RBAC<br>- Quota management UI |
| **ConfigMap/Secret** | Same | + **Secrets in Web Console**<br>+ **Vault integration** |
| **Pod** | Same | + **Security Context Constraints (SCC)** — replaces PSP |
| **kubectl** | **oc** | - `oc new-app`<br>- `oc expose`<br>- `oc set env`<br>- `oc rsh` |
| **Metrics Server** | **Monitoring Stack** | - Prometheus<br>- Grafana<br>- Alertmanager (built-in) |
| **Helm** | **Operators** | - OperatorHub (1-click install)<br>- Lifecycle automation |
| **Container Runtime** | CRI-O (not Docker) | Immutable OS (RHCOS) |

---

### 🔧 প্র্যাকটিক্যাল কম্পোনেন্ট এক্সেস

#### 1. **Routes (Ingress বদলে)**
```bash
oc get routes -A
oc describe route <name> -n <namespace>
```

#### 2. **Operators (Helm-এর বদলে)**
- Web Console → **Operators** → **OperatorHub**  
- CLI: `oc get csv -n openshift-operators`

#### 3. **Monitoring (Prometheus/Grafana)**
```bash
oc get pods -n openshift-monitoring
# Web: Administrator → Observe
```

#### 4. **Security Context Constraints (SCC)**
```bash
oc get scc
oc describe scc restricted
```

#### 5. **ImageStreams (Container Registry Abstraction)**
```bash
oc get is -n openshift  # pre-loaded images
oc tag nginx:latest my-nginx:prod -n demo
```

> 💡 ImageStream = OpenShift-এর internal image registry reference

#### 6. **BuildConfigs (S2I)**
```bash
oc get bc
oc start-build nodejs-app
```

---

## ✅ সারসংক্ষেপ

- ✅ **Kubernetes কম্পোনেন্টস সব OpenShift-এ কাজ করে**  
- ✅ **+ এন্টারপ্রাইজ ফিচার**: Routes, Operators, S2I, Monitoring, SCC  
- ✅ **YAML একই**, কিন্তু `oc` CLI দিয়ে অনেক কাজ সহজ  
- ✅ **Web Console**-এ সবকিছু ভিজ্যুয়ালি ম্যানেজ করা যায়

---

> 📚 **সব YAML এবং গাইড একসাথে**: [https://github.com/SumonPaul18/openshift](https://github.com/SumonPaul18/openshift)

আপনি এখন **Kubernetes জানলেই OpenShift-এর 80% জানেন** — বাকি 20% (Routes, Operators, S2I) দিয়ে আপনি **enterprise DevOps** হয়ে উঠবেন! 🚀

প্রয়োজনে আরও প্রজেক্ট (MySQL, CI/CD, GitOps) চাইলে জানান!

---
অবশ্যই! নিচে আমি আপনাকে **OpenShift-এর সম্পূর্ণ ফিচার সেট** কভার করে **ধারাবাহিক, প্র্যাকটিক্যাল প্রজেক্ট সিরিজ** দিচ্ছি — প্রতিটি প্রজেক্ট আপনার **CRC v2.57.0 (OpenShift 4.20.5)**-এ চলবে।

প্রতিটি প্রজেক্টে থাকবে:
- ✅ **উদ্দেশ্য** (কেন এটি গুরুত্বপূর্ণ?)
- ✅ **YAML / CLI কমান্ড**
- ✅ **চালানোর ধাপ**
- ✅ **ভেরিফাই করার উপায়**
- ✅ **বাস্তব ব্যবহারের উদাহরণ**

---

## 🧪 প্রজেক্ট ১: **ImageStream + BuildConfig (S2I — OpenShift Exclusive)**

> 🎯 **উদ্দেশ্য**: OpenShift-এর **Source-to-Image (S2I)** ব্যবহার করে GitHub থেকে অটো বিল্ড ও ডিপ্লয়মেন্ট — **Kubernetes-এ নেই!**

### ধাপ ১: প্রজেক্ট তৈরি
```bash
oc new-project s2i-demo
```

### ধাপ ২: S2I বিল্ড শুরু করুন
```bash
oc new-app https://github.com/sclorg/nodejs-ex --name=nodejs-app
```

> ✅ OpenShift অটো:
> - `nodejs-ex` clone করবে  
> - Node.js বিল্ড করবে  
> - ইমেজ তৈরি করবে (`nodejs-app:latest` in **ImageStream**)  
> - Deployment চালু করবে

### ধাপ ৩: Route তৈরি করুন
```bash
oc expose svc/nodejs-app
oc get route
# Output: nodejs-app-s2i-demo.apps-crc.testing
```

### ধাপ ৪: ভেরিফাই
- ব্রাউজারে যান: [https://nodejs-app-s2i-demo.apps-crc.testing](https://nodejs-app-s2i-demo.apps-crc.testing)
- দেখুন: "Welcome to your Node.js application on OpenShift"

### ধাপ ৫: ImageStream চেক করুন
```bash
oc get is
oc describe is nodejs-app
```

> 🌐 **বাস্তব ব্যবহার**:  
> ডেভেলপার শুধু GitHub-এ কোড পুশ করে — OpenShift অটো বিল্ড → ডিপ্লয়!

### ডিলিট:
```bash
oc delete project s2i-demo
```

---

## 🧪 প্রজেক্ট ২: **OperatorHub থেকে Redis ইনস্টল**

> 🎯 **উদ্দেশ্য**: **Operators** ব্যবহার করে stateful সার্ভিস (Redis) এক ক্লিকে চালু করা — **Helm-এর enterprise-grade বিকল্প**

### ধাপ ১: Redis Operator ইনস্টল (CLI)
```bash
oc new-project redis-demo

# OperatorGroup তৈরি
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: redis-operators
  namespace: redis-demo
spec:
  targetNamespaces:
  - redis-demo
EOF

# Subscription তৈরি
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: redis-operator
  namespace: redis-demo
spec:
  channel: alpha
  name: redis-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
EOF
```

> ⏳ ২-৩ মিনিট অপেক্ষা করুন

### ধাপ ২: Redis ইনস্ট্যান্স তৈরি
```bash
cat <<EOF | oc apply -f -
apiVersion: redis.redis.opstreelabs.io/v1beta1
kind: Redis
metadata:
  name: my-redis
  namespace: redis-demo
spec:
  mode: standalone
  size: 1
  storage:
    volumeSize: 1Gi
EOF
```

### ধাপ ৩: ভেরিফাই
```bash
oc get pods -n redis-demo
# আউটপুট: my-redis-standalone-0 (Running)

# কানেক্ট করুন
oc run redis-client --image=redis --rm -it --restart=Never -- redis-cli -h my-redis-standalone
```

> 🌐 **বাস্তব ব্যবহার**:  
> ব্যাংক, ই-কমার্স সাইট — সবাই Redis ব্যবহার করে caching ও session storage-এর জন্য।

### ডিলিট:
```bash
oc delete project redis-demo
```

---

## 🧪 প্রজেক্ট ৩: **Built-in Monitoring (Prometheus + Grafana)**

> 🎯 **উদ্দেশ্য**: OpenShift-এর **in-cluster monitoring stack** ব্যবহার করে পডের CPU, memory, network usage দেখা

### ধাপ ১: একটি অ্যাপ ডিপ্লয় করুন
```bash
oc new-project monitor-demo
oc new-app nginx:latest
```

### ধাপ ২: Web Console-এ যান
- **Administrator** → **Observe** → **Metrics**
- **Namespace**: `monitor-demo`
- দেখুন:
  - CPU Usage
  - Memory Usage
  - Network I/O

### ধাপ ৩: CLI দিয়ে মেট্রিক্স চেক (অপশনাল)
```bash
# Prometheus endpoint চেক (advanced)
oc get route -n openshift-monitoring
# তারপর bearer token ব্যবহার করে API call
```

> 🌐 **বাস্তব ব্যবহার**:  
> SRE টিম Prometheus alert সেট করে — যখনই CPU 90% ছাড়ায়, Slack-এ alert যায়!

### ডিলিট:
```bash
oc delete project monitor-demo
```

---

## 🧪 প্রজেক্ট ৪: **Security Context Constraints (SCC)**

> 🎯 **উদ্দেশ্য**: Kubernetes-এর **PodSecurityPolicy (PSP)**-এর বদলে OpenShift-এ **SCC** ব্যবহার করা

### ধাপ ১: ডিফল্ট SCC চেক করুন
```bash
oc get scc
# Output: anyuid, restricted, privileged, etc.
```

### ধাপ ২: একটি পড চালান যেটা root হিসেবে চলবে
```yaml
# root-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
spec:
  containers:
  - name: alpine
    image: alpine
    command: ["/bin/sleep", "3600"]
    securityContext:
      runAsUser: 0  # root
```

### ধাপ ৩: চালানো ও এরর দেখুন
```bash
oc apply -f root-pod.yaml
oc get pods
# Status: CreateContainerConfigError
oc describe pod root-pod
# Reason: containers with UID 0 (root) not allowed in 'restricted' SCC
```

### ধাপ ৪: SCC পরিবর্তন করুন (অপশনাল, শুধু টেস্টের জন্য!)
```bash
oc adm policy add-scc-to-user anyuid -z default -n default
oc delete pod root-pod
oc apply -f root-pod.yaml  # এবার চলবে
```

> 🛡️ **বাস্তব ব্যবহার**:  
> ব্যাংকিং অ্যাপে container কখনো root-এ চলবে না — SCC enforce করে security policy.

### ডিলিট:
```bash
oc delete -f root-pod.yaml
```

---

## 🧪 প্রজেক্ট ৫: **GitOps with ArgoCD (Advanced)**

> 🎯 **উদ্দেশ্য**: OpenShift-এ **continuous delivery** সেটআপ করা — যেখানে Git repo আপডেট হলে অটো ডিপ্লয় হয়

### ধাপ ১: ArgoCD Operator ইনস্টল (OperatorHub থেকে)
- Web Console → **Operators** → **OperatorHub** → Search "Argo CD" → Install

### ধাপ ২: ArgoCD ইনস্ট্যান্স তৈরি
```yaml
# argocd.yaml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: argocd
  namespace: argocd
spec: {}
```

```bash
oc create namespace argocd
oc apply -f argocd.yaml
```

### ধাপ ৩: ArgoCD UI অ্যাক্সেস করুন
```bash
oc get route -n argocd
# পাসওয়ার্ড:
oc extract secret/argocd-cluster -n argocd --to=-
```

### ধাপ ৪: একটি অ্যাপ যোগ করুন (GitHub repo লিঙ্ক দিয়ে)
- UI-তে "Create Application"
- Repo URL: `https://github.com/yourname/openshift-manifests.git`
- Path: `nginx/`
- Cluster: `https://kubernetes.default.svc`
- Namespace: `gitops-demo`

> 🌐 **বাস্তব ব্যবহার**:  
> bKash, DBBL — সবাই GitOps ব্যবহার করে production deployment automate করে।

### ডিলিট:
```bash
oc delete project argocd
oc delete project gitops-demo
```

---

## ✅ সারসংক্ষেপ: আপনি এখন পারেন

| ফিচার | কী শিখলেন | বাস্তব প্রয়োগ |
|--------|------------|----------------|
| **S2I** | GitHub → Auto Build → Deploy | Developer productivity |
| **Operators** | 1-click Redis, PostgreSQL | Stateful app management |
| **Monitoring** | Prometheus + Grafana | Observability, Alerting |
| **SCC** | Security policy enforcement | Compliance (banking, govt) |
| **GitOps** | Git-driven deployment | CI/CD, Audit trail |

---

> 📚 **সব কোড, YAML, গাইড**: [https://github.com/SumonPaul18/openshift](https://github.com/SumonPaul18/openshift)  
> 🎯 **পরবর্তী লক্ষ্য**:  
> - OpenShift Pipelines (Tekton)  
> - Service Mesh (Istio)  
> - Cluster Logging (EFK)

আপনি যদি চান, আমি **পরবর্তী ৫টি প্রজেক্ট** (CI/CD, Logging, Service Mesh, Quotas, NetworkPolicy) দিতে পারি।  
শুধু বলুন: **"চালিয়ে যান!"** 😊

---



