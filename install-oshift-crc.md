# 🚀 OpenShift CRC (CodeReady Containers) – Complete Home Lab Guide

> **Goal**: Run OpenShift 4.x on your local machine for learning, development, and team practice  
> **Target Machine**: Ubuntu 24.04.1, 4 CPU, 16GB RAM, 250GB SSD  
> **CRC Version**: v2.57.0 (OpenShift 4.20.5)  
> **Author**: Sumon (DevOps & Cloud Engineer)  
> **Last Updated**: December 20, 2025  

---

## 📌 Table of Contents

1. [Why Use CRC?](#1-why-use-crc)
2. [System Requirements](#2-system-requirements)
3. [Step 1: Get Red Hat Pull Secret](#3-step-1-get-red-hat-pull-secret)
4. [Step 2: Install Dependencies](#4-step-2-install-dependencies)
5. [Step 3: Create Non-Root User](#5-step-3-create-non-root-user)
6. [Step 4: Download & Install CRC](#6-step-4-download--install-crc)
7. [Step 5: Configure CRC (RAM, CPU)](#7-step-5-configure-crc-ram-cpu)
8. [Step 6: First-Time Setup & Start](#8-step-6-first-time-setup--start)
9. [Step 7: Access Web Console (Local & Remote)](#9-step-7-access-web-console-local--remote)
10. [Step 8: Set Up `oc` CLI](#10-step-8-set-up-oc-cli)
11. [Step 9: Manage Users & Passwords](#11-step-9-manage-users--passwords)
12. [Step 10: Recover After Reboot or Crash](#12-step-10-recover-after-reboot-or-crash)
13. [Step 11: Deploy Your First App (Nginx)](#13-step-11-deploy-your-first-app-nginx)
14. [Step 12: Expose Apps with Routes](#14-step-12-expose-apps-with-routes)
15. [Step 13: Use ConfigMaps and Secrets](#15-step-13-use-configmaps-and-secrets)
16. [Step 14: Monitor with Built-in Tools](#16-step-14-monitor-with-built-in-tools)
17. [Step 15: Backup & Restore Projects](#17-step-15-backup--restore-projects)
18. [Step 16: Automate with Shell Scripts](#18-step-16-automate-with-shell-scripts)
19. [Step 17: Understand CRC Architecture](#19-step-17-understand-crc-architecture)
20. [Step 18: Common Errors & Fixes](#20-step-18-common-errors--fixes)
21. [Step 19: Best Practices for Home Lab](#21-step-19-best-practices-for-home-lab)
22. [Step 20: What’s Next? Learning Path](#22-step-20-whats-next-learning-path)
23. [References](#references)

---

## 1. Why Use CRC?

**CodeReady Containers (CRC)** is Red Hat’s official tool to run a **single-node OpenShift cluster** on your laptop or home server.

✅ **Perfect for**:
- Learning OpenShift/Kubernetes
- Testing YAML manifests
- Practicing CI/CD, Operators, GitOps
- Creating demos for your team

❌ **Not for**:
- Production workloads
- Multi-node clusters
- High-availability setups

> 💡 **Real-life example**:  
> A DevOps trainer uses CRC to teach Kubernetes concepts without cloud costs.  
> A developer tests an app before deploying to AWS OpenShift.

---

## 2. System Requirements

| Component | Requirement |
|---------|-------------|
| OS | Ubuntu 24.04.1 (Desktop or Server) |
| CPU | 4 cores (Intel VT-x or AMD-V enabled in BIOS) |
| RAM | 16 GB (CRC needs **10.5 GB minimum** from v2.57+) |
| Disk | 250 GB SSD (CRC bundle = ~7 GB, plus OS and apps) |
| Network | 1 NIC with static or DHCP IP (e.g., `192.168.0.30`) |
| Internet | Required for first setup (to download bundle) |

> ⚠️ **Important**:  
> If `podman` is installed, CRC may run in **podman mode** → **no remote access**.  
> **Solution**: Uninstall `podman`:
> ```bash
> sudo apt remove -y podman
> ```

---

## 3. Step 1: Get Red Hat Pull Secret

1. Go to [https://developers.redhat.com](https://developers.redhat.com)
2. Sign up for a **free account**
3. Download your **pull secret** from:  
   [https://cloud.redhat.com/openshift/create/local](https://cloud.redhat.com/openshift/create/local)
4. Save it as `~/openshift-secret.txt`

> 🔐 This secret lets CRC download OpenShift images from Red Hat’s registry.

---

## 4. Step 2: Install Dependencies

Run these commands as **root or sudo user**:

```bash
sudo apt update
sudo apt install -y curl wget tar jq libvirt-daemon-system libvirt-clients qemu-kvm
```

Add your user to required groups:

```bash
# Replace 'osu' with your username
sudo usermod -aG libvirt,kvm osu
```

> 💡 Reboot or log out/in so group changes take effect.

---

## 5. Step 3: Create Non-Root User

**Never run CRC as root!** Create a regular user:

```bash
sudo adduser osu
sudo usermod -aG sudo,libvirt,kvm osu
```

Now switch to that user:

```bash
su - osu
```

> ✅ All future steps must be done as this user.

---

## 6. Step 4: Download & Install CRC

```bash
cd ~
wget https://mirror.openshift.com/pub/openshift-v4/clients/crc/2.57.0/crc-linux-amd64.tar.xz
tar -xf crc-linux-amd64.tar.xz
cd crc-linux-2.57.0-amd64
sudo cp crc /usr/local/bin/
```

Verify:

```bash
crc version
# Output: CRC version: 2.57.0+..., OpenShift version: 4.20.5
```

---

## 7. Step 5: Configure CRC (RAM, CPU)

CRC v2.57+ **requires at least 10.5 GB RAM**:

```bash
crc config set memory 10752  # in MiB
crc config set cpus 4
```

> 💡 On 16GB RAM machines, enable swap to avoid crashes:
> ```bash
> sudo fallocate -l 4G /swapfile
> sudo chmod 600 /swapfile
> sudo mkswap /swapfile
> sudo swapon /swapfile
> echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
> ```

---

## 8. Step 6: First-Time Setup & Start

```bash
crc setup
crc start
```

When prompted, **paste the full content** of your `openshift-secret.txt`.

> ✅ Success looks like:
> ```
> Started the OpenShift cluster.
> Web console URL: https://console-openshift-console.apps-crc.testing
> Username: kubeadmin
> Password: xxxxx-xxxxx-xxxxx-xxxxx
> ```

---

## 9. Step 7: Access Web Console (Local & Remote)

### On CRC Host (`192.168.0.30`)
Open browser → [https://console-openshift-console.apps-crc.testing](https://console-openshift-console.apps-crc.testing)  
→ Click **"Advanced" → "Proceed anyway"** (SSL warning)

### From Another Machine (e.g., your laptop at `192.168.0.35`)

#### Step 1: Create SSH Tunnel
```bash
ssh -L 8443:console-openshift-console.apps-crc.testing:443 osu@192.168.0.30
```
> Keep this terminal open.

#### Step 2: Edit `hosts` file on your laptop
- **Linux/macOS**: `sudo nano /etc/hosts`
- **Windows**: Open `C:\Windows\System32\drivers\etc\hosts` as Administrator

Add this line:
```text
127.0.0.1 console-openshift-console.apps-crc.testing
```

#### Step 3: Open Browser
Go to: [https://console-openshift-console.apps-crc.testing](https://console-openshift-console.apps-crc.testing)

✅ You’ll see OpenShift login page!

---

## 10. Step 8: Set Up `oc` CLI

The `oc` tool is OpenShift’s command-line interface.

```bash
# Make it executable
chmod +x ~/.crc/bin/oc

# Add to PATH (temporary)
export PATH="$HOME/.crc/bin:$PATH"

# Or permanently
echo 'export PATH="$HOME/.crc/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Set up CLI
eval $(crc oc-env)

# Log in (first time only)
crc console --credentials  # copy password
oc login -u kubeadmin -p 'YOUR_PASSWORD' https://api.crc.testing:6443
```

Now test:
```bash
oc get nodes
# Output: crc (Ready)
```

---

## 11. Step 9: Manage Users & Passwords

### Default Users
- **`kubeadmin`**: Temporary admin (password changes every `crc start`)
- **`developer`**: Standard user (password: `developer`)

Get credentials anytime:
```bash
crc console --credentials
```

### Create Permanent User (HTPasswd)

```bash
# Install tool
sudo apt install -y apache2-utils

# Create user 'sumon'
htpasswd -c -B -b users.htpasswd sumon MySecurePass123

# Create secret in OpenShift
oc create secret generic htpass-secret -n openshift-config --from-file=htpasswd=users.htpasswd

# Enable HTPasswd login
cat <<EOF | oc apply -f -
apiVersion: config.openshift.io/v1
kind: OAuth
meta
  name: cluster
spec:
  identityProviders:
  - name: htpasswd
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
EOF
```

Wait 1–2 minutes. Now log in via web console → select **"htpasswd"** → use `sumon` / `MySecurePass123`.

> ✅ You can now disable `kubeadmin` in production (not in CRC).

---

## 12. Step 10: Recover After Reboot or Crash

If your machine reboots, CRC may not start automatically.

### Recovery Steps:
```bash
# Switch to your user
su - osu

# Set environment (needed if using 'su -')
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"

# Start CRC daemon manually
nohup ~/.crc/bin/crc daemon > /tmp/crc-daemon.log 2>&1 &

# Start cluster
crc start
```

> ✅ All your projects, pods, and configs will be **restored automatically**.

> 💡 **Pro Tip**: Always run `crc stop` before shutting down your machine.

---

## 13. Step 11: Deploy Your First App (Nginx)

### Via Web Console:
1. Click **+ Create Project** → Name: `my-app`
2. Go to **Developer** view → **+Add** → **Container Image**
3. Image: `nginx:latest` → **Create**

### Via CLI:
```bash
oc new-project my-app
oc new-app nginx:latest
oc get pods -w  # watch until "Running"
```

---

## 14. Step 12: Expose Apps with Routes

OpenShift uses **Routes** (like Ingress in Kubernetes) to expose apps.

```bash
# Expose the nginx service
oc expose svc/nginx

# Get URL
oc get route
# Output: nginx-my-app.apps-crc.testing

# Now visit: https://nginx-my-app.apps-crc.testing
```

> 🔒 All routes use HTTPS by default in CRC.

---

## 15. Step 13: Use ConfigMaps and Secrets

### Create ConfigMap:
```bash
oc create configmap app-config --from-literal=ENV=dev --from-literal=LOG_LEVEL=info
```

### Create Secret:
```bash
oc create secret generic db-secret --from-literal=password=12345
```

### Use in Pod:
```yaml
# In your deployment YAML
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: db-secret
```

---

## 16. Step 14: Monitor with Built-in Tools

CRC includes **Prometheus, Grafana, and Alertmanager**.

1. In web console → **Administrator** → **Observe**
2. View:
   - **Metrics**: CPU, memory, network
   - **Logs**: Real-time pod logs
   - **Alerts**: System warnings

> 💡 No extra setup needed — it’s all built-in!

---

## 17. Step 15: Backup & Restore Projects

### Backup a project:
```bash
oc get all -n my-app -o yaml > my-app-backup.yaml
```

### Restore:
```bash
oc new-project my-app-restored
oc apply -f my-app-backup.yaml -n my-app-restored
```

> ⚠️ This doesn’t include persistent data (PVCs). For full backup, consider **Velero** (advanced).

---

## 18. Step 16: Automate with Shell Scripts

Create `start-crc.sh`:

```bash
#!/bin/bash
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
nohup ~/.crc/bin/crc daemon > /tmp/crc-daemon.log 2>&1 &
sleep 3
crc start
echo "OpenShift is ready at https://console-openshift-console.apps-crc.testing"
```

Make executable:
```bash
chmod +x start-crc.sh
./start-crc.sh
```

> ✅ Perfect for quick recovery after reboot.

---

## 19. Step 17: Understand CRC Architecture

| Component | Technology |
|---------|------------|
| Core | Kubernetes 1.27+ |
| OS | RHCOS (Red Hat CoreOS) |
| Runtime | CRI-O (not Docker) |
| Networking | OVN-Kubernetes |
| Web UI | React.js |
| CLI | Go (`oc`) |
| Build System | Source-to-Image (S2I) |
| Monitoring | Prometheus + Grafana |

> 🧠 **Key Insight**:  
> OpenShift = Kubernetes + Enterprise features (Routes, Operators, Web Console, RBAC, etc.)

---

## 20. Step 18: Common Errors & Fixes

| Error | Solution |
|------|--------|
| `crc should not be run as root` | Use non-root user (`osu`) |
| `Unable to find virtiofsd` | Install: `sudo apt install -y qemu-system-x86 virtiofsd` |
| `Application is not available` | Check `hosts` file + SSH tunnel |
| `oc: command not found` | Run: `chmod +x ~/.crc/bin/oc` and add to PATH |
| `crc daemon not running` | Run: `nohup ~/.crc/bin/crc daemon &` |

---

## 21. Step 19: Best Practices for Home Lab

1. ✅ Always use a **non-root user**
2. ✅ Run `crc stop` before shutdown
3. ✅ Use **HTPasswd users** instead of `kubeadmin`
4. ✅ Keep `~/.crc/cache/` — avoids re-downloading 7GB bundle
5. ✅ Use **SSH login** (not `su -`) to avoid systemd issues
6. ✅ Monitor RAM usage — enable SWAP on 16GB systems
7. ✅ Practice **GitOps** with ArgoCD or Flux later

---

## 22. Step 20: What’s Next? Learning Path

| Day | Topic | Resource |
|-----|------|--------|
| 1 | Pods, Deployments, Services | [OpenShift Basics](https://www.youtube.com/watch?v=K3Xg7y8Wm6w) |
| 2 | Routes, ConfigMaps, Secrets | This guide! |
| 3 | BuildConfigs & ImageStreams | [Red Hat Docs](https://docs.openshift.com) |
| 4 | Operators & OperatorHub | Web Console → Operators |
| 5 | Monitoring & Logging | Administrator → Observe |
| 6 | CI/CD Pipelines | OpenShift Pipelines (Tekton) |
| 7 | GitOps with ArgoCD | Install ArgoCD on CRC |

> 🎓 **Goal**: After 1 week, you can deploy a full app with DB, config, and monitoring!

---

## References

- [Red Hat CRC Official Docs](https://access.redhat.com/documentation/en-us/red_hat_codeready_containers)
- [OpenShift 4 Tutorial (YouTube)](https://www.youtube.com/watch?v=K3Xg7y8Wm6w)
- [CRC GitHub](https://github.com/code-ready/crc)
- [OpenShift Cheat Sheet](https://learn.openshift.com/cheatsheets/)

---

> ✨ **You’re ready!**  
> You now have a fully working OpenShift home lab — for learning, teaching, or prototyping.  
> Save this guide. Share it with your team. And happy DevOps-ing! 🚀

---
