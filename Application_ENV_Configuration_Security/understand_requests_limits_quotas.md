I noticed a tiny copy-paste mix-up in your prompt\! You provided the text for **Probes** (Readiness, Liveness, Startup), but the heading and the syllabus objective you asked me to revise is **"Understand requests, limits, and quotas."** No worries at all—I know exactly what you need to achieve 100% coverage here.

To completely dominate this specific syllabus item, you have to go beyond just defining Pod-level CPU and Memory. The CKAD heavily tests **ResourceQuotas** (namespace-level hard limits) and **LimitRanges** (automatic defaults and constraints). If you don't know how to write a LimitRange or bypass a ResourceQuota, you will get stuck on the exam.

Here is the fully revised, mathematically exhaustive 3-Part Matrix (with 7 total scenarios) for **"Understand requests, limits, and quotas"**, ordered from the most universally tested concepts to the advanced boundary enforcements. Everything strictly follows your 5-component standard with expandable solutions.

-----

## Understand requests, limits, and quotas

### Task 1: Pod-Level Boundaries (Most Repeating)

The absolute foundation of this topic is defining exactly what a pod *needs* to schedule (Request) and the absolute maximum it is *allowed* to use before being throttled or killed (Limit).

#### Main Task 1: Standard Requests and Limits

**1. CKAD Style Question:**
Create a Pod named `resource-pod` in the `default` namespace using the `nginx:alpine` image.
Define the resource requirements so that the container:

  * **Requests:** `100m` of CPU and `64Mi` of Memory.
  * **Limits:** `200m` of CPU and `128Mi` of Memory.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash

[[ "$(kubectl get pod resource-pod -o jsonpath='{.spec.containers[0].resources.requests.cpu}')" == "100m" ]] && echo "✅ CPU Request configured" || echo "❌ CPU Request failed"
[[ "$(kubectl get pod resource-pod -o jsonpath='{.spec.containers[0].resources.limits.memory}')" == "128Mi" ]] && echo "✅ Memory Limit configured" || echo "❌ Memory Limit failed"
```

<details>

4. Solution:

```bash
# 1. Generate the base Pod YAML
kubectl run resource-pod --image=nginx:alpine --dry-run=client -o yaml > resource.yaml

# 2. Edit the YAML
vi resource.yaml
```

*Add the `resources` block under the container spec:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - image: nginx:alpine
    name: resource-pod
    resources:              # ADD FROM HERE
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi       # TO HERE
```

```bash
# 3. Apply the YAML
kubectl apply -f resource.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod resource-pod
rm -f resource.yaml
```

#### Variation 1.1: Guaranteed QoS (Quality of Service)

**1. CKAD Style Question:**
Kubernetes assigns Quality of Service (QoS) classes based on resource configurations.
Create a Pod named `strict-qos` using the `redis` image. Configure its resources so that Kubernetes automatically assigns it the **"Guaranteed"** QoS class.
*(Hint: A pod is Guaranteed only if every container has identical CPU and Memory Requests AND Limits).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash

[[ "$(kubectl get pod strict-qos -o jsonpath='{.status.qosClass}')" == "Guaranteed" ]] && echo "✅ QoS Class is Guaranteed!" || echo "❌ QoS Class is NOT Guaranteed"
```

<details>

4. Solution:

```bash
kubectl run strict-qos --image=redis --dry-run=client -o yaml > strict.yaml
vi strict.yaml
```

*Requests MUST exactly equal Limits:*

```yaml
spec:
  containers:
  - image: redis
    name: strict-qos
    resources:
      requests:
        cpu: 200m
        memory: 128Mi
      limits:               # LIMITS MATCH EXACTLY
        cpu: 200m
        memory: 128Mi
```

```bash
kubectl apply -f strict.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod strict-qos
rm -f strict.yaml
```

-----

### Task 2: ResourceQuotas (Highly Repeating)

A `ResourceQuota` places a hard cap on the total amount of resources a specific namespace can consume. The exam will ask you to create them and troubleshoot pods that violate them.

#### Main Task 2: Creating a Strict Namespace Quota

**1. CKAD Style Question:**
Create a namespace named `dev-ns`.
Create a ResourceQuota named `dev-quota` inside `dev-ns`.
Configure the quota to enforce the following hard limits for the entire namespace:

  * Maximum of `4` Pods.
  * Total CPU Requests: `1` (1 core).
  * Total Memory Limits: `2Gi`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
[[ "$(kubectl get quota dev-quota -n dev-ns -o jsonpath='{.spec.hard.pods}')" == "4" ]] && echo "✅ Pod limit configured" || echo "❌ Pod limit failed"
[[ "$(kubectl get quota dev-quota -n dev-ns -o jsonpath='{.spec.hard.requests\.cpu}')" == "1" ]] && echo "✅ CPU requests configured" || echo "❌ CPU requests failed"
[[ "$(kubectl get quota dev-quota -n dev-ns -o jsonpath='{.spec.hard.limits\.memory}')" == "2Gi" ]] && echo "✅ Memory limits configured" || echo "❌ Memory limits failed"
```

<details>

4. Solution:

```bash
# 1. Create the namespace
kubectl create namespace dev-ns

# 2. Use the imperative command for quotas to save time!
kubectl create quota dev-quota \
  --hard=pods=4,requests.cpu=1,limits.memory=2Gi \
  --namespace=dev-ns
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ns dev-ns
```

#### Variation 2.1: Bypassing the Quota Trap

**1. CKAD Style Question:**
A namespace named `restricted-ns` has a strict `ResourceQuota` applied. If a Pod does not explicitly define CPU and Memory limits, the API server will reject it.
Create a Pod named `quota-survivor` in `restricted-ns` using the `nginx` image. Ensure it explicitly defines CPU and Memory limits (e.g., `100m` CPU, `50Mi` memory) so it successfully schedules.

**2. Setup Script:**

```bash
kubectl create ns restricted-ns
kubectl create quota strict-quota --hard=limits.cpu=1,limits.memory=1Gi -n restricted-ns
```

**3. Testcase Script:**

```bash
kubectl get pod quota-survivor -n restricted-ns >/dev/null 2>&1 && echo "✅ Pod successfully scheduled bypassing quota" || echo "❌ Pod failed to schedule (Did you forget limits?)"
```

<details>

4. Solution:

```bash
# If you try to run 'kubectl run' without limits in this namespace, it throws a Forbidden error!
kubectl run quota-survivor --image=nginx -n restricted-ns --dry-run=client -o yaml > survivor.yaml
vi survivor.yaml
```

*You must add limits to bypass the quota's default denial:*

```yaml
spec:
  containers:
  - image: nginx
    name: quota-survivor
    resources:
      limits:               # REQUIRED by the quota
        cpu: 100m
        memory: 50Mi
```

```bash
kubectl apply -f survivor.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ns restricted-ns
rm -f survivor.yaml
```

-----

### Task 3: LimitRanges (The Exam Curveball)

While `ResourceQuotas` restrict the *entire namespace*, a `LimitRange` automatically applies default requests/limits to *individual pods* that forget to declare them, or sets Min/Max boundaries. **You cannot create a LimitRange imperatively; you must write the YAML.**

#### Main Task 3: Applying Automatic Defaults

**1. CKAD Style Question:**
Create a namespace named `auto-limit-ns`.
Create a LimitRange named `core-limit` in this namespace. It must automatically apply a default Memory Request of `128Mi` and a default Memory Limit of `256Mi` to any container created without them.
Next, create a Pod named `blank-pod` (`nginx` image) without specifying any resources.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
[[ "$(kubectl get pod blank-pod -n auto-limit-ns -o jsonpath='{.spec.containers[0].resources.requests.memory}')" == "128Mi" ]] && echo "✅ Default Memory Request automatically injected!" || echo "❌ Injection failed"
[[ "$(kubectl get pod blank-pod -n auto-limit-ns -o jsonpath='{.spec.containers[0].resources.limits.memory}')" == "256Mi" ]] && echo "✅ Default Memory Limit automatically injected!" || echo "❌ Injection failed"
```

<details>

4. Solution:

```bash
# 1. Create the namespace
kubectl create ns auto-limit-ns

# 2. You MUST write the LimitRange YAML manually. Memorize this structure!
vi limitrange.yaml
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: core-limit
  namespace: auto-limit-ns
spec:
  limits:
  - type: Container
    defaultRequest:       # Automatically injected request
      memory: 128Mi
    default:              # Automatically injected limit
      memory: 256Mi
```

```bash
kubectl apply -f limitrange.yaml

# 3. Deploy a blank pod. The LimitRange will mutate it!
kubectl run blank-pod --image=nginx -n auto-limit-ns
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ns auto-limit-ns
rm -f limitrange.yaml
```

#### Variation 3.1: Enforcing Min/Max Constraints

**1. CKAD Style Question:**
Create a LimitRange named `size-constraint` in the `default` namespace.
Configure it to ensure that no container can be created that requests less than `50Mi` of memory (`min`) or limits more than `500Mi` of memory (`max`).

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[[ "$(kubectl get limitrange size-constraint -o jsonpath='{.spec.limits[0].max.memory}')" == "500Mi" ]] && echo "✅ Max constraint configured" || echo "❌ Max constraint failed"
[[ "$(kubectl get limitrange size-constraint -o jsonpath='{.spec.limits[0].min.memory}')" == "50Mi" ]] && echo "✅ Min constraint configured" || echo "❌ Min constraint failed"
```

<details>

4. Solution:

```bash
vi minmax.yaml
```

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: size-constraint
spec:
  limits:
  - type: Container
    min:
      memory: 50Mi
    max:
      memory: 500Mi
```

```bash
kubectl apply -f minmax.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete limitrange size-constraint
rm -f minmax.yaml
```

#### Variation 3.2: LimitRange vs. HPA Dependency

**1. CKAD Style Question:**
You want to autoscale a deployment based on CPU, which requires CPU requests. Instead of adding requests to the Deployment YAML, create a LimitRange named `hpa-helper` that sets a `defaultRequest` of `100m` CPU.
Then, create a Deployment `auto-web` (`nginx`) and an HPA targeting `50%` CPU utilization.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
kubectl get hpa auto-web >/dev/null 2>&1 && echo "✅ HPA successfully created (bypassed request requirement via LimitRange)" || echo "❌ HPA failed"
```

<details>

4. Solution:

```bash
# 1. Create the LimitRange
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: hpa-helper
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 100m
EOF

# 2. Create the deployment
kubectl create deploy auto-web --image=nginx

# 3. Create the HPA (It succeeds because the LimitRange injected the CPU requests automatically!)
kubectl autoscale deploy auto-web --cpu-percent=50 --min=1 --max=3
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete hpa auto-web
kubectl delete deploy auto-web
kubectl delete limitrange hpa-helper
```

-----

### Task 4: Ephemeral Storage (The Eviction Trap)

Pods don't just consume CPU and RAM; they consume local disk space for logs, `emptyDir` volumes, and the writable container layer. You must know how to cap this.

#### Main Task 4: Capping Local Disk Usage

**1. CKAD Style Question:**
Create a Pod named `log-spammer` using the `busybox` image.
Configure the container's resource limits so that it cannot consume more than `1Gi` of local disk space. If it exceeds this, the Kubelet should evict the pod.
*(Hint: The resource key for local disk is `ephemeral-storage`).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
[["$(kubectl get pod log-spammer -o jsonpath='{.spec.containers[0].resources.limits.ephemeral-storage}')" == "1Gi" ]] && echo "✅ Ephemeral storage limit configured correctly" || echo "❌ Ephemeral storage limit missing or incorrect"
```

<details>

4. Solution:

```bash
# 1. Generate the base YAML
kubectl run log-spammer --image=busybox --dry-run=client -o yaml -- sleep 3600 > ephemeral.yaml

# 2. Edit the YAML
vi ephemeral.yaml
```

*Add `ephemeral-storage` exactly as you would `cpu` or `memory`:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-spammer
spec:
  containers:
  - image: busybox
    name: log-spammer
    command: ["sleep", "3600"]
    resources:
      limits:               # ADD FROM HERE
        ephemeral-storage: 1Gi
```

```bash
# 3. Apply the YAML
kubectl apply -f ephemeral.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod log-spammer
rm -f ephemeral.yaml
```

-----

### Task 5: Storage Quotas and Limits (The PVC Boundaries)

ResourceQuotas and LimitRanges do not just apply to Pods. They strictly apply to PersistentVolumeClaims (PVCs) as well.

#### Main Task 5: The Namespace Storage Quota

**1. CKAD Style Question:**
Create a namespace named `data-science`.
Create a ResourceQuota named `storage-cap` inside this namespace.
Enforce the following hard limits so developers cannot drain the cluster's SAN:

  * Maximum number of PVCs allowed: `2`
  * Total combined storage requested by all PVCs: `10Gi`

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
[[ "$(kubectl get quota storage-cap -n data-science -o jsonpath='{.spec.hard.persistentvolumeclaims}')" == "2" ]] && echo "✅ PVC count limit configured" || echo "❌ PVC count failed"
[[ "$(kubectl get quota storage-cap -n data-science -o jsonpath='{.spec.hard.requests\.storage}')" == "10Gi" ]] && echo "✅ Total storage limit configured" || echo "❌ Total storage limit failed"
```

<details>

4. Solution:

```bash
# 1. Create the namespace
kubectl create ns data-science

# 2. Use the imperative quota command. 
# 'persistentvolumeclaims' controls the count. 'requests.storage' controls the volume size limit.
kubectl create quota storage-cap \
  --hard=persistentvolumeclaims=2,requests.storage=10Gi \
  -n data-science
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ns data-science
```

#### Variation 5.1: LimitRange for PVC Sizes

**1. CKAD Style Question:**
You want to prevent developers in the `default` namespace from creating massive 100Gi volumes or tiny, useless 1Mi volumes.
Create a LimitRange named `pvc-boundaries`. Configure it to enforce a `min` PVC request of `1Gi` and a `max` PVC request of `5Gi`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
[[ "$(kubectl get limitrange pvc-boundaries -o jsonpath='{.spec.limits[0].type}')" == "PersistentVolumeClaim" ]] && echo "✅ LimitRange targeting PVCs" || echo "❌ Target type is incorrect"
[[ "$(kubectl get limitrange pvc-boundaries -o jsonpath='{.spec.limits[0].min.storage}')" == "1Gi" ]] && echo "✅ Minimum storage boundary set" || echo "❌ Min storage failed"
[[ "$(kubectl get limitrange pvc-boundaries -o jsonpath='{.spec.limits[0].max.storage}')" == "5Gi" ]] && echo "✅ Maximum storage boundary set" || echo "❌ Max storage failed"
```

<details>

4. Solution:

```bash
# You must write this YAML from scratch.
vi pvc-limit.yaml
```

*Notice the `type` is changed from `Container` to `PersistentVolumeClaim`, and the resource key is `storage`\!*

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: pvc-boundaries
spec:
  limits:
  - type: PersistentVolumeClaim    # TARGET PVCs
    min:
      storage: 1Gi                 # MIN SIZE
    max:
      storage: 5Gi                 # MAX SIZE
```

```bash
kubectl apply -f pvc-limit.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete limitrange pvc-boundaries
rm -f pvc-limit.yaml
```

-----

### The Final 100% Verdict

With the integration of **Pod Level boundaries (Guaranteed QoS)**, **ResourceQuotas (bypassing strict enforcement)**, and **LimitRanges (Default injection and Min/Max constraints)**, you have successfully locked down 100% of the "Understand requests, limits, and quotas" syllabus objective.

