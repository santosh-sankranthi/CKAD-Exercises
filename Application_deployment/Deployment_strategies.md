## Deployment strategies

When the syllabus asks you to "use Kubernetes primitives to implement common deployment strategies," it is a trick question. **There is no `kind: BlueGreen` or `kind: Canary` in Kubernetes.** Instead, the exam tests your ability to manipulate standard **Deployments, Labels, and Services** to manually route traffic and control rollouts. 

Here are the three deployment strategies that make up 100% of this specific subtopic.

---

### Task 1: The Tuned Rolling Update (The Built-in Strategy)
You already know how to trigger a Rolling Update by changing an image. But what if the exam tells you that an application is extremely resource-heavy, and you are **not allowed** to have more than the desired number of Pods running during an update? You have to tune the deployment strategy.

**1. CKAD Style Question:**
A Deployment named `heavy-api` exists in the `default` namespace with 4 replicas.
Modify the deployment's update strategy to ensure that:
* **Zero** extra Pods are created above the desired replica count during an update (`maxSurge`).
* Up to **50%** of the Pods can be taken down simultaneously during the update (`maxUnavailable`).

**2. Setup Script:**
```bash
kubectl create deployment heavy-api --image=nginx:1.24 --replicas=4
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get deploy heavy-api -o jsonpath='{.spec.strategy.rollingUpdate.maxSurge}')" == "0" ] && echo "✅ maxSurge is 0" || echo "❌ maxSurge failed"
[ "$(kubectl get deploy heavy-api -o jsonpath='{.spec.strategy.rollingUpdate.maxUnavailable}')" == "50%" ] && echo "✅ maxUnavailable is 50%" || echo "❌ maxUnavailable failed"
```

<details>

**4. Solution:**
```bash
# You cannot set these specific parameters imperatively, so you must edit the live object.
kubectl edit deploy heavy-api
```
*In the vim editor, find the `strategy` block (usually right above the `template` block) and modify it to look like this:*
```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0          # Changed to 0
      maxUnavailable: 50%  # Changed to 50%
```
*(Save and exit vim. The Deployment is now tuned!)*

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy heavy-api
```

---

### Task 2: The Blue/Green Deployment (The Instant Switch)
In a Blue/Green deployment, you deploy a completely separate "Version 2" environment alongside "Version 1". Once Version 2 is ready, you instantly flip the traffic switch (the Service) to point to the new version.

**1. CKAD Style Question:**
A Deployment named `app-blue` (v1) is running, and a Service named `app-svc` is currently routing traffic to it.
1. Create a new Deployment named `app-green` using the `nginx:1.25` image with 2 replicas. Ensure it has the label `version=green`.
2. Update the `app-svc` Service so that it stops sending traffic to `app-blue` and instantly routes 100% of traffic to `app-green`.

**2. Setup Script:**
```bash
# Create the Blue environment and expose it
kubectl create deployment app-blue --image=nginx:1.24
kubectl expose deployment app-blue --name=app-svc --port=80
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get deploy app-green -o jsonpath='{.spec.replicas}')" == "2" ] && echo "✅ Green deployment created" || echo "❌ Green deployment missing"
[ "$(kubectl get svc app-svc -o jsonpath='{.spec.selector.app}')" == "app-green" ] && echo "✅ Service selector switched to Green" || echo "❌ Service is still pointing to Blue"
```

<details>

**4. Solution:**
```bash
# 1. Create the Green deployment
kubectl create deployment app-green --image=nginx:1.25 --replicas=2

# 2. Add the requested label to the new deployment's pod template
# (Using edit is safest to ensure it applies to the pod template, not just the deployment object)
kubectl edit deploy app-green
# -> Add `version: green` under spec.template.metadata.labels

# 3. The Magic Imperative Command: Instantly update the Service to point to the Green deployment's labels!
# The 'create deployment' command automatically adds the label 'app=app-green', so we tell the service to look for that.
kubectl set selector svc app-svc 'app=app-green'
```

</details>
  
**5. Clean-up Script:**
```bash
kubectl delete deploy app-blue app-green
kubectl delete svc app-svc
```

---

### Task 3: The Canary Release (The Traffic Split)
Instead of a 100% switch, a Canary release sends just a fraction of traffic to the new version. You achieve this by having two Deployments that **share the same Service label**, but have different replica counts.

**1. CKAD Style Question:**
A Deployment named `web-stable` (v1) is running with 3 replicas. A Service named `web-svc` routes traffic to it using the selector `app=web`.
Create a Canary Deployment named `web-canary` (v2) using the `nginx:1.25` image with **1 replica**. 
Ensure that `web-svc` automatically routes exactly 25% of its traffic to the `web-canary` Pod.
*(Hint: To do this, the Canary Pods must possess the exact label that the Service is looking for).*

**2. Setup Script:**
```bash
# Build the stable environment with the specific shared label
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-stable
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      track: stable
  template:
    metadata:
      labels:
        app: web        # This is the label the Service looks for
        track: stable
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  ports:
  - port: 80
  selector:
    app: web          # Service routes to ANY pod with 'app: web'
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
CANARY_LABEL=$(kubectl get deploy web-canary -o jsonpath='{.spec.template.metadata.labels.app}')
[ "$CANARY_LABEL" == "web" ] && echo "✅ Canary has the shared Service label" || echo "❌ Canary label missing"
ENDPOINTS=$(kubectl get endpoints web-svc -o jsonpath='{.subsets[0].addresses[*].ip}' | wc -w)
[ "$ENDPOINTS" == "4" ] && echo "✅ Service is routing to 4 total pods (75% stable, 25% canary)" || echo "❌ Service endpoint math is incorrect"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base YAML for the Canary deployment
kubectl create deployment web-canary --image=nginx:1.25 --replicas=1 --dry-run=client -o yaml > canary.yaml

# 2. Edit the YAML to inject the shared label
vi canary.yaml
```
*Modify the labels so they match the Service selector (`app: web`), but keep the deployment separate by using a different name.*
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web             # MATCH THE SERVICE SELECTOR
      track: canary        # Unique track identifier
  template:
    metadata:
      labels:
        app: web           # MATCH THE SERVICE SELECTOR
        track: canary
    spec:
      containers:
      - image: nginx:1.25
        name: nginx
```
```bash
# 3. Apply the YAML
kubectl apply -f canary.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy web-stable web-canary
kubectl delete svc web-svc
rm canary.yaml
```

---

### The Strategy Breakdown
If you understand the underlying logic here, you are golden:
* **Rolling Update:** 1 Deployment, 1 Service. Kubernetes handles the swap.
* **Blue/Green:** 2 Deployments, 1 Service. You flip the Service selector.
* **Canary:** 2 Deployments, 1 Service. Both Deployments share a label, the Service routes to both, and you control the percentage by altering replica counts.


## More variations:-

Here are the three advanced variations that round out this topic completely.

---

### Variation 1: The Botched Rollout (Rollback / Undo)
The exam loves to test your ability to panic-fix a broken deployment. You will be dropped into a scenario where an update was pushed with a bad image, and the Pods are stuck in `ImagePullBackOff` or `CrashLoopBackOff`. You must use the rollout history to save the day.

**1. CKAD Style Question:**
A Deployment named `critical-processor` in the `default` namespace was recently updated, but the new Pods are failing to start.
1. Inspect the rollout history of the deployment.
2. Undo the rollout to return the deployment to its previous stable revision.
3. Verify that the deployment is successfully running the previous image (`nginx:1.24`).

**2. Setup Script:**
```bash
# We create a stable deployment, record it, then intentionally break it
kubectl create deployment critical-processor --image=nginx:1.24 --replicas=2
kubectl annotate deployment critical-processor kubernetes.io/change-cause="Initial stable release"
kubectl set image deployment/critical-processor nginx=nginx:9.9.9
kubectl annotate deployment critical-processor kubernetes.io/change-cause="Updated to v9.9.9"
sleep 3 # Give it a moment to break
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1 ---"
[ "$(kubectl get deploy critical-processor -o jsonpath='{.spec.template.spec.containers[0].image}')" == "nginx:1.24" ] && echo "✅ Successfully rolled back to nginx:1.24" || echo "❌ Image is still broken"
[ "$(kubectl get deploy critical-processor -o jsonpath='{.status.readyReplicas}')" == "2" ] && echo "✅ Pods are fully ready" || echo "❌ Pods are not ready"
```

<details>

**4. Solution:**
```bash
# 1. Check the status and see the error
kubectl rollout status deployment/critical-processor

# 2. View the history to see the revisions
kubectl rollout history deployment/critical-processor

# 3. The magic imperative command to instantly revert to the previous working version
kubectl rollout undo deployment/critical-processor

# (Optional: If they ask you to roll back to a SPECIFIC revision, e.g., revision 1)
# kubectl rollout undo deployment/critical-processor --to-revision=1
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy critical-processor
```

---

### Variation 2: The Two-Tier Blue/Green (Staging Swap)
In a real Blue/Green scenario, you don't just blindly switch traffic. You test the Green deployment on a private Service first. The exam will ask you to swap the endpoints of *two* services simultaneously.

**1. CKAD Style Question:**
You have a Blue deployment (`store-v1`) serving production traffic via `prod-svc`.
You have a Green deployment (`store-v2`) serving testing traffic via `test-svc`.
The QA team has approved the Green deployment. Promote `store-v2` to production by updating `prod-svc` to route to it. Simultaneously, update `test-svc` to route to `store-v1` so the developers can debug the old version.

**2. Setup Script:**
```bash
# Deploy v1 (Blue) and its Service
kubectl create deployment store-v1 --image=nginx:alpine --replicas=2
kubectl label deploy store-v1 version=v1
kubectl expose deploy store-v1 --name=prod-svc --port=80
# Deploy v2 (Green) and its Service
kubectl create deployment store-v2 --image=nginx:latest --replicas=2
kubectl label deploy store-v2 version=v2
kubectl expose deploy store-v2 --name=test-svc --port=8080 --target-port=80
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2 ---"
[ "$(kubectl get svc prod-svc -o jsonpath='{.spec.selector.app}')" == "store-v2" ] && echo "✅ prod-svc routing to Green (v2)" || echo "❌ prod-svc still routing to Blue"
[ "$(kubectl get svc test-svc -o jsonpath='{.spec.selector.app}')" == "store-v1" ] && echo "✅ test-svc routing to Blue (v1)" || echo "❌ test-svc still routing to Green"
```

<details>

**4. Solution:**
```bash
# 1. Update the Prod Service to point to Green. 
# The deployment 'store-v2' automatically got the label 'app=store-v2', so we use that.
kubectl set selector svc prod-svc 'app=store-v2'

# 2. Update the Test Service to point to Blue.
kubectl set selector svc test-svc 'app=store-v1'
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy store-v1 store-v2
kubectl delete svc prod-svc test-svc
```

---

### Variation 3: Finalizing the Canary (Scale and Record)
We practiced splitting the traffic for a Canary. But what happens when the Canary is successful? You have to finish the deployment by scaling the Canary up, scaling the Stable down, and heavily annotating your actions.

**1. CKAD Style Question:**
A Canary deployment (`api-canary`) is running successfully alongside a Stable deployment (`api-stable`).
1. Scale the `api-canary` deployment up to 4 replicas to handle full production load.
2. Scale the `api-stable` deployment down to 0 replicas to decommission it.
3. Add an annotation to the `api-canary` deployment with the key `kubernetes.io/change-cause` and the value `Promoted canary to full production`.

**2. Setup Script:**
```bash
kubectl create deployment api-stable --image=nginx:1.24 --replicas=3
kubectl create deployment api-canary --image=nginx:1.25 --replicas=1
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3 ---"
[ "$(kubectl get deploy api-canary -o jsonpath='{.spec.replicas}')" == "4" ] && echo "✅ Canary scaled to 4" || echo "❌ Canary scale failed"
[ "$(kubectl get deploy api-stable -o jsonpath='{.spec.replicas}')" == "0" ] && echo "✅ Stable scaled to 0" || echo "❌ Stable scale failed"
kubectl describe deploy api-canary | grep -q "Promoted canary to full production" && echo "✅ Annotation recorded successfully" || echo "❌ Annotation missing"
```

<details>
  
**4. Solution:**
```bash
# 1. Scale the Canary up imperatively
kubectl scale deployment api-canary --replicas=4

# 2. Scale the Stable down imperatively
kubectl scale deployment api-stable --replicas=0

# 3. Add the annotation imperatively
kubectl annotate deployment api-canary kubernetes.io/change-cause="Promoted canary to full production"
```

</details>
  
**5. Clean-up Script:**
```bash
kubectl delete deploy api-stable api-canary
```

---

### The Mastery Check
By covering the baseline strategies *and* these three lifecycle variations (Undo, Swap, and Finalize), you are literally unshakeable on this topic. You have seen every angle the examiners can possibly test using pure Kubernetes primitives.
