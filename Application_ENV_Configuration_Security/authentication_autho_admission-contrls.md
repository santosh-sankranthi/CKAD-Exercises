## Understanding Authentication, Authorization and Admission Controllers

For the **Authentication, Authorization, and Admission Controllers** domain, the exam focuses strictly on **RBAC (Role-Based Access Control)** and the **ServiceAccount Admission Controller**.

There are massive traps here that we missed in the first pass:

1.  **Multiple API Groups:** Real-world roles don't just access Pods; they access Deployments (`apps` group) and CronJobs (`batch` group).
2.  **The ClusterRole to RoleBinding Trap:** Using a ClusterRole inside a specific namespace.
3.  **ServiceAccount-Level Automount:** Disabling token mounts globally for an identity, not just a pod.

Here is the fully revised, mathematically exhaustive 4-Part Matrix (with 8 total scenarios) for **Understanding Authentication, Authorization, and Admission Controllers**.

-----

### Task 1: The Core RBAC Triangle (Most Repeating)

This is practically guaranteed to be on the exam. You must link a Subject (User or ServiceAccount) to a Role using a RoleBinding, all within a specific namespace.

#### Main Task 1: The Standard ServiceAccount Binding

**1. CKAD Style Question:**
Create a new namespace named `sec-ns`.
In this namespace, create a ServiceAccount named `api-accessor`.
Next, create a Role named `pod-reader` that has permission to `get`, `watch`, and `list` Pods.
Bind the `api-accessor` ServiceAccount to the `pod-reader` Role using a RoleBinding named `api-access-binding`.
Finally, deploy a Pod named `test-pod` using the `nginx` image that runs as the `api-accessor` ServiceAccount.

**2. Setup Script:**
*(None required, creating the namespace is part of the task)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[[ "$(kubectl auth can-i list pods --as=system:serviceaccount:sec-ns:api-accessor -n sec-ns)" == "yes" ]] && echo "✅ RBAC successfully configured" || echo "❌ RBAC failed"
[["$(kubectl get pod test-pod -n sec-ns -o jsonpath='{.spec.serviceAccountName}')" == "api-accessor" ]] && echo "✅ Pod is using correct ServiceAccount" || echo "❌ Pod ServiceAccount failed"
```

<details>
  
4. Solution:

```bash
# 1. Create the namespace and ServiceAccount
kubectl create ns sec-ns
kubectl create serviceaccount api-accessor -n sec-ns

# 2. Create the Role imperatively
kubectl create role pod-reader --verb=get,watch,list --resource=pods -n sec-ns

# 3. Create the RoleBinding imperatively
kubectl create rolebinding api-access-binding --role=pod-reader --serviceaccount=sec-ns:api-accessor -n sec-ns

# 4. Generate and edit the Pod YAML
kubectl run test-pod --image=nginx -n sec-ns --dry-run=client -o yaml > sec-pod.yaml
vi sec-pod.yaml
```

*Add the `serviceAccountName` under the pod's `spec`:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
  namespace: sec-ns
spec:
  serviceAccountName: api-accessor   # ADD THIS LINE
  containers:
  - image: nginx
    name: test-pod
```

```bash
# 5. Apply the Pod
kubectl apply -f sec-pod.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ns sec-ns
rm -f sec-pod.yaml
```

#### Variation 1.1: Multiple API Groups

**1. CKAD Style Question:**
Create a Role named `app-manager` in the `default` namespace.
This Role must grant the `create`, `update`, and `delete` verbs. However, it must apply to **Deployments** and **StatefulSets**.
*(Hint: These resources do not live in the core API group; they live in the `apps` API group).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[[ "$(kubectl get role app-manager -o jsonpath='{.rules[0].apiGroups[0]}')" == "apps" ]] && echo "✅ API Group 'apps' correctly identified" || echo "❌ API Group failed"
[[ "$(kubectl get role app-manager -o jsonpath='{.rules[0].resources[*]}')" == "deployments statefulsets" ]] && echo "✅ Resources correctly identified" || echo "❌ Resources failed"
```

<details>
  
4. Solution:

```bash
# You specify the API group imperatively by appending it to the resource name with a dot!
kubectl create role app-manager --verb=create,update,delete --resource=deployments.apps,statefulsets.apps
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete role app-manager
```

#### Variation 1.2: Binding a Human User

**1. CKAD Style Question:**
You have a human developer named `jane`.
Create a RoleBinding named `jane-binding` in the `default` namespace that binds the human user `jane` to an existing Role named `developer-role`.

**2. Setup Script:**

```bash
kubectl create role developer-role --verb=create --resource=pods
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[[ "$(kubectl get rolebinding jane-binding -o jsonpath='{.subjects[0].kind}')" == "User" ]] && echo "✅ Subject Kind is User" || echo "❌ Subject Kind is not User"
[[ "$(kubectl get rolebinding jane-binding -o jsonpath='{.subjects[0].name}')" == "jane" ]] && echo "✅ Subject Name is jane" || echo "❌ Subject Name is incorrect"
```

<details>
  
4. Solution:

```bash
# Use the --user flag instead of the --serviceaccount flag!
kubectl create rolebinding jane-binding --role=developer-role --user=jane
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete rolebinding jane-binding
kubectl delete role developer-role
```

-----

### Task 2: Cluster-Wide RBAC vs. Cross-Namespace (Highly Repeating)

Sometimes you need to grant an application access to cluster-scoped resources (like PersistentVolumes or Nodes). Other times, you want to use a global template to grant access to a single namespace.

#### Main Task 2: Pure Cluster-Wide RBAC

**1. CKAD Style Question:**
Create a ServiceAccount named `storage-manager` in the `default` namespace.
Create a ClusterRole named `pv-admin` that has permissions to `create`, `delete`, and `list` `persistentvolumes`.
Bind the `storage-manager` ServiceAccount to the `pv-admin` ClusterRole globally using a ClusterRoleBinding named `pv-admin-binding`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[[ "$(kubectl auth can-i list pv --as=system:serviceaccount:default:storage-manager)" == "yes" ]] && echo "✅ Cluster-wide access granted" || echo "❌ Cluster RBAC failed"
```

<details>

4. Solution:

```bash
# 1. Create the ServiceAccount
kubectl create serviceaccount storage-manager

# 2. Create the ClusterRole imperatively
kubectl create clusterrole pv-admin --verb=create,delete,list --resource=persistentvolumes

# 3. Create the ClusterRoleBinding imperatively
kubectl create clusterrolebinding pv-admin-binding --clusterrole=pv-admin --serviceaccount=default:storage-manager
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete clusterrolebinding pv-admin-binding
kubectl delete clusterrole pv-admin
kubectl delete serviceaccount storage-manager
```

#### Variation 2.1: The "ClusterRole to RoleBinding" Trap

**1. CKAD Style Question:**
You want to grant the ServiceAccount `auditor` (in the `default` namespace) standard "view" permissions, but **only** for resources inside the `finance` namespace.
Kubernetes has a built-in ClusterRole named `view`. Do not create a new Role. Bind the existing `view` ClusterRole to the `auditor` ServiceAccount so the permissions are isolated strictly to the `finance` namespace. Name the binding `finance-auditor-binding`.

**2. Setup Script:**

```bash
kubectl create ns finance
kubectl create serviceaccount auditor
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl auth can-i get pods --as=system:serviceaccount:default:auditor -n finance)" == "yes" ] && echo "✅ Access granted in finance namespace" || echo "❌ Access failed in finance namespace"
[ "$(kubectl auth can-i get pods --as=system:serviceaccount:default:auditor -n default)" == "no" ] && echo "✅ Access properly denied in other namespaces (Trap Avoided!)" || echo "❌ FAILED: Access granted globally. You used a ClusterRoleBinding instead of a RoleBinding!"
```

<details>
  
4. Solution:

```bash
# THE TRAP: You must use a standard 'RoleBinding' to constrain a 'ClusterRole' to a single namespace. 
# If you use a ClusterRoleBinding, the auditor gets access to the whole cluster!

kubectl create rolebinding finance-auditor-binding \
  --clusterrole=view \
  --serviceaccount=default:auditor \
  -n finance
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete rolebinding finance-auditor-binding -n finance
kubectl delete serviceaccount auditor
kubectl delete ns finance
```

-----

### Task 3: Managing the Admission Controller (Moderately Repeating)

The Kubernetes "ServiceAccount Admission Controller" automatically mounts API credentials into every single Pod you create. The exam frequently tests your ability to bypass this default behavior for security reasons.

#### Main Task 3: Pod-Level Token Shielding

**1. CKAD Style Question:**
Create a Pod named `isolated-pod` in the `default` namespace using the `busybox` image and the command `sleep 3600`.
Configure the Pod so that the default ServiceAccount token is **not** automatically mounted into the container filesystem.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(kubectl get pod isolated-pod -o jsonpath='{.spec.automountServiceAccountToken}')" == "false" ] && echo "✅ Token automount disabled on Pod" || echo "❌ Token is still mounting"
```

<details>

4. Solution:

```bash
# 1. Generate the base Pod YAML
kubectl run isolated-pod --image=busybox --dry-run=client -o yaml -- sleep 3600 > isolated.yaml

# 2. Edit the YAML
vi isolated.yaml
```

*Add `automountServiceAccountToken: false` directly under the `spec`:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: isolated-pod
spec:
  automountServiceAccountToken: false   # ADD THIS LINE
  containers:
  - image: busybox
    name: isolated-pod
    command: ["sleep", "3600"]
```

```bash
# 3. Apply the YAML
kubectl apply -f isolated.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod isolated-pod
rm -f isolated.yaml
```

#### Variation 3.1: ServiceAccount-Level Token Shielding

**1. CKAD Style Question:**
Instead of disabling the token mount on every single Pod individually, you can disable it at the source.
Create a ServiceAccount named `paranoid-sa`. Modify the ServiceAccount object itself so that no Pod using this account will automatically mount an API token.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get serviceaccount paranoid-sa -o jsonpath='{.automountServiceAccountToken}')" == "false" ] && echo "✅ Token automount disabled on ServiceAccount" || echo "❌ ServiceAccount configuration failed"
```

<details>

4. Solution:

```bash
# 1. Create the ServiceAccount
kubectl create serviceaccount paranoid-sa

# 2. Patch the ServiceAccount to add the automount setting
kubectl patch serviceaccount paranoid-sa -p '{"automountServiceAccountToken": false}'

# (Alternatively, you can use 'kubectl edit serviceaccount paranoid-sa')
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete serviceaccount paranoid-sa
```

-----

### Task 4: Troubleshooting Authorization (The `auth can-i` Lifeline)

When RBAC is broken, you cannot guess the problem. You must use the API server to test permissions directly.

#### Main Task 4: Diagnosing Broken Verbs

**1. CKAD Style Question:**
A ServiceAccount named `ci-cd-bot` is supposed to be able to delete Deployments in the `default` namespace, but the pipeline is failing with a "Forbidden" error.

1.  Use the CLI to verify if `ci-cd-bot` can delete deployments.
2.  Investigate the `ci-role` and `ci-binding` to find the issue.
3.  Fix the `ci-role` so the pipeline succeeds.

**2. Setup Script:**

```bash
kubectl create serviceaccount ci-cd-bot
kubectl create role ci-role --verb=create,get,update --resource=deployments.apps
kubectl create rolebinding ci-binding --role=ci-role --serviceaccount=default:ci-cd-bot
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 4 ---"
[ "$(kubectl auth can-i delete deployments --as=system:serviceaccount:default:ci-cd-bot)" == "yes" ] && echo "✅ ci-cd-bot can now delete deployments!" || echo "❌ Authorization still failing"
```

<details>
  
4. Solution:

```bash
# 1. Verify the current state
kubectl auth can-i delete deployments --as=system:serviceaccount:default:ci-cd-bot
# Output: no

# 2. Check the role. You will see 'delete' is missing from the verbs list.
kubectl describe role ci-role

# 3. Edit the role live to add the 'delete' verb
kubectl edit role ci-role
```

*In vim, add "delete" to the verbs array:*

```yaml
rules:
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - create
  - get
  - update
  - delete     # ADDED THIS
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete rolebinding ci-binding
kubectl delete role ci-role
kubectl delete serviceaccount ci-cd-bot
```


### Task 5: Pod Security Admission Controllers (The Modern Standard)

The exam will ask you to configure the cluster's admission controller to automatically reject insecure pods in a specific namespace.

#### Main Task 5: Enforcing Namespace Security Standards

**1. CKAD Style Question:**
Create a namespace named `secure-workload`.
Configure the built-in Pod Security Admission controller to **enforce** the `restricted` security standard for all pods deployed in this namespace.
*(Note: Use the latest version for the standard, which can be specified as `latest`)*.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 5 ---"
[ "$(kubectl get ns secure-workload -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}')" == "restricted" ] && echo "✅ Admission Controller enforcement label applied" || echo "❌ Enforcement label missing"
```

<details>
  
4. Solution:

```bash
# 1. Create the namespace
kubectl create ns secure-workload

# 2. Pod Security Admission is entirely controlled via Namespace Labels!
# The syntax is: pod-security.kubernetes.io/<mode>=<standard>
kubectl label namespace secure-workload pod-security.kubernetes.io/enforce=restricted

# (Optional but good practice: you can also set the version)
kubectl label namespace secure-workload pod-security.kubernetes.io/enforce-version=latest
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ns secure-workload
```

#### Variation 5.1: The "Warn" and "Audit" Modes

**1. CKAD Style Question:**
A namespace named `legacy-apps` exists. You want to know if the apps inside it violate the `baseline` security standard, but you do **not** want the admission controller to block or kill them.
Configure the namespace so the admission controller only triggers a **warning** to the user and logs an **audit** event when a violation occurs.

**2. Setup Script:**

```bash
kubectl create ns legacy-apps
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 5.1 ---"
[ "$(kubectl get ns legacy-apps -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/warn}')" == "baseline" ] && echo "✅ Warn mode successfully configured" || echo "❌ Warn mode missing"
[ "$(kubectl get ns legacy-apps -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/audit}')" == "baseline" ] && echo "✅ Audit mode successfully configured" || echo "❌ Audit mode missing"
```

<details>

4. Solution:

```bash
# You can apply multiple modes (enforce, warn, audit) simultaneously using labels.
kubectl label namespace legacy-apps pod-security.kubernetes.io/warn=baseline
kubectl label namespace legacy-apps pod-security.kubernetes.io/audit=baseline
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ns legacy-apps
```

-----


-----

### Pro-Tip for the Exam

Never waste time writing Role or RoleBinding YAMLs from scratch. The imperative commands (`kubectl create role ...` and `kubectl create rolebinding ...`) are significantly faster and eliminate the chance of formatting errors.

If you master those imperative RBAC creation commands and the `kubectl auth can-i` command to check your work, you will absolutely breeze through this section.
