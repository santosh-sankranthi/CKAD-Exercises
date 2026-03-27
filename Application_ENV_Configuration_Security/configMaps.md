## ConfigMaps

ConfigMaps are the absolute backbone of the "Configuration" domain on the CKAD. The exam tests this topic to ensure you know how to decouple configuration artifacts (like database URLs, environment variables, or `.conf` files) from your container images.

On the exam, you will be tested on **two primary ways** to consume a ConfigMap:
1.  Injecting specific keys as **Environment Variables**.
2.  Mounting the entire ConfigMap as a **Volume** (so the keys become files).

If you can master these three tasks, you have 100% coverage of the ConfigMap objective. Let's lock it down.

---

### Task 1: Specific Environment Variable Injection (Most Repeating)
The exam will often ask you to create a ConfigMap imperatively and then selectively pluck specific keys out of it to serve as environment variables inside a Pod.

**1. CKAD Style Question:**
Create a ConfigMap named `db-config` in the `default` namespace with two key-value pairs: `DB_HOST=cluster-db` and `DB_PORT=5432`.
Create a Pod named `backend-api` using the `nginx:alpine` image.
Configure the Pod to create an environment variable named `DATABASE_URL` that pulls its value directly from the `DB_HOST` key in the `db-config` ConfigMap.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get cm db-config -o jsonpath='{.data.DB_HOST}')" == "cluster-db" ] && echo "✅ ConfigMap DB_HOST created" || echo "❌ ConfigMap missing or incorrect"
[ "$(kubectl get pod backend-api -o jsonpath='{.spec.containers[0].env[0].name}')" == "DATABASE_URL" ] && echo "✅ Env Var name correct" || echo "❌ Env Var name failed"
[ "$(kubectl get pod backend-api -o jsonpath='{.spec.containers[0].env[0].valueFrom.configMapKeyRef.key}')" == "DB_HOST" ] && echo "✅ Env Var references correct ConfigMap key" || echo "❌ ConfigMap reference failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the ConfigMap imperatively (Never write this YAML by hand!)
kubectl create configmap db-config --from-literal=DB_HOST=cluster-db --from-literal=DB_PORT=5432

# 2. Generate the base Pod YAML
kubectl run backend-api --image=nginx:alpine --dry-run=client -o yaml > backend.yaml

# 3. Edit the YAML
vi backend.yaml
```
*Add the `env` block under the container spec:*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-api
spec:
  containers:
  - image: nginx:alpine
    name: backend-api
    env:                                # ADD FROM HERE
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: db-config
          key: DB_HOST                  # TO HERE
```
```bash
# 4. Apply the YAML
kubectl apply -f backend.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod backend-api
kubectl delete cm db-config
rm backend.yaml
```

---

### Task 2: Mounting a ConfigMap as a Volume (Highly Repeating)
Instead of environment variables, the exam might hand you a configuration file (like an `nginx.conf` or a `settings.json`) and ask you to mount it directly into the container's file system using a ConfigMap.

**1. CKAD Style Question:**
A file named `custom.conf` exists on your node. Create a ConfigMap named `proxy-config` from this file. 
Next, create a Pod named `proxy-server` using the `nginx` image. 
Mount the `proxy-config` ConfigMap as a volume so that the `custom.conf` file appears inside the container at the path `/etc/nginx/conf.d/custom.conf`.

**2. Setup Script:**
```bash
# Create the dummy conf file on your system
echo "server { listen 8080; }" > custom.conf
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get cm proxy-config -o jsonpath='{.data.custom\.conf}')" == "server { listen 8080; }" ] && echo "✅ ConfigMap created from file" || echo "❌ ConfigMap creation failed"
[ "$(kubectl get pod proxy-server -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/etc/nginx/conf.d" ] && echo "✅ Volume mounted to correct path" || echo "❌ Volume mount path failed"
kubectl exec proxy-server -- cat /etc/nginx/conf.d/custom.conf | grep -q "listen 8080" && echo "✅ File successfully injected into container" || echo "❌ File injection failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the ConfigMap from the file
kubectl create configmap proxy-config --from-file=custom.conf

# 2. Generate the base Pod YAML
kubectl run proxy-server --image=nginx --dry-run=client -o yaml > proxy.yaml

# 3. Edit the YAML to add the volumes and volumeMounts (identical to standard storage volume syntax)
vi proxy.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: proxy-server
spec:
  volumes:                              # 1. Define the volume
  - name: config-vol
    configMap:
      name: proxy-config
  containers:
  - image: nginx
    name: proxy-server
    volumeMounts:                       # 2. Mount it in the container
    - name: config-vol
      mountPath: /etc/nginx/conf.d
```
```bash
# 4. Apply the YAML
kubectl apply -f proxy.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod proxy-server
kubectl delete cm proxy-config
rm proxy.yaml custom.conf
```

---

### Task 3: The Bulk Environment Import (`envFrom`)
If a ConfigMap has 20 keys, writing 20 `valueFrom` blocks in your YAML is a nightmare. The exam will test if you know how to import the entire ConfigMap as environment variables in just two lines of code.

**1. CKAD Style Question:**
Create a ConfigMap named `app-settings` with the keys `THEME=dark` and `LOG_LEVEL=debug`.
Create a Pod named `bulk-app` using the `busybox` image running the command `sleep 3600`.
Configure the Pod to import **all** keys from the `app-settings` ConfigMap as environment variables using a single block of YAML.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get pod bulk-app -o jsonpath='{.spec.containers[0].envFrom[0].configMapRef.name}')" == "app-settings" ] && echo "✅ envFrom configured correctly" || echo "❌ envFrom failed"
kubectl exec bulk-app -- env | grep -q "THEME=dark" && echo "✅ THEME injected successfully" || echo "❌ THEME missing"
kubectl exec bulk-app -- env | grep -q "LOG_LEVEL=debug" && echo "✅ LOG_LEVEL injected successfully" || echo "❌ LOG_LEVEL missing"
```

<details>

**4. Solution:**
```bash
# 1. Create the ConfigMap imperatively
kubectl create configmap app-settings --from-literal=THEME=dark --from-literal=LOG_LEVEL=debug

# 2. Generate the base Pod YAML
kubectl run bulk-app --image=busybox --dry-run=client -o yaml -- sleep 3600 > bulk.yaml

# 3. Edit the YAML
vi bulk.yaml
```
*Instead of `env:`, you will use `envFrom:`*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bulk-app
spec:
  containers:
  - image: busybox
    name: bulk-app
    command: ["sleep", "3600"]
    envFrom:                      # ADD FROM HERE
    - configMapRef:
        name: app-settings        # TO HERE
```
```bash
# 4. Apply the YAML
kubectl apply -f bulk.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod bulk-app
kubectl delete cm app-settings
rm bulk.yaml
```

---

### The Secret Weapon
Whenever you need to remember the exact syntax for `valueFrom` or `envFrom` during the exam, don't rely purely on memory. Open your terminal and run:
`kubectl explain pod.spec.containers.env.valueFrom`

