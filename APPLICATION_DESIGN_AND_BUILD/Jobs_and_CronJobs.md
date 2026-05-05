## Jobs and CronJobs

In the CKAD exam, **Jobs** and **CronJobs** are tested under the "Application Design and Build" domain. A Job runs a task to completion (batch processing), while a CronJob runs Jobs on a schedule (like cron in Linux). The exam focuses on: creating Jobs with specific completion/parallelism settings, managing CronJob schedules and concurrency policies, and extracting logs from completed Job pods.

*(Exam Pro-Tip: `kubectl create job` and `kubectl create cronjob` are fully imperative commands! You can scaffold almost everything from the command line and only use `vi` for the few fields that have no flags.)*

---

### Task 1: The Basic Batch Job (The Most Repeating)
The most common Job question asks you to run a task to completion with a specific number of successful completions.

**1. CKAD Style Question:**
Create a Job named `data-export` using the `busybox` image.
The Job must run the command `echo "Export batch complete"`.
Configure it to require `4` successful completions, running `2` pods in parallel at a time.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get job data-export -o jsonpath='{.spec.completions}')" == "4" ] && echo "✅ Completions set to 4" || echo "❌ Completions incorrect"
[ "$(kubectl get job data-export -o jsonpath='{.spec.parallelism}')" == "2" ] && echo "✅ Parallelism set to 2" || echo "❌ Parallelism incorrect"
```

<details>

**4. Solution:**
```bash
# 1. Generate base YAML. The imperative command supports completions and parallelism!
kubectl create job data-export --image=busybox --dry-run=client -o yaml -- echo "Export batch complete" > task1.yaml

# 2. Edit the YAML to add completions and parallelism under spec:
vi task1.yaml
```
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-export
spec:
  completions: 4             # ADD: 4 pods must succeed
  parallelism: 2             # ADD: Run 2 at a time
  template:
    spec:
      containers:
      - image: busybox
        name: data-export
        command: ["echo", "Export batch complete"]
      restartPolicy: Never
```
```bash
# 3. Apply the YAML
kubectl apply -f task1.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete job data-export; rm task1.yaml
```

#### Variation 1.1: The Failure Budget (`backoffLimit`)
**1. CKAD Style Question:**
Create a Job named `flaky-import` using the `busybox` image running `sh -c 'exit 1'` (it always fails).
Set `backoffLimit` to `3` so Kubernetes only retries 3 times before giving up on the Job entirely.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get job flaky-import -o jsonpath='{.spec.backoffLimit}')" == "3" ] && echo "✅ backoffLimit set to 3" || echo "❌ backoffLimit incorrect"
```

<details>

**4. Solution:**
```bash
kubectl create job flaky-import --image=busybox --dry-run=client -o yaml -- sh -c 'exit 1' > var11.yaml

vi var11.yaml
```
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: flaky-import
spec:
  backoffLimit: 3             # ADD: Only allow 3 retries
  template:
    spec:
      containers:
      - image: busybox
        name: flaky-import
        command: ["sh", "-c", "exit 1"]
      restartPolicy: Never
```
```bash
kubectl apply -f var11.yaml
```

</details>

**5. Clean-up Script:** `kubectl delete job flaky-import; rm var11.yaml`

#### Variation 1.2: The Timeout Guard (`activeDeadlineSeconds`)
**1. CKAD Style Question:**
Create a Job named `timeout-job` using the `busybox` image running `sleep 600`.
The Job must be automatically terminated if it has not completed within `30` seconds. Set `activeDeadlineSeconds` to `30`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get job timeout-job -o jsonpath='{.spec.activeDeadlineSeconds}')" == "30" ] && echo "✅ activeDeadlineSeconds set to 30" || echo "❌ activeDeadlineSeconds incorrect"
```

<details>

**4. Solution:**
```bash
kubectl create job timeout-job --image=busybox --dry-run=client -o yaml -- sleep 600 > var12.yaml

vi var12.yaml
```
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: timeout-job
spec:
  activeDeadlineSeconds: 30   # ADD: Hard timeout at 30 seconds
  template:
    spec:
      containers:
      - image: busybox
        name: timeout-job
        command: ["sleep", "600"]
      restartPolicy: Never
```
```bash
kubectl apply -f var12.yaml
```

</details>

**5. Clean-up Script:** `kubectl delete job timeout-job; rm var12.yaml`

---

### Task 2: The CronJob (Scheduled Execution)
CronJobs are the second most tested pattern. You must know the cron schedule syntax and the imperative command.

**1. CKAD Style Question:**
Create a CronJob named `db-backup` using the `busybox` image.
It must run the command `echo "Database backup initiated"` every `5 minutes`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get cronjob db-backup -o jsonpath='{.spec.schedule}')" == "*/5 * * * *" ] && echo "✅ Schedule is every 5 minutes" || echo "❌ Schedule incorrect"
[ "$(kubectl get cronjob db-backup -o jsonpath='{.spec.jobTemplate.spec.template.spec.containers[0].command[0]}')" == "echo" ] && echo "✅ Command configured" || echo "❌ Command missing"
```

<details>

**4. Solution:**
```bash
# The imperative command for CronJob is the fastest way!
kubectl create cronjob db-backup --image=busybox --schedule="*/5 * * * *" -- echo "Database backup initiated"
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete cronjob db-backup
```

#### Variation 2.1: Concurrency Policy (`Forbid`)
**1. CKAD Style Question:**
Create a CronJob named `exclusive-sync` using the `busybox` image running `sleep 120` on a schedule of `*/2 * * * *` (every 2 minutes).
Because the job takes longer than 2 minutes, overlapping runs will occur. Set the `concurrencyPolicy` to `Forbid` so that if a previous Job is still running, the next scheduled run is skipped entirely.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get cronjob exclusive-sync -o jsonpath='{.spec.concurrencyPolicy}')" == "Forbid" ] && echo "✅ concurrencyPolicy is Forbid" || echo "❌ concurrencyPolicy incorrect"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base CronJob
kubectl create cronjob exclusive-sync --image=busybox --schedule="*/2 * * * *" --dry-run=client -o yaml -- sleep 120 > var21.yaml

# 2. Edit to add the concurrencyPolicy (no imperative flag exists for this!)
vi var21.yaml
```
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: exclusive-sync
spec:
  schedule: "*/2 * * * *"
  concurrencyPolicy: Forbid    # ADD THIS LINE
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - image: busybox
            name: exclusive-sync
            command: ["sleep", "120"]
          restartPolicy: OnFailure
```
```bash
kubectl apply -f var21.yaml
```

</details>

**5. Clean-up Script:** `kubectl delete cronjob exclusive-sync; rm var21.yaml`

#### Variation 2.2: Concurrency Policy (`Replace`)
**1. CKAD Style Question:**
Create a CronJob named `replace-sync` using the `busybox` image running `sleep 90` on a schedule of `* * * * *` (every minute).
Set the `concurrencyPolicy` to `Replace` so that if a previous Job is still running, it is killed and the new one starts.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.2 ---"
[ "$(kubectl get cronjob replace-sync -o jsonpath='{.spec.concurrencyPolicy}')" == "Replace" ] && echo "✅ concurrencyPolicy is Replace" || echo "❌ concurrencyPolicy incorrect"
```

<details>

**4. Solution:**
```bash
kubectl create cronjob replace-sync --image=busybox --schedule="* * * * *" --dry-run=client -o yaml -- sleep 90 > var22.yaml

vi var22.yaml
# Add under spec: concurrencyPolicy: Replace
```
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: replace-sync
spec:
  schedule: "* * * * *"
  concurrencyPolicy: Replace   # ADD THIS LINE
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - image: busybox
            name: replace-sync
            command: ["sleep", "90"]
          restartPolicy: OnFailure
```
```bash
kubectl apply -f var22.yaml
```

</details>

**5. Clean-up Script:** `kubectl delete cronjob replace-sync; rm var22.yaml`

---

### Task 3: CronJob Suspend & History Limits (The Gotchas)
The exam loves testing whether you know how to pause a CronJob without deleting it, and how to manage the history of child Jobs.

**1. CKAD Style Question:**
Create a CronJob named `report-gen` using the `busybox` image running `echo "Report generated"` on a schedule of `0 * * * *` (every hour).
Configure the CronJob so that:
1. Only `2` successful Job histories are retained (`successfulJobsHistoryLimit`).
2. Only `1` failed Job history is retained (`failedJobsHistoryLimit`).

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get cronjob report-gen -o jsonpath='{.spec.successfulJobsHistoryLimit}')" == "2" ] && echo "✅ successfulJobsHistoryLimit is 2" || echo "❌ History limit incorrect"
[ "$(kubectl get cronjob report-gen -o jsonpath='{.spec.failedJobsHistoryLimit}')" == "1" ] && echo "✅ failedJobsHistoryLimit is 1" || echo "❌ Failed history limit incorrect"
```

<details>

**4. Solution:**
```bash
kubectl create cronjob report-gen --image=busybox --schedule="0 * * * *" --dry-run=client -o yaml -- echo "Report generated" > task3.yaml

vi task3.yaml
```
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: report-gen
spec:
  schedule: "0 * * * *"
  successfulJobsHistoryLimit: 2    # ADD: Keep only 2 successful Job records
  failedJobsHistoryLimit: 1        # ADD: Keep only 1 failed Job record
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - image: busybox
            name: report-gen
            command: ["echo", "Report generated"]
          restartPolicy: OnFailure
```
```bash
kubectl apply -f task3.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete cronjob report-gen; rm task3.yaml
```

#### Variation 3.1: The Suspend Toggle (Pause & Resume)
**1. CKAD Style Question:**
A CronJob named `hourly-cleanup` is currently running on a schedule. The operations team needs to pause it immediately without deleting it. Suspend the CronJob, then verify it is suspended.

**2. Setup Script:**
```bash
kubectl create cronjob hourly-cleanup --image=busybox --schedule="0 * * * *" -- echo "Cleaning up"
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get cronjob hourly-cleanup -o jsonpath='{.spec.suspend}')" == "true" ] && echo "✅ CronJob is suspended" || echo "❌ CronJob is NOT suspended"
```

<details>

**4. Solution:**
```bash
# The fastest way to suspend a CronJob is a targeted patch!
kubectl patch cronjob hourly-cleanup -p '{"spec":{"suspend":true}}'

# To resume later:
# kubectl patch cronjob hourly-cleanup -p '{"spec":{"suspend":false}}'
```

</details>

**5. Clean-up Script:** `kubectl delete cronjob hourly-cleanup`

#### Variation 3.2: Manual Job Trigger from CronJob
**1. CKAD Style Question:**
A CronJob named `nightly-etl` exists but is scheduled to run only at midnight. You need to trigger it immediately for testing purposes without waiting for the schedule. Create a one-off Job from the existing CronJob named `etl-manual-run`.

**2. Setup Script:**
```bash
kubectl create cronjob nightly-etl --image=busybox --schedule="0 0 * * *" -- echo "ETL complete"
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.2 ---"
[ "$(kubectl get job etl-manual-run -o jsonpath='{.spec.template.spec.containers[0].command[0]}')" == "echo" ] && echo "✅ Manual Job created from CronJob" || echo "❌ Manual trigger failed"
```

<details>

**4. Solution:**
```bash
# This is the imperative command to create a one-off Job from an existing CronJob!
kubectl create job etl-manual-run --from=cronjob/nightly-etl
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete job etl-manual-run
kubectl delete cronjob nightly-etl
```

---

### Task 4: Job + Completions + Logs Extraction (The Composite Drill)
The exam will ask you to create a Job, wait for it, and then prove it worked by extracting its logs.

**1. CKAD Style Question:**
1. Create a Job named `audit-scan` using the `busybox` image running `echo "Scan: 0 vulnerabilities found"`. It must complete exactly `3` times.
2. After the Job completes, extract the logs from any one of the completed pods and save them to `/opt/scan-results.txt`.

**2. Setup Script:**
```bash
sudo mkdir -p /opt
sudo chmod 777 /opt
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 4 ---"
[ "$(kubectl get job audit-scan -o jsonpath='{.spec.completions}')" == "3" ] && echo "✅ Job completions set to 3" || echo "❌ Completions incorrect"
grep -q "0 vulnerabilities" /opt/scan-results.txt && echo "✅ Logs successfully extracted from completed Job" || echo "❌ Log extraction failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the Job
kubectl create job audit-scan --image=busybox --dry-run=client -o yaml -- echo "Scan: 0 vulnerabilities found" > task4.yaml
vi task4.yaml
# Add: completions: 3 under spec:
kubectl apply -f task4.yaml

# 2. Wait for the job to complete
kubectl wait --for=condition=complete job/audit-scan --timeout=60s

# 3. Extract logs from the Job's pods using the job-name label
kubectl logs job/audit-scan > /opt/scan-results.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete job audit-scan; rm task4.yaml /opt/scan-results.txt
```

#### Variation 4.1: The `restartPolicy` Trap
**1. CKAD Style Question:**
You try to create a Job with `restartPolicy: Always`, but it fails validation.
Create a Job named `policy-test` using the `busybox` image running `echo "done"`. Ensure you use the correct `restartPolicy` for a Job.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 4.1 ---"
POLICY=$(kubectl get job policy-test -o jsonpath='{.spec.template.spec.restartPolicy}')
[ "$POLICY" == "Never" ] || [ "$POLICY" == "OnFailure" ] && echo "✅ Valid restartPolicy ($POLICY) for Job" || echo "❌ Invalid restartPolicy: $POLICY"
```

<details>

**4. Solution:**
```bash
# Jobs ONLY support restartPolicy: Never or restartPolicy: OnFailure
# Using 'Always' (the default for Pods and Deployments) will cause a validation error!

kubectl create job policy-test --image=busybox -- echo "done"

# The imperative 'kubectl create job' automatically sets restartPolicy: Never for you!
```
*(Exam Trap: If you generate a Pod YAML and manually change Kind to Job, you MUST also change the restartPolicy from 'Always' to 'Never' or 'OnFailure', or the apply will fail.)*

</details>

**5. Clean-up Script:** `kubectl delete job policy-test`

---

### Task 5: Troubleshooting Failed Jobs (The Boss Fight)
The exam will drop you into a cluster where a Job or CronJob is broken and you must diagnose it.

**1. CKAD Style Question:**
A Job named `broken-etl` exists but its pods keep failing and restarting. Investigate the issue.
You will find the `backoffLimit` has been reached and the Job has failed. The command being run is incorrect.
1. Delete the failed Job.
2. Recreate the Job with the correct command: `echo "ETL success"` (it was previously running `echo_typo "ETL success"`).
3. Save the logs from the successful run to `/opt/etl-fix.txt`.

**2. Setup Script:**
```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: broken-etl
spec:
  backoffLimit: 2
  template:
    spec:
      containers:
      - name: etl
        image: busybox
        command: ["echo_typo", "ETL success"]
      restartPolicy: Never
EOF
echo "Waiting 15s for Job to fail..."
sleep 15
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 5 ---"
[ "$(kubectl get job broken-etl -o jsonpath='{.status.succeeded}')" == "1" ] && echo "✅ Job is now succeeding" || echo "❌ Job is still failing"
grep -q "ETL success" /opt/etl-fix.txt && echo "✅ Fixed logs extracted" || echo "❌ Logs missing"
```

<details>

**4. Solution:**
```bash
# 1. Investigate the failed job
kubectl describe job broken-etl
kubectl get pods -l job-name=broken-etl
# You'll see pods in Error state. Check the events to find the broken command.

# 2. Delete the failed job (this also cleans up the dead pods)
kubectl delete job broken-etl

# 3. Recreate with the correct command
kubectl create job broken-etl --image=busybox -- echo "ETL success"

# 4. Wait for success and extract logs
kubectl wait --for=condition=complete job/broken-etl --timeout=30s
kubectl logs job/broken-etl > /opt/etl-fix.txt
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete job broken-etl; rm -f /opt/etl-fix.txt
```

#### Variation 5.1: The Deadline Autopsy
**1. CKAD Style Question:**
A Job named `deadline-exceeded` was supposed to finish in 10 seconds but its `activeDeadlineSeconds` expired, causing it to be terminated by Kubernetes.
1. Investigate the Job and save the reason for failure to `/opt/deadline-reason.txt`. (Hint: check `.status.conditions`).
2. Delete the broken Job and recreate it with a more generous `activeDeadlineSeconds` of `120`.

**2. Setup Script:**
```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: deadline-exceeded
spec:
  activeDeadlineSeconds: 2
  template:
    spec:
      containers:
      - name: worker
        image: busybox
        command: ["sleep", "60"]
      restartPolicy: Never
EOF
echo "Waiting 8s for deadline to expire..."
sleep 8
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 5.1 ---"
grep -q "DeadlineExceeded" /opt/deadline-reason.txt && echo "✅ Failure reason correctly identified" || echo "❌ Failure reason missing"
[ "$(kubectl get job deadline-exceeded -o jsonpath='{.spec.activeDeadlineSeconds}')" == "120" ] && echo "✅ Job recreated with 120s deadline" || echo "❌ Deadline not fixed"
```

<details>

**4. Solution:**
```bash
# 1. Extract the failure reason from the Job status
kubectl get job deadline-exceeded -o jsonpath='{.status.conditions[0].reason}' > /opt/deadline-reason.txt
# Output will be: DeadlineExceeded

# 2. Delete and recreate with a longer deadline
kubectl delete job deadline-exceeded
kubectl create job deadline-exceeded --image=busybox --dry-run=client -o yaml -- sleep 60 > fix.yaml
vi fix.yaml
# Add: activeDeadlineSeconds: 120
kubectl apply -f fix.yaml
rm fix.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete job deadline-exceeded; rm -f /opt/deadline-reason.txt
```

---

That gives you complete mastery over Jobs and CronJobs. You know how to create them imperatively, tune completions/parallelism/backoffLimit, manage CronJob schedules and concurrency, suspend and resume, manually trigger, and troubleshoot failures. This is the single most overlooked CKAD topic—don't skip it.
