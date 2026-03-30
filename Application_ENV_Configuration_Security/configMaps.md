## Understand ConfigMaps

ConfigMaps are the absolute backbone of the "Configuration" domain on the CKAD. The exam tests this topic to ensure you know how to decouple configuration artifacts (like database URLs, environment variables, or `.conf` files) from your container images.

-----

### Task 1: Environment Variable Injection (Most Repeating)

The exam will often ask you to create a ConfigMap imperatively and then selectively pluck specific keys out of it to serve as environment variables, or bulk-import the entire thing.

#### Main Task 1: Specific Key Injection (`valueFrom`)

**1. CKAD Style Question:**
Create a ConfigMap named `db-config` in the `default` namespace with two key-value pairs: `DB_HOST=cluster-db` and `DB_PORT=5432`.
Create a Pod named `backend-api` using the `nginx:alpine` image.
Configure the Pod to create an environment variable named `DATABASE_URL` that pulls its value directly from the `DB_HOST` key in the `db-config` ConfigMap.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get cm db-config -o jsonpath='{.data.DB_HOST}')" == "cluster-db" ] && echo "✅ ConfigMap DB_HOST created" || echo "❌ ConfigMap missing or incorrect"
[ "$(kubectl get pod backend-api -o jsonpath='{.spec.containers[0].env[0].name}')" == "DATABASE_URL" ] && echo "✅ Env Var name correct" || echo "❌ Env Var name failed"
[ "$(kubectl get pod backend-api -o jsonpath='{.spec.containers[0].env[0].valueFrom.configMapKeyRef.key}')" == "DB_HOST" ] && echo "✅ Env Var references correct ConfigMap key" || echo "❌ ConfigMap reference failed"
```

<details>
  
4. Solution:

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
rm -f backend.yaml
```

#### Variation 1.1: The Bulk Import (`envFrom`)

**1. CKAD Style Question:**
Create a ConfigMap named `app-settings` with the keys `THEME=dark` and `LOG_LEVEL=debug`.
Create a Pod named `bulk-app` using the `busybox` image running the command `sleep 3600`.
Configure the Pod to import **all** keys from the `app-settings` ConfigMap as environment variables using a single block of YAML.

**2. Setup Script:**

```bash
kubectl create configmap app-settings --from-literal=THEME=dark --from-literal=LOG_LEVEL=debug
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get pod bulk-app -o jsonpath='{.spec.containers[0].envFrom[0].configMapRef.name}')" == "app-settings" ] && echo "✅ envFrom configured correctly" || echo "❌ envFrom failed"
kubectl exec bulk-app -- env | grep -q "THEME=dark" && echo "✅ THEME injected successfully" || echo "❌ THEME missing"
```

<details>
  
4. Solution:

```bash
kubectl run bulk-app --image=busybox --dry-run=client -o yaml -- sleep 3600 > bulk.yaml
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
kubectl apply -f bulk.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod bulk-app
kubectl delete cm app-settings
rm -f bulk.yaml
```

#### Variation 1.2: Bulk Import with Prefix

**1. CKAD Style Question:**
Modify the requirements for the `bulk-app`: Import all variables from `app-settings` using `envFrom`, but automatically append the prefix `CONFIG_` to every variable name inside the container (e.g., it becomes `CONFIG_THEME=dark`).

**2. Setup Script:**

```bash
kubectl create configmap app-settings --from-literal=THEME=dark --from-literal=LOG_LEVEL=debug
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get pod prefix-app -o jsonpath='{.spec.containers[0].envFrom[0].prefix}')" == "CONFIG_" ] && echo "✅ Prefix configured correctly" || echo "❌ Prefix failed"
kubectl exec prefix-app -- env | grep -q "CONFIG_THEME=dark" && echo "✅ Prefixed variable verified live" || echo "❌ Live prefix missing"
```

<details>

4. Solution:

```bash
kubectl run prefix-app --image=busybox --dry-run=client -o yaml -- sleep 3600 > prefix.yaml
vi prefix.yaml
```

*Add the `prefix` key inside the `envFrom` array item:*

```yaml
    envFrom:
    - prefix: CONFIG_             # ADD THIS PREFIX
      configMapRef:
        name: app-settings
```

```bash
kubectl apply -f prefix.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod prefix-app
kubectl delete cm app-settings
rm -f prefix.yaml
```

-----

### Task 2: Mounting a ConfigMap as a Volume (Highly Repeating)

Instead of environment variables, the exam might hand you a configuration file and ask you to mount it directly into the container's file system using a ConfigMap.

#### Main Task 2: Standard Directory Mount

**1. CKAD Style Question:**
A file named `custom.conf` exists on your node. Create a ConfigMap named `proxy-config` from this file.
Next, create a Pod named `proxy-server` using the `nginx` image.
Mount the `proxy-config` ConfigMap as a volume so that the `custom.conf` file appears inside the container at the path `/etc/nginx/conf.d/custom.conf`.

**2. Setup Script:**

```bash
echo "server { listen 8080; }" > custom.conf
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get pod proxy-server -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/etc/nginx/conf.d" ] && echo "✅ Volume mounted to correct path" || echo "❌ Volume mount path failed"
kubectl exec proxy-server -- cat /etc/nginx/conf.d/custom.conf | grep -q "listen 8080" && echo "✅ File successfully injected into container" || echo "❌ File injection failed"
```

<details>

4. Solution:

```bash
# 1. Create the ConfigMap from the file
kubectl create configmap proxy-config --from-file=custom.conf

# 2. Generate the base Pod YAML
kubectl run proxy-server --image=nginx --dry-run=client -o yaml > proxy.yaml

# 3. Edit the YAML to add volumes and volumeMounts
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
kubectl apply -f proxy.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod proxy-server
kubectl delete cm proxy-config
rm -f proxy.yaml custom.conf
```

#### Variation 2.1: Mapping Specific Keys (`items`)

**1. CKAD Style Question:**
A ConfigMap named `multi-config` exists with two keys: `nginx.conf` and `cache.conf`.
Create a Pod named `strict-mount` using `nginx`. Mount the ConfigMap as a volume at `/etc/nginx/conf.d`, but **ONLY** project the `nginx.conf` key into the directory, and rename it to `app.conf` upon mounting.

**2. Setup Script:**

```bash
kubectl create configmap multi-config --from-literal=nginx.conf="listen 80;" --from-literal=cache.conf="size 50;"
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get pod strict-mount -o jsonpath='{.spec.volumes[0].configMap.items[0].key}')" == "nginx.conf" ] && echo "✅ Specific key targeted" || echo "❌ Key targeting failed"
[ "$(kubectl get pod strict-mount -o jsonpath='{.spec.volumes[0].configMap.items[0].path}')" == "app.conf" ] && echo "✅ Path renamed successfully" || echo "❌ Path rename failed"
```

<details>

4. Solution:

```bash
kubectl run strict-mount --image=nginx --dry-run=client -o yaml > strict.yaml
vi strict.yaml
```

*Use the `items` array under the volume definition to specify exact mappings:*

```yaml
spec:
  volumes:
  - name: config-vol
    configMap:
      name: multi-config
      items:                      # ADD FROM HERE
      - key: nginx.conf
        path: app.conf            # TO HERE
  containers:
  - image: nginx
    name: strict-mount
    volumeMounts:
    - name: config-vol
      mountPath: /etc/nginx/conf.d
```

```bash
kubectl apply -f strict.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod strict-mount
kubectl delete cm multi-config
rm -f strict.yaml
```

#### Variation 2.2: Setting File Permissions (`defaultMode`)

**1. CKAD Style Question:**
Modify a volume mount so that the configuration files projected from the `auth-config` ConfigMap are mounted with strict read-only permissions for the owner only (`0400`).

**2. Setup Script:**

```bash
kubectl create configmap auth-config --from-literal=keys.txt="secret-data"
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.2 ---"
[ "$(kubectl get pod perm-mount -o jsonpath='{.spec.volumes[0].configMap.defaultMode}')" == "256" ] && echo "✅ Permissions set successfully (256 is decimal for octal 0400)" || echo "❌ Permissions failed"
```

<details>

4. Solution:

```bash
kubectl run perm-mount --image=nginx --dry-run=client -o yaml > perm.yaml
vi perm.yaml
```

*Add `defaultMode` to the volume definition. Note: You type it in octal (0400), but Kubernetes converts and stores it internally as decimal (256).*

```yaml
spec:
  volumes:
  - name: config-vol
    configMap:
      name: auth-config
      defaultMode: 0400           # ADD THIS LINE
  containers:
  - image: nginx
    name: perm-mount
    volumeMounts:
    - name: config-vol
      mountPath: /etc/auth
```

```bash
kubectl apply -f perm.yaml
```

</details>
**5. Clean-up Script:**

```bash
kubectl delete pod perm-mount
kubectl delete cm auth-config
rm -f perm.yaml
```

-----

### Task 3: Advanced Creation & Troubleshooting (Moderately Repeating)

The exam will test your ability to ingest multiple files and handle missing configurations gracefully.

#### Main Task 3: Creating from Directories and Env-Files

**1. CKAD Style Question:**
You have a `.env` file containing environment variables (e.g., `PORT=8080`).
Create a ConfigMap named `env-config` using the `--from-env-file` flag (which treats each line as a discrete key/value pair, rather than creating one giant key containing the whole file).

**2. Setup Script:**

```bash
echo -e "PORT=8080\nMODE=prod" > settings.env
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(kubectl get cm env-config -o jsonpath='{.data.PORT}')" == "8080" ] && echo "✅ File parsed correctly as env vars" || echo "❌ env-file parsing failed"
kubectl get cm env-config -o jsonpath='{.data.settings\.env}' 2>&1 | grep -q "not found" || echo "❌ FAILED: You used --from-file instead of --from-env-file!"
```

<details>
  
4. Solution:

```bash
# This is a massive exam trap! 
# --from-file creates a single key named "settings.env".
# --from-env-file reads the file line-by-line and creates multiple keys (PORT, MODE).
kubectl create configmap env-config --from-env-file=settings.env
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete cm env-config
rm -f settings.env
```

#### Variation 3.1: The `optional` Flag

**1. CKAD Style Question:**
A Pod named `flex-app` uses `envFrom` to import a ConfigMap named `future-config`.
However, this ConfigMap does not exist yet\! If you deploy the Pod, it will crash. Update the Pod YAML so that importing the ConfigMap is **optional**, allowing the Pod to start cleanly even if the ConfigMap is missing.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get pod flex-app -o jsonpath='{.spec.containers[0].envFrom[0].configMapRef.optional}')" == "true" ] && echo "✅ Optional flag configured correctly" || echo "❌ Optional flag missing"
[ "$(kubectl get pod flex-app -o jsonpath='{.status.phase}')" == "Running" ] && echo "✅ Pod is running successfully without the ConfigMap" || echo "❌ Pod crashed or failed to start"
```

<details>

4. Solution:

```bash
kubectl run flex-app --image=busybox --dry-run=client -o yaml -- sleep 3600 > flex.yaml
vi flex.yaml
```

*Add the `optional: true` parameter inside the configMapRef:*

```yaml
spec:
  containers:
  - image: busybox
    name: flex-app
    command: ["sleep", "3600"]
    envFrom:
    - configMapRef:
        name: future-config
        optional: true            # ADD THIS LINE
```

```bash
kubectl apply -f flex.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod flex-app
rm -f flex.yaml
```
----


#### Variation 3.2: The Immutable ConfigMap Trap

**1. CKAD Style Question:**
Create a ConfigMap named `locked-config` containing the key-value pair `environment=production`.
Configure this ConfigMap so that its data cannot be accidentally updated or altered by other users. (You must make the ConfigMap immutable).

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.2 ---"
[ "$(kubectl get cm locked-config -o jsonpath='{.immutable}')" == "true" ] && echo "✅ ConfigMap successfully set to immutable" || echo "❌ Immutable flag missing or false"
# Try to break it to prove it works
kubectl patch cm locked-config -p '{"data":{"environment":"staging"}}' 2>&1 | grep -q "Forbidden" && echo "✅ Verified: API server rejected the modification" || echo "❌ FAILED: ConfigMap allowed modification"
```

<details>

4. Solution:

```bash
# There is no imperative flag for '--immutable', so you must generate the YAML and edit it!
kubectl create configmap locked-config --from-literal=environment=production --dry-run=client -o yaml > immutable.yaml

vi immutable.yaml
```

*Add the `immutable: true` flag at the root level of the YAML (at the same indentation as `apiVersion` and `kind`):*

```yaml
apiVersion: v1
data:
  environment: production
kind: ConfigMap
metadata:
  name: locked-config
immutable: true             # ADD THIS EXACT LINE
```

```bash
kubectl apply -f immutable.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete cm locked-config
rm -f immutable.yaml
```

