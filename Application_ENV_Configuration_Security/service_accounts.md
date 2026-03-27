## Service Accounts 

On the CKAD, "Configure Service Accounts" is fundamentally about identity management for your Pods. If you don't explicitly assign one, Kubernetes assigns the `default` ServiceAccount, which is generally a security bad practice. 

The exam tests your ability to create them, attach them to workloads (both new and existing), and manage their legacy token behaviors (a major curveball introduced in recent Kubernetes versions).

---

### Task 1: The Standard Assignment (Most Repeating)
This is the bread and butter of the topic. You must create a ServiceAccount and explicitly tell a new Pod to use it instead of the default one.

**1. CKAD Style Question:**
Create a new namespace named `app-auth`. 
Inside this namespace, create a ServiceAccount named `backend-sa`. 
Finally, create a Pod named `backend-api` using the `nginx:alpine` image in this namespace, and configure it to run using the `backend-sa` ServiceAccount.

**2. Setup Script:**
*(None required, creating the namespace is part of the task)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get sa backend-sa -n app-auth -o jsonpath='{.metadata.name}')" == "backend-sa" ] && echo "✅ ServiceAccount created" || echo "❌ ServiceAccount missing"
[ "$(kubectl get pod backend-api -n app-auth -o jsonpath='{.spec.serviceAccountName}')" == "backend-sa" ] && echo "✅ Pod is using correct ServiceAccount" || echo "❌ Pod ServiceAccount failed"
```

<details>

**4. Solution:**
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
rm sa-pod.yaml
```

---

### Task 2: The Live Injection (Highly Repeating / Imperative Trick)
The exam loves to test if you know how to update a running workload. They will give you an existing Deployment and ask you to swap out its ServiceAccount. **Do not edit the YAML for this; there is an imperative shortcut that saves massive amounts of time.**

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
echo "--- Testing Task 2 ---"
[ "$(kubectl get deploy payment-processor -o jsonpath='{.spec.template.spec.serviceAccountName}')" == "stripe-sa" ] && echo "✅ Deployment updated to use stripe-sa" || echo "❌ Deployment ServiceAccount failed"
# Ensure the active pods actually received the update
POD_SA=$(kubectl get pods -l app=payment-processor -o jsonpath='{.items[0].spec.serviceAccountName}')
[ "$POD_SA" == "stripe-sa" ] && echo "✅ Pods rolled out with new ServiceAccount" || echo "❌ Pod rollout failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the new ServiceAccount
kubectl create serviceaccount stripe-sa

# 2. Use the 'set serviceaccount' imperative command to instantly patch the deployment
kubectl set serviceaccount deployment/payment-processor stripe-sa
```
*(That's it. Two lines of code, no vim required!)*

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy payment-processor
kubectl delete sa stripe-sa
```

---

### Task 3: The Modern Token Generator (The v1.24+ Curveball)
Historically, whenever you created a ServiceAccount, Kubernetes automatically generated an infinite-lifespan Secret containing a bearer token. **As of Kubernetes v1.24, this no longer happens.** The exam will test if you know how to manually create a long-lived token Secret for a legacy application.

**1. CKAD Style Question:**
Create a ServiceAccount named `ci-cd-bot` in the `default` namespace.
Because a legacy CI/CD pipeline requires a permanent API token, manually create a Secret named `ci-cd-token` that acts as a long-lived token specifically tied to the `ci-cd-bot` ServiceAccount.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get secret ci-cd-token -o jsonpath='{.type}')" == "kubernetes.io/service-account-token" ] && echo "✅ Secret is the correct type" || echo "❌ Secret type failed"
[ "$(kubectl get secret ci-cd-token -o jsonpath='{.metadata.annotations.kubernetes\.io/service-account\.name}')" == "ci-cd-bot" ] && echo "✅ Secret is linked to ci-cd-bot" || echo "❌ Secret linking failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the ServiceAccount
kubectl create serviceaccount ci-cd-bot

# 2. You cannot create this specific type of secret imperatively. You must write the YAML.
vi sa-token.yaml
```
*Create this precise Secret structure. The annotation is what tells Kubernetes to generate and inject the token data into this secret.*
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
rm sa-token.yaml
```

---

If you know how to assign it in a Pod YAML (`serviceAccountName`), how to hot-swap it in a Deployment (`kubectl set serviceaccount`), and how to generate a manual token Secret, you have completely conquered this objective. 

