## Overlapping Topics.
---

### Task 1: The Batch Processing Pipeline (Job + InitContainer + PVC + emptyDir)

This is the ultimate test of the Pod lifecycle combined with storage. You have to pass data from an init container to a main container, and then permanently save the result.

**1. CKAD Style Question:**
Create a PersistentVolumeClaim named `pipeline-pvc` requesting `1Gi` of storage with `ReadWriteOnce` access in the `default` namespace.
Next, create a Job named `data-pipeline`.

* **Init Container:** Name it `fetcher` (image: `busybox`). Command: `sh -c 'echo "RAW_DATA" > /tmp/shared/data.txt'`.
* **Main Container:** Name it `processor` (image: `alpine`). Command: `sh -c 'cat /tmp/shared/data.txt | sed s/RAW/PROCESSED/ > /output/final.txt'`.
* **Volumes:** You must use an `emptyDir` volume named `scratch-pad` mounted at `/tmp/shared` in *both* containers. You must also mount the `pipeline-pvc` to the main container *only*, at the path `/output`.

**2. Setup Script:**
*(None required)*

**3. Testcase Runner Script:**

```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get pvc pipeline-pvc -o jsonpath='{.status.phase}')" == "Bound" ] || [ "$(kubectl get pvc pipeline-pvc -o jsonpath='{.status.phase}')" == "Pending" ] && echo "✅ PVC exists" || echo "❌ PVC missing"
[ "$(kubectl get job data-pipeline -o jsonpath='{.spec.template.spec.initContainers[0].name}')" == "fetcher" ] && echo "✅ Init container configured" || echo "❌ Init container missing"
[ "$(kubectl get job data-pipeline -o jsonpath='{.spec.template.spec.containers[0].volumeMounts[?(@.name=="pipeline-pvc")].mountPath}')" == "/output" ] && echo "✅ PVC mounted correctly to main container" || echo "❌ PVC mount failed"

```

<details>

**4. Solution:**

```bash
# 1. Create the PVC YAML manually
vi pipeline-pvc.yaml

```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pipeline-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

```bash
kubectl apply -f pipeline-pvc.yaml

# 2. Generate the base Job YAML
kubectl create job data-pipeline --image=alpine --dry-run=client -o yaml > pipeline-job.yaml

# 3. Edit the Job YAML to add the init container, volumes, and mounts
vi pipeline-job.yaml

```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-pipeline
spec:
  template:
    spec:
      volumes:
      - name: scratch-pad
        emptyDir: {}
      - name: pipeline-pvc
        persistentVolumeClaim:
          claimName: pipeline-pvc
      initContainers:
      - name: fetcher
        image: busybox
        command: ["sh", "-c", "echo 'RAW_DATA' > /tmp/shared/data.txt"]
        volumeMounts:
        - name: scratch-pad
          mountPath: /tmp/shared
      containers:
      - image: alpine
        name: processor
        command: ["sh", "-c", "cat /tmp/shared/data.txt | sed s/RAW/PROCESSED/ > /output/final.txt"]
        volumeMounts:
        - name: scratch-pad
          mountPath: /tmp/shared
        - name: pipeline-pvc
          mountPath: /output
      restartPolicy: Never

```

```bash
kubectl apply -f pipeline-job.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete job data-pipeline
kubectl delete pvc pipeline-pvc
rm pipeline-pvc.yaml pipeline-job.yaml

```

---

### Task 2: The High-Availability Legacy Adapter (Deployment + Sidecar + Memory emptyDir)

This tests your ability to scale a workload while simultaneously handling local, rapid-access ephemeral storage across multiple containers.

**1. CKAD Style Question:**
Create a Deployment named `legacy-web` with 2 replicas in the `default` namespace.

* **Main Container:** Name it `web-app` (image: `busybox`). Command: `sh -c 'while true; do echo "Access logged" >> /var/log/access.log; sleep 5; done'`.
* **Sidecar Container:** Name it `log-shipper` (image: `busybox`). Command: `sh -c 'tail -f /var/log/access.log'`.
* **Volumes:** Both containers must share an `emptyDir` volume named `mem-logs` mounted at `/var/log`. This volume **must** be backed by node memory (tmpfs), not disk.

**2. Setup Script:**
*(None required)*

**3. Testcase Runner Script:**

```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get deploy legacy-web -o jsonpath='{.spec.replicas}')" == "2" ] && echo "✅ Replicas configured" || echo "❌ Replicas failed"
[ "$(kubectl get deploy legacy-web -o jsonpath='{.spec.template.spec.volumes[0].emptyDir.medium}')" == "Memory" ] && echo "✅ Memory medium configured" || echo "❌ Memory medium failed"
[ "$(kubectl get deploy legacy-web -o jsonpath='{.spec.template.spec.containers[1].name}')" == "log-shipper" ] && echo "✅ Sidecar configured" || echo "❌ Sidecar missing"

```

<details>

**4. Solution:**

```bash
# 1. Generate the base Deployment YAML
kubectl create deployment legacy-web --image=busybox --replicas=2 --dry-run=client -o yaml -- sh -c "while true; do echo 'Access logged' >> /var/log/access.log; sleep 5; done" > legacy-deploy.yaml

# 2. Edit the YAML
vi legacy-deploy.yaml

```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: legacy-web
  name: legacy-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: legacy-web
  template:
    metadata:
      labels:
        app: legacy-web
    spec:
      volumes:
      - name: mem-logs
        emptyDir:
          medium: Memory
      containers:
      - image: busybox
        name: web-app
        command: ["sh", "-c", "while true; do echo 'Access logged' >> /var/log/access.log; sleep 5; done"]
        volumeMounts:
        - name: mem-logs
          mountPath: /var/log
      - name: log-shipper
        image: busybox
        command: ["sh", "-c", "tail -f /var/log/access.log"]
        volumeMounts:
        - name: mem-logs
          mountPath: /var/log

```

```bash
kubectl apply -f legacy-deploy.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete deploy legacy-web
rm legacy-deploy.yaml

```

---

### Task 3: The Stateful Autoprovisioner (StatefulSet + volumeClaimTemplates + subPath)

This is the hardest storage question on the exam. Instead of creating a PVC manually, you must configure a StatefulSet to automatically generate PVCs for each replica using a `volumeClaimTemplate`, and mount it surgically.

**1. CKAD Style Question:**
Create a headless service named `cache-svc` (port 80).
Next, create a StatefulSet named `cache-nodes` with 2 replicas using the `nginx:alpine` image.

* You must associate it with the `cache-svc` service.
* Instead of a standard volume, you must use a `volumeClaimTemplate` named `cache-data` that requests `1Gi` of `ReadWriteOnce` storage.
* Mount this storage to the container at `/usr/share/nginx/html`, but only mount the `subPath` named `node-data`.

**2. Setup Script:**

```bash
# Create the headless service
kubectl create service clusterip cache-svc --clusterip="None" --tcp=80:80

```

**3. Testcase Runner Script:**

```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get sts cache-nodes -o jsonpath='{.spec.volumeClaimTemplates[0].metadata.name}')" == "cache-data" ] && echo "✅ volumeClaimTemplate exists" || echo "❌ volumeClaimTemplate missing"
[ "$(kubectl get sts cache-nodes -o jsonpath='{.spec.template.spec.containers[0].volumeMounts[0].subPath}')" == "node-data" ] && echo "✅ subPath configured" || echo "❌ subPath missing"
[ "$(kubectl get sts cache-nodes -o jsonpath='{.spec.serviceName}')" == "cache-svc" ] && echo "✅ Headless service attached" || echo "❌ Service attachment failed"

```

<details>

**4. Solution:**

```bash
# 1. Generate a Deployment skeleton to hack into a StatefulSet
kubectl create deployment cache-nodes --image=nginx:alpine --replicas=2 --dry-run=client -o yaml > sts-cache.yaml

# 2. Edit the YAML
vi sts-cache.yaml

```

*Changes to make in vim:*

1. Change `kind` to `StatefulSet`.
2. Add `serviceName: "cache-svc"` under `spec`.
3. Remove the `strategy` block.
4. Add the `volumeMounts` and the `volumeClaimTemplates` block at the bottom.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: cache-nodes
  name: cache-nodes
spec:
  serviceName: "cache-svc"
  replicas: 2
  selector:
    matchLabels:
      app: cache-nodes
  template:
    metadata:
      labels:
        app: cache-nodes
    spec:
      containers:
      - image: nginx:alpine
        name: nginx
        volumeMounts:
        - name: cache-data
          mountPath: /usr/share/nginx/html
          subPath: node-data
  volumeClaimTemplates:
  - metadata:
      name: cache-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi

```

```bash
kubectl apply -f sts-cache.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete sts cache-nodes
kubectl delete svc cache-svc
# StatefulSets do not auto-delete their PVCs!
kubectl delete pvc -l app=cache-nodes 
rm sts-cache.yaml

```

---
