## Kustomize
---

### Task 1: The Image Swap and Common Label
The most frequent Kustomize scenario is taking a base Deployment and mutating its image tag and labels before applying it to the cluster.

**1. CKAD Style Question:**
You have been provided a directory at `/opt/kustomize-base` containing a `deployment.yaml` file. 
Without modifying the `deployment.yaml` file, create a `kustomization.yaml` file in the same directory that does the following:
1.  Includes the `deployment.yaml` as a resource.
2.  Overrides the image of the deployment to use `nginx:1.25`.
3.  Adds a common label `env: production` to all resources.
Finally, apply this kustomization to the cluster.

**2. Setup Script:**
```bash
sudo mkdir -p /opt/kustomize-base
sudo chmod 777 /opt/kustomize-base
cat <<EOF > /opt/kustomize-base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-base
spec:
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
        image: nginx:1.24
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get deploy web-base -o jsonpath='{.spec.template.spec.containers[0].image}')" == "nginx:1.25" ] && echo "✅ Image successfully mutated via Kustomize" || echo "❌ Image mutation failed"
[ "$(kubectl get deploy web-base -o jsonpath='{.metadata.labels.env}')" == "production" ] && echo "✅ Common label added" || echo "❌ Common label missing"
```

<details>

**4. Solution:**
```bash
# 1. Navigate to the directory
cd /opt/kustomize-base

# 2. Create the kustomization.yaml file
vi kustomization.yaml
```
*Write the following configuration:*
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
commonLabels:
  env: production
images:
  - name: nginx        # The original image name in the base file
    newName: nginx     # The new image name (same in this case)
    newTag: "1.25"     # The new tag
```
```bash
# 3. Apply the directory using the -k flag (NOT -f)
kubectl apply -k .
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete -k /opt/kustomize-base
rm -rf /opt/kustomize-base
```

---

### Task 2: The ConfigMap Generator
Instead of creating a ConfigMap imperatively and attaching it, Kustomize can generate a ConfigMap and automatically hash its name so that updates trigger Pod restarts.

**1. CKAD Style Question:**
In the directory `/opt/kustomize-cm`, create a `kustomization.yaml` file. 
Configure it to generate a ConfigMap named `app-settings` containing the literal key-value pair `THEME=dark`. 
Apply the kustomization to the cluster.

**2. Setup Script:**
```bash
sudo mkdir -p /opt/kustomize-cm
sudo chmod 777 /opt/kustomize-cm
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
# Kustomize appends a hash to the generated ConfigMap name
CM_NAME=$(kubectl get cm | grep app-settings | awk '{print $1}')
[ -n "$CM_NAME" ] && echo "✅ ConfigMap generated: $CM_NAME" || echo "❌ ConfigMap missing"
[ "$(kubectl get cm $CM_NAME -o jsonpath='{.data.THEME}')" == "dark" ] && echo "✅ Literal data is correct" || echo "❌ Literal data failed"
```

<details>

**4. Solution:**
```bash
# 1. Navigate to the directory
cd /opt/kustomize-cm

# 2. Create the kustomization.yaml file
vi kustomization.yaml
```
*Write the following configuration:*
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
configMapGenerator:
  - name: app-settings
    literals:
      - THEME=dark
```
```bash
# 3. Apply the directory
kubectl apply -k .
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete -k /opt/kustomize-cm
rm -rf /opt/kustomize-cm
```

---

### Task 3: The Strategic Merge Patch
This is the classic "Dev vs. Prod" scenario. You have a base deployment, and you need to apply a patch that changes a specific field (like replicas) without touching the original file.

**1. CKAD Style Question:**
You are in `/opt/kustomize-patch` which contains a `deployment.yaml` with 1 replica.
1. Create a file named `replica-patch.yaml` that increases the replicas of the `web-patch` deployment to 3.
2. Create a `kustomization.yaml` that includes the base `deployment.yaml` and applies the `replica-patch.yaml`.
3. Apply the kustomization to the cluster.

**2. Setup Script:**
```bash
sudo mkdir -p /opt/kustomize-patch
sudo chmod 777 /opt/kustomize-patch
cat <<EOF > /opt/kustomize-patch/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-patch
spec:
  replicas: 1
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
        image: nginx:alpine
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get deploy web-patch -o jsonpath='{.spec.replicas}')" == "3" ] && echo "✅ Replicas successfully patched to 3" || echo "❌ Patch failed"
```

<details>

**4. Solution:**
```bash
# 1. Navigate to the directory
cd /opt/kustomize-patch

# 2. Create the patch file (It must contain enough metadata to target the correct resource)
vi replica-patch.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-patch
spec:
  replicas: 3
```
```bash
# 3. Create the kustomization.yaml file
vi kustomization.yaml
```
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
patchesStrategicMerge:
  - replica-patch.yaml
```
```bash
# 4. Apply the directory
kubectl apply -k .
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete -k /opt/kustomize-patch
rm -rf /opt/kustomize-patch
```

---
