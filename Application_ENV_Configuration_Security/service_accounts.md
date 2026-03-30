
## Understand Service Accounts

On the CKAD, "Configure Service Accounts" is fundamentally about identity management for your Pods. If you don't explicitly assign one, Kubernetes assigns the `default` ServiceAccount, which grants identical baseline permissions to everything in the namespace (a major security risk).

-----

### Task 1: The Standard Assignment (Most Repeating)

This is the bread and butter of the topic. You must create a ServiceAccount and explicitly tell a workload to use it instead of the default one.

#### Main Task 1: Single Pod Assignment

**1. CKAD Style Question:**
Create a new namespace named `app-auth`.
Inside this namespace, create a ServiceAccount named `backend-sa`.
Finally, create a Pod named `backend-api` using the `nginx:alpine` image in this namespace, and configure it to run using the `backend-sa` ServiceAccount.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get sa backend-sa -n app-auth -o jsonpath='{.metadata.name}')" == "backend-sa" ] && echo "✅ ServiceAccount created" || echo "❌ ServiceAccount missing"
[ "$(kubectl get pod backend-api -n app-auth -o jsonpath='{.spec.serviceAccountName}')" == "backend-sa" ] && echo "✅ Pod is using correct ServiceAccount" || echo "❌ Pod ServiceAccount failed"
```

<details>

4. Solution:

```bash
# 1. Create the namespace and ServiceAccount imperatively
kubectl create namespace app-auth
kubectl create serviceaccount backend-sa -n app-auth

# 2. Generate the base Pod YAML
kubectl run backend-api --image=nginx:alpine -n app-auth --dry-run=client -o yaml > sa-pod.yaml

# 3. Edit the YAML
vi sa-pod.yaml
```

*Add the `serviceAccountName` under the pod's `spec` block:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend-api
  namespace: app-auth
spec:
  serviceAccountName: backend-sa    # ADD THIS LINE
  containers:
  - image: nginx:alpine
    name: backend-api
```

```bash
# 4. Apply the YAML
kubectl apply -f sa-pod.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete namespace app-auth
rm -f sa-pod.yaml
```

#### Variation 1.1: Nested Workload Assignment

**1. CKAD Style Question:**
Create a ServiceAccount named `batch-executor` in the `default` namespace.
Create a CronJob named `nightly-job` that runs the `busybox` image with the command `echo "Done"` on the schedule `0 0 * * *`.
Configure the CronJob so that the pods it spawns run as the `batch-executor` ServiceAccount.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get cronjob nightly-job -o jsonpath='{.spec.jobTemplate.spec.template.spec.serviceAccountName}')" == "batch-executor" ] && echo "✅ Nested ServiceAccount correctly assigned" || echo "❌ Nested ServiceAccount missing or incorrect"
```

<details>

4. Solution:

```bash
# 1. Create the ServiceAccount
kubectl create serviceaccount batch-executor

# 2. Generate the base CronJob YAML
kubectl create cronjob nightly-job --image=busybox --schedule="0 0 * * *" --dry-run=client -o yaml -- echo "Done" > batch-job.yaml

# 3. Edit the YAML
vi batch-job.yaml
```

*You must place the `serviceAccountName` in the deepest `spec` block (the Pod template), NOT the Job or CronJob spec\!*

```yaml
spec:
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: batch-executor    # ADD HERE
          containers:
          - image: busybox
```

```bash
# 4. Apply
kubectl apply -f batch-job.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete cronjob nightly-job
kubectl delete sa batch-executor
rm -f batch-job.yaml
```

#### Variation 1.2: The Cross-Namespace Trap

**1. CKAD Style Question:**
A Pod named `isolated-worker` needs to run in the `finance` namespace, but it has been instructed to use a ServiceAccount named `global-reader` located in the `default` namespace.
*(Hint: This is impossible. ServiceAccounts are strictly namespace-bound).*
Fix the architecture: Create the `finance` namespace, create the necessary `global-reader` ServiceAccount *inside* it, and deploy the `isolated-worker` Pod (using `nginx`) correctly bound to it.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get sa global-reader -n finance -o jsonpath='{.metadata.name}')" == "global-reader" ] && echo "✅ SA created in correct namespace" || echo "❌ SA missing in finance namespace"
[ "$(kubectl get pod isolated-worker -n finance -o jsonpath='{.spec.serviceAccountName}')" == "global-reader" ] && echo "✅ Pod bound correctly" || echo "❌ Pod binding failed"
```

<details>
  
4. Solution:

```bash
# The trap is realizing you cannot reference a ServiceAccount across namespaces. You must create a duplicate in the target namespace.
kubectl create ns finance
kubectl create sa global-reader -n finance

# Create the pod directly without YAML generation using imperative overrides (Advanced trick):
kubectl run isolated-worker --image=nginx -n finance --overrides='{"spec":{"serviceAccountName":"global-reader"}}'
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ns finance
```

-----

### Task 2: Live Injection and Updates (Highly Repeating)

The exam loves to test if you know how to update a running workload. They will give you an existing object and ask you to swap out its ServiceAccount or attach credentials to the SA itself.

#### Main Task 2: The Imperative Deployment Swap

**1. CKAD Style Question:**
A Deployment named `payment-processor` exists in the `default` namespace. It is currently using the `default` ServiceAccount.
Create a new ServiceAccount named `stripe-sa`.
Update the `payment-processor` Deployment to use the `stripe-sa` ServiceAccount, triggering a rolling update of the Pods.

**2. Setup Script:**

```bash
kubectl create deployment payment-processor --image=busybox -- sleep 3600
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get deploy payment-processor -o jsonpath='{.spec.template.spec.serviceAccountName}')" == "stripe-sa" ] && echo "✅ Deployment updated to use stripe-sa" || echo "❌ Deployment ServiceAccount failed"
POD_SA=$(kubectl get pods -l app=payment-processor -o jsonpath='{.items[0].spec.serviceAccountName}')
[ "$POD_SA" == "stripe-sa" ] && echo "✅ Pods rolled out with new ServiceAccount" || echo "❌ Pod rollout failed"
```

<details>

4. Solution:

```bash
# 1. Create the new ServiceAccount
kubectl create serviceaccount stripe-sa

# 2. Use the 'set serviceaccount' imperative command to instantly patch the deployment
kubectl set serviceaccount deployment/payment-processor stripe-sa
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy payment-processor
kubectl delete sa stripe-sa
```

#### Variation 2.1: Attaching Registry Credentials (`imagePullSecrets`)

**1. CKAD Style Question:**
Instead of adding `imagePullSecrets` to every single Pod individually, you can attach the registry secret directly to the identity.
Create a ServiceAccount named `deployer-sa`. Attach an existing secret named `corp-registry-key` to this ServiceAccount so that any Pod using `deployer-sa` automatically inherits the registry credentials.

**2. Setup Script:**

```bash
kubectl create secret generic corp-registry-key
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get sa deployer-sa -o jsonpath='{.imagePullSecrets[0].name}')" == "corp-registry-key" ] && echo "✅ imagePullSecret attached to ServiceAccount" || echo "❌ ServiceAccount attachment failed"
```

<details>

4. Solution:

```bash
# 1. Create the ServiceAccount
kubectl create sa deployer-sa

# 2. Patch the ServiceAccount to include the imagePullSecret array
kubectl patch sa deployer-sa -p '{"imagePullSecrets": [{"name": "corp-registry-key"}]}'

# (Alternatively, you can use `kubectl edit sa deployer-sa`)
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete sa deployer-sa
kubectl delete secret corp-registry-key
```

#### Variation 2.2: Global Automount Shielding

**1. CKAD Style Question:**
Create a ServiceAccount named `paranoid-sa`. Modify the ServiceAccount object itself so that Kubernetes will **not** automatically mount an API token into any Pod that uses this identity.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.2 ---"
[ "$(kubectl get serviceaccount paranoid-sa -o jsonpath='{.automountServiceAccountToken}')" == "false" ] && echo "✅ Token automount disabled globally on the ServiceAccount" || echo "❌ ServiceAccount configuration failed"
```

<details>

4. Solution:

```bash
# 1. Create the ServiceAccount
kubectl create serviceaccount paranoid-sa

# 2. Patch the ServiceAccount to disable automounting
kubectl patch serviceaccount paranoid-sa -p '{"automountServiceAccountToken": false}'
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete serviceaccount paranoid-sa
```

-----

### Task 3: The Token Generator Lifecycle (The v1.24+ Curveball)

Historically, whenever you created a ServiceAccount, Kubernetes automatically generated an infinite-lifespan Secret containing a bearer token. **As of Kubernetes v1.24, this no longer happens.** You must know how to generate tokens manually via the API or via legacy Secrets.

#### Main Task 3: The Legacy Long-Lived Secret

**1. CKAD Style Question:**
Create a ServiceAccount named `ci-cd-bot` in the `default` namespace.
Because a legacy CI/CD pipeline requires a permanent API token that never expires, manually create a Secret named `ci-cd-token` that acts as a long-lived token specifically tied to the `ci-cd-bot` ServiceAccount.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(kubectl get secret ci-cd-token -o jsonpath='{.type}')" == "kubernetes.io/service-account-token" ] && echo "✅ Secret is the correct type" || echo "❌ Secret type failed"
[ "$(kubectl get secret ci-cd-token -o jsonpath='{.metadata.annotations.kubernetes\.io/service-account\.name}')" == "ci-cd-bot" ] && echo "✅ Secret is correctly linked to ci-cd-bot" || echo "❌ Secret linking failed"
```

<details>

4. Solution:

```bash
# 1. Create the ServiceAccount
kubectl create serviceaccount ci-cd-bot

# 2. Write the specialized YAML. (This cannot be done imperatively).
vi sa-token.yaml
```

*The annotation is what tells the API server to populate the `data` field with a permanent token.*

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ci-cd-token
  annotations:
    kubernetes.io/service-account.name: "ci-cd-bot"   # Links it to the SA
type: kubernetes.io/service-account-token             # Triggers the token generation
```

```bash
# 3. Apply the YAML
kubectl apply -f sa-token.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete secret ci-cd-token
kubectl delete sa ci-cd-bot
rm -f sa-token.yaml
```

#### Variation 3.1: The Ephemeral TokenRequest API

**1. CKAD Style Question:**
You need to grant a temporary script access to the cluster without leaving a permanent Secret lying around.
Create a ServiceAccount named `temp-script`. Use the CLI to dynamically request a token for this ServiceAccount that strictly expires in `1 hour`. Save the raw token string to `/opt/temp-token.txt`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ -s /opt/temp-token.txt ] && echo "✅ Token requested and saved" || echo "❌ Token generation failed"
```

<details>

4. Solution:

```bash
# 1. Create the ServiceAccount
kubectl create sa temp-script

# 2. Use the 'create token' command to hit the TokenRequest API
kubectl create token temp-script --duration=1h > /opt/temp-token.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete sa temp-script
rm -f /opt/temp-token.txt
```

#### Variation 3.2: Forensic Token Extraction

**1. CKAD Style Question:**
A Pod named `vault-agent` is running using the `default` ServiceAccount.
Without viewing the Pod's YAML, use `kubectl exec` to extract the automatically mounted ServiceAccount token directly from the container's filesystem and save it to `/opt/mounted-token.txt`.

**2. Setup Script:**

```bash
kubectl run vault-agent --image=busybox -- sleep 3600
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.2 ---"
[ -s /opt/mounted-token.txt ] && echo "✅ Token successfully extracted from container filesystem" || echo "❌ Extraction failed. Did you check the correct mount path?"
```

<details>

4. Solution:

```bash
# You must memorize this exact path for the exam! It is where K8s injects identity.
kubectl exec vault-agent -- cat /var/run/secrets/kubernetes.io/serviceaccount/token > /opt/mounted-token.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod vault-agent --force
rm -f /opt/mounted-token.txt
```

-----


#### Main Task 4: The Projected Token Mount

**1. CKAD Style Question:**
Create a Pod named `vault-auth-pod` using the `nginx:alpine` image.
Disable the default token automount (`automountServiceAccountToken: false`).
Instead, manually configure a `projected` volume to mount a ServiceAccount token into the container at the path `/var/run/custom-token`.
The token must have an `audience` of `vault` and an `expirationSeconds` of `7200` (2 hours).

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 4 ---"
[ "$(kubectl get pod vault-auth-pod -o jsonpath='{.spec.automountServiceAccountToken}')" == "false" ] && echo "✅ Default automount successfully disabled" || echo "❌ Default automount is still active"
[ "$(kubectl get pod vault-auth-pod -o jsonpath='{.spec.volumes[0].projected.sources[0].serviceAccountToken.audience}')" == "vault" ] && echo "✅ Projected token audience set to 'vault'" || echo "❌ Projected token audience incorrect or missing"
[ "$(kubectl get pod vault-auth-pod -o jsonpath='{.spec.volumes[0].projected.sources[0].serviceAccountToken.expirationSeconds}')" == "7200" ] && echo "✅ Projected token expiration set to 7200" || echo "❌ Projected token expiration incorrect"
```

<details>

4. Solution:

```bash
# Generate the base YAML
kubectl run vault-auth-pod --image=nginx:alpine --dry-run=client -o yaml > projected.yaml

vi projected.yaml
```

*You must build out the `projected` volume structure block carefully:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-auth-pod
spec:
  automountServiceAccountToken: false   # 1. Disable default mount
  volumes:
  - name: token-vol
    projected:                          # 2. Define the projected volume
      sources:
      - serviceAccountToken:
          path: token                   # The filename inside the mount path
          expirationSeconds: 7200
          audience: vault
  containers:
  - image: nginx:alpine
    name: vault-auth-pod
    volumeMounts:
    - name: token-vol
      mountPath: /var/run/custom-token  # 3. Mount it
```

```bash
kubectl apply -f projected.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod vault-auth-pod
rm -f projected.yaml
```

-----

With these additions—specifically the `imagePullSecrets` attachment, the `automountServiceAccountToken` global shield, the nested assignment in CronJobs, the ephemeral `kubectl create token` command, and the forensic `/var/run/secrets/...` extraction—you have now covered 100% of the Service Account domain mechanics.
