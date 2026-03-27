## understand requests, limits and quotas 

---

### Task 1: The Readiness Probe (Most Repeating)
Readiness probes tell Kubernetes when your Pod is ready to be added to a Service and receive user traffic. If it fails, the Pod isn't killed; it's just temporarily removed from the network balancer.

**1. CKAD Style Question:**
Create a Pod named `web-ready` in the `default` namespace using the `nginx:alpine` image.
Configure a Readiness Probe that checks the HTTP path `/` on port `80`. 
The probe must have an initial delay of `5` seconds and check every `10` seconds.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get pod web-ready -o jsonpath='{.spec.containers[0].readinessProbe.httpGet.path}')" == "/" ] && echo "✅ Readiness path is /" || echo "❌ Readiness path failed"
[ "$(kubectl get pod web-ready -o jsonpath='{.spec.containers[0].readinessProbe.initialDelaySeconds}')" == "5" ] && echo "✅ Initial delay is 5s" || echo "❌ Initial delay failed"
[ "$(kubectl get pod web-ready -o jsonpath='{.spec.containers[0].readinessProbe.periodSeconds}')" == "10" ] && echo "✅ Period is 10s" || echo "❌ Period failed"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base YAML
kubectl run web-ready --image=nginx:alpine --dry-run=client -o yaml > readiness.yaml

# 2. Edit the YAML
vi readiness.yaml
```
*Add the `readinessProbe` block at the same indentation level as `image` and `name`:*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-ready
spec:
  containers:
  - image: nginx:alpine
    name: web-ready
    readinessProbe:             # ADD FROM HERE
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10         # TO HERE
```
```bash
# 3. Apply the YAML
kubectl apply -f readiness.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod web-ready
rm readiness.yaml
```

---

### Task 2: The Liveness Probe with Exec (Highly Repeating)
Liveness probes tell Kubernetes if your container has crashed or deadlocked. If it fails, Kubernetes ruthlessly murders the container and restarts it. The exam frequently tests the `exec` command variant of this.

**1. CKAD Style Question:**
Create a Pod named `health-check` using the `busybox` image. 
The container should run the following command: `/bin/sh, -c, touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600`.
Configure a Liveness Probe that executes the command `cat /tmp/healthy`. 
Set the initial delay to `5` seconds and the period to `5` seconds.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get pod health-check -o jsonpath='{.spec.containers[0].livenessProbe.exec.command[0]}')" == "cat" ] && echo "✅ Liveness exec command configured" || echo "❌ Liveness command failed"
# If you wait about 45 seconds after creating the pod, you will see the RESTARTS count increase!
echo "Current Restarts: $(kubectl get pod health-check -o jsonpath='{.status.containerStatuses[0].restartCount}')"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base YAML
kubectl run health-check --image=busybox --dry-run=client -o yaml --command -- /bin/sh -c "touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600" > liveness.yaml

# 2. Edit the YAML
vi liveness.yaml
```
*Add the `livenessProbe` block:*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: health-check
spec:
  containers:
  - image: busybox
    name: health-check
    command:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:              # ADD FROM HERE
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5          # TO HERE
```
```bash
# 3. Apply the YAML
kubectl apply -f liveness.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod health-check
rm liveness.yaml
```

---

### Task 3: The Startup Probe (Moderately Repeating)
Sometimes legacy applications take a long time to boot up. If you use a Liveness probe, it might kill the app before it even finishes starting! A Startup probe disables the other probes until the app is fully online.

**1. CKAD Style Question:**
Create a Pod named `slow-app` using the `nginx` image. 
Configure a Startup Probe that checks the HTTP path `/` on port `80`. 
Because this app is "slow", configure the probe to try up to `30` times (`failureThreshold`), checking every `10` seconds (`periodSeconds`). This gives the app a maximum of 5 minutes to start.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get pod slow-app -o jsonpath='{.spec.containers[0].startupProbe.httpGet.path}')" == "/" ] && echo "✅ Startup path is correct" || echo "❌ Startup path failed"
[ "$(kubectl get pod slow-app -o jsonpath='{.spec.containers[0].startupProbe.failureThreshold}')" == "30" ] && echo "✅ Failure threshold is 30" || echo "❌ Failure threshold failed"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base YAML
kubectl run slow-app --image=nginx --dry-run=client -o yaml > startup.yaml

# 2. Edit the YAML
vi startup.yaml
```
*Add the `startupProbe` block:*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-app
spec:
  containers:
  - image: nginx
    name: slow-app
    startupProbe:               # ADD FROM HERE
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 10         # TO HERE
```
```bash
# 3. Apply the YAML
kubectl apply -f startup.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod slow-app
rm startup.yaml
```

---

If you can jump into YAML and quickly type out those three probe blocks (`readinessProbe`, `livenessProbe`, `startupProbe`), you have totally mastered this segment.
