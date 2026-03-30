
## Utilize Persistent and Ephemeral Volumes

### Task 1: The Core Storage Chain (Most Repeating)

This is the classic CKAD storage question. You must manually create a PersistentVolume, create a PersistentVolumeClaim to bind to it, and finally attach that Claim to a Pod.

#### Main Task 1: The PV, PVC, and Pod Assembly

**1. CKAD Style Question:**

1.  Create a PersistentVolume named `app-data-pv`. Configure it with a capacity of `2Gi`, access mode `ReadWriteMany`, and use the `hostPath` `/opt/app-data` on the node.
2.  Create a PersistentVolumeClaim named `app-data-pvc` in the `default` namespace. It must request `1Gi` of storage and use `ReadWriteMany`.
3.  Create a Pod named `data-writer` using the `nginx:alpine` image. Mount the `app-data-pvc` volume to the path `/usr/share/nginx/html`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt/app-data && sudo chmod 777 /opt/app-data
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get pv app-data-pv -o jsonpath='{.spec.capacity.storage}')" == "2Gi" ] && echo "✅ PV capacity correct" || echo "❌ PV capacity failed"
[ "$(kubectl get pvc app-data-pvc -o jsonpath='{.status.phase}')" == "Bound" ] && echo "✅ PVC successfully Bound to PV" || echo "❌ PVC Binding failed"
[ "$(kubectl get pod data-writer -o jsonpath='{.spec.volumes[0].persistentVolumeClaim.claimName}')" == "app-data-pvc" ] && echo "✅ Pod volume references correct PVC" || echo "❌ Pod volume reference failed"
[ "$(kubectl get pod data-writer -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/usr/share/nginx/html" ] && echo "✅ Pod volume mounted to correct path" || echo "❌ Pod mount path failed"
```

<details>

4. Solution:

```bash
# 1. You MUST write the PV and PVC YAML from scratch or copy from the k8s docs.
vi storage.yaml
```

*Write the PV and PVC in the same file to save time:*

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: app-data-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/opt/app-data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f storage.yaml

# 2. Generate the Pod YAML
kubectl run data-writer --image=nginx:alpine --dry-run=client -o yaml > pod.yaml
vi pod.yaml
```

*Add the `volumes` and `volumeMounts`:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-writer
spec:
  volumes:                              # 1. Define the Volume
  - name: pvc-mount
    persistentVolumeClaim:
      claimName: app-data-pvc
  containers:
  - image: nginx:alpine
    name: data-writer
    volumeMounts:                       # 2. Mount it in the container
    - name: pvc-mount
      mountPath: /usr/share/nginx/html
```

```bash
kubectl apply -f pod.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod data-writer
kubectl delete pvc app-data-pvc
kubectl delete pv app-data-pv
rm -f storage.yaml pod.yaml
```

#### Variation 1.1: Explicit Binding via StorageClassName

**1. CKAD Style Question:**
A PV named `slow-disk` exists.
Create a PVC named `slow-claim` requesting `500Mi` of `ReadWriteOnce` storage. You must ensure that `slow-claim` binds **specifically** to the `slow-disk` PV and not to any other available PVs in the cluster.
*(Hint: Use `storageClassName` to explicitly link them).*

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: slow-disk
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: slow-tier    # Explicit storage class
  hostPath:
    path: "/tmp/slow"
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get pvc slow-claim -o jsonpath='{.spec.storageClassName}')" == "slow-tier" ] && echo "✅ PVC requests specific StorageClass" || echo "❌ StorageClass missing"
sleep 2
[ "$(kubectl get pvc slow-claim -o jsonpath='{.spec.volumeName}')" == "slow-disk" ] && echo "✅ PVC explicitly bound to slow-disk PV!" || echo "❌ Binding failed or bound to wrong PV"
```

<details>

4. Solution:

```bash
vi specific-pvc.yaml
```

*The `storageClassName` in the PVC must exactly match the `storageClassName` of the target PV:*

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: slow-claim
spec:
  storageClassName: slow-tier     # THIS LINKS IT SPECIFICALLY
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl apply -f specific-pvc.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pvc slow-claim
kubectl delete pv slow-disk
rm -f specific-pvc.yaml
```

-----

### Task 2: Ephemeral Volumes (`emptyDir`)

When a workload needs scratch space that doesn't need to survive a Pod crash (like a cache, a sorting directory, or sharing files between containers), you use `emptyDir`.

#### Main Task 2: The Standard Ephemeral Cache

**1. CKAD Style Question:**
Create a Pod named `data-processor` using the `busybox` image and command `sleep 3600`.
The application requires a temporary scratch space. Configure the Pod with an `emptyDir` volume named `scratch-space` and mount it to the container at `/var/cache/app`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get pod data-processor -o jsonpath='{.spec.volumes[0].emptyDir}')" != "" ] && echo "✅ emptyDir volume successfully created" || echo "❌ emptyDir volume missing"
[ "$(kubectl get pod data-processor -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/var/cache/app" ] && echo "✅ emptyDir successfully mounted" || echo "❌ Mount path incorrect"
```

<details>

4. Solution:

```bash
kubectl run data-processor --image=busybox --dry-run=client -o yaml -- sleep 3600 > empty.yaml
vi empty.yaml
```

*Add the `emptyDir` definition. Notice the curly braces `{}` representing an empty object:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
spec:
  volumes:
  - name: scratch-space
    emptyDir: {}                        # This creates the ephemeral volume
  containers:
  - image: busybox
    name: data-processor
    command: ["sleep", "3600"]
    volumeMounts:
    - name: scratch-space
      mountPath: /var/cache/app
```

```bash
kubectl apply -f empty.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod data-processor
rm -f empty.yaml
```

#### Variation 2.1: Memory-Backed Ephemeral Volumes

**1. CKAD Style Question:**
Modify the `emptyDir` requirement. The application needs ultra-fast scratch space backed by RAM, not the node's disk.
Create a Pod named `ram-cache` (`nginx` image). Configure an `emptyDir` volume mounted at `/tmp/ram`.
Set the `emptyDir` medium to `Memory` and enforce a strict `sizeLimit` of `128Mi` so it doesn't consume all the node's RAM.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get pod ram-cache -o jsonpath='{.spec.volumes[0].emptyDir.medium}')" == "Memory" ] && echo "✅ Volume is backed by RAM (Memory)" || echo "❌ Medium is not Memory"
[ "$(kubectl get pod ram-cache -o jsonpath='{.spec.volumes[0].emptyDir.sizeLimit}')" == "128Mi" ] && echo "✅ Size limit of 128Mi successfully applied" || echo "❌ Size limit missing or incorrect"
```

<details>

4. Solution:

```bash
kubectl run ram-cache --image=nginx --dry-run=client -o yaml > ram.yaml
vi ram.yaml
```

*Expand the `emptyDir` block to include `medium` and `sizeLimit`:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ram-cache
spec:
  volumes:
  - name: cache-vol
    emptyDir:
      medium: Memory            # RAM-backed
      sizeLimit: 128Mi          # Prevents out-of-memory errors
  containers:
  - image: nginx
    name: ram-cache
    volumeMounts:
    - name: cache-vol
      mountPath: /tmp/ram
```

```bash
kubectl apply -f ram.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod ram-cache
rm -f ram.yaml
```

-----

### Task 3: Dynamic Provisioning (StorageClasses)

In modern Kubernetes, you rarely create `PersistentVolumes` manually. Instead, you create a PVC that points to a `StorageClass`, and the cloud provider automatically creates the underlying PV for you.

#### Main Task 3: The Dynamic PVC

**1. CKAD Style Question:**
The cluster has a default StorageClass installed.
Do NOT create a PersistentVolume.
Create a PVC named `dynamic-claim` in the `default` namespace requesting `1Gi` of `ReadWriteOnce` storage. Because you omit the `storageClassName` (or set it to the default), the cluster will dynamically provision a PV for you.

**2. Setup Script:**
*(Assumes a standard Minikube/Kubeadm cluster which has a default storage class. If the environment lacks one, the test case handles the Pending state).*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
kubectl get pvc dynamic-claim >/dev/null 2>&1 && echo "✅ PVC created" || echo "❌ PVC missing"
PHASE=$(kubectl get pvc dynamic-claim -o jsonpath='{.status.phase}')
[ "$PHASE" == "Bound" ] || [ "$PHASE" == "Pending" ] && echo "✅ PVC successfully recognized by provisioner (Status: $PHASE)" || echo "❌ PVC configuration is invalid"
```

<details>

4. Solution:

```bash
vi dynamic-pvc.yaml
```

*Create a standard PVC without specifying a specific PV to bind to:*

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f dynamic-pvc.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pvc dynamic-claim
rm -f dynamic-pvc.yaml
```

#### Variation 3.1: Expanding an Existing PVC

**1. CKAD Style Question:**
A dynamically provisioned PVC named `app-storage` is currently requesting `1Gi` of space. The database is growing quickly.
Modify the PVC to expand its requested storage to `3Gi`.
*(Note: This only works if the underlying StorageClass has `allowVolumeExpansion: true`, which is configured in the setup script).*

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-sc
provisioner: k8s.io/minikube-hostpath
allowVolumeExpansion: true
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-storage
spec:
  accessModes: [ "ReadWriteOnce" ]
  storageClassName: expandable-sc
  resources:
    requests:
      storage: 1Gi
EOF
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get pvc app-storage -o jsonpath='{.spec.resources.requests.storage}')" == "3Gi" ] && echo "✅ PVC successfully expanded to 3Gi!" || echo "❌ PVC expansion failed"
```

<details>

4. Solution:

```bash
# You can live-edit a PVC to expand its size!
kubectl edit pvc app-storage
```

*In vim, scroll down to the `resources.requests.storage` field and change `1Gi` to `3Gi`.*

```yaml
  resources:
    requests:
      storage: 3Gi      # Changed from 1Gi
```

*(Save and quit. Kubernetes will instruct the storage provider to expand the disk).*

</details>

**5. Clean-up Script:**

```bash
kubectl delete pvc app-storage
kubectl delete sc expandable-sc
```

-----

### Task 4: PersistentVolume Reclaim Policies (Edge Cases)

When a user deletes a PVC, what happens to the PV and the actual data? The exam tests your ability to protect data from accidental deletion.

#### Main Task 4: Changing the Reclaim Policy to Retain

**1. CKAD Style Question:**
A PersistentVolume named `critical-data-pv` currently has a `persistentVolumeReclaimPolicy` of `Delete`. This means if the attached PVC is deleted, the data is destroyed.
Patch the PV to change its reclaim policy to `Retain` to protect the data.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: critical-data-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    path: "/tmp/critical"
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 4 ---"
[ "$(kubectl get pv critical-data-pv -o jsonpath='{.spec.persistentVolumeReclaimPolicy}')" == "Retain" ] && echo "✅ Reclaim policy successfully changed to Retain!" || echo "❌ Reclaim policy change failed"
```

<details>

4. Solution:

```bash
# You can use the imperative 'patch' command or 'edit'
kubectl patch pv critical-data-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# Alternatively:
# kubectl edit pv critical-data-pv
# (Find persistentVolumeReclaimPolicy and change it to Retain)
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pv critical-data-pv
```

-----

### Task 5: Advanced Mounting Mechanics (`subPath` and `readOnly`)

The exam will test your ability to surgically mount specific folders from a volume, and how to protect data from being accidentally overwritten by a compromised container.

#### Main Task 5: The `subPath` Directory Isolation

**1. CKAD Style Question:**
A PersistentVolumeClaim named `shared-data-claim` exists.
Create a Pod named `multi-writer` in the `default` namespace with two containers: `app-a` and `app-b` (both using the `busybox` image).
Both containers must mount the `shared-data-claim` volume to their respective `/var/data` directories.
However, to prevent them from overwriting each other's files, you must use the `subPath` property:

  * `app-a` must write to a sub-directory on the volume named `dir-a`.
  * `app-b` must write to a sub-directory on the volume named `dir-b`.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data-claim
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 1Gi
EOF
sleep 2
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 5 ---"
[ "$(kubectl get pod multi-writer -o jsonpath='{.spec.containers[0].volumeMounts[0].subPath}')" == "dir-a" ] && echo "✅ Container A subPath configured correctly" || echo "❌ Container A subPath missing or incorrect"
[ "$(kubectl get pod multi-writer -o jsonpath='{.spec.containers[1].volumeMounts[0].subPath}')" == "dir-b" ] && echo "✅ Container B subPath configured correctly" || echo "❌ Container B subPath missing or incorrect"
```

<details>

4. Solution:

```bash
# 1. Generate the base YAML
kubectl run multi-writer --image=busybox --dry-run=client -o yaml -- sleep 3600 > subpath.yaml

# 2. Edit the YAML to add the second container and the specific mounts
vi subpath.yaml
```

*You define the volume once, but in the `volumeMounts`, you use the `subPath` key to target specific folders inside the PVC:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-writer
spec:
  volumes:
  - name: shared-vol
    persistentVolumeClaim:
      claimName: shared-data-claim
  containers:
  - name: app-a
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: shared-vol
      mountPath: /var/data
      subPath: dir-a              # ISOLATES TO dir-a
  - name: app-b
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: shared-vol
      mountPath: /var/data
      subPath: dir-b              # ISOLATES TO dir-b
```

```bash
# 3. Apply the YAML
kubectl apply -f subpath.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod multi-writer
kubectl delete pvc shared-data-claim
rm -f subpath.yaml
```

#### Variation 5.1: The `readOnly` Mount Shield

**1. CKAD Style Question:**
A PVC named `reference-data` contains static datasets.
Create a Pod named `data-reader` using the `nginx` image. Mount the `reference-data` PVC to `/usr/share/nginx/html`.
To ensure the web server cannot accidentally corrupt the datasets, you must enforce a **read-only** constraint directly at the volume mount level.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: reference-data
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 100Mi
EOF
sleep 2
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 5.1 ---"
[ "$(kubectl get pod data-reader -o jsonpath='{.spec.containers[0].volumeMounts[0].readOnly}')" == "true" ] && echo "✅ readOnly enforcement successfully applied to the mount!" || echo "❌ readOnly parameter is missing or false"
```

<details>

4. Solution:

```bash
kubectl run data-reader --image=nginx --dry-run=client -o yaml > readonly.yaml
vi readonly.yaml
```

*Add the `readOnly: true` flag specifically under the `volumeMounts` block:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-reader
spec:
  volumes:
  - name: ref-vol
    persistentVolumeClaim:
      claimName: reference-data
  containers:
  - name: data-reader
    image: nginx
    volumeMounts:
    - name: ref-vol
      mountPath: /usr/share/nginx/html
      readOnly: true                # THIS IS THE EXAM REQUIREMENT
```

```bash
kubectl apply -f readonly.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod data-reader
kubectl delete pvc reference-data
rm -f readonly.yaml
```

-----

### The Final 100% Verdict

By mastering the **manual PV -\> PVC -\> Pod chain**, executing **Memory-backed `emptyDir` caches**, triggering **Dynamic Provisioning** (and subsequent volume expansion), and safeguarding data via **Reclaim Policies**, you have definitively walled off the entirety of the "Utilize persistent and ephemeral volumes" objective.

