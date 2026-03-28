## Probes and Health Checks

In Kubernetes, **Readiness** dictates if a Pod gets network traffic (it removes it from the Service endpoint if it fails). **Liveness** dictates if a Pod needs to be murdered and restarted. **Startup** shields slow applications from being killed by Liveness before they even finish booting.

---

### Task 1: The HTTP Readiness Probe (The Most Repeating)
The absolute most common probe question on the exam is ensuring an HTTP application is ready to receive web traffic.

**1. CKAD Style Question:**
Create a Pod named `web-ready` using the `nginx:alpine` image.
Configure a `readinessProbe` that performs an HTTP GET request to the path `/` on port `80`.
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
# 1. Generate base YAML
kubectl run web-ready --image=nginx:alpine --dry-run=client -o yaml > task1.yaml

# 2. Edit YAML to add the readinessProbe at the container level
vi task1.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-ready
spec:
  containers:
  - image: nginx:alpine
    name: web-ready
    readinessProbe:             # ADD THIS BLOCK
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
```
```bash
# 3. Apply the YAML
kubectl apply -f task1.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod web-ready; rm task1.yaml
```

#### Variation 1.1: Live Patching a Deployment
**1. CKAD Style Question:**
A Deployment `api-deploy` is currently running. Add a `readinessProbe` to its container that checks the path `/health` on port `8080`.

**2. Setup Script:**
```bash
kubectl create deployment api-deploy --image=nginx:alpine
```

**3. Testcase Script:**
```bash
[ "$(kubectl get deploy api-deploy -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.httpGet.path}')" == "/health" ] && echo "✅ Probe successfully added to Deployment" || echo "❌ Probe missing"
```

<details>

**4. Solution:**
```bash
# Edit the live deployment. Scroll down to the container spec.
kubectl edit deploy api-deploy
# Add:
#         readinessProbe:
#           httpGet:
#             path: /health
#             port: 8080
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy api-deploy
```

#### Variation 1.2: Custom HTTP Headers
**1. CKAD Style Question:**
Create a Pod `secure-probe` (image `nginx`). Its `readinessProbe` must check `/` on port `80`, but it must also pass an HTTP header named `X-Custom-Header` with the value `Awesome`.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get pod secure-probe -o jsonpath='{.spec.containers[0].readinessProbe.httpGet.httpHeaders[0].name}')" == "X-Custom-Header" ] && echo "✅ Custom header configured" || echo "❌ Header missing"
```

<details>

**4. Solution:**
```bash
kubectl run secure-probe --image=nginx --dry-run=client -o yaml > var12.yaml
vi var12.yaml
# Add under the container spec:
#     readinessProbe:
#       httpGet:
#         path: /
#         port: 80
#         httpHeaders:
#         - name: X-Custom-Header
#           value: Awesome
kubectl apply -f var12.yaml
```

</details>

**5. Clean-up Script:** `kubectl delete pod secure-probe; rm var12.yaml`

---

### Task 2: The Exec Liveness Probe (The Command Checker)
If your app doesn't have an HTTP endpoint, you can have Kubernetes execute a shell command inside the container. If the command returns an exit code of `0`, it's healthy. Anything else triggers a restart.

**1. CKAD Style Question:**
Create a Pod named `worker-liveness` using the `busybox` image running `sleep 3600`.
Configure a `livenessProbe` that executes the command `cat /tmp/healthy`.
Set the initial delay to `5` seconds and the period to `5` seconds.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get pod worker-liveness -o jsonpath='{.spec.containers[0].livenessProbe.exec.command[0]}')" == "cat" ] && echo "✅ Exec command configured" || echo "❌ Command failed"
[ "$(kubectl get pod worker-liveness -o jsonpath='{.spec.containers[0].livenessProbe.periodSeconds}')" == "5" ] && echo "✅ Period configured" || echo "❌ Period failed"
```

<details>

**4. Solution:**
```bash
kubectl run worker-liveness --image=busybox --dry-run=client -o yaml -- sleep 3600 > task2.yaml
vi task2.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: worker-liveness
spec:
  containers:
  - image: busybox
    name: worker-liveness
    command: ["sleep", "3600"]
    livenessProbe:              # ADD THIS BLOCK
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
```bash
kubectl apply -f task2.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod worker-liveness; rm task2.yaml
```

#### Variation 2.1: Complex Shell Execution
**1. CKAD Style Question:**
Create Pod `complex-probe` (image `busybox`, command `sleep 3600`). The liveness probe must run: `sh -c 'ps aux | grep sleep'`.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get pod complex-probe -o jsonpath='{.spec.containers[0].livenessProbe.exec.command[2]}')" == "ps aux | grep sleep" ] && echo "✅ Complex shell command configured" || echo "❌ Command failed"
```

<details>

**4. Solution:**
```bash
kubectl run complex-probe --image=busybox --dry-run=client -o yaml -- sleep 3600 > var21.yaml
vi var21.yaml
# Add under container:
#     livenessProbe:
#       exec:
#         command:
#         - sh
#         - -c
#         - "ps aux | grep sleep"
kubectl apply -f var21.yaml
```

</details>

**5. Clean-up Script:** `kubectl delete pod complex-probe; rm var21.yaml`

#### Variation 2.2: Combined Liveness & Readiness
**1. CKAD Style Question:**
Create Pod `dual-probe` (`nginx`). Add a `readinessProbe` (HTTP GET `/` on `80`) AND a `livenessProbe` (TCP Socket on `80`).

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get pod dual-probe -o jsonpath='{.spec.containers[0].livenessProbe.tcpSocket.port}')" == "80" ] && echo "✅ Liveness TCP port 80 found" || echo "❌ Liveness failed"
[ "$(kubectl get pod dual-probe -o jsonpath='{.spec.containers[0].readinessProbe.httpGet.path}')" == "/" ] && echo "✅ Readiness HTTP found" || echo "❌ Readiness failed"
```

<details>

**4. Solution:**
```bash
kubectl run dual-probe --image=nginx --dry-run=client -o yaml > var22.yaml
vi var22.yaml
# Add BOTH blocks under container:
#     readinessProbe:
#       httpGet:
#         path: /
#         port: 80
#     livenessProbe:
#       tcpSocket:
#         port: 80
kubectl apply -f var22.yaml
```

</details>

**5. Clean-up Script:** `kubectl delete pod dual-probe; rm var22.yaml`

---

### Task 3: The TCP Socket Probe (Database/Cache Checks)
If you are running a database or a caching layer that doesn't speak HTTP, you use a TCP probe just to check if the port accepts connections.

**1. CKAD Style Question:**
Create a Pod named `cache-db` using the `redis:alpine` image.
Configure a `livenessProbe` that checks if the TCP socket on port `6379` is open.
Set the `timeoutSeconds` to `2`.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get pod cache-db -o jsonpath='{.spec.containers[0].livenessProbe.tcpSocket.port}')" == "6379" ] && echo "✅ TCP Socket on 6379 configured" || echo "❌ TCP Socket failed"
[ "$(kubectl get pod cache-db -o jsonpath='{.spec.containers[0].livenessProbe.timeoutSeconds}')" == "2" ] && echo "✅ Timeout configured to 2s" || echo "❌ Timeout failed"
```

<details>

**4. Solution:**
```bash
kubectl run cache-db --image=redis:alpine --dry-run=client -o yaml > task3.yaml
vi task3.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-db
spec:
  containers:
  - image: redis:alpine
    name: cache-db
    livenessProbe:              # ADD THIS BLOCK
      tcpSocket:
        port: 6379
      timeoutSeconds: 2
```
```bash
kubectl apply -f task3.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod cache-db; rm task3.yaml
```

#### Variation 3.1: Named Ports
**1. CKAD Style Question:**
Create Pod `named-probe` (`nginx`). Name the container's port `80` as `web-port`. Configure a `readinessProbe` to use the TCP socket targeting the string name `web-port` instead of the number.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get pod named-probe -o jsonpath='{.spec.containers[0].readinessProbe.tcpSocket.port}')" == "web-port" ] && echo "✅ Probe uses named port" || echo "❌ Named port failed"
```

<details>

**4. Solution:**
```bash
kubectl run named-probe --image=nginx --port=80 --dry-run=client -o yaml > var31.yaml
vi var31.yaml
# Edit ports and add probe:
#     ports:
#     - containerPort: 80
#       name: web-port
#     readinessProbe:
#       tcpSocket:
#         port: web-port
kubectl apply -f var31.yaml
```

</details>

**5. Clean-up Script:** `kubectl delete pod named-probe; rm var31.yaml`


#### Variation 3.2: Threshold Tuning
**1. CKAD Style Question:**
Modify a TCP `livenessProbe` on port `80` for an existing pod named `strict-probe` (running `nginx`). It is currently failing too quickly. Update the probe so it must fail `5` times before restarting (`failureThreshold`), but only requires `1` success to be considered healthy (`successThreshold`).

**2. Setup Script:**
```bash
# We must create the pod with the baseline probe first
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: strict-probe
spec:
  containers:
  - image: nginx
    name: strict-probe
    livenessProbe:
      tcpSocket:
        port: 80
      failureThreshold: 3
      successThreshold: 1
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.2 ---"
[ "$(kubectl get pod strict-probe -o jsonpath='{.spec.containers[0].livenessProbe.failureThreshold}')" == "5" ] && echo "✅ failureThreshold tuned to 5" || echo "❌ failureThreshold is incorrect"
[ "$(kubectl get pod strict-probe -o jsonpath='{.spec.containers[0].livenessProbe.successThreshold}')" == "1" ] && echo "✅ successThreshold is 1" || echo "❌ successThreshold is incorrect"
```

<details>

**4. Solution:**
```bash
# Note: You generally cannot live-patch probe fields on a bare Pod. 
# You must extract the YAML, delete the Pod, edit the YAML, and recreate it.
kubectl get pod strict-probe -o yaml > strict.yaml
kubectl delete pod strict-probe --force

vi strict.yaml
```
*(In vim, strip out the cluster-specific metadata like `uid` and `resourceVersion`, then modify the probe thresholds):*
```yaml
    livenessProbe:
      tcpSocket:
        port: 80
      failureThreshold: 5     # Change from 3 to 5
      successThreshold: 1     # Leave as 1 (Liveness probes MUST be 1)
```
```bash
kubectl apply -f strict.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod strict-probe
rm strict.yaml
```


---

### Task 4: The Startup Probe (The Time Shield)
Legacy Java apps or heavy databases take a long time to boot. If a `livenessProbe` checks too early, it will kill the pod while it's still booting! A `startupProbe` disables the other probes until it succeeds.

**1. CKAD Style Question:**
Create a Pod named `slow-boot` using the `nginx` image.
It takes up to 2 minutes to start. Create a `startupProbe` that checks HTTP GET `/` on port `80`.
Configure the probe to check every `10` seconds, with a `failureThreshold` of `12` (10s * 12 = 120 seconds buffer).

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 4 ---"
[ "$(kubectl get pod slow-boot -o jsonpath='{.spec.containers[0].startupProbe.failureThreshold}')" == "12" ] && echo "✅ failureThreshold is 12" || echo "❌ Threshold incorrect"
[ "$(kubectl get pod slow-boot -o jsonpath='{.spec.containers[0].startupProbe.periodSeconds}')" == "10" ] && echo "✅ periodSeconds is 10" || echo "❌ Period incorrect"
```

<details>

**4. Solution:**
```bash
kubectl run slow-boot --image=nginx --dry-run=client -o yaml > task4.yaml
vi task4.yaml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: slow-boot
spec:
  containers:
  - image: nginx
    name: slow-boot
    startupProbe:               # ADD THIS BLOCK
      httpGet:
        path: /
        port: 80
      failureThreshold: 12
      periodSeconds: 10
```
```bash
kubectl apply -f task4.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod slow-boot; rm task4.yaml
```

---

#### Variation 4.1: The Triple Threat (Startup + Liveness + Readiness)
**1. CKAD Style Question:**
Create a Pod named `triple-app` using the `nginx` image. Configure all three probes simultaneously:
1. `startupProbe`: HTTP GET to `/` on port `80`, failure threshold `10`, period `5` seconds.
2. `livenessProbe`: HTTP GET to `/` on port `80`, failure threshold `3`, period `10` seconds.
3. `readinessProbe`: TCP Socket on port `80`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 4.1 ---"
[ "$(kubectl get pod triple-app -o jsonpath='{.spec.containers[0].startupProbe.failureThreshold}')" == "10" ] && echo "✅ startupProbe configured correctly" || echo "❌ startupProbe failed"
[ "$(kubectl get pod triple-app -o jsonpath='{.spec.containers[0].livenessProbe.periodSeconds}')" == "10" ] && echo "✅ livenessProbe configured correctly" || echo "❌ livenessProbe failed"
[ "$(kubectl get pod triple-app -o jsonpath='{.spec.containers[0].readinessProbe.tcpSocket.port}')" == "80" ] && echo "✅ readinessProbe configured correctly" || echo "❌ readinessProbe failed"
```

<details>

**4. Solution:**
```bash
kubectl run triple-app --image=nginx --dry-run=client -o yaml > triple.yaml

vi triple.yaml
```
*(Stack all three blocks securely under the container spec):*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: triple-app
spec:
  containers:
  - image: nginx
    name: triple-app
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 10
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 3
      periodSeconds: 10
    readinessProbe:
      tcpSocket:
        port: 80
```
```bash
kubectl apply -f triple.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod triple-app
rm triple.yaml
```

---

#### Variation 4.2: Port Mismatch Trap
**1. CKAD Style Question:**
Create a Pod named `mismatch-pod` using the `nginx` image. 
Explicitly expose the `containerPort` as `8080`. However, the health-check daemon runs on a different port. Configure a `startupProbe` using HTTP GET to the path `/healthz` on port `9090`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 4.2 ---"
[ "$(kubectl get pod mismatch-pod -o jsonpath='{.spec.containers[0].ports[0].containerPort}')" == "8080" ] && echo "✅ Container port is 8080" || echo "❌ Container port failed"
[ "$(kubectl get pod mismatch-pod -o jsonpath='{.spec.containers[0].startupProbe.httpGet.port}')" == "9090" ] && echo "✅ Probe successfully pointing to alternative port 9090" || echo "❌ Probe port failed"
```

<details>

**4. Solution:**
```bash
kubectl run mismatch-pod --image=nginx --port=8080 --dry-run=client -o yaml > mismatch.yaml

vi mismatch.yaml
```
*(Add the startup probe pointing to the alternate port):*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mismatch-pod
spec:
  containers:
  - image: nginx
    name: mismatch-pod
    ports:
    - containerPort: 8080
    startupProbe:             # Add this block
      httpGet:
        path: /healthz
        port: 9090            # Probes can point to ports different from the main containerPort!
```
```bash
kubectl apply -f mismatch.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod mismatch-pod
rm mismatch.yaml
```

---

### Task 5: Troubleshooting Broken Probes (The Boss Fight)
The exam will drop you into a cluster where pods are in a `CrashLoopBackOff` or `Running` but `0/1 Ready` state because the probes were poorly written.

**1. CKAD Style Question:**
A Deployment named `faulty-api` is running, but the Pods are never becoming "Ready" (0/1).
Investigate the Pods to determine why the `readinessProbe` is failing. Fix the Deployment so the Pods become Ready.

**2. Setup Script:**
```bash
kubectl create deployment faulty-api --image=nginx
kubectl set env deploy/faulty-api dummy=force
# Intentionally break the readiness probe by pointing to a non-existent path
kubectl patch deploy faulty-api -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","readinessProbe":{"httpGet":{"path":"/does-not-exist","port":80},"periodSeconds":5}}]}}}}'
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 5 ---"
[ "$(kubectl get deploy faulty-api -o jsonpath='{.status.readyReplicas}')" == "1" ] && echo "✅ Deployment is fully ready" || echo "❌ Pods are still not ready"
```

<details>

**4. Solution:**
```bash
# 1. Investigate the pod events. You will see: "Warning  Unhealthy ... Readiness probe failed: HTTP probe failed with statuscode: 404"
kubectl describe pods -l app=faulty-api

# 2. Fix the deployment. Edit the readiness probe path to point to '/' instead of '/does-not-exist'
kubectl edit deploy faulty-api
```
*(In vim, scroll down to `readinessProbe.httpGet.path` and change it to `/`)*

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy faulty-api
```

#### Variation 5.1: The CrashLoop Liveness Trap
**1. CKAD Style Question:**
A Pod named `dying-pod` keeps restarting. The `livenessProbe` is executing `cat /tmp/status`, but that file doesn't exist. Fix the pod's live YAML to check `/etc/hostname` instead.

**2. Setup Script:**
```bash
kubectl run dying-pod --image=busybox --command -- sleep 3600
kubectl patch pod dying-pod -p '{"spec":{"containers":[{"name":"dying-pod","livenessProbe":{"exec":{"command":["cat","/tmp/status"]},"periodSeconds":5}}]}}'
```

**3. Testcase Script:**
```bash
[ "$(kubectl get pod dying-pod -o jsonpath='{.spec.containers[0].livenessProbe.exec.command[1]}')" == "/etc/hostname" ] && echo "✅ Exec path fixed" || echo "❌ Exec path still broken"
```

<details>

**4. Solution:**
```bash
# Pod specs are mostly immutable, but you CANNOT patch probes on a live pod directly!
# You must export, delete, edit, and recreate.
kubectl get pod dying-pod -o yaml > fix.yaml
kubectl delete pod dying-pod --force
vi fix.yaml # Fix the command array, remove cluster-specific metadata
kubectl apply -f fix.yaml
```

</details>



#### Variation 5.2: The Timeout Fix
**1. CKAD Style Question:**
A Deployment named `slow-db` is failing its health checks. The database takes 4 seconds to respond to a ping, but its `readinessProbe` has a `timeoutSeconds` of `1`. 
Fix the `slow-db` Deployment by increasing the probe's timeout to `5` seconds.

**2. Setup Script:**
```bash
# Deploy with an intentionally broken (too short) timeout
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slow-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: slow-db
  template:
    metadata:
      labels:
        app: slow-db
    spec:
      containers:
      - name: redis
        image: redis:alpine
        readinessProbe:
          tcpSocket:
            port: 6379
          timeoutSeconds: 1
EOF
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 5.2 ---"
[ "$(kubectl get deploy slow-db -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.timeoutSeconds}')" == "5" ] && echo "✅ Timeout successfully increased to 5" || echo "❌ Timeout is still incorrect"
```

<details>

**4. Solution:**
```bash
# Because it is a Deployment (not a bare Pod), you CAN edit the probes live.
kubectl edit deploy slow-db
```
*(In vim, scroll down to the `readinessProbe` block and edit the timeout):*
```yaml
        readinessProbe:
          tcpSocket:
            port: 6379
          timeoutSeconds: 5    # Change from 1 to 5
```
*(Save and exit. The deployment will automatically roll out the fix).*

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy slow-db
```

---

That gives you absolute 100% mastery over Probes. You know how to build them from scratch, patch them live, combine them, and fix them when they break.
