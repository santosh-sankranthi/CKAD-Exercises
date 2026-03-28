## Repeating and multi-task based of Application deployment.

---

### Task 1: The Rollout Rescue & Scale (Rollouts + Scaling + History)
The exam will intentionally give you a broken environment or ask you to perform an update, realize it's wrong, fix it, and then prepare it for high traffic.

**1. CKAD Style Question:**
A Deployment named `payment-gateway` is running in the `default` namespace. 
1. Update the deployment to use the image `nginx:invalid-tag` and record the change in the rollout history.
2. Realize the error and **undo** the rollout to restore the previous working image.
3. Once restored, scale the deployment to exactly `5` replicas to handle an incoming traffic spike.

**2. Setup Script:**
```bash
kubectl create deployment payment-gateway --image=nginx:1.24 --replicas=2
kubectl annotate deployment payment-gateway kubernetes.io/change-cause="Initial stable release"
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get deploy payment-gateway -o jsonpath='{.spec.template.spec.containers[0].image}')" == "nginx:1.24" ] && echo "✅ Image successfully rolled back" || echo "❌ Rollback failed"
[ "$(kubectl get deploy payment-gateway -o jsonpath='{.spec.replicas}')" == "5" ] && echo "✅ Successfully scaled to 5" || echo "❌ Scaling failed"
```

<details>

**4. Solution:**
```bash
# 1. Update the image and record the cause
kubectl set image deployment/payment-gateway nginx=nginx:invalid-tag
kubectl annotate deployment payment-gateway kubernetes.io/change-cause="Updated to invalid tag"

# 2. Undo the rollout (reverts to the previous revision)
kubectl rollout undo deployment/payment-gateway

# 3. Scale the deployment
kubectl scale deployment payment-gateway --replicas=5
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy payment-gateway
```

---

### Task 2: The Blue/Green Production Cutover (Deployments + Labels + Services)
You will be asked to build the "Green" environment and then instantly route production traffic to it by manipulating a Service selector.

**1. CKAD Style Question:**
A Blue deployment (`auth-v1`) is serving traffic via a Service named `auth-svc`. 
1. Create a Green deployment named `auth-v2` using the `nginx:1.25` image with `2` replicas. Ensure it has the label `app=auth-v2`.
2. Update the existing `auth-svc` Service to stop routing to `auth-v1` and instantly route 100% of its traffic to `auth-v2`.

**2. Setup Script:**
```bash
kubectl create deployment auth-v1 --image=nginx:1.24 --replicas=2
kubectl expose deployment auth-v1 --name=auth-svc --port=80
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get deploy auth-v2 -o jsonpath='{.spec.replicas}')" == "2" ] && echo "✅ Green deployment created" || echo "❌ Green deployment missing"
[ "$(kubectl get svc auth-svc -o jsonpath='{.spec.selector.app}')" == "auth-v2" ] && echo "✅ Service successfully cutover to Green" || echo "❌ Service still pointing to Blue"
```

<details>

**4. Solution:**
```bash
# 1. Create the Green deployment (the label app=auth-v2 is automatically added by this command)
kubectl create deployment auth-v2 --image=nginx:1.25 --replicas=2

# 2. Update the Service selector imperatively
kubectl set selector svc auth-svc 'app=auth-v2'
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy auth-v1 auth-v2
kubectl delete svc auth-svc
```

---

### Task 3: The Canary Traffic Split (Deployments + Shared Selectors)
This tests your understanding of how Services route traffic based on labels, regardless of the Deployment name.

**1. CKAD Style Question:**
A Stable deployment (`checkout-stable`) is running with 3 replicas. A Service (`checkout-svc`) routes traffic to it using the selector `tier=frontend`.
1. Create a Canary deployment named `checkout-canary` using the `nginx:alpine` image with exactly `1` replica.
2. Configure the `checkout-canary` pods so they share the `tier=frontend` label, ensuring the Service routes 25% of traffic to the Canary.

**2. Setup Script:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-stable
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
---
apiVersion: v1
kind: Service
metadata:
  name: checkout-svc
spec:
  ports:
  - port: 80
  selector:
    tier: frontend
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get deploy checkout-canary -o jsonpath='{.spec.template.metadata.labels.tier}')" == "frontend" ] && echo "✅ Canary has correct shared label" || echo "❌ Canary label missing"
ENDPOINTS=$(kubectl get endpoints checkout-svc -o jsonpath='{.subsets[0].addresses[*].ip}' | wc -w)
[ "$ENDPOINTS" == "4" ] && echo "✅ Service routing to 4 total pods (75/25 split)" || echo "❌ Endpoint math incorrect"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base YAML
kubectl create deployment checkout-canary --image=nginx:alpine --replicas=1 --dry-run=client -o yaml > canary.yaml

# 2. Edit the YAML to ensure the pod template labels match the Service selector
vi canary.yaml
```
*Modify the `labels` and `matchLabels`:*
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend     # Must match service
  template:
    metadata:
      labels:
        tier: frontend   # Must match service
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
```
```bash
# 3. Apply the YAML
kubectl apply -f canary.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy checkout-stable checkout-canary
kubectl delete svc checkout-svc
rm canary.yaml
```

---

### Task 4: The Helm Isolation & Extraction (Helm + Namespaces + Architecture)
This task combines creating namespaces, installing Helm charts with custom values, and immediately extracting the applied values for auditing.

**1. CKAD Style Question:**
1. Create a namespace named `data-tier`.
2. Add the `bitnami` Helm repo and install the `bitnami/redis` chart into the `data-tier` namespace. Name the release `cache-store`.
3. During installation, set `architecture=standalone` and `auth.enabled=false`.
4. Once installed, extract the user-supplied values of this release and save them to `/opt/redis-values.yaml`.

**2. Setup Script:**
```bash
sudo mkdir -p /opt
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 4 ---"
[ "$(helm list -n data-tier -qf "^cache-store$")" == "cache-store" ] && echo "✅ Chart installed in correct namespace" || echo "❌ Chart installation failed"
grep -q "architecture: standalone" /opt/redis-values.yaml && echo "✅ Values successfully extracted to file" || echo "❌ Value extraction failed"
```

<details>

**4. Solution:**
```bash
# 1. Add repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# 2. Install with namespace creation and custom values
helm install cache-store bitnami/redis \
  --namespace data-tier \
  --create-namespace \
  --set architecture=standalone,auth.enabled=false

# 3. Extract the user-supplied values to a file
helm get values cache-store -n data-tier > /opt/redis-values.yaml
```

</details>

**5. Clean-up Script:**
```bash
helm uninstall cache-store -n data-tier
kubectl delete ns data-tier
rm -f /opt/redis-values.yaml
```

---

### Task 5: The Helm Upgrade & Dry-Run Audit (Helm Upgrades + Templating)
The exam will test if you can upgrade a live release, and then simulate a completely different installation without affecting the cluster.

**1. CKAD Style Question:**
A Helm release named `web-ui` (using `bitnami/nginx`) is running in the `default` namespace.
1. Upgrade the `web-ui` release to change its `service.type` to `NodePort`.
2. You have been asked to audit what the YAML would look like if you installed the `bitnami/apache` chart with `replicaCount=3`. **Do not install it.** Generate the raw YAML template and save it to `/opt/apache-audit.yaml`.

**2. Setup Script:**
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install web-ui bitnami/nginx
sudo mkdir -p /opt
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 5 ---"
[ "$(kubectl get svc web-ui-nginx -o jsonpath='{.spec.type}')" == "NodePort" ] && echo "✅ Helm release successfully upgraded" || echo "❌ Release upgrade failed"
grep -q "replicas: 3" /opt/apache-audit.yaml && echo "✅ Template generated successfully" || echo "❌ Template generation failed"
helm list -qf "apache" | grep -q "apache" && echo "❌ FAILED: You installed Apache instead of dry-running!" || echo "✅ Apache kept out of the cluster"
```

<details>

**4. Solution:**
```bash
# 1. Upgrade the existing release
helm upgrade web-ui bitnami/nginx --set service.type=NodePort

# 2. Use the template command for the audit (Dry-run)
helm template audit-release bitnami/apache --set replicaCount=3 > /opt/apache-audit.yaml
```

</details>

**5. Clean-up Script:**
```bash
helm uninstall web-ui
rm -f /opt/apache-audit.yaml
```

---
