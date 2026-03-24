## Understanding Authentication, Authorization and Admission Controllers.


### Task 1: The Core RBAC Triangle (Most Repeating)

This is practically guaranteed to be on the exam. You must link a ServiceAccount to a Role using a RoleBinding, all within a specific namespace.

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
echo "--- Testing Task 1 ---"
# The 'auth can-i' command is your best friend for testing RBAC
[ "$(kubectl auth can-i list pods --as=system:serviceaccount:sec-ns:api-accessor -n sec-ns)" == "yes" ] && echo "✅ RBAC successfully configured" || echo "❌ RBAC failed"
[ "$(kubectl get pod test-pod -n sec-ns -o jsonpath='{.spec.serviceAccountName}')" == "api-accessor" ] && echo "✅ Pod is using correct ServiceAccount" || echo "❌ Pod ServiceAccount failed"

```

<details>

**4. Solution:**

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
rm sec-pod.yaml

```

---

### Task 2: Cluster-Wide RBAC (Highly Repeating)

Sometimes you need to grant an application access to cluster-scoped resources (like PersistentVolumes or Nodes), which cannot be done with a standard Role. You must use a ClusterRole.

**1. CKAD Style Question:**
Create a ServiceAccount named `storage-manager` in the `default` namespace.
Create a ClusterRole named `pv-admin` that has permissions to `create`, `delete`, and `list` `persistentvolumes`.
Bind the `storage-manager` ServiceAccount to the `pv-admin` ClusterRole using a ClusterRoleBinding named `pv-admin-binding`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl auth can-i list pv --as=system:serviceaccount:default:storage-manager)" == "yes" ] && echo "✅ Cluster-wide access granted" || echo "❌ Cluster RBAC failed"

```

<details>

**4. Solution:**

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

---

### Task 3: Managing the Admission Controller (Moderately Repeating)

The Kubernetes "ServiceAccount Admission Controller" automatically mounts API credentials into every single Pod you create. The exam frequently tests your ability to bypass this default behavior for security reasons.

**1. CKAD Style Question:**
Create a Pod named `isolated-pod` in the `default` namespace using the `busybox` image and the command `sleep 3600`.
Configure the Pod so that the default ServiceAccount token is **not** automatically mounted into the container filesystem.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
[ "$(kubectl get pod isolated-pod -o jsonpath='{.spec.automountServiceAccountToken}')" == "false" ] && echo "✅ Token automount disabled" || echo "❌ Token is still mounting"

```

<details>

**4. Solution:**

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
rm isolated.yaml

```

---

### Task 4: Troubleshooting Quotas and Scheduling (The "Pending" Trap)

**1. CKAD Style Question:**
You have been assigned to fix a broken environment. A Deployment named `heavy-calc` in the `crunch-ns` namespace was supposed to scale to 2 replicas, but developers are reporting that the application is down.

1. Investigate the namespace and the deployment to find out why the Pods are failing to schedule.
2. Edit the Deployment to fix the underlying issue.
3. The containers must be reconfigured to request exactly `100m` of CPU and `50Mi` of memory, with a strict limit of `200m` CPU and `100Mi` memory.
4. Verify that exactly 2 Pods are in the `Running` state.

**2. Setup Script:**
*(Run this to intentionally break your cluster for the drill)*

```bash
kubectl create ns crunch-ns
# Create a strict quota
kubectl create quota crunch-quota --hard=pods=2,requests.cpu=1,requests.memory=1Gi,limits.cpu=2,limits.memory=2Gi -n crunch-ns
# Create a deployment that violently violates this quota
cat <<EOF > broken-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: heavy-calc
  namespace: crunch-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: heavy-calc
  template:
    metadata:
      labels:
        app: heavy-calc
    spec:
      containers:
      - name: calc
        image: nginx:alpine
        resources:
          requests:
            cpu: "5"  # This is asking for 5 whole CPU cores per pod!
EOF
kubectl apply -f broken-deploy.yaml
rm broken-deploy.yaml

```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 4 ---"
[ "$(kubectl get deploy heavy-calc -n crunch-ns -o jsonpath='{.status.readyReplicas}')" == "2" ] && echo "✅ 2 Replicas are Running" || echo "❌ Pods are not fully running"
[ "$(kubectl get deploy heavy-calc -n crunch-ns -o jsonpath='{.spec.template.spec.containers[0].resources.requests.cpu}')" == "100m" ] && echo "✅ CPU Request fixed" || echo "❌ CPU Request is incorrect"
[ "$(kubectl get deploy heavy-calc -n crunch-ns -o jsonpath='{.spec.template.spec.containers[0].resources.limits.memory}')" == "100Mi" ] && echo "✅ Memory Limit fixed" || echo "❌ Memory Limit is incorrect"

```

<details>

**4. Solution:**

```bash
# 1. Investigate the Deployment. 
# You will see 0/2 ready. Why? Let's check the events of the ReplicaSet.
kubectl describe deploy heavy-calc -n crunch-ns
# Look closely at the events or conditions. If it doesn't tell you enough, check the ReplicaSet directly:
kubectl get rs -n crunch-ns
kubectl describe rs <insert-replicaset-name> -n crunch-ns
# The Events will scream: "Warning  FailedCreate ... Error creating: pods "heavy-calc-xxx" is forbidden: exceeded quota: crunch-quota, requested: requests.cpu=5, used: requests.cpu=0, limited: requests.cpu=1"

# 2. Fix the Deployment live in the cluster
kubectl edit deploy heavy-calc -n crunch-ns

```

*In the vim editor that opens, scroll down to the `resources` block and fix it to match the requested parameters:*

```yaml
        resources:
          requests:
            cpu: 100m        # Changed from "5"
            memory: 50Mi     # Added
          limits:            # Added limits block
            cpu: 200m
            memory: 100Mi

```

```bash
# Save and quit vim (:wq). The deployment will automatically roll out the new, fixed pods.

# 3. Verify they are running
kubectl get pods -n crunch-ns

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete ns crunch-ns

```

---


### Pro-Tip for the Exam

Never waste time writing Role or RoleBinding YAMLs from scratch. The imperative commands (`kubectl create role ...` and `kubectl create rolebinding ...`) are significantly faster and eliminate the chance of indentation errors.

If you master those imperative RBAC creation commands and the `kubectl auth can-i` command to check your work, you will breeze through this section.
