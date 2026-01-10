# 🚀 OpenShift Practical Guide: Secure Nginx Deployment on CRC

> **Author**: Sumon  
> **Platform**: OpenShift CodeReady Containers (CRC)  
> **Goal**: Deploy a secure, non-root Nginx web server with TLS edge termination using best practices.

---

## 📁 What This Project Does

This project demonstrates how to deploy the official [`nginxinc/nginx-unprivileged`](https://hub.docker.com/r/nginxinc/nginx-unprivileged) image on **OpenShift CRC** with strong security settings:

✅ Runs as **non-root user** (`runAsNonRoot: true`)  
✅ Drops all Linux capabilities (`drop: ["ALL"]`)  
✅ Disables privilege escalation (`allowPrivilegeEscalation: false`)  
✅ Uses `seccompProfile: RuntimeDefault`  
✅ Exposes the app via **OpenShift Route** with **TLS edge termination**  
✅ Isolates resources in a dedicated namespace: `demo-app`

> 🔒 **Why unprivileged?**  
> OpenShift enforces strict Pod Security by default. Most containers **must not run as root**. The `nginx-unprivileged` image listens on port `8080` (non-privileged port), making it compatible out-of-the-box.

---

## ❓ What is `oc`? How is it different from `kubectl`?

- **`oc` = OpenShift Client**  
  It’s a superset of `kubectl` with **extra commands for OpenShift-specific features** like:
  - `Route` (instead of Ingress)
  - `Project` (OpenShift term for Namespace)
  - `BuildConfig`, `DeploymentConfig`
  - Security Context Constraints (`SCC`)
  - Integrated developer workflows (`oc new-app`, `oc login`, etc.)

- **`kubectl`** only manages standard Kubernetes resources.

> ✅ You can use `kubectl` on OpenShift too—but for full functionality (especially Routes), **always prefer `oc`**.

Example:
```bash
oc get route          # OpenShift-native
kubectl get ingress   # Not used in OpenShift (Routes replace Ingress)
```

---

## 🛠️ Prerequisites

Ensure the following before starting:

1. **OpenShift CRC is running**  
   ```bash
   crc status
   # Should show: CRC VM: Running, OpenShift: Running
   ```

2. **You’re logged in via `oc`**  
   ```bash
   oc login -u kubeadmin -p <your-password> https://api.crc.testing:6443
   ```

3. **`oc` CLI is installed** (comes with CRC)

---

## 📥 Step 1: Clone the Repository

```bash
git clone https://github.com/your-username/openshift-practical.git
cd openshift-practical/projects/nginx
```

> ⚠️ Replace `your-username` with your actual GitHub username.

---

## 🌐 Step 2: Create the Namespace

Your YAML defines a namespace called `demo-app`. Apply it:

```bash
oc apply -f ns-demo-app.yaml
```

✅ **Expected output**:
```
namespace/demo-app created
```

Verify:
```bash
oc get ns demo-app
```

---

## ▶️ Step 3: Deploy Nginx (Deployment + Service + Route)

> 💡 Note: Your `nginx-deploy.yaml` already includes **Deployment**, **Service**, and **Route**.  
> The separate `route.yaml` file is **redundant**—ignore it to avoid conflicts.

Apply everything in one go:

```bash
oc apply -f nginx-deploy.yaml
```

✅ **Expected output**:
```
deployment.apps/nginx-deploy created
service/nginx-svc created
route.route.openshift.io/nginx-route created
```

---

## 🔍 Step 4: Verify the Deployment

### Check if the pod is running:
```bash
oc get pods -n demo-app
```

✅ Healthy output:
```
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-7d8f5b9c8-xk2lq    1/1     Running   0          45s
```

If you see `CrashLoopBackOff` or `Error`, check logs:
```bash
oc logs -l app=nginx -n demo-app --tail=20
```

### Get the public URL:
```bash
oc get route nginx-route -n demo-app -o jsonpath='https://{.spec.host}{"\n"}'
```

✅ Example output:
```
https://nginx-route-demo-app.apps-crc.testing
```

👉 Open this URL in your browser. You should see the **default Nginx welcome page** over **HTTPS**.

> 🔒 Because `insecureEdgeTerminationPolicy: Redirect` is set, HTTP requests automatically redirect to HTTPS.

---

## 🛠️ Step 5: Operate & Manage

### Scale the deployment:
```bash
oc scale deployment/nginx-deploy --replicas=3 -n demo-app
```

### View real-time logs:
```bash
oc logs -f -l app=nginx -n demo-app
```

### Describe the pod (for debugging):
```bash
oc describe pod -l app=nginx -n demo-app
```

---

## 🗑️ Step 6: Clean Up (Full Removal)

To delete everything:

```bash
# Option 1: Delete via manifest
oc delete -f nginx-deploy.yaml -n demo-app
oc delete -f ns-demo-app.yaml

# Option 2: Delete entire namespace (recommended)
oc delete namespace demo-app
```

> ✅ Deleting the namespace removes **all** associated resources automatically.

---

## 🧩 Beyond the Basics: Recommended Enhancements

Your current setup works—but for **real-world use**, add these improvements.

---

### ✅ 1. Add Resource Requests & Limits

Prevent resource starvation and improve scheduling.

Create `nginx-resources.yaml`:
```bash
cat > nginx-resources.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
  namespace: demo-app
spec:
  template:
    spec:
      containers:
      - name: nginx
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
```

Apply it:
```bash
oc apply -f nginx-resources.yaml
```

> 💡 **Tip**: Always set `requests` and `limits` in production.

---

### ✅ 2. Add Health Checks (Liveness & Readiness)

Improve reliability and traffic routing.

Update your `nginx-deploy.yaml` under the container spec:

```yaml
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 2
          periodSeconds: 5
```

Then reapply:
```bash
oc apply -f nginx-deploy.yaml
```

> ✅ This ensures OpenShift only sends traffic to healthy pods.

---

### ✅ 3. Use a Fixed Image Tag (Not `:latest`)

Avoid unpredictable updates.

Edit `nginx-deploy.yaml`:
```diff
- image: nginxinc/nginx-unprivileged:latest
+ image: nginxinc/nginx-unprivileged:1.25-alpine
```

> 📌 **Best Practice**: Pin to a specific version in production.

---

### ✅ 4. Add Custom Nginx Configuration (Optional)

Use a ConfigMap to inject custom server blocks.

Create `nginx-configmap.yaml`:
```bash
cat > nginx-configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: demo-app
data:
  default.conf: |
    server {
        listen 8080;
        location / {
            return 200 'Hello from OpenShift Nginx!';
            add_header Content-Type text/plain;
        }
    }
EOF
```

Apply it:
```bash
oc apply -f nginx-configmap.yaml
```

Then mount it in your `nginx-deploy.yaml` under `containers`:
```yaml
        volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-config
```

Reapply:
```bash
oc apply -f nginx-deploy.yaml
```

Now your app returns a custom message!

---

### ✅ 5. Monitor with Built-in OpenShift Tools

View metrics in the Web Console:

1. Go to **OpenShift Web Console** → `https://console-openshift-console.apps-crc.testing`
2. Switch to **Developer** perspective
3. Go to **Topology** → Click your Nginx pod → **Metrics**

Or use CLI:
```bash
oc adm top pods -n demo-app
```

> Requires Metrics Operator (enabled by default in CRC).

---

## 📈 Future Roadmap

| Enhancement | Why It Matters |
|-----------|----------------|
| ✅ **Persistent Logging** | Store access/error logs on PV for audit |
| ✅ **NetworkPolicy** | Restrict pod-to-pod traffic |
| ✅ **Pod Disruption Budget** | Ensure availability during node maintenance |
| ✅ **CI/CD Pipeline** | Auto-deploy on Git push (using Tekton or GitHub Actions) |
| ✅ **External DNS** | Map route to your custom domain |

---

## 💡 Pro Tips for OpenShift CRC Users

- **Don’t use `hostPath` or `oc cp`** for data — use **PersistentVolumeClaims (PVC)** instead.
- Always test with **non-root images** — most public images (e.g., `httpd`, `tomcat`) have `-unprivileged` variants.
- Use `oc explain <resource>` to learn about any OpenShift object:
  ```bash
  oc explain route.spec.tls
  ```

---

## 📬 Need Help?

Collect diagnostics with:
```bash
oc describe pod -l app=nginx -n demo-app
oc get events -n demo-app --sort-by=.metadata.creationTimestamp
oc get route nginx-route -n demo-app -o yaml
```

---

## ✅ Summary

You now have a **production-grade-ready**, secure Nginx deployment on OpenShift CRC that:

- Follows security best practices  
- Uses native OpenShift networking (Route + TLS)  
- Is easy to manage, scale, and debug  
- Can be enhanced with monitoring, config, and CI/CD

> 🌟 **Next Step**: Try deploying your own Flask app (`sumonpaul/testflask`) using the same pattern!

---

> Happy OpenShifting!  
> — Sumon

---