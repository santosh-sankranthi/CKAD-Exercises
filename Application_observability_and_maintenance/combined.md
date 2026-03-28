## Combined

Every single task combines 2 to 3 syllabus objectives and strictly adheres to your 5-component standard with expandable solutions.

-----

### Task 1: The Probes & Logs Rescue (Readiness + Forensic Logging)

This is the most standard observability question on the exam. You must fix an application's health check so it can receive traffic, and then extract its recent logs for the audit team.

**1. CKAD Style Question:**
A Deployment named `api-gateway` is running, but its Pods are not becoming ready.

1.  Fix the Deployment by updating the `readinessProbe` to use the correct path `/healthz` (it is currently pointing to a non-existent path).
2.  Extract the logs from the `api-gateway` deployment, including **only** logs from the last `10 minutes`, and save them to `/opt/gateway-logs.txt`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
kubectl create deployment api-gateway --image=nginx
kubectl set env deploy/api-gateway test=logs
kubectl patch deploy api-gateway -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","readinessProbe":{"httpGet":{"path":"/broken","port":80},"periodSeconds":5}}]}}}}'
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get deploy api-gateway -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.httpGet.path}')" == "/healthz" ] && echo "✅ Readiness probe path fixed" || echo "❌ Readiness probe path incorrect"
[ -s /opt/gateway-logs.txt ] && echo "✅ Log file created and populated" || echo "❌ Log file missing or empty"
```

<details>
  
4. Solution:

```bash
# 1. Edit the deployment to fix the probe
kubectl edit deploy api-gateway
# In vim, scroll to readinessProbe and change path from /broken to /healthz

# 2. Extract logs directly from the deployment object with the time filter
kubectl logs deployment/api-gateway --since=10m > /opt/gateway-logs.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy api-gateway
rm -f /opt/gateway-logs.txt
```

#### Variation 1.1: Multi-Container Probes & Extraction

**1. CKAD Style Question:**
A Pod named `cache-bundle` has two containers: `redis-main` and `log-shipper`.

1.  Add a TCP socket `livenessProbe` on port `6379` specifically to the `redis-main` container (initial delay: 5s).
2.  Extract the logs specifically from the `log-shipper` container and filter them to only include lines with the word "INFO". Save to `/opt/shipper.txt`.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cache-bundle
spec:
  containers:
  - name: redis-main
    image: redis:alpine
  - name: log-shipper
    image: busybox
    command: ["sh", "-c", "echo 'INFO: Shipper started'; sleep 3600"]
EOF
sleep 5
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get pod cache-bundle -o jsonpath='{.spec.containers[0].livenessProbe.tcpSocket.port}')" == "6379" ] && echo "✅ TCP Probe configured on redis-main" || echo "❌ TCP Probe missing"
grep -q "INFO" /opt/shipper.txt && echo "✅ Shipper logs successfully extracted and filtered" || echo "❌ Log extraction failed"
```

<details>
  
4. Solution:

```bash
# 1. You cannot patch probes on a live pod. Extract, delete, edit, recreate.
kubectl get pod cache-bundle -o yaml > cache.yaml
kubectl delete pod cache-bundle --force
vi cache.yaml 
# Add livenessProbe under the redis-main container spec.
kubectl apply -f cache.yaml

# 2. Extract logs from the specific container and filter
kubectl logs cache-bundle -c log-shipper | grep "INFO" > /opt/shipper.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod cache-bundle --force; rm cache.yaml /opt/shipper.txt
```

#### Variation 1.2: CrashLoop Shielding & Previous Logs

**1. CKAD Style Question:**
A Deployment `slow-app` keeps crash-looping.

1.  Extract the logs from the **previously terminated** instance of the pod and save them to `/opt/crash.txt`.
2.  The app is crashing because liveness checks happen too early. Add a `startupProbe` to the deployment (HTTP GET `/` on port `80`, failureThreshold `15`, period `5s`).

**2. Setup Script:**

```bash
kubectl create deployment slow-app --image=nginx
kubectl patch deploy slow-app -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","command":["sh","-c","sleep 2; exit 1"],"livenessProbe":{"httpGet":{"path":"/","port":80},"initialDelaySeconds":1}}]}}}}'
echo "Waiting 10s for crash..." && sleep 10
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ -s /opt/crash.txt ] && echo "✅ Previous logs extracted" || echo "❌ Previous logs missing"
[ "$(kubectl get deploy slow-app -o jsonpath='{.spec.template.spec.containers[0].startupProbe.failureThreshold}')" == "15" ] && echo "✅ Startup probe added" || echo "❌ Startup probe missing"
```

<details>
  
4. Solution:

```bash
# 1. Get the pod name, then extract previous logs
POD_NAME=$(kubectl get pods -l app=slow-app -o jsonpath='{.items[0].metadata.name}')
kubectl logs $POD_NAME -p > /opt/crash.txt

# 2. Edit the deployment live to add the startup shield
kubectl edit deploy slow-app
# Add startupProbe block alongside the livenessProbe block
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy slow-app; rm /opt/crash.txt
```

-----

### Task 2: API Updates & Metric Gathering (Deprecations + Top)

You must be able to revive legacy configurations and immediately check the cluster's physical resource consumption.

**1. CKAD Style Question:**

1.  A file at `/opt/legacy-cron.yaml` uses `batch/v1beta1`. Fix the API version to the modern stable version and apply it.
2.  Using the built-in CLI tools, identify the Pod in the `kube-system` namespace consuming the highest amount of CPU. Write its name to `/opt/high-cpu.txt`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
cat <<EOF > /opt/legacy-cron.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: legacy-backup
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: busybox
            command: ["echo", "backing up"]
          restartPolicy: OnFailure
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get cronjob legacy-backup -o jsonpath='{.apiVersion}')" == "batch/v1" ] && echo "✅ CronJob deployed with modern API" || echo "❌ API deprecation fix failed"
[ -s /opt/high-cpu.txt ] && echo "✅ CPU log file created" || echo "❌ CPU log file missing"
```

<details>
  
4. Solution:

```bash
# 1. Edit the file, change 'batch/v1beta1' to 'batch/v1', and apply
vi /opt/legacy-cron.yaml
kubectl apply -f /opt/legacy-cron.yaml

# 2. Use top to find the highest CPU pod
kubectl top pod -n kube-system --sort-by=cpu
# Visually identify the top pod and write it to the file
echo "kube-apiserver-..." > /opt/high-cpu.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete cronjob legacy-backup; rm /opt/legacy-cron.yaml /opt/high-cpu.txt
```

#### Variation 2.1: Structural Legacy Fix & Node Memory

**1. CKAD Style Question:**

1.  Fix `/opt/legacy-ingress.yaml` (`networking.k8s.io/v1beta1`). Update it to `v1` and fix the mandatory structural changes (`pathType: Prefix` and the `backend` schema). Apply it.
2.  Identify the Node with the highest Memory consumption and save its name to `/opt/high-mem-node.txt`.

**2. Setup Script:**

```bash
cat <<EOF > /opt/legacy-ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: old-ingress
spec:
  rules:
  - http:
      paths:
      - path: /test
        backend:
          serviceName: test-svc
          servicePort: 80
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get ingress old-ingress -o jsonpath='{.spec.rules[0].http.paths[0].pathType}')" == "Prefix" ] && echo "✅ Ingress structure fixed and applied" || echo "❌ Ingress structural fix failed"
[ -s /opt/high-mem-node.txt ] && echo "✅ Node memory file created" || echo "❌ Node file missing"
```

<details>
  
4. Solution:

```bash
# 1. Fix the schema
vi /opt/legacy-ingress.yaml
# Change to v1. Add 'pathType: Prefix'. Rewrite backend to:
# backend:
#   service:
#     name: test-svc
#     port:
#       number: 80
kubectl apply -f /opt/legacy-ingress.yaml

# 2. Use top for nodes
kubectl top node --sort-by=memory
echo "node-name" > /opt/high-mem-node.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ingress old-ingress; rm /opt/legacy-ingress.yaml /opt/high-mem-node.txt
```

#### Variation 2.2: RBAC Deprecation & Chronological Events

**1. CKAD Style Question:**

1.  Fix `/opt/legacy-role.yaml` (`rbac.authorization.k8s.io/v1beta1`) to the stable API and apply it.
2.  Retrieve all events in the `default` namespace. Sort them chronologically by `creationTimestamp` and save to `/opt/sorted-events.txt`.

**2. Setup Script:**

```bash
cat <<EOF > /opt/legacy-role.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: old-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.2 ---"
[ "$(kubectl get role old-role -o jsonpath='{.apiVersion}')" == "rbac.authorization.k8s.io/v1" ] && echo "✅ Role API fixed" || echo "❌ Role API failed"
grep -q "LAST SEEN" /opt/sorted-events.txt && echo "✅ Sorted events extracted" || echo "❌ Events missing"
```

<details>

4. Solution:

```bash
# 1. Edit the file, update API to v1, and apply
vi /opt/legacy-role.yaml
kubectl apply -f /opt/legacy-role.yaml

# 2. Extract and sort events
kubectl get events --sort-by=.metadata.creationTimestamp > /opt/sorted-events.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete role old-role; rm /opt/legacy-role.yaml /opt/sorted-events.txt
```

-----

### Task 3: The Init Container Forensic Debug (Init Logs + Exec)

This tests the most heavily failed logging objective alongside live container interaction.

**1. CKAD Style Question:**
A Pod named `db-init-tester` is stuck in `Init:Error`.

1.  Extract the logs specifically from the failing Init Container (`schema-builder`) and save them to `/opt/init-fail.txt`.
2.  A separate healthy Pod named `app-runner` is running. Use `kubectl exec` to run the command `date` inside `app-runner` and save the output to `/opt/app-date.txt`.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: db-init-tester
spec:
  initContainers:
  - name: schema-builder
    image: busybox
    command: ["sh", "-c", "echo 'Error: Schema validation failed'; exit 1"]
  containers:
  - name: main
    image: nginx
EOF
kubectl run app-runner --image=nginx
sleep 5
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
grep -q "Schema validation failed" /opt/init-fail.txt && echo "✅ Init logs correctly extracted" || echo "❌ Init logs missing"
[ -s /opt/app-date.txt ] && echo "✅ Date command executed via exec" || echo "❌ Exec output missing"
```

<details>
  
>4. Solution:

```bash
# 1. Target the init container specifically
kubectl logs db-init-tester -c schema-builder > /opt/init-fail.txt

# 2. Exec into the running pod
kubectl exec app-runner -- date > /opt/app-date.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod db-init-tester app-runner --force; rm /opt/init-fail.txt /opt/app-date.txt
```

#### Variation 3.1: All-Containers Prefix & Threshold Tuning

**1. CKAD Style Question:**

1.  Pod `multi-log` has 2 containers. Dump the logs of **all** containers, prefixing each line with the container name, and save to `/opt/all-logs.txt`.
2.  Edit the existing `multi-log` pod (you must recreate it) to change its liveness probe `failureThreshold` from 3 to 5.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-log
spec:
  containers:
  - name: web
    image: nginx
    livenessProbe:
      tcpSocket: {port: 80}
      failureThreshold: 3
  - name: helper
    image: busybox
    command: ["sh", "-c", "echo 'Helper running'; sleep 3600"]
EOF
sleep 4
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
grep -q "\[helper\] Helper running" /opt/all-logs.txt && echo "✅ Prefixed multi-logs extracted" || echo "❌ Multi-logs missing"
[ "$(kubectl get pod multi-log -o jsonpath='{.spec.containers[0].livenessProbe.failureThreshold}')" == "5" ] && echo "✅ Threshold tuned" || echo "❌ Threshold incorrect"
```

<details>
  
4. Solution:

```bash
# 1. Extract prefixed logs
kubectl logs multi-log --all-containers --prefix > /opt/all-logs.txt

# 2. Extract, edit, and recreate the pod
kubectl get pod multi-log -o yaml > fix.yaml
kubectl delete pod multi-log --force
vi fix.yaml # Change failureThreshold: 5
kubectl apply -f fix.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod multi-log --force; rm fix.yaml /opt/all-logs.txt
```

#### Variation 3.2: Aggregated Logs & Temporary DNS Test

**1. CKAD Style Question:**

1.  Extract the last `20` lines of logs from ALL pods with the label `tier=backend` and save to `/opt/aggregated.txt`.
2.  Spin up a temporary, self-deleting pod (`--rm`) named `net-test` (image `busybox:1.28`). Run `nslookup kubernetes.default` and save the output to `/opt/dns-test.txt`.

**2. Setup Script:**

```bash
kubectl run backend-a --image=busybox --labels=tier=backend -- sh -c 'echo "Backend A"; sleep 3600'
kubectl run backend-b --image=busybox --labels=tier=backend -- sh -c 'echo "Backend B"; sleep 3600'
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.2 ---"
grep -q "Backend" /opt/aggregated.txt && echo "✅ Aggregated logs extracted" || echo "❌ Aggregated logs missing"
grep -q "Address" /opt/dns-test.txt && echo "✅ DNS resolution successful" || echo "❌ DNS resolution failed"
kubectl get pod net-test 2>&1 | grep -q "NotFound" && echo "✅ Temporary pod self-deleted" || echo "❌ Pod failed to delete"
```

<details>
  
4. Solution:

```bash
# 1. Aggregate logs using label selector and tail
kubectl logs -l tier=backend --tail=20 > /opt/aggregated.txt

# 2. Temporary pod execution
kubectl run net-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default > /opt/dns-test.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod backend-a backend-b --force; rm /opt/aggregated.txt /opt/dns-test.txt
```

-----

### Task 4: Ephemeral Injection & Event Warning Triage

This combines the newest debugging feature in Kubernetes with advanced event filtering.

**1. CKAD Style Question:**

1.  Pod `locked-front` has no shell. Inject an ephemeral container (image `busybox`) into it, and from that shell, run `echo "hacked" > /tmp/flag`. (Exit the shell when done).
2.  Filter the cluster events to show **only** events of type `Warning` and save them to `/opt/warnings.txt`.

**2. Setup Script:**

```bash
kubectl run locked-front --image=nginx
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 4 ---"
[ "$(kubectl get pod locked-front -o jsonpath='{.spec.ephemeralContainers[0].image}')" == "busybox" ] && echo "✅ Ephemeral container injected" || echo "❌ Ephemeral container missing"
[ -f /opt/warnings.txt ] && echo "✅ Warning file created" || echo "❌ Warning file missing"
```

<details>
  
4. Solution:

```bash
# 1. Inject the ephemeral container and get a shell
kubectl debug locked-front -it --image=busybox -- sh
# Inside the container: echo "hacked" > /tmp/flag
# exit

# 2. Filter events
kubectl get events --field-selector type=Warning > /opt/warnings.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod locked-front --force; rm /opt/warnings.txt
```

#### Variation 4.1: Pod Copying & Timestamps

**1. CKAD Style Question:**

1.  Pod `crash-loop-app` keeps dying due to a bad command. Use `kubectl debug` to create a copy of this pod named `app-debug`. Override the copy's command to `sh` so it stays alive.
2.  Extract the logs from a pod named `steady-app`, appending RFC3339 timestamps, and save to `/opt/timestamps.txt`.

**2. Setup Script:**

```bash
kubectl run crash-loop-app --image=busybox --command -- false
kubectl run steady-app --image=busybox -- sh -c 'echo "Steady state"; sleep 3600'
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 4.1 ---"
[ "$(kubectl get pod app-debug -o jsonpath='{.spec.containers[0].command[0]}')" == "sh" ] && echo "✅ Pod copied and command overridden" || echo "❌ Copy or override failed"
grep -qE "^[0-9]{4}-[0-9]{2}-[0-9]{2}T" /opt/timestamps.txt && echo "✅ Timestamps appended" || echo "❌ Timestamps missing"
```

<details>
  
4. Solution:

```bash
# 1. Copy the pod and override the command
kubectl debug crash-loop-app -it --copy-to=app-debug --container=crash-loop-app -- sh
# exit

# 2. Extract logs with timestamps
kubectl logs steady-app --timestamps > /opt/timestamps.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod crash-loop-app app-debug steady-app --force; rm /opt/timestamps.txt
```

#### Variation 4.2: Node Debugging & Specific Object Events

**1. CKAD Style Question:**

1.  Create a privileged node-level debugging pod on your primary node using `busybox`. Save its generated name to `/opt/node-debugger.txt`.
2.  Extract the events specifically related to the `auth-deployment` object using a field selector, and save to `/opt/auth-events.txt`.

**2. Setup Script:**

```bash
kubectl create deployment auth-deployment --image=nginx
sleep 2
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 4.2 ---"
POD_NAME=$(cat /opt/node-debugger.txt | tr -d '[:space:]')
kubectl get pod $POD_NAME | grep -q "node-debugger" && echo "✅ Node debugger pod recorded" || echo "❌ Node debugger missing"
grep -q "auth-deployment" /opt/auth-events.txt && echo "✅ Object-specific events extracted" || echo "❌ Event extraction failed"
```

<details>
  
4. Solution:

```bash
# 1. Debug the node (find node name first with 'kubectl get nodes')
kubectl debug node/<your-node-name> -it --image=busybox
# exit
kubectl get pods # find the name of the new node-debugger pod
echo "node-debugger-xxx" > /opt/node-debugger.txt

# 2. Filter events by object
kubectl get events --field-selector involvedObject.name=auth-deployment > /opt/auth-events.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod $(cat /opt/node-debugger.txt | tr -d '[:space:]') --force; kubectl delete deploy auth-deployment; rm /opt/node-debugger.txt /opt/auth-events.txt
```

-----

### Task 5: The Full Stack Maintenance Drill

This is the ultimate capstone. You must deploy legacy configurations, secure them with probes, and test the internal network.

**1. CKAD Style Question:**

1.  Fix the legacy `extensions/v1beta1` Deployment at `/opt/legacy-deploy.yaml` by updating it to `apps/v1` and adding the mandatory `selector` block. Apply it.
2.  The deployed pods need a TCP Socket `readinessProbe` on port `80`. Edit the deployment live to add this.
3.  Spin up an interactive, self-deleting pod (`--rm`, image `busybox:1.28`) and execute a `wget -O-` command targeting the pod IP of the deployed application to prove it is serving traffic. Save output to `/opt/wget-test.txt`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
cat <<EOF > /opt/legacy-deploy.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: legacy-app
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: legacy-app
    spec:
      containers:
      - name: nginx
        image: nginx
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 5 ---"
[ "$(kubectl get deploy legacy-app -o jsonpath='{.spec.selector.matchLabels.app}')" == "legacy-app" ] && echo "✅ Legacy deployment fixed and running" || echo "❌ Legacy deployment failed"
[ "$(kubectl get deploy legacy-app -o jsonpath='{.spec.template.spec.containers[0].readinessProbe.tcpSocket.port}')" == "80" ] && echo "✅ Readiness probe live-patched" || echo "❌ Readiness probe failed"
grep -q "Welcome to nginx" /opt/wget-test.txt && echo "✅ Wget test successful" || echo "❌ Wget test failed"
```

<details>
  
4. Solution:

```bash
# 1. Fix the schema and apply
vi /opt/legacy-deploy.yaml
# Change to apps/v1, add selector block matching the template labels
kubectl apply -f /opt/legacy-deploy.yaml

# 2. Live patch the probe
kubectl edit deploy legacy-app
# Add readinessProbe -> tcpSocket -> port: 80

# 3. Network test
POD_IP=$(kubectl get pods -l app=legacy-app -o jsonpath='{.items[0].status.podIP}')
kubectl run wget-test --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- $POD_IP > /opt/wget-test.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy legacy-app; rm /opt/legacy-deploy.yaml /opt/wget-test.txt
```

*(Because this is the final capstone, it represents the absolute ceiling of complexity you will face on the exam. Variations 5.1 and 5.2 are not necessary here, as Tasks 1-4 exhaustively covered every other edge case in this domain).*

