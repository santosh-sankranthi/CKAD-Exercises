## Use built-in CLI tools to monitor Kubernetes applications

The subtopic **"Use built-in CLI tools to monitor Kubernetes applications"** revolves around three primary command families: `kubectl logs` (Layer 7 application output), `kubectl top` (Hardware resource consumption), and `kubectl get events` (Cluster-level orchestrator actions). 

To ensure absolute coverage, I have expanded this to **4 Main Tasks** with strict variations, ordered from the most frequently tested scenarios down to the advanced filtering techniques. Every single scenario follows the strict 5-component standard.

*(Exam Pro-Tip: The `kubectl top` command requires the Kubernetes Metrics Server to be installed. On the real exam, it is already running. If you are practicing on a local cluster like Minikube, you may need to run `minikube addons enable metrics-server` first!)*

---

### Task 1: Container Log Extraction (`kubectl logs`)
This is the single most tested monitoring command. You will be asked to extract logs, filter them, and save them to a file for review.

#### Main Task 1: The Standard Log Filter
**1. CKAD Style Question:**
A Pod named `error-generator` is producing massive amounts of log output. 
Extract all logs from this pod, filter them to include **only** lines containing the word `ERROR`, and save the filtered output to a file located at `/opt/filtered-logs.txt`.

**2. Setup Script:**
```bash
sudo mkdir -p /opt
sudo chmod 777 /opt
kubectl run error-generator --image=busybox -- sh -c 'while true; do echo "INFO: system nominal"; sleep 1; echo "ERROR: connection timeout"; sleep 1; done'
sleep 3 # Give it time to generate logs
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ -s /opt/filtered-logs.txt ] && echo "✅ File created and not empty" || echo "❌ File missing or empty"
grep -q "ERROR" /opt/filtered-logs.txt && echo "✅ File contains ERROR logs" || echo "❌ ERROR logs missing"
grep -q "INFO" /opt/filtered-logs.txt && echo "❌ FAILED: File contains INFO logs, filtering was incorrect" || echo "✅ File is correctly filtered"
```

<details>

**4. Solution:**
```bash
# Standard output redirection combined with grep
kubectl logs error-generator | grep "ERROR" > /opt/filtered-logs.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod error-generator --force
rm -f /opt/filtered-logs.txt
```

#### Variation 1.1: The Multi-Container Log Target
**1. CKAD Style Question:**
A Pod named `web-bundle` contains two containers: `nginx-app` and `fluentd-sidecar`. 
Extract the logs specifically from the `fluentd-sidecar` container and save them to `/opt/sidecar-logs.txt`.

**2. Setup Script:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: web-bundle
spec:
  containers:
  - name: nginx-app
    image: nginx:alpine
  - name: fluentd-sidecar
    image: busybox
    command: ["sh", "-c", "echo 'Sidecar initialized'; sleep 3600"]
EOF
sleep 4
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
grep -q "Sidecar initialized" /opt/sidecar-logs.txt && echo "✅ Specific container logs successfully extracted" || echo "❌ Container logs missing or incorrect"
```

<details>

**4. Solution:**
```bash
# You MUST use the -c flag to specify the container name when there is more than one!
kubectl logs web-bundle -c fluentd-sidecar > /opt/sidecar-logs.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod web-bundle --force
rm -f /opt/sidecar-logs.txt
```

#### Variation 1.2: The "Previous" Instance (Crash Debugging)
**1. CKAD Style Question:**
A Pod named `crashing-app` keeps restarting. You cannot debug the live container because it dies instantly. 
Extract the logs from the **previously terminated** instance of the container and save them to `/opt/crash-report.txt`.

**2. Setup Script:**
```bash
# Create a pod that intentionally crashes and restarts
kubectl run crashing-app --image=busybox --restart=Always -- sh -c 'echo "FATAL EXCEPTION: Out of memory"; exit 1'
echo "Waiting 10 seconds for the pod to crash and restart..."
sleep 10 
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
grep -q "FATAL EXCEPTION" /opt/crash-report.txt && echo "✅ Previous instance logs successfully extracted" || echo "❌ Previous logs missing"
```

<details>

**4. Solution:**
```bash
# The -p (or --previous) flag accesses the logs of the container instance that died prior to the current one.
kubectl logs crashing-app -p > /opt/crash-report.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod crashing-app --force
rm -f /opt/crash-report.txt
```

---

### Task 2: Resource Consumption Metrics (`kubectl top`)
You must be able to identify which workloads are hogging CPU or Memory. The exam will ask you to identify the highest consumer and record its name.

#### Main Task 2: The CPU Hog (Pod Level)
**1. CKAD Style Question:**
Identify the Pod in the `kube-system` namespace that is consuming the most **CPU**. 
Write the name of this pod to a file located at `/opt/highest-cpu-pod.txt`.

**2. Setup Script:**
*(Requires Metrics Server to be running on your cluster).*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ -s /opt/highest-cpu-pod.txt ] && echo "✅ File created" || echo "❌ File missing"
# We just check if a pod name from kube-system is in the file
POD_NAME=$(cat /opt/highest-cpu-pod.txt | tr -d '[:space:]')
kubectl get pod $POD_NAME -n kube-system >/dev/null 2>&1 && echo "✅ Valid kube-system pod name recorded: $POD_NAME" || echo "❌ Invalid pod name recorded"
```

<details>

**4. Solution:**
```bash
# 1. Run the top command, sorting by cpu
kubectl top pod -n kube-system --sort-by=cpu

# 2. Visually identify the top pod from the output, then copy its name into the file
# (e.g., echo "kube-apiserver-control-plane" > /opt/highest-cpu-pod.txt)
```

</details>
  
**5. Clean-up Script:**
```bash
rm -f /opt/highest-cpu-pod.txt
```

#### Variation 2.1: The Memory Hog (Node Level)
**1. CKAD Style Question:**
Identify the Node in your cluster that is consuming the most **Memory**.
Write the exact name of this node to `/opt/highest-mem-node.txt`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ -s /opt/highest-mem-node.txt ] && echo "✅ File created" || echo "❌ File missing"
NODE_NAME=$(cat /opt/highest-mem-node.txt | tr -d '[:space:]')
kubectl get node $NODE_NAME >/dev/null 2>&1 && echo "✅ Valid node name recorded: $NODE_NAME" || echo "❌ Invalid node name recorded"
```

<details>

**4. Solution:**
```bash
# Sort nodes by memory consumption
kubectl top node --sort-by=memory

# Visually copy the top node name into the file
# (e.g., echo "minikube" > /opt/highest-mem-node.txt)
```

</details>

**5. Clean-up Script:**
```bash
rm -f /opt/highest-mem-node.txt
```

#### Variation 2.2: Deep Container Metrics
**1. CKAD Style Question:**
A Pod named `data-processor` has multiple containers. You need to see the resource consumption of *each individual container* inside that pod. 
Write the command you would use to view container-level metrics into a script at `/opt/top-containers.sh`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.2 ---"
grep -q "\--containers" /opt/top-containers.sh && echo "✅ --containers flag correctly identified" || echo "❌ Command incorrect"
```

<details>

**4. Solution:**
```bash
# The --containers flag breaks down the metrics per container instead of grouping them by Pod.
echo "kubectl top pod data-processor --containers" > /opt/top-containers.sh
```

</details>

**5. Clean-up Script:**
```bash
rm -f /opt/top-containers.sh
```

---

### Task 3: Cluster Events (`kubectl get events`)
When logs don't tell the story (e.g., a Pod is stuck in `Pending` and hasn't generated logs yet), you must look at the orchestrator's Events.

#### Main Task 3: Chronological Event Sorting
**1. CKAD Style Question:**
Retrieve all events in the `default` namespace. Sort them chronologically by their **creation time** (oldest to newest) and save the raw output to `/opt/cluster-events.txt`.

**2. Setup Script:**
```bash
# Create and delete a dummy pod to force some event generation
kubectl run dummy-event --image=nginx
sleep 2
kubectl delete pod dummy-event --force
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ -s /opt/cluster-events.txt ] && echo "✅ Event file created" || echo "❌ Event file missing"
# If the command was run correctly, it should contain normal event headers
grep -q "LAST SEEN" /opt/cluster-events.txt && echo "✅ Valid event output found" || echo "❌ Output format incorrect"
```

<details>

**4. Solution:**
```bash
# The --sort-by flag combined with JSONPath is required here!
kubectl get events --sort-by=.metadata.creationTimestamp > /opt/cluster-events.txt
```

</details>

**5. Clean-up Script:**
```bash
rm -f /opt/cluster-events.txt
```

#### Variation 3.1: Filtering by Specific Object
**1. CKAD Style Question:**
You need to see the events *only* for a specific Deployment named `frontend-deploy`. 
Use the `--field-selector` flag to extract only the events where the involved object name is `frontend-deploy` and save them to `/opt/deploy-events.txt`.

**2. Setup Script:**
```bash
kubectl create deployment frontend-deploy --image=nginx
sleep 2
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
grep -q "frontend-deploy" /opt/deploy-events.txt && echo "✅ Specific object events successfully filtered" || echo "❌ Filter failed or missing"
```

<details>

**4. Solution:**
```bash
# Field selectors allow you to query the Kubernetes API directly for specific fields
kubectl get events --field-selector involvedObject.name=frontend-deploy > /opt/deploy-events.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy frontend-deploy
rm -f /opt/deploy-events.txt
```

#### Variation 3.2: Filtering for Warnings (The Troubleshooting Lifeline)
**1. CKAD Style Question:**
Something is failing in the `default` namespace. Retrieve all events in the `default` namespace that have a type of **Warning**, and save them to `/opt/warnings.txt`.

**2. Setup Script:**
```bash
# Intentionally trigger a warning (ImagePullBackOff)
kubectl run bad-image-pod --image=nginx:non-existent-tag-123
sleep 5
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.2 ---"
grep -q "Warning" /opt/warnings.txt && echo "✅ Warning events successfully captured" || echo "❌ Warnings missing"
grep -q "Normal" /opt/warnings.txt && echo "❌ FAILED: Normal events were included, filtering incorrect" || echo "✅ Output correctly filtered for Warnings only"
```

<details>

**4. Solution:**
```bash
kubectl get events --field-selector type=Warning > /opt/warnings.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod bad-image-pod --force
rm -f /opt/warnings.txt
```

---

### Task 4: Advanced Aggregated Logging
When an application is scaled across 5 pods, checking logs one by one is impossible. You must know how to aggregate logs using labels.

#### Main Task 4: Logging by Label Selector
**1. CKAD Style Question:**
An application has 3 pods running with the label `tier=backend`. 
Extract the logs from **all** pods bearing this label simultaneously, and save them to `/opt/aggregated-logs.txt`.

**2. Setup Script:**
```bash
kubectl run backend-1 --image=busybox --labels=tier=backend -- sh -c 'echo "Log from backend-1"; sleep 3600'
kubectl run backend-2 --image=busybox --labels=tier=backend -- sh -c 'echo "Log from backend-2"; sleep 3600'
sleep 3
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 4 ---"
grep -q "backend-1" /opt/aggregated-logs.txt && grep -q "backend-2" /opt/aggregated-logs.txt && echo "✅ Logs from multiple pods successfully aggregated" || echo "❌ Aggregation failed"
```

<details>

**4. Solution:**
```bash
# Use the -l (label selector) flag to grab logs from multiple pods at once!
kubectl logs -l tier=backend > /opt/aggregated-logs.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod backend-1 backend-2 --force
rm -f /opt/aggregated-logs.txt
```

#### Variation 4.1: Tailing specific line counts
**1. CKAD Style Question:**
A Pod named `heavy-logger` has thousands of lines of logs. You only care about the most recent activity. 
Extract exactly the **last 20 lines** of logs from this pod and save them to `/opt/tail-logs.txt`.

**2. Setup Script:**
```bash
kubectl run heavy-logger --image=busybox -- sh -c 'for i in $(seq 1 100); do echo "Log line $i"; done; sleep 3600'
sleep 3
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 4.1 ---"
LINE_COUNT=$(wc -l < /opt/tail-logs.txt)
[ "$LINE_COUNT" -eq "20" ] && echo "✅ Exactly 20 lines extracted" || echo "❌ Line count is $LINE_COUNT, expected 20"
```

<details>

**4. Solution:**
```bash
# Use the --tail flag to restrict output size
kubectl logs heavy-logger --tail=20 > /opt/tail-logs.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod heavy-logger --force
rm -f /opt/tail-logs.txt
```

---

This completely covers the CLI troubleshooting toolset required for the CKAD. If you know how to use `logs` (with `-c`, `-p`, `-l`, and `--tail`), `top`, and `get events --sort-by` / `--field-selector`, you can diagnose any broken cluster object they throw at you.
