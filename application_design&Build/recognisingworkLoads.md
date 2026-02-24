# CKAD Workload Resource Patterns: The 2024-2026 Strategy Guide

The Certified Kubernetes Application Developer (CKAD) exam strictly tests your execution speed and ability to recognize operational patterns. You are rarely asked to "just create" a resource. Instead, you are given a scenario where choosing the wrong resource—or the wrong configuration—costs you time and points.

Below are the definitive workload patterns tested on the CKAD, complete with the traps to avoid, mock questions, and imperative-first solutions.

---

## 1. The "Stateless App & Quick Mutation" Pattern (Deployments)
The exam heavily tests your ability to generate a Deployment imperatively and rapidly mutate the resulting YAML to inject configurations like ServiceAccounts, resource limits, or environment variables.

**The Trap:** Wasting time writing YAML from scratch, or placing Pod-level specifications (like `serviceAccountName`) at the Deployment level by mistake.
**The Pattern:** You need to run a web server or standard application and immediately bind it to an existing cluster configuration.

### 🌟 CKAD Style Question
**Task:** Create a deployment named `api-gateway` in the `staging` namespace using the `nginx:1.23` image. It should have 3 replicas. The pods must run under an existing service account named `gateway-sa`. 

### 💡 Solution
1. Generate the base YAML imperatively to save time:
   ```bash
   kubectl create deployment api-gateway --image=nginx:1.23 --replicas=3 -n staging --dry-run=client -o yaml > deploy.yaml
   ```
2. Edit `deploy.yaml` to add the `serviceAccountName` under the pod template spec (NOT the deployment spec):
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: api-gateway
     namespace: staging
   spec:
     replicas: 3
     selector:
       matchLabels:
         app: api-gateway
     template:
       metadata:
         labels:
           app: api-gateway
       spec:
         serviceAccountName: gateway-sa # Add this line here
         containers:
         - image: nginx:1.23
           name: nginx
   ```
3. Apply the configuration:
   ```bash
   kubectl apply -f deploy.yaml
   ```

---

## 2. The "Parallel Worker" Pattern (Jobs)
Jobs are not just for single tasks. The exam will test if you know how to process a queue or batch of items concurrently to save time.

**The Trap:** Creating a standard Job that processes a queue sequentially (one pod at a time), which loses points for inefficiency.
**The Pattern:** The prompt mentions processing "multiple items," "queues," or explicitly states a number of successful completions required alongside a concurrency limit.

### 🌟 CKAD Style Question
**Task:** Create a Job named `image-processor` that runs the `busybox` image and executes the command `sleep 5`. The job should run a total of 8 times to completion, but it must process 2 pods in parallel at any given time.

### 💡 Solution
1. Generate the base Job YAML:
   ```bash
   kubectl create job image-processor --image=busybox --dry-run=client -o yaml -- sh -c "sleep 5" > job.yaml
   ```
2. Edit `job.yaml` to add `completions` and `parallelism` to the Job spec:
   ```yaml
   apiVersion: batch/v1
   kind: Job
   metadata:
     name: image-processor
   spec:
     completions: 8 # Add this
     parallelism: 2 # Add this
     template:
       spec:
         containers:
         - command:
           - sh
           - -c
           - sleep 5
           image: busybox
           name: image-processor
         restartPolicy: Never
   ```
3. Apply the Job:
   ```bash
   kubectl apply -f job.yaml
   ```

---

## 3. The "Scheduled Task & Retention" Pattern (CronJobs)
The trick with CronJobs isn't the schedule itself, but managing the history of the jobs it creates so it doesn't clutter the cluster.

**The Trap:** Only configuring the schedule and forgetting to set the history limits, leaving dead pods to accumulate.
**The Pattern:** Running a backup or sync at a specific time, combined with strict instructions on how many past job logs to keep.

### 🌟 CKAD Style Question
**Task:** Create a CronJob named `db-backup` that runs every day at midnight (`0 0 * * *`). It should use the `alpine` image and run the command `echo "backup complete"`. Configure the CronJob so it retains only the 1 most recently failed job and the 3 most recently successful jobs.

### 💡 Solution
1. Generate the base CronJob:
   ```bash
   kubectl create cronjob db-backup --image=alpine --schedule="0 0 * * *" --dry-run=client -o yaml -- sh -c 'echo "backup complete"' > cronjob.yaml
   ```
2. Edit `cronjob.yaml` to add the history limits:
   ```yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: db-backup
   spec:
     schedule: "0 0 * * *"
     successfulJobsHistoryLimit: 3 # Add this
     failedJobsHistoryLimit: 1     # Add this
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - command:
               - sh
               - -c
               - echo "backup complete"
               image: alpine
               name: db-backup
             restartPolicy: OnFailure
   ```
3. Apply the CronJob:
   ```bash
   kubectl apply -f cronjob.yaml
   ```

---

## 4. The "Agent on Every Node" Pattern (DaemonSets)
There is no imperative `kubectl create daemonset` command. The CKAD tests if you know how to convert a Deployment YAML into a DaemonSet YAML manually.

**The Trap:** Spending time searching for an imperative command, or forgetting to delete the `replicas` and `strategy` fields when converting a Deployment YAML, which will cause the apply command to fail.
**The Pattern:** Ensuring a specific pod (usually logging, monitoring, or security agents) runs on *every* single node in the cluster.

### 🌟 CKAD Style Question
**Task:** A deployment YAML file at `/opt/course/fluentd.yaml` was created by mistake. Modify this file to convert the workload into a DaemonSet named `fluentd-agent` running in the `kube-system` namespace. Ensure it schedules exactly one pod per node. 

### 💡 Solution
1. Open `/opt/course/fluentd.yaml` in your editor (`vim /opt/course/fluentd.yaml`).
2. Make the following three critical changes:
   * Change `kind: Deployment` to `kind: DaemonSet`.
   * Delete the `replicas: X` line (DaemonSets don't use replicas).
   * Delete the `strategy: {}` block (DaemonSets use `updateStrategy`, not `strategy`).
   ```yaml
   apiVersion: apps/v1
   kind: DaemonSet # Changed from Deployment
   metadata:
     name: fluentd-agent
     namespace: kube-system
   spec:
     # replicas: 3 <-- DELETED
     # strategy: <-- DELETED
     #   type: RollingUpdate <-- DELETED
     selector:
       matchLabels:
         app: fluentd
     template:
       metadata:
         labels:
           app: fluentd
       spec:
         containers:
         - image: fluentd:v1.14
           name: fluentd
   ```
3. Apply the converted file:
   ```bash
   kubectl apply -f /opt/course/fluentd.yaml
   ```

---

## 5. The "Traffic Switch / Blue-Green" Pattern (Services & Deployments)
This tests your understanding of how Services route traffic to Pods via labels, completely decoupling the network layer from the workload rollout.

**The Trap:** Attempting a Rolling Update (`kubectl set image`) when the prompt demands an *instant* shift of traffic to a new version without dropping connections.
**The Pattern:** Redirecting live traffic from an old version of an app to a new version instantly via a Service selector modification.

### 🌟 CKAD Style Question
**Task:** A service named `frontend-svc` is currently routing traffic to a deployment named `frontend-v1` (using the label `version: v1`). A new deployment named `frontend-v2` (using the label `version: v2`) has been successfully rolled out. Reconfigure `frontend-svc` to instantly route all traffic to `frontend-v2`. Do not delete any deployments.

### 💡 Solution
You can do this directly from the command line by patching or editing the Service. 

Using `kubectl edit`:
```bash
kubectl edit svc frontend-svc
```
Find the `selector` block and change `version: v1` to `version: v2`:
```yaml
spec:
  # ... other fields
  selector:
    app: frontend
    version: v2 # Changed from v1
```
Save and exit. Traffic snaps to the new deployment instantly.

---

## 6. The "Lifecycle & Restart" Trap (Pods vs Jobs)
Understanding default behaviors is crucial. If a container finishes its process, Kubernetes will try to restart it unless told otherwise.

**The Trap:** Using a standard Pod or Deployment for a one-off script. It defaults to `restartPolicy: Always`, meaning once the script finishes, Kubernetes will endlessly restart the container in a `CrashLoopBackOff`.
**The Pattern:** Running a one-off database migration or initialization script that must run once and then stay in a `Completed` state.

### 🌟 CKAD Style Question
**Task:** Run a one-off pod named `db-init` using the `postgres:13` image that runs the command `psql -U admin -c "CREATE DATABASE mydb"`. Ensure that once the command finishes successfully, the pod does NOT restart.

### 💡 Solution
You must explicitly set the `restartPolicy` to `Never` or `OnFailure`. 
```bash
kubectl run db-init --image=postgres:13 --restart=Never --command -- psql -U admin -c "CREATE DATABASE mydb"
```
*(Note: Generating a Job instead of a bare pod is also an acceptable pattern here, as Jobs default to `OnFailure`/`Never`, but the imperative Pod command with `--restart=Never` is the fastest).*

---

## 7. The "Stable Identity" Pattern (StatefulSets)
While you likely won't build one from scratch, you must know when they are required over a Deployment.

**The Trap:** Using a Deployment for a clustered application. Deployments generate random pod name hashes (e.g., `es-7f8a9b-xyz`), meaning pods cannot reliably discover or communicate with each other upon restart.
**The Pattern:** The application requires sticky network identities, ordered rollouts, or stable storage that survives pod rescheduling (e.g., clustered databases like MongoDB, Elasticsearch, or ZooKeeper).

### 🌟 CKAD Style Question
**Task:** (Multiple Choice / Scenario) You are tasked with deploying a 3-node Elasticsearch cluster. Each node must have a predictable DNS name (e.g., `es-0`, `es-1`, `es-2`) so the nodes can discover each other on the network. Which workload resource and service combination should you use?

### 💡 Solution
**Answer:** A **StatefulSet** paired with a **Headless Service** (`clusterIP: None`). 
StatefulSets guarantee ordered, numbered, and sticky identities, allowing the headless service to create reliable DNS records for each individual pod.
