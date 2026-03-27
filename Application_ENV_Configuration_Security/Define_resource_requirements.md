## Define resource requiremnets

---

### Task 1: The Multi-Container Resource Math
The exam loves to test if you understand how Kubernetes calculates resources for a Pod with both InitContainers and regular containers. (Hint: It takes the *highest* InitContainer requirement, compares it to the *sum* of the regular containers, and reserves whichever is larger).

**1. CKAD Style Question:**
Create a Pod named `boot-heavy` in the `default` namespace.
* **InitContainer:** Name it `db-setup`, use the `busybox` image, and command `sleep 5`. It must **request** `500m` CPU and `128Mi` Memory.
* **Main Container:** Name it `app-server`, use the `nginx` image. It must **request** `200m` CPU and `64Mi` Memory.
* *Note: Because of how K8s math works, this Pod will require 500m CPU to schedule, not 700m.*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get pod boot-heavy -o jsonpath='{.spec.initContainers[0].resources.requests.cpu}')" == "500m" ] && echo "✅ InitContainer CPU correct" || echo "❌ InitContainer CPU failed"
[ "$(kubectl get pod boot-heavy -o jsonpath='{.spec.containers[0].resources.requests.memory}')" == "64Mi" ] && echo "✅ Main Container Memory correct" || echo "❌ Main Container Memory failed"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base YAML
kubectl run boot-heavy --image=nginx --dry-run=client -o yaml > boot.yaml

# 2. Edit the YAML
vi boot.yaml
```
*Add the initContainer and the resources blocks:*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: boot-heavy
spec:
  initContainers:
  - name: db-setup
    image: busybox
    command: ["sleep", "5"]
    resources:
      requests:
        cpu: 500m
        memory: 128Mi
  containers:
  - image: nginx
    name: app-server
    resources:
      requests:
        cpu: 200m
        memory: 64Mi
```
```bash
# 3. Apply the YAML
kubectl apply -f boot.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod boot-heavy
rm boot.yaml
```

---

### Task 2: The Autoscaler Dependency (The Imperative Trick)
You **cannot** autoscale a Deployment based on CPU usage unless you have explicitly defined CPU `requests` first. The exam will ask you to do both simultaneously. Here is how you do it entirely from the command line without writing any YAML.

**1. CKAD Style Question:**
Create a Deployment named `dynamic-web` using the `nginx` image.
Define the resource requirements so that each container **requests** exactly `100m` of CPU.
Next, create a Horizontal Pod Autoscaler (HPA) for this deployment. The HPA should scale up when average CPU utilization hits `50%`, with a minimum of `1` replica and a maximum of `3` replicas.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get deploy dynamic-web -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')" == "100m" ] && echo "✅ CPU Request configured" || echo "❌ CPU Request failed"
[ "$(kubectl get hpa dynamic-web -o jsonpath='{.spec.maxReplicas}')" == "3" ] && echo "✅ HPA Max Replicas: 3" || echo "❌ HPA Max Replicas failed"
[ "$(kubectl get hpa dynamic-web -o jsonpath='{.spec.metrics[0].resource.target.averageUtilization}')" == "50" ] && echo "✅ HPA Target CPU: 50%" || echo "❌ HPA Target CPU failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the deployment
kubectl create deploy dynamic-web --image=nginx

# 2. Set the resource requests IMPERATIVELY (This saves you from opening vim!)
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

---

By adding the multi-container math and the HPA imperative commands to what we covered in the "Quotas and Limits" section, you have definitively walled off every possible angle of the "Resource Requirements" topic.
