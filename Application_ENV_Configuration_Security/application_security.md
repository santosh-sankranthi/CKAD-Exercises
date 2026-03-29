## Understand Application Security

When the CKAD syllabus says "Understand Application Security," it is talking specifically about the **`securityContext`** block. The exam wants to know if you can lock down a container so that if a hacker breaches your application, they are trapped in a powerless, restricted environment.

There is a massive trap in this topic that catches candidates every time: **Some security settings belong at the Pod level, and some belong at the Container level.** If you put them in the wrong place, the YAML will fail to apply.

-----

### Task 1: User Identity & Execution (Most Repeating)

By default, most containers run as the `root` user (User ID 0). The exam will ask you to force the entire Pod, or specific containers, to run as restricted, non-privileged users.

#### Main Task 1: Non-Root Execution (Pod-Level)

**1. CKAD Style Question:**
Create a Pod named `non-root-pod` in the `default` namespace using the `redis:alpine` image.
Configure the Pod's security context so that all containers within it run as User ID `1000` and Group ID `2000`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get pod non-root-pod -o jsonpath='{.spec.securityContext.runAsUser}')" == "1000" ] && echo "✅ runAsUser is 1000" || echo "❌ runAsUser failed"
[ "$(kubectl get pod non-root-pod -o jsonpath='{.spec.securityContext.runAsGroup}')" == "2000" ] && echo "✅ runAsGroup is 2000" || echo "❌ runAsGroup failed"
# Live verification inside the container
kubectl exec non-root-pod -- id -u | grep -q "1000" && echo "✅ Container is actually running as UID 1000" || echo "❌ Container is running as wrong user"
```

<details>
  
4. Solution:

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
rm -f non-root.yaml
```

#### Variation 1.1: Enforcing `runAsNonRoot`

**1. CKAD Style Question:**
Create a Pod named `strict-user-pod` using the `busybox` image and command `sleep 3600`.
At the **container level**, explicitly set `runAsNonRoot: true`. Because the image runs as root by default, you must also specify `runAsUser: 1000` so the container successfully starts without violating the rule.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get pod strict-user-pod -o jsonpath='{.spec.containers[0].securityContext.runAsNonRoot}')" == "true" ] && echo "✅ runAsNonRoot enforced" || echo "❌ runAsNonRoot failed"
```

<details>
  
4. Solution:

```bash
kubectl run strict-user-pod --image=busybox --dry-run=client -o yaml -- sleep 3600 > strict-user.yaml
vi strict-user.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: strict-user-pod
spec:
  containers:
  - image: busybox
    name: strict-user-pod
    command: ["sleep", "3600"]
    securityContext:               # CONTAINER LEVEL
      runAsNonRoot: true
      runAsUser: 1000
```

```bash
kubectl apply -f strict-user.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod strict-user-pod
rm -f strict-user.yaml
```

#### Variation 1.2: The Container Override Trap

**1. CKAD Style Question:**
Create a Pod named `override-pod` using the `nginx:alpine` image.
Set the **Pod-level** security context to `runAsUser: 1000`.
However, set the **Container-level** security context to `runAsUser: 2000`. (Kubernetes will prioritize the container-level setting).

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get pod override-pod -o jsonpath='{.spec.securityContext.runAsUser}')" == "1000" ] && echo "✅ Pod-level user is 1000" || echo "❌ Pod-level user failed"
[ "$(kubectl get pod override-pod -o jsonpath='{.spec.containers[0].securityContext.runAsUser}')" == "2000" ] && echo "✅ Container-level override is 2000" || echo "❌ Container-level override failed"
```

<details>

4. Solution:

```bash
kubectl run override-pod --image=nginx:alpine --dry-run=client -o yaml > override.yaml
vi override.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: override-pod
spec:
  securityContext:        # POD LEVEL
    runAsUser: 1000
  containers:
  - image: nginx:alpine
    name: override-pod
    securityContext:      # CONTAINER LEVEL OVERRIDE
      runAsUser: 2000
```

```bash
kubectl apply -f override.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod override-pod
rm -f override.yaml
```

#### Variation 1.3: Volume Ownership (`fsGroup`)

**1. CKAD Style Question:**
Create a Pod named `shared-vol-pod` using the `nginx:alpine` image.
Mount an `emptyDir` volume named `data-vol` to `/usr/share/data`.
Configure the Pod's security context so that the mounted volume is owned by Group ID `3000`.
*(Note: You do not need to change the `runAsUser` for this specific task, only the filesystem group).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.3 ---"
[ "$(kubectl get pod shared-vol-pod -o jsonpath='{.spec.securityContext.fsGroup}')" == "3000" ] && echo "✅ fsGroup successfully configured" || echo "❌ fsGroup missing or incorrect"
# Live verification (checking the permissions of the mounted directory)
kubectl exec shared-vol-pod -- stat -c "%g" /usr/share/data | grep -q "3000" && echo "✅ Volume ownership physically verified inside container" || echo "❌ Volume ownership failed"
```

<details>

4. Solution:

```bash
# 1. Generate the base Pod YAML
kubectl run shared-vol-pod --image=nginx:alpine --dry-run=client -o yaml > fsgroup.yaml

# 2. Edit the YAML
vi fsgroup.yaml
```

*Add the `volumes` block, the `volumeMounts`, and the `fsGroup` at the Pod-level `securityContext`:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-vol-pod
spec:
  securityContext:        # POD LEVEL
    fsGroup: 3000
  volumes:
  - name: data-vol
    emptyDir: {}
  containers:
  - image: nginx:alpine
    name: shared-vol-pod
    volumeMounts:
    - name: data-vol
      mountPath: /usr/share/data
```

```bash
# 3. Apply the YAML
kubectl apply -f fsgroup.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod shared-vol-pod
rm -f fsgroup.yaml
```

-----

### Task 2: Filesystem & Privilege Escalation (Highly Repeating)

Hackers often try to download malware into a compromised container or escalate their privileges. You stop this by making the filesystem read-only and explicitly blocking escalation.

#### Main Task 2: Immutable Filesystems

**1. CKAD Style Question:**
Create a Pod named `locked-down-app` using the `nginx:alpine` image.
Configure the container's security context to enforce a **read-only root filesystem** and explicitly set **allowPrivilegeEscalation to false**.
*(Note: Because Nginx needs to write PID files to run, you must also mount a standard `emptyDir` volume to `/var/run` so the app doesn't crash).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get pod locked-down-app -o jsonpath='{.spec.containers[0].securityContext.readOnlyRootFilesystem}')" == "true" ] && echo "✅ Read-only filesystem configured" || echo "❌ Read-only filesystem failed"
[ "$(kubectl get pod locked-down-app -o jsonpath='{.spec.containers[0].securityContext.allowPrivilegeEscalation}')" == "false" ] && echo "✅ Privilege escalation blocked" || echo "❌ Privilege escalation block failed"
```

<details>

4. Solution:

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
rm -f locked.yaml
```

#### Variation 2.1: The Privileged Container

**1. CKAD Style Question:**
Sometimes infrastructure tools require full host-level access. Create a Pod named `sys-admin-tool` using the `busybox` image and command `sleep 3600`.
Configure the container to run in **privileged mode**. *(Note: This is different from `allowPrivilegeEscalation`).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get pod sys-admin-tool -o jsonpath='{.spec.containers[0].securityContext.privileged}')" == "true" ] && echo "✅ Privileged mode enabled" || echo "❌ Privileged mode failed"
```

<details>
  
4. Solution:

```bash
kubectl run sys-admin-tool --image=busybox --dry-run=client -o yaml -- sleep 3600 > priv.yaml
vi priv.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sys-admin-tool
spec:
  containers:
  - image: busybox
    name: sys-admin-tool
    command: ["sleep", "3600"]
    securityContext:               # CONTAINER LEVEL
      privileged: true
```

```bash
kubectl apply -f priv.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod sys-admin-tool
rm -f priv.yaml
```

-----

### Task 3: Kernel Capabilities & Seccomp Profiles

By default, containerd grants containers a default set of Linux kernel capabilities and syscall permissions. For ultimate security, you must restrict these.

#### Main Task 3: Dropping/Adding Linux Capabilities

**1. CKAD Style Question:**
Create a Pod named `cap-pod` using the `busybox` image. The container should run the command `sleep 3600`.
Modify the container's security context to **drop ALL** Linux capabilities, and explicitly **add the `NET_ADMIN`** capability.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(kubectl get pod cap-pod -o jsonpath='{.spec.containers[0].securityContext.capabilities.drop[0]}')" == "ALL" ] && echo "✅ Capabilities explicitly dropped" || echo "❌ Capabilities drop failed"
[ "$(kubectl get pod cap-pod -o jsonpath='{.spec.containers[0].securityContext.capabilities.add[0]}')" == "NET_ADMIN" ] && echo "✅ NET_ADMIN explicitly added" || echo "❌ NET_ADMIN add failed"
```

<details>

4. Solution:

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
rm -f cap.yaml
```

#### Variation 3.1: Enforcing Seccomp Profiles

**1. CKAD Style Question:**
Create a Pod named `seccomp-pod` using the `nginx` image.
Ensure the Pod's system calls are restricted by explicitly setting the `seccompProfile` type to `RuntimeDefault` at the **Pod level**.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get pod seccomp-pod -o jsonpath='{.spec.securityContext.seccompProfile.type}')" == "RuntimeDefault" ] && echo "✅ Seccomp profile configured" || echo "❌ Seccomp profile failed"
```

<details>

4. Solution:

```bash
kubectl run seccomp-pod --image=nginx --dry-run=client -o yaml > seccomp.yaml
vi seccomp.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-pod
spec:
  securityContext:        # POD LEVEL
    seccompProfile:
      type: RuntimeDefault
  containers:
  - image: nginx
    name: seccomp-pod
```

```bash
kubectl apply -f seccomp.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod seccomp-pod
rm -f seccomp.yaml
```

-----

### Task 4: API Attack Surface Reduction

If a hacker breaches your Pod, they will look for the Service Account token mounted at `/var/run/secrets/kubernetes.io/serviceaccount` to attack the API server. If the app doesn't need to talk to the K8s API, you should block this.

#### Main Task 4: Disabling Token Automount

**1. CKAD Style Question:**
Create a Pod named `no-api-access` using the `nginx` image.
Ensure that Kubernetes does **not** automatically mount a ServiceAccount API token into this pod.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 4 ---"
[ "$(kubectl get pod no-api-access -o jsonpath='{.spec.automountServiceAccountToken}')" == "false" ] && echo "✅ Token automount disabled" || echo "❌ Token automount is still active"
# Live verification
kubectl exec no-api-access -- ls /var/run/secrets/kubernetes.io/serviceaccount 2>&1 | grep -q "No such file" && echo "✅ Verified: Secret is not mounted in container" || echo "❌ Verified: Secret leaked into container"
```

<details>

4. Solution:

```bash
kubectl run no-api-access --image=nginx --dry-run=client -o yaml > no-api.yaml
vi no-api.yaml
```

*Add `automountServiceAccountToken: false` at the Pod spec level:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-api-access
spec:
  automountServiceAccountToken: false   # POD LEVEL (Under spec:)
  containers:
  - image: nginx
    name: no-api-access
```

```bash
kubectl apply -f no-api.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod no-api-access
rm -f no-api.yaml
```
