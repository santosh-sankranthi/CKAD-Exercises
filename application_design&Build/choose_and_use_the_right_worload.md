## Choose and use the Right Workload.


### Task 1: Deployments & Rollouts

**CKAD Question:**
Create a Deployment named `front-end` in the `web-ns` namespace using the `nginx:1.24` image. Configure it to run 3 replicas and expose container port 80. Once it is running, perform a rolling update to change the image to `nginx:1.25`. Ensure the update is recorded in the resource's rollout history.

**Setup Script:**

```bash
kubectl create namespace web-ns

```
<details>
  
**Solution:**

```bash
# 1. Create the deployment
kubectl create deployment front-end --image=nginx:1.24 --replicas=3 --port=80 -n web-ns

# 2. Update the image
kubectl set image deployment/front-end nginx=nginx:1.25 -n web-ns 

# 3. Record the change cause
kubectl annotate deployment front-end kubernetes.io/change-cause="Updated image to nginx:1.25" -n web-ns

```

</details>

**Testcase Runner & Cleanup:**

```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get deploy front-end -n web-ns -o jsonpath='{.spec.replicas}')" == "3" ] && echo "✅ Replicas: 3" || echo "❌ Replicas failed"
[ "$(kubectl get deploy front-end -n web-ns -o jsonpath='{.spec.template.spec.containers[0].image}')" == "nginx:1.25" ] && echo "✅ Image: nginx:1.25" || echo "❌ Image failed"
kubectl describe deploy front-end -n web-ns | grep -q "change-cause" && echo "✅ Rollout history recorded" || echo "❌ Rollout history missing"
echo "--- Cleaning up ---"
kubectl delete namespace web-ns

```

---

### Task 2: CronJobs & History Limits

**CKAD Question:**
Create a CronJob named `db-cleanup` in the `default` namespace. It must run on a schedule of `*/15 * * * *` using the `busybox:1.31` image. The container command should be `sh -c 'echo "Cleaning up..." && sleep 10'`. Configure the CronJob to retain exactly 3 successful job histories and 1 failed job history.

**Setup Script:**
*(None required)*

<details>

**Solution:**

```bash
# 1. Generate base YAML
kubectl create cronjob db-cleanup --image=busybox:1.31 --schedule="*/15 * * * *" --dry-run=client -o yaml > cj.yaml

# 2. Edit the YAML (vi cj.yaml) to match this structure:

```

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-cleanup
spec:
  successfulJobsHistoryLimit: 3   
  failedJobsHistoryLimit: 1       
  schedule: "*/15 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - image: busybox:1.31
            name: db-cleanup
            command: ["sh", "-c", "echo 'Cleaning up...' && sleep 10"]
          restartPolicy: OnFailure

```

```bash
# 3. Apply
kubectl apply -f cj.yaml

```

</details>

**Testcase Runner & Cleanup:**

```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get cj db-cleanup -o jsonpath='{.spec.schedule}')" == "*/15 * * * *" ] && echo "✅ Schedule matches" || echo "❌ Schedule failed"
[ "$(kubectl get cj db-cleanup -o jsonpath='{.spec.successfulJobsHistoryLimit}')" == "3" ] && echo "✅ Success limit: 3" || echo "❌ Success limit failed"
[ "$(kubectl get cj db-cleanup -o jsonpath='{.spec.failedJobsHistoryLimit}')" == "1" ] && echo "✅ Failed limit: 1" || echo "❌ Failed limit failed"
echo "--- Cleaning up ---"
kubectl delete cronjob db-cleanup
rm cj.yaml

```

---

### Task 3: Jobs, Parallelism, & Completions

**CKAD Question:**
Create a Job named `file-processor` in the `default` namespace using the `ubuntu` image. The container should run the command `sh -c 'echo "Processing files..." && sleep 5'`. The Job must run exactly 6 times to be considered complete, and it must run 2 Pods simultaneously.

**Setup Script:**
*(None required)*

<details>

**Solution:**

```bash
# 1. Generate base YAML
kubectl create job file-processor --image=ubuntu --dry-run=client -o yaml > job.yaml

# 2. Edit the YAML (vi job.yaml) to match this structure:

```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: file-processor
spec:
  completions: 6     
  parallelism: 2     
  template:
    spec:
      containers:
      - image: ubuntu
        name: file-processor
        command: ["sh", "-c", "echo 'Processing files...' && sleep 5"]
      restartPolicy: Never

```

```bash
# 3. Apply
kubectl apply -f job.yaml

```

</details>

**Testcase Runner & Cleanup:**

```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get job file-processor -o jsonpath='{.spec.completions}')" == "6" ] && echo "✅ Completions: 6" || echo "❌ Completions failed"
[ "$(kubectl get job file-processor -o jsonpath='{.spec.parallelism}')" == "2" ] && echo "✅ Parallelism: 2" || echo "❌ Parallelism failed"
echo "--- Cleaning up ---"
kubectl delete job file-processor
rm job.yaml

```

---

### Task 4: DaemonSets (The YAML Hack)

**CKAD Question:**
Deploy a workload that runs exactly one Pod per Node in the cluster. Name the resource `sec-agent` and use the `busybox` image. The container command should be `sh -c 'sleep 3600'`. Deploy this resource in the `security-ns` namespace.

**Setup Script:**

```bash
kubectl create namespace security-ns

```

<details>

**Solution:**

```bash
# 1. Generate a Deployment YAML to act as your skeleton
kubectl create deployment sec-agent --image=busybox -n security-ns --dry-run=client -o yaml > ds.yaml

# 2. Edit the YAML (vi ds.yaml). Change kind to DaemonSet, remove replicas, remove strategy block, add command:

```

```yaml
apiVersion: apps/v1
kind: DaemonSet   
metadata:
  labels:
    app: sec-agent
  name: sec-agent
  namespace: security-ns
spec:
  selector:
    matchLabels:
      app: sec-agent
  template:
    metadata:
      labels:
        app: sec-agent
    spec:
      containers:
      - image: busybox
        name: busybox
        command: ["sh", "-c", "sleep 3600"]

```

```bash
# 3. Apply
kubectl apply -f ds.yaml

```

</details>

**Testcase Runner & Cleanup:**

```bash
#!/bin/bash
echo "--- Testing Task 4 ---"
[ "$(kubectl get ds sec-agent -n security-ns -o jsonpath='{.kind}')" == "DaemonSet" ] && echo "✅ Resource is a DaemonSet" || echo "❌ Not a DaemonSet"
[ "$(kubectl get ds sec-agent -n security-ns -o jsonpath='{.spec.template.spec.containers[0].command[2]}')" == "sleep 3600" ] && echo "✅ Command matches" || echo "❌ Command failed"
echo "--- Cleaning up ---"
kubectl delete ds sec-agent -n security-ns
kubectl delete namespace security-ns
rm ds.yaml

```

---

### Task 5: StatefulSets & Headless Services

**CKAD Question:**
Create a StatefulSet named `web-db` in the `default` namespace using the `nginx:alpine` image. Configure it with 2 replicas. It must be associated with an existing headless service named `web-db-hl`.

**Setup Script:**

```bash
kubectl create service clusterip web-db-hl --clusterip="None" --tcp=80:80

```

<details>

**Solution:**

```bash
# 1. Generate a Deployment YAML to use as a base
kubectl create deployment web-db --image=nginx:alpine --replicas=2 --dry-run=client -o yaml > sts.yaml

# 2. Edit the YAML (vi sts.yaml). Change kind to StatefulSet, add serviceName, remove strategy block:

```

```yaml
apiVersion: apps/v1
kind: StatefulSet   
metadata:
  labels:
    app: web-db
  name: web-db
spec:
  serviceName: "web-db-hl"  
  replicas: 2
  selector:
    matchLabels:
      app: web-db
  template:
    metadata:
      labels:
        app: web-db
    spec:
      containers:
      - image: nginx:alpine
        name: nginx

```

```bash
# 3. Apply
kubectl apply -f sts.yaml

```

</details>

**Testcase Runner & Cleanup:**

```bash
#!/bin/bash
echo "--- Testing Task 5 ---"
[ "$(kubectl get sts web-db -o jsonpath='{.kind}')" == "StatefulSet" ] && echo "✅ Resource is a StatefulSet" || echo "❌ Not a StatefulSet"
[ "$(kubectl get sts web-db -o jsonpath='{.spec.serviceName}')" == "web-db-hl" ] && echo "✅ ServiceName attached" || echo "❌ ServiceName failed"
[ "$(kubectl get sts web-db -o jsonpath='{.spec.replicas}')" == "2" ] && echo "✅ Replicas: 2" || echo "❌ Replicas failed"
echo "--- Cleaning up ---"
kubectl delete sts web-db
kubectl delete svc web-db-hl
rm sts.yaml

```

---
