## Combined concepts based tasks.
---

### Task 1: The "Fort Knox" Pod (The Kitchen Sink Overlap)
This is the ultimate test of your YAML structure. You must combine a ServiceAccount, a ConfigMap (as environment variables), a Secret (as a volume), and a SecurityContext all into a single Pod definition. If your indentation is off by two spaces anywhere, the whole thing fails.

**1. CKAD Style Question:**
A ServiceAccount (`app-sa`), a ConfigMap (`app-config`), and a Secret (`app-vault`) already exist in the `default` namespace.
Create a Pod named `secure-processor` using the `nginx:alpine` image.
1. It must run under the `app-sa` ServiceAccount.
2. It must run as User ID `1000` (Pod-level security context).
3. It must import all keys from the `app-config` ConfigMap as environment variables.
4. It must mount the `app-vault` Secret as a volume at the path `/etc/secrets`.

**2. Setup Script:**
```bash
# Run this to build the prerequisites 
kubectl create serviceaccount app-sa
kubectl create configmap app-config --from-literal=ENV=production --from-literal=REGION=us-east
kubectl create secret generic app-vault --from-literal=api-key=12345ABCDE
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get pod secure-processor -o jsonpath='{.spec.serviceAccountName}')" == "app-sa" ] && echo "✅ ServiceAccount attached" || echo "❌ ServiceAccount failed"
[ "$(kubectl get pod secure-processor -o jsonpath='{.spec.securityContext.runAsUser}')" == "1000" ] && echo "✅ SecurityContext configured" || echo "❌ SecurityContext failed"
[ "$(kubectl get pod secure-processor -o jsonpath='{.spec.containers[0].envFrom[0].configMapRef.name}')" == "app-config" ] && echo "✅ ConfigMap envFrom configured" || echo "❌ ConfigMap import failed"
[ "$(kubectl get pod secure-processor -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/etc/secrets" ] && echo "✅ Secret volume mounted" || echo "❌ Secret mount failed"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base Pod YAML
kubectl run secure-processor --image=nginx:alpine --dry-run=client -o yaml > fort-knox.yaml

# 2. Edit the YAML carefully. Pay strict attention to where each block goes!
vi fort-knox.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-processor
spec:
  serviceAccountName: app-sa      # 1. ServiceAccount
  securityContext:                # 2. Pod-Level Security
    runAsUser: 1000
  volumes:                        # 3a. Define the Secret Volume
  - name: vault-vol
    secret:
      secretName: app-vault
  containers:
  - image: nginx:alpine
    name: secure-processor
    envFrom:                      # 4. Import ConfigMap as Env Vars
    - configMapRef:
        name: app-config
    volumeMounts:                 # 3b. Mount the Secret Volume
    - name: vault-vol
      mountPath: /etc/secrets
```
```bash
# 3. Apply the YAML
kubectl apply -f fort-knox.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod secure-processor
kubectl delete sa app-sa
kubectl delete cm app-config
kubectl delete secret app-vault
rm fort-knox.yaml
```

---

### Task 2: The Config Update Cycle (The Restart Pattern)
Here is a massive real-world and exam "gotcha": If you inject a ConfigMap into a Pod as an *environment variable*, and then you update the ConfigMap, **the Pod does not automatically pick up the new value**. You have to know how to force a Deployment to pull the fresh configuration.

**1. CKAD Style Question:**
A Deployment named `api-gateway` is currently running and pulling environment variables from a ConfigMap named `gateway-config`.
1. Update the `gateway-config` ConfigMap so that the key `TIMEOUT` is changed from `30s` to `60s`.
2. Ensure the running `api-gateway` Pods immediately reflect this new environment variable by triggering a rollout.

**2. Setup Script:**
```bash
kubectl create configmap gateway-config --from-literal=TIMEOUT=30s
kubectl create deployment api-gateway --image=nginx:alpine --dry-run=client -o yaml > setup-deploy.yaml
# Injecting the ConfigMap into the Deployment
sed -i '/image: nginx:alpine/a \        envFrom:\n        - configMapRef:\n            name: gateway-config' setup-deploy.yaml
kubectl apply -f setup-deploy.yaml
rm setup-deploy.yaml
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get cm gateway-config -o jsonpath='{.data.TIMEOUT}')" == "60s" ] && echo "✅ ConfigMap updated" || echo "❌ ConfigMap update failed"
# Get the newest pod created by the rollout
NEW_POD=$(kubectl get pods -l app=api-gateway --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
kubectl exec $NEW_POD -- env | grep -q "TIMEOUT=60s" && echo "✅ Deployment rolled out with new config!" || echo "❌ Pods still have the old config"
```

<details>

**4. Solution:**
```bash
# 1. Update the ConfigMap. The fastest way is to recreate it using the imperative command and piping it to apply.
kubectl create configmap gateway-config --from-literal=TIMEOUT=60s --dry-run=client -o yaml | kubectl apply -f -

# 2. Force the Deployment to restart its Pods so they read the new environment variables
kubectl rollout restart deployment/api-gateway

# 3. (Optional) Watch the rollout happen
kubectl rollout status deployment/api-gateway
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy api-gateway
kubectl delete cm gateway-config
```

---

### The Verdict
If you can build the "Fort Knox" Pod without getting lost in the YAML hierarchy, and you remember to use `kubectl rollout restart` when configuration data changes, you have totally boxed out this 20% chunk of the syllabus. 
