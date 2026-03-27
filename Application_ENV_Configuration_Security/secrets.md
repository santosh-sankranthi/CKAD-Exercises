## Create and consume secrets 

In Kubernetes, **Secrets** are consumed exactly like ConfigMaps—either injected as Environment Variables or mounted as Volumes. The only major difference is how they are created and stored (they use Base64 encoding). 

The CKAD exam will test your ability to create them imperatively, mount them securely, and most importantly, **decode them** on the fly. 

---

### Task 1: The Literal Secret (Environment Variables)
Just like ConfigMaps, you will often need to pass database credentials or API keys into a container as environment variables.

**1. CKAD Style Question:**
Create a Secret named `db-credentials` in the `default` namespace containing two key-value pairs: `username=admin` and `password=supersecret`.
Create a Pod named `backend-api` using the `nginx:alpine` image.
Configure the Pod to inject these secrets as environment variables named `DB_USER` and `DB_PASS`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get secret db-credentials -o jsonpath='{.data.username}')" == "$(echo -n 'admin' | base64)" ] && echo "✅ Secret username created and encoded" || echo "❌ Secret username failed"
[ "$(kubectl get pod backend-api -o jsonpath='{.spec.containers[0].env[0].valueFrom.secretKeyRef.name}')" == "db-credentials" ] && echo "✅ Env Var references correct Secret" || echo "❌ Secret reference failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the Secret imperatively (Notice it is 'secret generic', not just 'secret')
kubectl create secret generic db-credentials --from-literal=username=admin --from-literal=password=supersecret

# 2. Generate the base Pod YAML
kubectl run backend-api --image=nginx:alpine --dry-run=client -o yaml > secret-env.yaml

# 3. Edit the YAML
vi secret-env.yaml
```
*Add the `env` block. Notice it is `secretKeyRef` instead of `configMapKeyRef`:*
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
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password                 # TO HERE
```
```bash
# 4. Apply the YAML
kubectl apply -f secret-env.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod backend-api
kubectl delete secret db-credentials
rm secret-env.yaml
```

---

### Task 2: The File-Based Secret (Volume Mounts)
For larger secrets, like TLS certificates or SSH keys, you will be asked to create a secret from a file and mount it securely into the container's file system.

**1. CKAD Style Question:**
A file named `tls.crt` exists on your node. Create a Secret named `app-tls` from this file. 
Next, create a Pod named `secure-app` using the `nginx` image. 
Mount the `app-tls` Secret as a volume so that the `tls.crt` file appears inside the container at the path `/etc/certs/tls.crt`.

**2. Setup Script:**
```bash
# Create the dummy certificate file on your system
echo "----BEGIN DUMMY CERTIFICATE----" > tls.crt
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get secret app-tls -o jsonpath='{.data.tls\.crt}')" == "$(echo -n '----BEGIN DUMMY CERTIFICATE----' | base64)" ] && echo "✅ Secret created from file" || echo "❌ Secret creation failed"
[ "$(kubectl get pod secure-app -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/etc/certs" ] && echo "✅ Volume mounted to correct path" || echo "❌ Volume mount path failed"
kubectl exec secure-app -- cat /etc/certs/tls.crt | grep -q "DUMMY CERTIFICATE" && echo "✅ File successfully injected into container" || echo "❌ File injection failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the Secret from the file
kubectl create secret generic app-tls --from-file=tls.crt

# 2. Generate the base Pod YAML
kubectl run secure-app --image=nginx --dry-run=client -o yaml > secret-vol.yaml

# 3. Edit the YAML to add the volumes and volumeMounts
vi secret-vol.yaml
```
*Add the `volumes` and `volumeMounts`. Notice it uses `secret` instead of `configMap`:*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  volumes:                              # 1. Define the volume
  - name: cert-vol
    secret:
      secretName: app-tls               # IMPORTANT: It is 'secretName', not just 'name'
  containers:
  - image: nginx
    name: secure-app
    volumeMounts:                       # 2. Mount it in the container
    - name: cert-vol
      mountPath: /etc/certs
```
```bash
# 4. Apply the YAML
kubectl apply -f secret-vol.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod secure-app
kubectl delete secret app-tls
rm secret-vol.yaml tls.crt
```

---

### Task 3: The Decoding Drill (The CKAD Curveball)
Because Secrets are only Base64 encoded (not actually encrypted by default), the exam frequently tests your ability to retrieve and decode a secret that someone else created in the cluster. 

**1. CKAD Style Question:**
A Secret named `hidden-vault` exists in the `mystery-ns` namespace. It contains a key called `access-token`. 
Retrieve the value of this token, decode it from Base64, and write the plain text value to a file at `/opt/token.txt`.

**2. Setup Script:**
```bash
kubectl create namespace mystery-ns
kubectl create secret generic hidden-vault --from-literal=access-token=SuperSecretData123! -n mystery-ns
sudo mkdir -p /opt
sudo chmod 777 /opt
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
if [ -f /opt/token.txt ]; then
    [ "$(cat /opt/token.txt)" == "SuperSecretData123!" ] && echo "✅ Secret successfully decoded and saved" || echo "❌ Decoded value is incorrect"
else
    echo "❌ File /opt/token.txt does not exist"
fi
```

<details>

**4. Solution:**
```bash
# 1. Use JSONPath to extract just the encoded value, then pipe it into the base64 decode utility.
# The '-d' flag stands for decode.
kubectl get secret hidden-vault -n mystery-ns -o jsonpath='{.data.access-token}' | base64 -d > /opt/token.txt

# Verify it worked
cat /opt/token.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete namespace mystery-ns
rm -f /opt/token.txt
```

---

If you can inject a secret as an env var, mount it as a volume, and use `base64 -d` in the terminal to read an existing one, you have entirely boxed out the "Create and consume Secrets" objective. 
