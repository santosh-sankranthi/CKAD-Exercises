
## Choose and Use the Right Workload Resource

### Task 1: Deployments and Update Strategies (Most Repeating)

Deployments are the default choice for stateless applications. The exam heavily tests your ability to configure *how* a deployment updates its pods when a new image is rolled out to ensure zero downtime (or strict downtime).

#### Main Task 1: Fine-Tuning a RollingUpdate

**1. CKAD Style Question:**
Create a Deployment named `web-frontend` in the `default` namespace using the `nginx:1.24` image with `3` replicas.
Configure the Deployment's update strategy to ensure **zero downtime**:

  * Strategy Type: `RollingUpdate`
  * Max Surge: `2` (It can burst 2 pods over the desired count during an update).
  * Max Unavailable: `0` (It cannot drop below the desired 3 replicas at any time).

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get deploy web-frontend -o jsonpath='{.spec.strategy.type}')" == "RollingUpdate" ] && echo "✅ Strategy is RollingUpdate" || echo "❌ Strategy is incorrect"
[ "$(kubectl get deploy web-frontend -o jsonpath='{.spec.strategy.rollingUpdate.maxSurge}')" == "2" ] && echo "✅ Max Surge is 2" || echo "❌ Max Surge failed"
[ "$(kubectl get deploy web-frontend -o jsonpath='{.spec.strategy.rollingUpdate.maxUnavailable}')" == "0" ] && echo "✅ Max Unavailable is 0" || echo "❌ Max Unavailable failed"
```

<details>

4. Solution:

```bash
# 1. Generate the base YAML imperatively
kubectl create deploy web-frontend --image=nginx:1.24 --replicas=3 --dry-run=client -o yaml > rollout.yaml

# 2. Edit the YAML
vi rollout.yaml
```

*Add the `strategy` block directly under `spec` (above `template`):*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  strategy:                     # ADD FROM HERE
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0         # TO HERE
  selector:
    matchLabels:
      app: web-frontend
```

```bash
# 3. Apply the YAML
kubectl apply -f rollout.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy web-frontend
rm -f rollout.yaml
```

#### Variation 1.1: The `Recreate` Strategy

**1. CKAD Style Question:**
A legacy Deployment named `data-processor` cannot handle having two versions running simultaneously (it corrupts the database).
Modify the existing Deployment so that during an update, it kills all existing pods *before* starting the new ones.

**2. Setup Script:**

```bash
kubectl create deploy data-processor --image=busybox -- sleep 3600
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get deploy data-processor -o jsonpath='{.spec.strategy.type}')" == "Recreate" ] && echo "✅ Strategy successfully changed to Recreate" || echo "❌ Strategy change failed"
```

<details>

4. Solution:

```bash
# 1. Edit the deployment live
kubectl edit deploy data-processor
```

*In vim, locate the `strategy` block. Delete the `rollingUpdate` block entirely, and change the type:*

```yaml
  strategy:
    type: Recreate       # Changed from RollingUpdate, deleted everything below it
  template:
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy data-processor
```

#### Variation 1.2: Rollout History and Undo

**1. CKAD Style Question:**
A Deployment named `api-gateway` was recently updated to `nginx:1.99` (a broken image), and the pods are failing.

1.  Check the rollout history of the Deployment.
2.  Roll back the Deployment to its previous working revision.

**2. Setup Script:**

```bash
kubectl create deploy api-gateway --image=nginx:1.24
sleep 2
kubectl set image deploy/api-gateway nginx=nginx:1.99
sleep 2
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get deploy api-gateway -o jsonpath='{.spec.template.spec.containers[0].image}')" == "nginx:1.24" ] && echo "✅ Deployment successfully rolled back" || echo "❌ Rollback failed"
```

<details>

4. Solution:

```bash
# 1. View the history (Optional, but good for verification)
kubectl rollout history deploy/api-gateway

# 2. Execute the rollback imperative command
kubectl rollout undo deploy/api-gateway

# (If they ask you to roll back to a specific revision, use: --to-revision=1)
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy api-gateway
```

-----

### Task 2: DaemonSets (Highly Repeating)

When a requirement says "Ensure exactly one copy of this pod runs on every node," you must choose a DaemonSet. The exam trick is that **you must write the YAML by hijacking a Deployment generation command.**

#### Main Task 2: Creating a DaemonSet from Scratch

**1. CKAD Style Question:**
The security team needs an audit agent running on every single node in the cluster.
Create a DaemonSet named `audit-daemon` in the `default` namespace using the `busybox` image and the command `sleep 3600`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
kubectl get daemonset audit-daemon >/dev/null 2>&1 && echo "✅ DaemonSet created successfully" || echo "❌ DaemonSet missing (Did you create a Deployment by mistake?)"
[ "$(kubectl get daemonset audit-daemon -o jsonpath='{.spec.template.spec.containers[0].image}')" == "busybox" ] && echo "✅ Image is correct" || echo "❌ Image is incorrect"
```

<details>

4. Solution:

```bash
# 1. There is no 'kubectl create ds'. Generate a Deployment YAML instead!
kubectl create deploy audit-daemon --image=busybox --dry-run=client -o yaml -- sleep 3600 > ds.yaml

# 2. Edit the YAML to morph it into a DaemonSet
vi ds.yaml
```

*You must change the `kind`, and delete `replicas` and `strategy` (DaemonSets don't use them):*

```yaml
apiVersion: apps/v1
kind: DaemonSet                 # 1. CHANGE THIS FROM Deployment
metadata:
  name: audit-daemon
spec:
  # replicas: 1                 # 2. DELETE THIS LINE
  selector:
    matchLabels:
      app: audit-daemon
  # strategy: {}                # 3. DELETE THIS LINE
  template:
    metadata:
      labels:
        app: audit-daemon
```

```bash
# 3. Apply the morphed YAML
kubectl apply -f ds.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete daemonset audit-daemon
rm -f ds.yaml
```

#### Variation 2.1: Targeted Node DaemonSet

**1. CKAD Style Question:**
Modify the `audit-daemon` DaemonSet so that it only schedules pods on nodes bearing the label `env=production`.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: audit-daemon
spec:
  selector:
    matchLabels:
      app: audit-daemon
  template:
    metadata:
      labels:
        app: audit-daemon
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ["sleep", "3600"]
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get daemonset audit-daemon -o jsonpath='{.spec.template.spec.nodeSelector.env}')" == "production" ] && echo "✅ NodeSelector successfully applied" || echo "❌ NodeSelector missing or incorrect"
```

<details>

4. Solution:

```bash
# 1. Edit the live DaemonSet
kubectl edit ds audit-daemon
```

*Add the `nodeSelector` block inside the pod template spec:*

```yaml
    spec:
      nodeSelector:             # ADD FROM HERE
        env: production         # TO HERE
      containers:
      - command:
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ds audit-daemon
```

-----

### Task 3: StatefulSets (Moderately Repeating)

When a requirement asks for "stable network identities" (e.g., `pod-0`, `pod-1`), strictly ordered creation, or unique storage volumes per replica, you must use a StatefulSet.

#### Main Task 3: StatefulSet and the Headless Service

**1. CKAD Style Question:**
A database application requires stable networking.

1.  Create a "Headless" Service named `db-service` on port `3306`.
2.  Create a StatefulSet named `db-cluster` with `2` replicas using the `mysql:8.0` image.
3.  Link the StatefulSet to the Headless Service using the `serviceName` field.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(kubectl get svc db-service -o jsonpath='{.spec.clusterIP}')" == "None" ] && echo "✅ Headless Service created correctly" || echo "❌ Service is not Headless"
[ "$(kubectl get statefulset db-cluster -o jsonpath='{.spec.serviceName}')" == "db-service" ] && echo "✅ StatefulSet properly linked to Service" || echo "❌ ServiceName link failed"
```

<details>

4. Solution:

```bash
# 1. Create the Headless Service (ClusterIP = None)
kubectl create service clusterip db-service --tcp=3306:3306 --clusterip="None"

# 2. Morph a Deployment into a StatefulSet
kubectl create deploy db-cluster --image=mysql:8.0 --replicas=2 --dry-run=client -o yaml > sts.yaml
vi sts.yaml
```

*Change the `kind`, delete `strategy`, and add `serviceName`:*

```yaml
apiVersion: apps/v1
kind: StatefulSet               # 1. CHANGE THIS FROM Deployment
metadata:
  name: db-cluster
spec:
  serviceName: "db-service"     # 2. ADD THIS REQUIRED FIELD
  replicas: 2
  selector:
    matchLabels:
      app: db-cluster
  # strategy: {}                # 3. DELETE THIS LINE
  template:
```

```bash
kubectl apply -f sts.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete statefulset db-cluster
kubectl delete svc db-service
rm -f sts.yaml
```

#### Variation 3.1: VolumeClaimTemplates (The Unique PVC Generator)

**1. CKAD Style Question:**
Modify the requirement for `db-cluster`.
Instead of sharing a volume, configure the StatefulSet so that every replica automatically provisions its own unique `1Gi` PersistentVolumeClaim named `data-disk`.
*(Note: Use standard storage class and ReadWriteOnce access mode).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get sts db-cluster -o jsonpath='{.spec.volumeClaimTemplates[0].metadata.name}')" == "data-disk" ] && echo "✅ volumeClaimTemplate created" || echo "❌ volumeClaimTemplate missing"
[ "$(kubectl get sts db-cluster -o jsonpath='{.spec.volumeClaimTemplates[0].spec.resources.requests.storage}')" == "1Gi" ] && echo "✅ Storage size is 1Gi" || echo "❌ Storage size incorrect"
```

<details>

4. Solution:

```bash
kubectl create deploy db-cluster --image=mysql:8.0 --dry-run=client -o yaml > sts-pvc.yaml
vi sts-pvc.yaml
```

*Add the `volumeClaimTemplates` array at the bottom of the StatefulSet `spec` (outside the template):*

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db-cluster
spec:
  serviceName: "db-service"
  replicas: 2
  selector:
    matchLabels:
      app: db-cluster
  template:
    metadata:
      labels:
        app: db-cluster
    spec:
      containers:
      - image: mysql:8.0
        name: mysql
        volumeMounts:                   # Mount the template into the container
        - name: data-disk
          mountPath: /var/lib/mysql
  volumeClaimTemplates:                 # ADD THIS ENTIRE BLOCK
  - metadata:
      name: data-disk
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

```bash
kubectl apply -f sts-pvc.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete sts db-cluster
rm -f sts-pvc.yaml
```

-----

### Task 4: Architectural Refactoring (The CKAD Capstone)

The ultimate test of "Choosing the right workload resource" is migrating an incorrectly deployed application into the correct architectural pattern without losing its configuration.

#### Main Task 4: Naked Pod to Deployment Migration

**1. CKAD Style Question:**
A junior developer deployed an application as a naked, standalone Pod named `legacy-api` in the `default` namespace. Because it is a naked pod, it has no self-healing capabilities and dies permanently if the node restarts.

1.  Extract the container configuration from the running Pod.
2.  Delete the naked Pod.
3.  Wrap the application in a highly-available Deployment named `legacy-api-deploy` with `3` replicas.

**2. Setup Script:**

```bash
kubectl run legacy-api --image=nginx:alpine --env="ENV=PROD" --labels="tier=backend"
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 4 ---"
kubectl get pod legacy-api >/dev/null 2>&1 && echo "❌ FAILED: Naked pod still exists" || echo "✅ Naked pod successfully deleted"
[ "$(kubectl get deploy legacy-api-deploy -o jsonpath='{.spec.replicas}')" == "3" ] && echo "✅ Deployment created with 3 replicas" || echo "❌ Deployment missing or replica count wrong"
[ "$(kubectl get deploy legacy-api-deploy -o jsonpath='{.spec.template.spec.containers[0].env[0].name}')" == "ENV" ] && echo "✅ Environment variables successfully migrated" || echo "❌ Config migration failed"
```

<details>

4. Solution:

```bash
# 1. Extract the pod's specific configuration
kubectl get pod legacy-api -o yaml > extract.yaml

# 2. Delete the naked pod
kubectl delete pod legacy-api

# 3. Create a new deployment scaffolding
kubectl create deploy legacy-api-deploy --image=nginx:alpine --replicas=3 --dry-run=client -o yaml > new-deploy.yaml

# 4. Use vim to migrate any specific settings (like env vars or volumes) from extract.yaml into new-deploy.yaml
vi new-deploy.yaml
# (In this specific case, you just need to add the ENV block that was present in the original pod).

kubectl apply -f new-deploy.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy legacy-api-deploy
rm -f extract.yaml new-deploy.yaml
```

-----

### Task 5: Advanced Workload Patterns (Canary & Lifecycle)

The exam will test your ability to run multiple workloads in parallel to share traffic safely, and how to manipulate the workload controllers without disrupting running applications.

#### Main Task 5: The Canary Deployment Pattern

**1. CKAD Style Question:**
A Deployment named `app-stable` is currently running `nginx:1.24` with `3` replicas. A Service named `app-svc` is routing traffic to it.
You need to test a new version (`nginx:latest`) on live traffic without taking down the stable version.
Create a new "Canary" Deployment named `app-canary` with `1` replica. Configure its labels so that the existing `app-svc` Service automatically routes approximately 25% of its traffic to this new canary pod, alongside the 3 stable pods.

**2. Setup Script:**

```bash
# Set up the stable environment
kubectl create deploy app-stable --image=nginx:1.24 --replicas=3
kubectl set env deploy/app-stable TRACK=stable
kubectl label deploy app-stable track=stable
# Set up the Service that targets 'app=app-stable'
kubectl expose deploy app-stable --name=app-svc --port=80 --target-port=80
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 5 ---"
[ "$(kubectl get deploy app-canary -o jsonpath='{.spec.replicas}')" == "1" ] && echo "✅ Canary deployment created with 1 replica" || echo "❌ Canary deployment missing or wrong replica count"
# The magic of Canary: Both deployments must share the same 'app' label that the Service is looking for!
STABLE_LABEL=$(kubectl get deploy app-stable -o jsonpath='{.spec.template.metadata.labels.app}')
CANARY_LABEL=$(kubectl get deploy app-canary -o jsonpath='{.spec.template.metadata.labels.app}')
[ "$STABLE_LABEL" == "$CANARY_LABEL" ] && echo "✅ Canary shares the Service selector label" || echo "❌ FAILED: Canary does not share the Service label"
[ "$(kubectl get endpoints app-svc -o jsonpath='{.subsets[0].addresses[*].ip}' | wc -w)" == "4" ] && echo "✅ Service is successfully routing to all 4 pods (3 Stable, 1 Canary)!" || echo "❌ Service is not routing to the canary pod"
```

<details>

4. Solution:

```bash
# 1. Create the base canary deployment
kubectl create deploy app-canary --image=nginx:latest --replicas=1 --dry-run=client -o yaml > canary.yaml

# 2. Edit the YAML to ensure the labels match the Service's selector
vi canary.yaml
```

*The `app-svc` is looking for `app: app-stable`. Your canary MUST have this label to receive traffic, plus a unique label like `track: canary` to distinguish it from the stable pods:*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-stable       # MUST MATCH THE SERVICE SELECTOR
      track: canary         # Distinguishes it from stable
  template:
    metadata:
      labels:
        app: app-stable     # MUST MATCH THE SERVICE SELECTOR
        track: canary
    spec:
      containers:
      - image: nginx:latest
        name: nginx
```

```bash
kubectl apply -f canary.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy app-stable app-canary
kubectl delete svc app-svc
rm -f canary.yaml
```

#### Variation 5.1: Orphan Deletion (The Cascade Trap)

**1. CKAD Style Question:**
A StatefulSet named `broken-db` is managing 2 database pods. The controller itself is misconfigured and needs to be deleted and recreated. However, tearing down the database pods will cause a massive outage.
Delete the `broken-db` StatefulSet controller **without** terminating its underlying Pods.

**2. Setup Script:**

```bash
kubectl create service clusterip dummy-svc --tcp=80:80 --clusterip="None"
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: broken-db
spec:
  serviceName: "dummy-svc"
  replicas: 2
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
sleep 5
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 5.1 ---"
kubectl get sts broken-db >/dev/null 2>&1 && echo "❌ FAILED: StatefulSet still exists" || echo "✅ StatefulSet successfully deleted"
kubectl get pod broken-db-0 >/dev/null 2>&1 && echo "✅ Pod broken-db-0 survived the deletion!" || echo "❌ FAILED: Pod broken-db-0 was killed"
kubectl get pod broken-db-1 >/dev/null 2>&1 && echo "✅ Pod broken-db-1 survived the deletion!" || echo "❌ FAILED: Pod broken-db-1 was killed"
```

<details>
  
4. Solution:

```bash
# By default, deleting a workload controller sends a cascading delete signal to its pods.
# You MUST use the --cascade=orphan flag to sever the link and leave the pods running.
kubectl delete statefulset broken-db --cascade=orphan
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod broken-db-0 broken-db-1 --force
kubectl delete svc dummy-svc
```

-----

### The *True* Final 100% Verdict

By mastering **Deployment rollout strategies** (`RollingUpdate` limits vs `Recreate`), understanding how to hijack generator commands to build **DaemonSets** and **StatefulSets**, mastering **Headless Services / Volume Templates**, and executing live architectural refactoring... you have walled off exactly 100% of the workload selection scenarios tested on the CKAD.
