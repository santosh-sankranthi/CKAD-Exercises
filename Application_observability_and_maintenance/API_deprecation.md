## API Deprecation

To be completely straightforward with you, the subtopic **"Understand API Deprecations"** is very narrow. While some domains required 5 or 9 tasks to cover, API deprecations really only boil down to three distinct scenarios. I am going to give you **3 Main Tasks** (with their strict variations) because extending this to 5 main tasks would just be creating artificial, repetitive filler. 

On the exam, this topic always presents as a "Fix the broken YAML" question. You will be given a file created for an older version of Kubernetes (like v1.15), and when you try to apply it to the exam cluster (v1.30+), it will throw an error. 


---

### Task 1: The Version String Swap (The Most Repeating)
Often, a resource's structure hasn't changed at all; only its API Group version has been promoted from "beta" to stable. You just need to fix the string at the top of the file.

**1. CKAD Style Question:**
You have been provided a file at `/opt/legacy-cron.yaml`. It was written for an old cluster and fails to apply because the `batch/v1beta1` API is no longer served. 
Fix the YAML file to use the modern, stable API version for CronJobs and deploy it to the cluster.

**2. Setup Script:**
```bash
sudo mkdir -p /opt
sudo chmod 777 /opt
cat <<EOF > /opt/legacy-cron.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: legacy-backup
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command: ["echo", "backing up"]
          restartPolicy: OnFailure
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get cronjob legacy-backup -o jsonpath='{.apiVersion}')" == "batch/v1" ] && echo "✅ CronJob successfully deployed with modern API" || echo "❌ Deployment failed or API incorrect"
```

<details>

**4. Solution:**
```bash
# 1. Open the file
vi /opt/legacy-cron.yaml

# 2. Change the very first line from 'batch/v1beta1' to 'batch/v1'
# (Save and exit)

# 3. Apply the fixed file
kubectl apply -f /opt/legacy-cron.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete cronjob legacy-backup
rm -f /opt/legacy-cron.yaml
```

#### Variation 1.1: The PodDisruptionBudget (PDB) Promotion
**1. CKAD Style Question:**
A file at `/opt/legacy-pdb.yaml` uses the deprecated `policy/v1beta1` API. Fix the file to use the stable API version and apply it to the cluster.

**2. Setup Script:**
```bash
cat <<EOF > /opt/legacy-pdb.yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: frontend
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get pdb app-pdb -o jsonpath='{.apiVersion}')" == "policy/v1" ] && echo "✅ PDB deployed with modern API" || echo "❌ PDB deployment failed"
```

<details>

**4. Solution:**
```bash
vi /opt/legacy-pdb.yaml
# Change 'policy/v1beta1' to 'policy/v1'
kubectl apply -f /opt/legacy-pdb.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pdb app-pdb
rm -f /opt/legacy-pdb.yaml
```

#### Variation 1.2: The RBAC Promotion
**1. CKAD Style Question:**
A file at `/opt/legacy-role.yaml` uses the deprecated `rbac.authorization.k8s.io/v1beta1` API. Update it to the stable version and deploy it.

**2. Setup Script:**
```bash
cat <<EOF > /opt/legacy-role.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: legacy-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get role legacy-role -o jsonpath='{.apiVersion}')" == "rbac.authorization.k8s.io/v1" ] && echo "✅ Role deployed with modern API" || echo "❌ Role deployment failed"
```

<details>

**4. Solution:**
```bash
vi /opt/legacy-role.yaml
# Change 'rbac.authorization.k8s.io/v1beta1' to 'rbac.authorization.k8s.io/v1'
kubectl apply -f /opt/legacy-role.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete role legacy-role
rm -f /opt/legacy-role.yaml
```

---

### Task 2: The Structural Paradigm Shift (The Boss Fight)
This is where the exam gets lethal. Sometimes, when an API moves from `v1beta1` to `v1`, Kubernetes changes the actual structure of the YAML. If you just change the top string, it will still fail to apply. You must fix the syntax.

**1. CKAD Style Question:**
A file at `/opt/legacy-ingress.yaml` was written for Kubernetes v1.18 using `networking.k8s.io/v1beta1`. 
It fails to apply on the modern cluster. 
1. Update the API version to `networking.k8s.io/v1`.
2. Fix the structural schema changes required by the `v1` API (specifically regarding `pathType`, `serviceName`, and `servicePort`).
3. Apply the file.

**2. Setup Script:**
```bash
cat <<EOF > /opt/legacy-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: legacy-ingress
spec:
  rules:
  - http:
      paths:
      - path: /app
        backend:
          serviceName: app-svc
          servicePort: 80
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get ingress legacy-ingress -o jsonpath='{.spec.rules[0].http.paths[0].pathType}')" == "Prefix" ] && echo "✅ PathType successfully added" || echo "❌ PathType missing"
[ "$(kubectl get ingress legacy-ingress -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.name}')" == "app-svc" ] && echo "✅ Modern service backend structure correct" || echo "❌ Legacy backend structure remains"
```

<details>

**4. Solution:**
```bash
vi /opt/legacy-ingress.yaml
```
*You must completely refactor the `backend` block and add `pathType`:*
```yaml
apiVersion: networking.k8s.io/v1    # 1. Change version
kind: Ingress
metadata:
  name: legacy-ingress
spec:
  rules:
  - http:
      paths:
      - path: /app
        pathType: Prefix            # 2. REQUIRED IN V1
        backend:                    # 3. RESTRUCTURED IN V1
          service:
            name: app-svc
            port:
              number: 80
```
```bash
kubectl apply -f /opt/legacy-ingress.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete ingress legacy-ingress
rm -f /opt/legacy-ingress.yaml
```

#### Variation 2.1: The Legacy Deployment Schema
**1. CKAD Style Question:**
A very old file at `/opt/legacy-deploy.yaml` uses `extensions/v1beta1`. 
In modern `apps/v1` Deployments, the `selector` block is absolutely mandatory, but it was optional in `v1beta1`. Fix the API version and add the missing required structure.

**2. Setup Script:**
```bash
cat <<EOF > /opt/legacy-deploy.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: old-deploy
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: old-app
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get deploy old-deploy -o jsonpath='{.spec.selector.matchLabels.app}')" == "old-app" ] && echo "✅ Required selector block added" || echo "❌ Deployment failed or missing selector"
```

<details>

**4. Solution:**
```bash
vi /opt/legacy-deploy.yaml
```
```yaml
apiVersion: apps/v1                 # 1. Update version
kind: Deployment
metadata:
  name: old-deploy
spec:
  replicas: 1
  selector:                         # 2. ADD THIS REQUIRED BLOCK
    matchLabels:
      app: old-app
  template:
    metadata:
      labels:
        app: old-app
    spec:
      containers:
      - name: nginx
        image: nginx
```
```bash
kubectl apply -f /opt/legacy-deploy.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy old-deploy
rm -f /opt/legacy-deploy.yaml
```

---

### Task 3: The CLI Discovery Strategy (When you blank on the exam)
You cannot memorize every API version. The exam expects you to use the command line to discover the correct, currently supported API version when presented with a broken file.

**1. CKAD Style Question:**
A file at `/opt/broken-hpa.yaml` has a completely made-up API version (`autoscaling/v999`). 
Without opening the Kubernetes documentation browser, use the `kubectl` CLI to discover the correct API group and version for a `HorizontalPodAutoscaler`. Fix the file and apply it.

**2. Setup Script:**
```bash
cat <<EOF > /opt/broken-hpa.yaml
apiVersion: autoscaling/v999
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: dummy-deploy
  minReplicas: 1
  maxReplicas: 3
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get hpa api-hpa -o jsonpath='{.apiVersion}')" == "autoscaling/v2" ] && echo "✅ HPA deployed with correct API via discovery" || echo "❌ Deployment failed"
```

<details>

**4. Solution:**
```bash
# 1. Use api-resources to search for the correct API version! This is your lifeline.
kubectl api-resources | grep HorizontalPodAutoscaler

# The output will show "autoscaling/v2" (or v1 depending on cluster version).

# 2. Edit the file
vi /opt/broken-hpa.yaml
# Change 'autoscaling/v999' to 'autoscaling/v2'

# 3. Apply the file
kubectl apply -f /opt/broken-hpa.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete hpa api-hpa
rm -f /opt/broken-hpa.yaml
```

#### Variation 3.1: Discovering Structural Schema Changes
**1. CKAD Style Question:**
You updated the API version of a file, but when you apply it, Kubernetes complains about an unknown field. Use the CLI to output the exact YAML schema blueprint for the `spec` of a modern `Ingress` resource so you know how to rewrite it. Save this blueprint to `/opt/ingress-schema.txt`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
grep -q "defaultBackend" /opt/ingress-schema.txt && echo "✅ Schema successfully extracted via CLI" || echo "❌ Schema extraction failed"
```

<details>

**4. Solution:**
```bash
# When you don't know the structure, 'kubectl explain' is your dictionary.
# The --recursive flag shows the entire YAML tree.
kubectl explain ingress.spec --recursive > /opt/ingress-schema.txt
```

</details>

**5. Clean-up Script:**
```bash
rm -f /opt/ingress-schema.txt
```

---

That gives you total coverage over API deprecations, including the deadly structural traps and the exact CLI tools (`api-resources` and `explain`) you need to survive if you forget a version string.
