## Discover and use resoruces that extend Kubernetes   

---

### Task 1: Discover, Deconstruct, and Deploy (Most Repeating)

This is the classic CRD exam question. You are dropped into a cluster, told a custom extension exists, and asked to create an object using it. You must use `kubectl get` and `kubectl explain` to figure out the schema on the fly.

**1. CKAD Style Question:**
A Custom Resource Definition (CRD) for managing specialized cron tasks is installed in the cluster.

1. Identify the name and API group of this CRD.
2. Create a new custom resource of this type named `nightly-backup` in the `default` namespace.
3. Under its `spec`, configure the `cronSpec` to `"0 0 * * *"` and the `image` to `"backup-runner:v2"`.

**2. Setup Script:**

```bash
# This script installs the CRD into your cluster so you can practice discovering it.
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
EOF
sleep 2 # Wait for the API server to register the new endpoint

```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get crontab nightly-backup -o jsonpath='{.spec.cronSpec}')" == "0 0 * * *" ] && echo "✅ cronSpec matches" || echo "❌ cronSpec failed"
[ "$(kubectl get crontab nightly-backup -o jsonpath='{.spec.image}')" == "backup-runner:v2" ] && echo "✅ image matches" || echo "❌ image failed"

```

<details>

**4. Solution:**

```bash
# 1. Discover the installed CRDs to find the API Group and Kind
kubectl get crds

# Look for the one related to cron tasks. You will see 'crontabs.stable.example.com'.
# The Kind is usually the singular form capitalized (CronTab), and the API group is before the first dot in the name (stable.example.com/v1).

# 2. Deconstruct the schema using explain to see what fields are required under 'spec'
kubectl explain crontab.spec

# 3. Create the YAML file manually (imperative commands do not work for custom resources)
vi custom-cron.yaml

```

```yaml
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: nightly-backup
spec:
  cronSpec: "0 0 * * *"
  image: "backup-runner:v2"

```

```bash
# 4. Apply the resource
kubectl apply -f custom-cron.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete crontab nightly-backup
kubectl delete crd crontabs.stable.example.com
rm custom-cron.yaml

```

---

### Task 2: Querying and Filtering Custom Resources (Highly Repeating)

Sometimes the exam flips the script. Instead of asking you to create a Custom Resource, they populate the cluster with dozens of them and test your ability to query and filter these non-standard objects just like you would native Pods or Deployments.

**1. CKAD Style Question:**
A CRD for a resource named `Database` (API group: `db.k8s.local/v1`) is installed. Several `Database` custom resources have been deployed across all namespaces.
Find the `Database` resource that has its `spec.engine` explicitly set to `postgres`. Write the name of that specific custom resource to the file `/opt/postgres-db-name.txt`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt
sudo chmod 777 /opt

# Install the CRD
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.db.k8s.local
spec:
  group: db.k8s.local
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
                storage:
                  type: string
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
    - db
EOF
sleep 2

# Create dummy custom resources
cat <<EOF | kubectl apply -f -
apiVersion: db.k8s.local/v1
kind: Database
metadata:
  name: user-db
  namespace: default
spec:
  engine: mysql
  storage: 50Gi
---
apiVersion: db.k8s.local/v1
kind: Database
metadata:
  name: analytics-db
  namespace: kube-public
spec:
  engine: postgres
  storage: 500Gi
EOF

```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
if [ -f /opt/postgres-db-name.txt ]; then
    [ "$(cat /opt/postgres-db-name.txt)" == "analytics-db" ] && echo "✅ Correct Database name written to file" || echo "❌ Incorrect Database name"
else
    echo "❌ File /opt/postgres-db-name.txt does not exist"
fi

```

<details>

**4. Solution:**

```bash
# 1. Query the custom resource across all namespaces. You can use the shortname if you discovered it, or just the full plural name.
kubectl get databases -A

# 2. Output the YAML or JSON to inspect the 'spec.engine' field for each, or use a jsonpath filter to find it immediately.
# Method A (Manual Inspection):
kubectl get databases -A -o yaml | grep -B 5 -A 2 "engine: postgres"

# Method B (Advanced JSONPath filter - highly recommended for speed):
kubectl get databases -A -o jsonpath='{range .items[?(@.spec.engine=="postgres")]}{.metadata.name}{"\n"}{end}' > /opt/postgres-db-name.txt

# 3. If you used Method A and found the name was 'analytics-db', write it to the file:
echo "analytics-db" > /opt/postgres-db-name.txt

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete database user-db -n default
kubectl delete database analytics-db -n kube-public
kubectl delete crd databases.db.k8s.local
rm -f /opt/postgres-db-name.txt

```

---

### The CRD Strategy

If you understand that CRDs behave exactly like native Kubernetes objects once installed—meaning they respond to `get`, `describe`, `explain`, and `apply -f` in the exact same way—you have mastered this topic.

