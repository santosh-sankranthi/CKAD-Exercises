## Create and Consume Secrets

In Kubernetes, Secrets are consumed exactly like ConfigMaps—either injected as Environment Variables or mounted as Volumes. The major differences are how they are stored (Base64 encoding) and specific specialized types (like Docker registry credentials).

-----

### Task 1: Environment Variable Injection (Most Repeating)

Just like ConfigMaps, you will often need to pass database credentials or API keys into a container as environment variables.

#### Main Task 1: Specific Key Injection (`valueFrom`)

**1. CKAD Style Question:**
Create a Secret named `db-credentials` in the `default` namespace containing two key-value pairs: `username=admin` and `password=supersecret`.
Create a Pod named `backend-api` using the `nginx:alpine` image.
Configure the Pod to inject these secrets as environment variables named `DB_USER` and `DB_PASS`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get secret db-credentials -o jsonpath='{.data.username}')" == "$(echo -n 'admin' | base64)" ] && echo "✅ Secret username created and encoded" || echo "❌ Secret username failed"
[ "$(kubectl get pod backend-api -o jsonpath='{.spec.containers[0].env[0].valueFrom.secretKeyRef.name}')" == "db-credentials" ] && echo "✅ Env Var references correct Secret" || echo "❌ Secret reference failed"
```

<details>

4. Solution:

```bash
# 1. Create the Secret imperatively ('secret generic', not just 'secret')
kubectl create secret generic db-credentials --from-literal=username=admin --from-literal=password=supersecret

# 2. Generate the base Pod YAML
kubectl run backend-api --image=nginx:alpine --dry-run=client -o yaml > secret-env.yaml

# 3. Edit the YAML
vi secret-env.yaml
```

*Add the `env` block. Notice it uses `secretKeyRef`:*

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
rm -f secret-env.yaml
```

#### Variation 1.1: Bulk Import (`envFrom`)

**1. CKAD Style Question:**
Create a Secret named `app-secrets` with the keys `API_TOKEN=xyz123` and `CLIENT_ID=app99`.
Create a Pod named `bulk-secret-app` using the `busybox` image running `sleep 3600`.
Configure the Pod to import **all** keys from the `app-secrets` Secret as environment variables using a single block of YAML.

**2. Setup Script:**

```bash
kubectl create secret generic app-secrets --from-literal=API_TOKEN=xyz123 --from-literal=CLIENT_ID=app99
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get pod bulk-secret-app -o jsonpath='{.spec.containers[0].envFrom[0].secretRef.name}')" == "app-secrets" ] && echo "✅ envFrom configured correctly" || echo "❌ envFrom failed"
kubectl exec bulk-secret-app -- env | grep -q "API_TOKEN=xyz123" && echo "✅ API_TOKEN injected successfully" || echo "❌ API_TOKEN missing"
```

<details>

4. Solution:

```bash
kubectl run bulk-secret-app --image=busybox --dry-run=client -o yaml -- sleep 3600 > bulk-secret.yaml
vi bulk-secret.yaml
```

*Use `envFrom` with `secretRef`:*

```yaml
spec:
  containers:
  - image: busybox
    name: bulk-secret-app
    command: ["sleep", "3600"]
    envFrom:                      # ADD FROM HERE
    - secretRef:
        name: app-secrets         # TO HERE
```

```bash
kubectl apply -f bulk-secret.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod bulk-secret-app
kubectl delete secret app-secrets
rm -f bulk-secret.yaml
```

#### Variation 1.2: Creation via `--from-env-file`

**1. CKAD Style Question:**
A file named `auth.env` contains several secret environment variables.
Create a Secret named `file-secrets` using the `--from-env-file` flag so that each line in the file becomes a discrete key in the Secret.

**2. Setup Script:**

```bash
echo -e "USER=test\nPASS=pass123" > auth.env
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get secret file-secrets -o jsonpath='{.data.USER}')" == "$(echo -n 'test' | base64)" ] && echo "✅ File parsed correctly as discrete keys" || echo "❌ env-file parsing failed"
```

<details>

4. Solution:

```bash
# Just like ConfigMaps, --from-env-file parses the file line-by-line.
# --from-file would create a single key named "auth.env".
kubectl create secret generic file-secrets --from-env-file=auth.env
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete secret file-secrets
rm -f auth.env
```

-----

### Task 2: File-Based Secrets (Volume Mounts)

For larger secrets, like TLS certificates or SSH keys, you will be asked to create a secret from a file and mount it securely into the container's file system.

#### Main Task 2: Standard Directory Mount

**1. CKAD Style Question:**
A file named `tls.crt` exists on your node. Create a Secret named `app-tls` from this file.
Next, create a Pod named `secure-app` using the `nginx` image.
Mount the `app-tls` Secret as a volume so that the `tls.crt` file appears inside the container at the path `/etc/certs/tls.crt`.

**2. Setup Script:**

```bash
echo "----BEGIN DUMMY CERTIFICATE----" > tls.crt
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get secret app-tls -o jsonpath='{.data.tls\.crt}')" == "$(echo -n '----BEGIN DUMMY CERTIFICATE----' | base64)" ] && echo "✅ Secret created from file" || echo "❌ Secret creation failed"
[ "$(kubectl get pod secure-app -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/etc/certs" ] && echo "✅ Volume mounted to correct path" || echo "❌ Volume mount path failed"
kubectl exec secure-app -- cat /etc/certs/tls.crt | grep -q "DUMMY CERTIFICATE" && echo "✅ File successfully injected into container" || echo "❌ File injection failed"
```

<details>

4. Solution:

```bash
# 1. Create the Secret from the file
kubectl create secret generic app-tls --from-file=tls.crt

# 2. Generate the base Pod YAML
kubectl run secure-app --image=nginx --dry-run=client -o yaml > secret-vol.yaml

# 3. Edit the YAML to add the volumes and volumeMounts
vi secret-vol.yaml
```

*Notice it uses `secret` instead of `configMap`:*

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
kubectl apply -f secret-vol.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod secure-app
kubectl delete secret app-tls
rm -f secret-vol.yaml tls.crt
```

#### Variation 2.1: Specific Key Mapping & Permissions (`items` & `defaultMode`)

**1. CKAD Style Question:**
A Secret named `ssh-key-secret` exists containing a key named `private-key`.
Create a Pod named `ssh-pod` using `busybox`. Mount the secret as a volume at `/root/.ssh`.
You must explicitly project ONLY the `private-key` into the directory, rename the file to `id_rsa`, and restrict the file permissions to `0400` (read-only by owner).

**2. Setup Script:**

```bash
kubectl create secret generic ssh-key-secret --from-literal=private-key="RSA-DATA-HERE"
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get pod ssh-pod -o jsonpath='{.spec.volumes[0].secret.items[0].key}')" == "private-key" ] && echo "✅ Specific key targeted" || echo "❌ Key targeting failed"
[ "$(kubectl get pod ssh-pod -o jsonpath='{.spec.volumes[0].secret.items[0].path}')" == "id_rsa" ] && echo "✅ File renamed to id_rsa" || echo "❌ Rename failed"
[ "$(kubectl get pod ssh-pod -o jsonpath='{.spec.volumes[0].secret.defaultMode}')" == "256" ] && echo "✅ Permissions set to 0400 (decimal 256)" || echo "❌ Permissions failed"
```

<details>

4. Solution:

```bash
kubectl run ssh-pod --image=busybox --dry-run=client -o yaml -- sleep 3600 > ssh.yaml
vi ssh.yaml
```

*Combine `defaultMode` (in octal) and `items` under the secret volume definition:*

```yaml
spec:
  volumes:
  - name: ssh-vol
    secret:
      secretName: ssh-key-secret
      defaultMode: 0400             # ADD Permissions
      items:                        # ADD Key Mapping
      - key: private-key
        path: id_rsa
  containers:
  - image: busybox
    name: ssh-pod
    command: ["sleep", "3600"]
    volumeMounts:
    - name: ssh-vol
      mountPath: /root/.ssh
```

```bash
kubectl apply -f ssh.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod ssh-pod
kubectl delete secret ssh-key-secret
rm -f ssh.yaml
```

-----

### Task 3: Decoding & Declarative Creation (The CKAD Tricks)

Because Secrets are only Base64 encoded (not encrypted), the exam tests your ability to retrieve/decode them, and your ability to write them declaratively without having to manually encode strings in your terminal.

#### Main Task 3: The Decoding Drill

**1. CKAD Style Question:**
A Secret named `hidden-vault` exists in the `mystery-ns` namespace. It contains a key called `access-token`.
Retrieve the value of this token, decode it from Base64, and write the plain text value to a file at `/opt/token.txt`.

**2. Setup Script:**

```bash
kubectl create namespace mystery-ns
kubectl create secret generic hidden-vault --from-literal=access-token=SuperSecretData123! -n mystery-ns
sudo mkdir -p /opt && sudo chmod 777 /opt
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
if [ -f /opt/token.txt ]; then
    [ "$(cat /opt/token.txt)" == "SuperSecretData123!" ] && echo "✅ Secret successfully decoded and saved" || echo "❌ Decoded value is incorrect"
else
    echo "❌ File /opt/token.txt does not exist"
fi
```

<details>

4. Solution:

```bash
# 1. Use JSONPath to extract just the encoded value, then pipe it into base64 decode.
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

#### Variation 3.1: The `stringData` Declarative Trick

**1. CKAD Style Question:**
You are asked to create a Secret declaratively using a YAML manifest file at `/opt/new-secret.yaml`.
The Secret must be named `yaml-secret` and contain the key `api-key` with the value `123456789`.
*Constraint:* You are NOT allowed to use the `base64` command in your terminal to encode the value before placing it in the YAML file. You must provide the plain text string directly in the YAML, and Kubernetes must encode it automatically.

**2. Setup Script:**

```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get secret yaml-secret -o jsonpath='{.data.api-key}')" == "$(echo -n '123456789' | base64)" ] && echo "✅ Secret successfully created and auto-encoded" || echo "❌ Secret missing or encoded incorrectly"
grep -q "stringData" /opt/new-secret.yaml && echo "✅ Used stringData correctly" || echo "❌ FAILED: You did not use stringData"
```

<details>

4. Solution:

```bash
# By substituting 'data:' with 'stringData:', Kubernetes allows you to type plain text.
# It automatically converts it to base64 upon creation!
vi /opt/new-secret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: yaml-secret
stringData:                 # Use stringData instead of data!
  api-key: "123456789"
```

```bash
kubectl apply -f /opt/new-secret.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete secret yaml-secret
rm -f /opt/new-secret.yaml
```

-----

### Task 4: Private Registry Access (`ImagePullSecrets`)

If you try to pull a container image from a private registry (like a private Docker Hub repo or an AWS ECR), Kubernetes will fail unless you provide a specific type of Secret containing the login credentials.

#### Main Task 4: Creating and Attaching Registry Secrets

**1. CKAD Style Question:**
Create a Secret named `registry-auth` of type `docker-registry`.
Use the following dummy credentials:

  * server: `https://index.docker.io/v1/`
  * username: `myuser`
  * password: `mypassword`
  * email: `myuser@example.com`

Next, create a Pod named `private-pod` using the `myuser/private-app:v1` image. Ensure the Pod is configured to use the `registry-auth` secret to pull the image. *(Note: The pod will fail to start because the image is fake, but the configuration must be correct).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 4 ---"
[ "$(kubectl get secret registry-auth -o jsonpath='{.type}')" == "kubernetes.io/dockerconfigjson" ] && echo "✅ Secret created with correct type" || echo "❌ Secret type is incorrect"
[ "$(kubectl get pod private-pod -o jsonpath='{.spec.imagePullSecrets[0].name}')" == "registry-auth" ] && echo "✅ imagePullSecrets configured on Pod" || echo "❌ imagePullSecrets missing"
```

<details>

4. Solution:

```bash
# 1. Create the specialized docker-registry secret
kubectl create secret docker-registry registry-auth \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myuser@example.com

# 2. Generate the base Pod YAML
kubectl run private-pod --image=myuser/private-app:v1 --dry-run=client -o yaml > private.yaml

# 3. Edit the YAML
vi private.yaml
```

*Add the `imagePullSecrets` array under the `spec` (same level as `containers`):*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:             # ADD FROM HERE
  - name: registry-auth         # TO HERE
  containers:
  - image: myuser/private-app:v1
    name: private-pod
```

```bash
kubectl apply -f private.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod private-pod
kubectl delete secret registry-auth
rm -f private.yaml
```

#### Variation 4.1: Attaching to a ServiceAccount

**1. CKAD Style Question:**
Instead of adding `imagePullSecrets` to every single Pod individually, you can attach the registry secret directly to a ServiceAccount.
Create a ServiceAccount named `deployer-sa`. Attach an existing secret named `corp-registry-key` to this ServiceAccount as an `imagePullSecret`.

**2. Setup Script:**

```bash
# Create a dummy secret
kubectl create secret generic corp-registry-key
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 4.1 ---"
[ "$(kubectl get sa deployer-sa -o jsonpath='{.imagePullSecrets[0].name}')" == "corp-registry-key" ] && echo "✅ imagePullSecret attached to ServiceAccount" || echo "❌ ServiceAccount attachment failed"
```

<details>

4. Solution:

```bash
# 1. Create the ServiceAccount
kubectl create sa deployer-sa

# 2. Patch the ServiceAccount to include the imagePullSecret
kubectl patch sa deployer-sa -p '{"imagePullSecrets": [{"name": "corp-registry-key"}]}'

# Alternatively, you can use `kubectl edit sa deployer-sa`
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete sa deployer-sa
kubectl delete secret corp-registry-key
```

-----

#### Variation 4.2: TLS Secrets (The Ingress Prerequisite)

**1. CKAD Style Question:**
You have been provided with a public certificate file (`cert.pem`) and a private key file (`key.pem`).
Create a specialized Secret named `web-tls` that is formatted correctly to be consumed by an Ingress controller for HTTPS termination.

**2. Setup Script:**

```bash
# Generate dummy certificate and key files
echo "----BEGIN CERTIFICATE----" > cert.pem
echo "----BEGIN PRIVATE KEY----" > key.pem
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 4.2 ---"
[ "$(kubectl get secret web-tls -o jsonpath='{.type}')" == "kubernetes.io/tls" ] && echo "✅ Secret created with correct TLS type" || echo "❌ Secret type is incorrect (Did you use 'generic'?)"
[ "$(kubectl get secret web-tls -o jsonpath='{.data.tls\.crt}')" == "$(echo -n '----BEGIN CERTIFICATE----' | base64)" ] && echo "✅ Certificate data mapped correctly" || echo "❌ Certificate mapping failed"
[ "$(kubectl get secret web-tls -o jsonpath='{.data.tls\.key}')" == "$(echo -n '----BEGIN PRIVATE KEY----' | base64)" ] && echo "✅ Private key data mapped correctly" || echo "❌ Private key mapping failed"
```

<details>
  
4. Solution:

```bash
# You MUST use 'secret tls', not 'secret generic'! 
# It requires the exact flags --cert and --key.
kubectl create secret tls web-tls --cert=cert.pem --key=key.pem
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete secret web-tls
rm -f cert.pem key.pem
```

