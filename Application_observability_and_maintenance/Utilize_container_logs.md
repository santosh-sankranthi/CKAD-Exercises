## Utilize container logs.

In the exam, they won't just ask you to grab standard logs. They will throw you into forensic debugging scenarios where Pod names change dynamically, applications are stuck in initialization, or you need to extract logs based on strict timeframes. 



Here is the deep-dive 6-part matrix (2 Main Tasks, 4 Variations) that isolates the most advanced and highly tested logging edge cases, strictly adhering to the 5-component format.

---

### Task 1: Forensic Time & Object Filtering
The exam will frequently ask you to extract logs without knowing the exact Pod name (e.g., from a Deployment or Job), or restrict the output to a specific chronological window to avoid massive files.

#### Main Task 1: Timeframes and Timestamps
**1. CKAD Style Question:**
A Pod named `audit-logger` has been running for days. 
Extract its logs, but **only** include logs generated within the last `15 minutes`. Furthermore, force Kubernetes to prepend RFC3339 **timestamps** to every log line. Save the output to `/opt/audit-recent.txt`.

**2. Setup Script:**
```bash
sudo mkdir -p /opt
sudo chmod 777 /opt
kubectl run audit-logger --image=busybox -- sh -c 'while true; do echo "Transaction processed"; sleep 2; done'
sleep 5
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ -s /opt/audit-recent.txt ] && echo "✅ Log file created" || echo "❌ Log file missing"
# Check if the output contains the Kubernetes timestamp format (e.g., 2026-03-28T...)
grep -qE "^[0-9]{4}-[0-9]{2}-[0-9]{2}T" /opt/audit-recent.txt && echo "✅ Timestamps successfully prepended" || echo "❌ Timestamps missing"
```

<details>

**4. Solution:**
```bash
# Combine --since to filter time and --timestamps to format the output
kubectl logs audit-logger --since=15m --timestamps > /opt/audit-recent.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod audit-logger --force
rm -f /opt/audit-recent.txt
```

#### Variation 1.1: Direct Deployment Logging
**1. CKAD Style Question:**
A Deployment named `auth-api` is running with multiple replicas. You do not have time to look up the individual Pod hashes. 
Extract the logs directly from the `auth-api` Deployment object and save them to `/opt/deploy-logs.txt`.

**2. Setup Script:**
```bash
kubectl create deployment auth-api --image=busybox -- sh -c 'while true; do echo "Auth success"; sleep 2; done'
sleep 5
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
grep -q "Auth success" /opt/deploy-logs.txt && echo "✅ Logs successfully extracted directly from Deployment" || echo "❌ Logs missing or empty"
```

<details>

**4. Solution:**
```bash
# You can prefix the resource type to bypass pod discovery!
kubectl logs deployment/auth-api > /opt/deploy-logs.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy auth-api
rm -f /opt/deploy-logs.txt
```

#### Variation 1.2: Job Execution Logs
**1. CKAD Style Question:**
A Job named `nightly-batch` has recently completed. 
Extract the execution logs directly from the `nightly-batch` Job object and save them to `/opt/job-logs.txt`.

**2. Setup Script:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: nightly-batch
spec:
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["echo", "Batch processing complete: 402 records"]
      restartPolicy: Never
EOF
sleep 5
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
grep -q "402 records" /opt/job-logs.txt && echo "✅ Job logs successfully extracted" || echo "❌ Job logs missing"
```

<details>

**4. Solution:**
```bash
# Similar to deployments, you can directly target the Job resource
kubectl logs job/nightly-batch > /opt/job-logs.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete job nightly-batch
rm -f /opt/job-logs.txt
```

---

### Task 2: The Init Container Traps (The Boss Fight)
This is the #1 reason people fail logging questions. If a Pod is stuck in `Init:Error` or `Init:CrashLoopBackOff`, running `kubectl logs <pod>` will return *nothing* because the main container hasn't started yet!

#### Main Task 2: Extracting Init Logs
**1. CKAD Style Question:**
A Pod named `db-migrator` is stuck and will not reach the `Running` state. 
Investigate the Pod. You will find it has an Init Container named `schema-loader` that is failing. Extract the logs specifically from this failing Init Container and save them to `/opt/init-error.txt`.

**2. Setup Script:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: db-migrator
spec:
  initContainers:
  - name: schema-loader
    image: busybox
    command: ["sh", "-c", "echo 'FATAL: Schema syntax error'; exit 1"]
  containers:
  - name: main-app
    image: nginx
EOF
sleep 5
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
grep -q "FATAL: Schema syntax error" /opt/init-error.txt && echo "✅ Init container logs successfully extracted" || echo "❌ Init logs missing"
```

<details>

**4. Solution:**
```bash
# 1. You would normally run 'kubectl describe pod db-migrator' to find the init container's name.
# 2. Use the -c flag targeting the init container.
kubectl logs db-migrator -c schema-loader > /opt/init-error.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod db-migrator --force
rm -f /opt/init-error.txt
```

#### Variation 2.1: The "All Containers" Dump
**1. CKAD Style Question:**
A Pod named `metrics-bundle` has 3 active containers. You do not know which one is throwing errors. 
Dump the logs from **all** containers inside the `metrics-bundle` pod simultaneously, prefixing each line with the container name, and save to `/opt/all-containers.txt`.

**2. Setup Script:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: metrics-bundle
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'App OK'; sleep 3600"]
  - name: sidecar
    image: busybox
    command: ["sh", "-c", "echo 'Sidecar OK'; sleep 3600"]
EOF
sleep 4
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
grep -q "\[app\] App OK" /opt/all-containers.txt && grep -q "\[sidecar\] Sidecar OK" /opt/all-containers.txt && echo "✅ Logs from all containers prefixed and extracted" || echo "❌ Multi-container log extraction failed"
```

<details>

**4. Solution:**
```bash
# The --all-containers flag grabs everything. 
# The --prefix flag ensures you know which container generated which line!
kubectl logs metrics-bundle --all-containers --prefix > /opt/all-containers.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod metrics-bundle --force
rm -f /opt/all-containers.txt
```

#### Variation 2.2: The Ghost Init Container (Previous)
**1. CKAD Style Question:**
The Init Container `config-puller` inside the Pod `ghost-app` keeps crashing (`Init:CrashLoopBackOff`). It dies too fast for you to view the live logs. 
Extract the logs from the **previously terminated** instance of the `config-puller` Init Container and save them to `/opt/ghost-init.txt`.

**2. Setup Script:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ghost-app
spec:
  initContainers:
  - name: config-puller
    image: busybox
    command: ["sh", "-c", "echo 'Network timeout pulling config'; exit 1"]
  containers:
  - name: main
    image: nginx
EOF
echo "Waiting 12 seconds for init container to crash and restart..."
sleep 12
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.2 ---"
grep -q "Network timeout pulling config" /opt/ghost-init.txt && echo "✅ Previous init container logs successfully extracted" || echo "❌ Ghost init logs missing"
```

<details>

**4. Solution:**
```bash
# You must combine the -p (previous) flag with the -c (container) flag!
kubectl logs ghost-app -p -c config-puller > /opt/ghost-init.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod ghost-app --force
rm -f /opt/ghost-init.txt
```

---

By mastering these precise combinations (`--since`, `--timestamps`, `-c <init-container>`, `--all-containers`, and `-p -c`), you are totally immune to the forensic traps they set for you in the "Utilize container logs" objective. 
