## understanding application security

When the CKAD syllabus says "Understand Application Security," it is talking specifically about the **`securityContext`** block. The exam wants to know if you can lock down a container so that if a hacker breaches your application, they are trapped in a powerless, restricted environment.

There is a massive trap in this topic that catches candidates every time: **Some security settings belong at the Pod level, and some belong at the Container level.** If you put them in the wrong place, the YAML will fail to apply. 



---

### Task 1: Non-Root Execution (Pod-Level Security)
By default, most containers run as the `root` user (User ID 0). The exam will ask you to force the entire Pod to run as a restricted, non-privileged user.

**1. CKAD Style Question:**
Create a Pod named `non-root-pod` in the `default` namespace using the `redis:alpine` image.
Configure the Pod's security context so that all containers within it run as User ID `1000` and Group ID `2000`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get pod non-root-pod -o jsonpath='{.spec.securityContext.runAsUser}')" == "1000" ] && echo "✅ runAsUser is 1000" || echo "❌ runAsUser failed"
[ "$(kubectl get pod non-root-pod -o jsonpath='{.spec.securityContext.runAsGroup}')" == "2000" ] && echo "✅ runAsGroup is 2000" || echo "❌ runAsGroup failed"
# Live verification inside the container
kubectl exec non-root-pod -- id -u | grep -q "1000" && echo "✅ Container is actually running as UID 1000" || echo "❌ Container is running as wrong user"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base Pod YAML
kubectl run non-root-pod --image=redis:alpine --dry-run=client -o yaml > non-root.yaml

# 2. Edit the YAML
vi non-root.yaml
```
*Add the `securityContext` block at the **Pod level** (same indentation as `containers:`):*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: non-root-pod
spec:
  securityContext:        # POD LEVEL
    runAsUser: 1000
    runAsGroup: 2000
  containers:
  - image: redis:alpine
    name: non-root-pod
```
```bash
# 3. Apply the YAML
kubectl apply -f non-root.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod non-root-pod
rm non-root.yaml
```

---

### Task 2: Immutable Filesystems (Container-Level Security)
Hackers often try to download malware into a compromised container. You can stop this by making the entire root filesystem read-only, and explicitly blocking privilege escalation. **These settings only work at the container level.**

**1. CKAD Style Question:**
Create a Pod named `locked-down-app` using the `nginx:alpine` image.
Configure the container's security context to enforce a **read-only root filesystem** and explicitly set **allowPrivilegeEscalation to false**.
*(Note: Because Nginx needs to write PID files to run, you must also mount a standard `emptyDir` volume to `/var/run` so the app doesn't crash. This is a classic exam crossover!)*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get pod locked-down-app -o jsonpath='{.spec.containers[0].securityContext.readOnlyRootFilesystem}')" == "true" ] && echo "✅ Read-only filesystem configured" || echo "❌ Read-only filesystem failed"
[ "$(kubectl get pod locked-down-app -o jsonpath='{.spec.containers[0].securityContext.allowPrivilegeEscalation}')" == "false" ] && echo "✅ Privilege escalation blocked" || echo "❌ Privilege escalation block failed"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base Pod YAML
kubectl run locked-down-app --image=nginx:alpine --dry-run=client -o yaml > locked.yaml

# 2. Edit the YAML
vi locked.yaml
```
*Add the `securityContext` inside the **containers list**, plus the volume workaround:*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: locked-down-app
spec:
  volumes:
  - name: tmp-run
    emptyDir: {}
  containers:
  - image: nginx:alpine
    name: locked-down-app
    volumeMounts:
    - name: tmp-run
      mountPath: /var/run
    securityContext:               # CONTAINER LEVEL
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
```
```bash
# 3. Apply the YAML
kubectl apply -f locked.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod locked-down-app
rm locked.yaml
```

---

### Task 3: Linux Capabilities (Container-Level Security)
By default, Docker/containerd grants containers a default set of Linux kernel capabilities. For ultimate security, the exam might ask you to drop all of them and only add back exactly what the app needs.

**1. CKAD Style Question:**
Create a Pod named `cap-pod` using the `busybox` image. The container should run the command `sleep 3600`.
Modify the container's security context to **drop ALL** Linux capabilities, and explicitly **add the `NET_ADMIN`** capability.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get pod cap-pod -o jsonpath='{.spec.containers[0].securityContext.capabilities.drop[0]}')" == "ALL" ] && echo "✅ Capabilities explicitly dropped" || echo "❌ Capabilities drop failed"
[ "$(kubectl get pod cap-pod -o jsonpath='{.spec.containers[0].securityContext.capabilities.add[0]}')" == "NET_ADMIN" ] && echo "✅ NET_ADMIN explicitly added" || echo "❌ NET_ADMIN add failed"
```

<details>

**4. Solution:**
```bash
# 1. Generate the base Pod YAML
kubectl run cap-pod --image=busybox --dry-run=client -o yaml -- sleep 3600 > cap.yaml

# 2. Edit the YAML
vi cap.yaml
```
*Add the `capabilities` block under the container's `securityContext`:*
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cap-pod
spec:
  containers:
  - image: busybox
    name: cap-pod
    command: ["sleep", "3600"]
    securityContext:               # CONTAINER LEVEL
      capabilities:
        drop:
          - ALL
        add:
          - NET_ADMIN
```
```bash
# 3. Apply the YAML
kubectl apply -f cap.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod cap-pod
rm cap.yaml
```

---

### The Final Verification
If you know that `runAsUser` generally goes at the Pod level, and `readOnlyRootFilesystem` / `capabilities` go at the Container level, you have bulletproofed this entire objective. If you ever forget during the exam, you can run `kubectl explain pod.spec.securityContext` and `kubectl explain pod.spec.containers.securityContext` to see exactly which fields belong where.
