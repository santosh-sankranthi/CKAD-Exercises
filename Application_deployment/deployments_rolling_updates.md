## understanding deployments and performing rolling updates.

The syllabus item **"Understand Deployments and how to perform rolling updates"** is the beating heart of Kubernetes operations. The exam tests your ability to create them, mutate them, track their history, and bail yourself out when things go wrong.

Here is your 9-part master drill sheet, broken into 3 core groups: **Creation, Mutation, and Time Travel (Rollbacks/History).**

---

### Group 1: Deployment Creation & Baseline Management
Before you can update a Deployment, you must be able to spin it up rapidly and manipulate its baseline state.

#### Main Task 1: The Annotated Creation
**1. CKAD Style Question:**
Create a Deployment named `core-api` in the `default` namespace using the `nginx:1.24` image. Configure it with 3 replicas. Ensure that the initial creation is recorded in the rollout history with the message "Initial deployment of core-api".

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get deploy core-api -o jsonpath='{.spec.replicas}')" == "3" ] && echo "✅ Replicas: 3" || echo "❌ Replicas failed"
kubectl describe deploy core-api | grep -q "Initial deployment of core-api" && echo "✅ Change-cause recorded" || echo "❌ Change-cause missing"
```

<details>

**4. Solution:**
```bash
kubectl create deployment core-api --image=nginx:1.24 --replicas=3
kubectl annotate deployment core-api kubernetes.io/change-cause="Initial deployment of core-api"
```

</details>

**5. Clean-up:** `kubectl delete deploy core-api`

#### Variation 1.1: The Rapid Scale
**1. CKAD Style Question:**
A Deployment named `batch-worker` exists. Due to heavy load, immediately scale it to `5` replicas without editing its YAML file.

**2. Setup Script:** `kubectl create deploy batch-worker --image=busybox -- sleep 3600`

**3. Testcase:** ```bash
[ "$(kubectl get deploy batch-worker -o jsonpath='{.spec.replicas}')" == "5" ] && echo "✅ Scaled to 5" || echo "❌ Scale failed" ```

<details>

**4. Solution:**
```bash
kubectl scale deployment batch-worker --replicas=5
```

</details>

**5. Clean-up:** `kubectl delete deploy batch-worker`

#### Variation 1.2: The Clean YAML Extraction
**1. CKAD Style Question:**
A Deployment named `legacy-app` is running. Extract its configuration into a clean YAML file named `new-app.yaml`. Strip out all cluster-specific metadata (like uid, resourceVersion, creationTimestamp, and status). Change the name inside the file to `modern-app` and deploy it.

**2. Setup Script:** `kubectl create deploy legacy-app --image=nginx`

**3. Testcase:** ```bash
[ "$(kubectl get deploy modern-app -o jsonpath='{.metadata.name}')" == "modern-app" ] && echo "✅ modern-app deployed" || echo "❌ modern-app missing" ```

<details>

**4. Solution:**
```bash
# Using --dry-run=client -o yaml on a GET command is a pro-trick to strip out cluster metadata!
kubectl get deploy legacy-app -o yaml | grep -v "uid\|resourceVersion\|creationTimestamp\|status" > new-app.yaml
vi new-app.yaml # Change metadata.name from legacy-app to modern-app
kubectl apply -f new-app.yaml
```
</details>

**5. Clean-up:** `kubectl delete deploy legacy-app modern-app; rm new-app.yaml`

---

### Group 2: Executing Rolling Updates (Mutation)
The exam tests multiple ways to trigger a rollout. It’s not always just about changing the image.

#### Main Task 2: The Standard Image Rollout
**1. CKAD Style Question:**
A Deployment named `web-front` is running `nginx:1.24`. Update the image to `nginx:1.25`. Record the update in the rollout history with the message "Updated to v1.25" and monitor the rollout status until it completes.

**2. Setup Script:** `kubectl create deploy web-front --image=nginx:1.24 --replicas=2`

**3. Testcase:**
```bash
[ "$(kubectl get deploy web-front -o jsonpath='{.spec.template.spec.containers[0].image}')" == "nginx:1.25" ] && echo "✅ Image updated" || echo "❌ Image update failed"
kubectl describe deploy web-front | grep -q "Updated to v1.25" && echo "✅ Annotation verified" || echo "❌ Annotation missing"
```

<details>

**4. Solution:**
```bash
kubectl set image deployment/web-front nginx=nginx:1.25
kubectl annotate deployment web-front kubernetes.io/change-cause="Updated to v1.25"
kubectl rollout status deployment/web-front
```

</details>

**5. Clean-up:** `kubectl delete deploy web-front`

#### Variation 2.1: The Environment Variable Trigger
**1. CKAD Style Question:**
A Deployment named `api-worker` is running. Trigger a rolling update by injecting a new environment variable `LOG_LEVEL=debug` directly from the command line, without using a YAML file.

**2. Setup Script:** `kubectl create deploy api-worker --image=busybox -- sleep 3600`

**3. Testcase:**
```bash
[ "$(kubectl get deploy api-worker -o jsonpath='{.spec.template.spec.containers[0].env[0].name}')" == "LOG_LEVEL" ] && echo "✅ Env Var injected & rollout triggered" || echo "❌ Env Var missing"
```

<details>

**4. Solution:**
```bash
kubectl set env deployment/api-worker LOG_LEVEL=debug
```

</details>

**5. Clean-up:** `kubectl delete deploy api-worker`

#### Variation 2.2: The Ghost Rollout (Force Restart)
**1. CKAD Style Question:**
A Deployment named `cache-node` relies on a ConfigMap. The ConfigMap has been updated, but the Pods have not picked up the changes. Force the deployment to execute a rolling update to recreate its Pods, without changing its image or configuration.

**2. Setup Script:** `kubectl create deploy cache-node --image=nginx --replicas=2`

**3. Testcase:**
```bash
kubectl describe deploy cache-node | grep -q "deployment.kubernetes.io/revision: 2" && echo "✅ Forced rollout executed" || echo "❌ Rollout did not occur"
```

<details>

**4. Solution:**
```bash
kubectl rollout restart deployment/cache-node
```

</details>

**5. Clean-up:** `kubectl delete deploy cache-node`

---

### Group 3: History, Rollbacks, and Traffic Control (Time Travel)
When an update breaks production, you need to know how to navigate the ReplicaSets to restore service.

#### Main Task 3: The Panic Button (Undo)
**1. CKAD Style Question:**
A Deployment named `db-proxy` was recently updated with a broken image, and the rollout is hung. Immediately undo the rollout to restore the deployment to its previous stable state.

**2. Setup Script:** ```bash
kubectl create deploy db-proxy --image=nginx:1.24
kubectl set image deploy/db-proxy nginx=nginx:broken-tag
```

**3. Testcase:**
```bash
[ "$(kubectl get deploy db-proxy -o jsonpath='{.spec.template.spec.containers[0].image}')" == "nginx:1.24" ] && echo "✅ Successfully reverted to previous state" || echo "❌ Rollback failed"
```

<details>

**4. Solution:**
```bash
kubectl rollout undo deployment/db-proxy
```

</details>

**5. Clean-up:** `kubectl delete deploy db-proxy`

#### Variation 3.1: The Specific Revision Rollback
**1. CKAD Style Question:**
A Deployment named `payment-gateway` has multiple revisions in its history. Inspect its history and roll it back explicitly to revision `1`.

**2. Setup Script:**
```bash
kubectl create deploy payment-gateway --image=nginx:1.23
kubectl set image deploy/payment-gateway nginx=nginx:1.24
kubectl set image deploy/payment-gateway nginx=nginx:1.25
```

**3. Testcase:**

```bash
[ "$(kubectl get deploy payment-gateway -o jsonpath='{.spec.template.spec.containers[0].image}')" == "nginx:1.23" ] && echo "✅ Reverted to Revision 1" || echo "❌ Rollback to specific revision failed"
```

<details>

**4. Solution:**
```bash
kubectl rollout history deployment/payment-gateway
kubectl rollout undo deployment/payment-gateway --to-revision=1
```

</details>

**5. Clean-up:** `kubectl delete deploy payment-gateway`

#### Variation 3.2: The Paused Rollout (The Multi-Edit)
**1. CKAD Style Question:**
A Deployment named `data-miner` is running. You need to update its image to `busybox:1.36` AND change its CPU request to `100m`. To avoid triggering two separate rolling updates (which creates messy history and double the ReplicaSets), **pause** the deployment, apply both changes imperatively, and then **resume** the deployment.

**2. Setup Script:** `kubectl create deploy data-miner --image=busybox:1.35 -- sleep 3600`

**3. Testcase:**
```bash
REVISIONS=$(kubectl get rs -l app=data-miner --no-headers | wc -l)
[ "$REVISIONS" == "2" ] && echo "✅ Only one new ReplicaSet created (Paused successfully)" || echo "❌ Too many ReplicaSets created"
[ "$(kubectl get deploy data-miner -o jsonpath='{.spec.template.spec.containers[0].image}')" == "busybox:1.36" ] && echo "✅ Image updated" || echo "❌ Image failed"
```

<details>
  
**4. Solution:**
```bash
# 1. Pause the rollout to suspend triggers
kubectl rollout pause deployment/data-miner

# 2. Make all necessary changes
kubectl set image deployment/data-miner busybox=busybox:1.36
kubectl set resources deployment/data-miner --requests=cpu=100m

# 3. Resume the rollout to trigger a single update cycle
kubectl rollout resume deployment/data-miner
```

</details>

**5. Clean-up:** `kubectl delete deploy data-miner`

---

If you can confidently execute all 9 of these tasks in a terminal without heavily relying on the documentation, you are not just prepared for this subtopic—you are a Kubernetes operator. 
