## Define Resource Requirements

### Task 1: Baseline Requests, Limits, and QoS (Most Repeating)

The absolute foundation of this topic is defining exactly what a pod *needs* to start (Request) and the absolute maximum it is *allowed* to use before being throttled or killed (Limit).

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
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get pod resource-pod -o jsonpath='{.spec.containers[0].resources.requests.cpu}')" == "100m" ] && echo "✅ CPU Request configured" || echo "❌ CPU Request failed"
[ "$(kubectl get pod resource-pod -o jsonpath='{.spec.containers[0].resources.limits.memory}')" == "128Mi" ] && echo "✅ Memory Limit configured" || echo "❌ Memory Limit failed"
```

<details>
  
4. Solution:

```bash
# 1. Generate the base YAML
kubectl run resource-pod --image=nginx:alpine --dry-run=client -o yaml > resource.yaml

# 2. Edit the YAML to add the resources block
vi resource.yaml
```

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

#### Variation 1.1: Live Patching a Deployment

**1. CKAD Style Question:**
A Deployment named `web-res` is running but currently has no resource limits defined.
Using an imperative command (do not edit the YAML), update the Deployment so that its containers have a **Memory Limit** of `256Mi` and a **CPU Limit** of `500m`.

**2. Setup Script:**

```bash
kubectl create deploy web-res --image=nginx:alpine
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get deploy web-res -o jsonpath='{.spec.template.spec.containers[0].resources.limits.memory}')" == "256Mi" ] && echo "✅ Memory Limit patched" || echo "❌ Memory Limit failed"
[ "$(kubectl get deploy web-res -o jsonpath='{.spec.template.spec.containers[0].resources.limits.cpu}')" == "500m" ] && echo "✅ CPU Limit patched" || echo "❌ CPU Limit failed"
```

<details>
  
4. Solution:

```bash
# Use the 'set resources' command to patch live deployments instantly
kubectl set resources deploy web-res --limits=cpu=500m,memory=256Mi
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy web-res
```

#### Variation 1.2: Achieving "Guaranteed" QoS

**1. CKAD Style Question:**
Kubernetes assigns Quality of Service (QoS) classes based on resources.
Create a Pod named `strict-qos` using the `redis` image. You must configure its resources so that Kubernetes automatically assigns it the **"Guaranteed"** QoS class.
*(Hint: A pod is Guaranteed only if every container has identical CPU and Memory Requests AND Limits).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get pod strict-qos -o jsonpath='{.status.qosClass}')" == "Guaranteed" ] && echo "✅ QoS Class is Guaranteed!" || echo "❌ QoS Class is NOT Guaranteed"
```

<details>

4. Solution:

```bash
kubectl run strict-qos --image=redis --dry-run=client -o yaml > strict.yaml
vi strict.yaml
```

*(To achieve Guaranteed QoS, Requests MUST exactly equal Limits):*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: strict-qos
spec:
  containers:
  - image: redis
    name: strict-qos
    resources:
      requests:
        cpu: 200m
        memory: 128Mi
      limits:               # LIMITS MUST MATCH REQUESTS EXACTLY
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

### Task 2: The Multi-Container Resource Math

The exam loves to test if you understand how Kubernetes calculates resources for a Pod with both InitContainers and multiple regular containers.

#### Main Task 2: InitContainer vs. Main Container Math

**1. CKAD Style Question:**
Create a Pod named `boot-heavy` in the `default` namespace.

  * **InitContainer:** Name it `db-setup`, use the `busybox` image, and command `sleep 5`. It must **request** `500m` CPU and `128Mi` Memory.
  * **Main Container:** Name it `app-server`, use the `nginx` image. It must **request** `200m` CPU and `64Mi` Memory.
    *(Note: Because of K8s math, this Pod takes the highest InitContainer request or the sum of Main containers, whichever is larger).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get pod boot-heavy -o jsonpath='{.spec.initContainers[0].resources.requests.cpu}')" == "500m" ] && echo "✅ InitContainer CPU correct" || echo "❌ InitContainer CPU failed"
[ "$(kubectl get pod boot-heavy -o jsonpath='{.spec.containers[0].resources.requests.memory}')" == "64Mi" ] && echo "✅ Main Container Memory correct" || echo "❌ Main Container Memory failed"
```

<details>
  
4. Solution:

```bash
kubectl run boot-heavy --image=nginx --dry-run=client -o yaml > boot.yaml
vi boot.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: boot-heavy
spec:
  initContainers:           # INIT BLOCK
  - name: db-setup
    image: busybox
    command: ["sleep", "5"]
    resources:
      requests:
        cpu: 500m
        memory: 128Mi
  containers:               # MAIN BLOCK
  - image: nginx
    name: app-server
    resources:
      requests:
        cpu: 200m
        memory: 64Mi
```

```bash
kubectl apply -f boot.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod boot-heavy
rm -f boot.yaml
```

#### Variation 2.1: Multi-Container Summation

**1. CKAD Style Question:**
Create a Pod named `dual-worker`. It must contain two containers:

  * Container 1: Name `worker-a`, image `busybox`, command `sleep 3600`, requesting `100m` CPU.
  * Container 2: Name `worker-b`, image `busybox`, command `sleep 3600`, requesting `200m` CPU.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get pod dual-worker -o jsonpath='{.spec.containers[0].resources.requests.cpu}')" == "100m" ] && echo "✅ Worker A CPU correct" || echo "❌ Worker A CPU failed"
[ "$(kubectl get pod dual-worker -o jsonpath='{.spec.containers[1].resources.requests.cpu}')" == "200m" ] && echo "✅ Worker B CPU correct" || echo "❌ Worker B CPU failed"
```

<details>
  
4. Solution:

```bash
kubectl run dual-worker --image=busybox --dry-run=client -o yaml -- sleep 3600 > dual.yaml
vi dual.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dual-worker
spec:
  containers:
  - image: busybox
    name: worker-a
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: 100m
  - image: busybox          # ADD SECOND CONTAINER
    name: worker-b
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: 200m
```

```bash
kubectl apply -f dual.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod dual-worker
rm -f dual.yaml
```

-----

### Task 3: Resource Integrations (HPAs and Quotas)

Setting resource requests isn't just for scheduling; it is a strict prerequisite for Horizontal Pod Autoscalers (HPAs) and bypassing Namespace ResourceQuotas.

#### Main Task 3: The Autoscaler Dependency

**1. CKAD Style Question:**
Create a Deployment named `dynamic-web` using the `nginx` image.
Define the resource requirements so that each container **requests** exactly `100m` of CPU.
Next, create a Horizontal Pod Autoscaler (HPA) for this deployment. The HPA should scale up when average CPU utilization hits `50%`, with a minimum of `1` replica and a maximum of `3` replicas.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(kubectl get deploy dynamic-web -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')" == "100m" ] && echo "✅ CPU Request configured" || echo "❌ CPU Request failed"
[ "$(kubectl get hpa dynamic-web -o jsonpath='{.spec.maxReplicas}')" == "3" ] && echo "✅ HPA Max Replicas: 3" || echo "❌ HPA Max Replicas failed"
[ "$(kubectl get hpa dynamic-web -o jsonpath='{.spec.metrics[0].resource.target.averageUtilization}')" == "50" ] && echo "✅ HPA Target CPU: 50%" || echo "❌ HPA Target CPU failed"
```

<details>

4. Solution:

```bash
# 1. Create the deployment
kubectl create deploy dynamic-web --image=nginx

# 2. Set the resource requests IMPERATIVELY (This satisfies the HPA requirement)
kubectl set resources deploy dynamic-web --requests=cpu=100m

# 3. Create the HPA imperatively
kubectl autoscale deploy dynamic-web --cpu-percent=50 --min=1 --max=3
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete hpa dynamic-web
kubectl delete deploy dynamic-web
```

#### Variation 3.1: Surviving a ResourceQuota

**1. CKAD Style Question:**
A namespace named `restricted-ns` has a strict `ResourceQuota` applied. If a Pod does not explicitly define CPU and Memory limits, the API server will reject it.
Create a Pod named `quota-survivor` in `restricted-ns` using the `nginx` image. Ensure it requests `100m` CPU / `50Mi` Mem, and limits `200m` CPU / `100Mi` Mem so it is accepted by the namespace quota.

**2. Setup Script:**

```bash
kubectl create ns restricted-ns
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: strict-quota
  namespace: restricted-ns
spec:
  hard:
    limits.cpu: "1"
    limits.memory: 1Gi
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
kubectl get pod quota-survivor -n restricted-ns >/dev/null 2>&1 && echo "✅ Pod successfully scheduled bypassing quota" || echo "❌ Pod failed to schedule (Missing limits?)"
```

<details>

4. Solution:

```bash
# If you try to run 'kubectl run' without limits in this namespace, it fails!
kubectl run quota-survivor --image=nginx -n restricted-ns --dry-run=client -o yaml > quota-pod.yaml
vi quota-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-survivor
  namespace: restricted-ns
spec:
  containers:
  - image: nginx
    name: quota-survivor
    resources:              # MUST ADD BOTH REQUESTS AND LIMITS
      requests:
        cpu: 100m
        memory: 50Mi
      limits:
        cpu: 200m
        memory: 100Mi
```

```bash
kubectl apply -f quota-pod.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod quota-survivor -n restricted-ns
kubectl delete ns restricted-ns
rm -f quota-pod.yaml
```
