## Utilize Persistent and Ephermal Volumes

---

### Task 1: The Standard Storage Claim (Most Repeating)

This is the most common storage scenario on the exam. Assuming a default StorageClass exists in the cluster (which provides dynamic provisioning), you just need to create a Claim and attach it to a Pod.

**1. CKAD Style Question:**
Create a PersistentVolumeClaim (PVC) named `app-data-pvc` in the `default` namespace. It should request `1Gi` of storage with the `ReadWriteOnce` access mode. Once created, deploy a Pod named `web-app` using the `nginx:alpine` image, and mount this PVC to the directory `/usr/share/nginx/html`.

**2. Setup Script:**
*(None required, relies on the standard cluster default StorageClass)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get pvc app-data-pvc -o jsonpath='{.spec.resources.requests.storage}')" == "1Gi" ] && echo "✅ PVC requests 1Gi" || echo "❌ PVC storage request failed"
[ "$(kubectl get pod web-app -o jsonpath='{.spec.volumes[0].persistentVolumeClaim.claimName}')" == "app-data-pvc" ] && echo "✅ Pod is using the correct PVC" || echo "❌ Pod volume mount failed"
[ "$(kubectl get pod web-app -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/usr/share/nginx/html" ] && echo "✅ Mount path is correct" || echo "❌ Mount path failed"

```

<details>

**4. Solution:**

```bash
# 1. Create the PVC YAML manually (no imperative command exists for PVCs)
vi pvc.yaml

```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

```bash
kubectl apply -f pvc.yaml

# 2. Generate the Pod YAML
kubectl run web-app --image=nginx:alpine --dry-run=client -o yaml > pod.yaml

# 3. Edit the Pod YAML to mount the PVC
vi pod.yaml

```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: app-data-pvc   # Attach the PVC here
  containers:
  - image: nginx:alpine
    name: web-app
    volumeMounts:
      - name: data-volume
        mountPath: /usr/share/nginx/html  # Mount it inside the container

```

```bash
kubectl apply -f pod.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod web-app
kubectl delete pvc app-data-pvc
rm pvc.yaml pod.yaml

```

---

### Task 2: The Full Static Chain (PV -> PVC -> Pod) (Highly Repeating)

Sometimes the exam will not let you rely on dynamic provisioning. They will ask you to create a specific `hostPath` PersistentVolume first, then create a PVC that specifically binds to it.

**1. CKAD Style Question:**

1. Create a PersistentVolume named `host-storage-pv` with a capacity of `2Gi` and access mode `ReadWriteMany`. Use the `hostPath` storage type pointing to `/opt/data` on the node.
2. Create a PersistentVolumeClaim named `host-storage-pvc` requesting `2Gi` and `ReadWriteMany` access.
3. Create a Pod named `data-writer` using the `busybox` image. It should run the command `sleep 3600`. Mount the PVC to `/data` in the container.

**2. Setup Script:**
*(None required. The `hostPath` will be created on the node automatically if it doesn't exist).*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get pv host-storage-pv -o jsonpath='{.spec.capacity.storage}')" == "2Gi" ] && echo "✅ PV capacity is 2Gi" || echo "❌ PV failed"
[ "$(kubectl get pvc host-storage-pvc -o jsonpath='{.status.phase}')" == "Bound" ] && echo "✅ PVC is Bound to the PV" || echo "❌ PVC is not Bound"
[ "$(kubectl get pod data-writer -o jsonpath='{.spec.volumes[0].persistentVolumeClaim.claimName}')" == "host-storage-pvc" ] && echo "✅ Pod is using the correct PVC" || echo "❌ Pod PVC failed"

```

<details>

**4. Solution:**

```bash
# 1. Create the PV and PVC YAMLs
vi storage.yaml

```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: host-storage-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  hostPath:                  # Crucial for this specific task
    path: /opt/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: host-storage-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi

```

```bash
kubectl apply -f storage.yaml

# 2. Generate and edit the Pod YAML
kubectl run data-writer --image=busybox --dry-run=client -o yaml -- sleep 3600 > writer.yaml
vi writer.yaml

```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-writer
spec:
  volumes:
    - name: storage-vol
      persistentVolumeClaim:
        claimName: host-storage-pvc
  containers:
  - image: busybox
    name: data-writer
    command: ["sleep", "3600"]
    volumeMounts:
      - name: storage-vol
        mountPath: /data

```

```bash
kubectl apply -f writer.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod data-writer
kubectl delete pvc host-storage-pvc
kubectl delete pv host-storage-pv
rm storage.yaml writer.yaml

```

---

### Task 3: Ephemeral Memory Volume (Moderately Repeating)

You already mastered the standard `emptyDir` in the multi-container section. The "gotcha" variation of this topic is asking you to back that ephemeral volume with node memory (RAM) instead of disk space.

**1. CKAD Style Question:**
Create a Pod named `cache-pod` using the `redis:alpine` image. Mount an ephemeral volume to the path `/data/cache`. Ensure this volume is backed by node memory (tmpfs) rather than the node's hard drive.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get pod cache-pod -o jsonpath='{.spec.volumes[0].emptyDir.medium}')" == "Memory" ] && echo "✅ emptyDir is backed by Memory" || echo "❌ Memory medium failed"
[ "$(kubectl get pod cache-pod -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/data/cache" ] && echo "✅ Mount path is correct" || echo "❌ Mount path failed"

```

<details>

**4. Solution:**

```bash
# 1. Generate the base Pod YAML
kubectl run cache-pod --image=redis:alpine --dry-run=client -o yaml > cache.yaml

# 2. Edit the YAML to add the emptyDir with the Memory medium
vi cache.yaml

```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
spec:
  volumes:
  - name: mem-cache
    emptyDir:
      medium: Memory   # This is the exact keyword they are testing for
  containers:
  - image: redis:alpine
    name: cache-pod
    volumeMounts:
    - name: mem-cache
      mountPath: /data/cache

```

```bash
# 3. Apply the configuration
kubectl apply -f cache.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod cache-pod
rm cache.yaml

```

---

### Task 4: The `subPath` Mount (The Curveball)

This tests your ability to surgically mount storage without destroying the container's existing file system.

**1. CKAD Style Question:**
Create a PersistentVolumeClaim named `app-logs-pvc` requesting `1Gi` of storage with `ReadWriteOnce` access.
Next, create a Pod named `logger-pod` using the `nginx:alpine` image. You must mount the PVC to the container's `/var/log/nginx` directory, but you must **only** mount a specific sub-directory from the volume named `processed-logs`. Do not mount the root of the volume.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 4 ---"
[ "$(kubectl get pvc app-logs-pvc -o jsonpath='{.spec.resources.requests.storage}')" == "1Gi" ] && echo "✅ PVC requests 1Gi" || echo "❌ PVC storage request failed"
[ "$(kubectl get pod logger-pod -o jsonpath='{.spec.containers[0].volumeMounts[0].subPath}')" == "processed-logs" ] && echo "✅ subPath is correctly configured" || echo "❌ subPath failed or is missing"
[ "$(kubectl get pod logger-pod -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/var/log/nginx" ] && echo "✅ Mount path is correct" || echo "❌ Mount path failed"

```

<details>

**4. Solution:**

```bash
# 1. Create the PVC YAML
vi pvc-logs.yaml

```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-logs-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

```bash
kubectl apply -f pvc-logs.yaml

# 2. Generate the base Pod YAML
kubectl run logger-pod --image=nginx:alpine --dry-run=client -o yaml > logger-pod.yaml

# 3. Edit the Pod YAML to add the volume, mountPath, AND subPath
vi logger-pod.yaml

```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: logger-pod
spec:
  volumes:
  - name: log-volume
    persistentVolumeClaim:
      claimName: app-logs-pvc
  containers:
  - image: nginx:alpine
    name: logger-pod
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/nginx
      subPath: processed-logs    # THIS is the magic line they test for

```

```bash
# 4. Apply the configuration
kubectl apply -f logger-pod.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod logger-pod
kubectl delete pvc app-logs-pvc
rm pvc-logs.yaml logger-pod.yaml

```
