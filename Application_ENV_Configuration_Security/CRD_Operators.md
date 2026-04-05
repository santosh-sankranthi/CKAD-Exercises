## Discover and use resoruces that extend Kubernetes   

---

It includes all 3 Main Tasks and their 6 Variations, ordered from the most repeating scenarios down to the schema validation edge cases, strictly following your 5-component format with expandable solutions.

-----

### Task 1: Discover, Deconstruct, and Deploy (Most Repeating)

This is the classic CRD exam question. You are dropped into a cluster, told a custom extension exists, and asked to create an object using it. You must use `kubectl get` and `kubectl explain` to figure out the schema on the fly.

#### Main Task 1: The Initial Creation

**1. CKAD Style Question:**
A Custom Resource Definition (CRD) for managing specialized cron tasks is installed in the cluster.

1.  Identify the name and API group of this CRD.
2.  Create a new custom resource of this type named `nightly-backup` in the `default` namespace.
3.  Under its `spec`, configure the `cronSpec` to `"0 0 * * *"` and the `image` to `"backup-runner:v2"`.

**2. Setup Script:**

```bash
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
sleep 2
```

**3. Testcase Script:**

```bash
echo "--- Testing Main Task 1 ---"
[[ "$(kubectl get crontab nightly-backup -o jsonpath='{.spec.cronSpec}' 2>/dev/null)" == "0 0 * * *" ]] && echo "✅ cronSpec matches" || echo "❌ cronSpec failed"
[[ "$(kubectl get crontab nightly-backup -o jsonpath='{.spec.image}' 2>/dev/null)" == "backup-runner:v2" ]] && echo "✅ image matches" || echo "❌ image failed"
```

<details>
  
4. Solution:

```bash
# 1. Discover the installed CRDs to find the API Group and Kind
kubectl get crds

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
rm -f custom-cron.yaml
# We leave the CRD installed for the variations!
```

#### Variation 1.1: Scope and Shortname Discovery

**1. CKAD Style Question:**
You need to document the `crontabs.stable.example.com` CRD for your team.
Use the CLI to discover its accepted short name and its scope (Namespaced or Cluster). Write the short name on the first line and the scope on the second line of a file at `/opt/crd-info.txt`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(sed -n '1p' /opt/crd-info.txt)" == "ct" ] && echo "✅ Short name is correct" || echo "❌ Short name incorrect"
[ "$(sed -n '2p' /opt/crd-info.txt)" == "Namespaced" ] && echo "✅ Scope is correct" || echo "❌ Scope incorrect"
```

<details>
  
4. Solution:

```bash
# Use api-resources to quickly find the shortnames and scope (NAMESPACED column)
kubectl api-resources | grep crontab

# Write the findings to the file
echo "ct" > /opt/crd-info.txt
echo "Namespaced" >> /opt/crd-info.txt
```

</details>

**5. Clean-up Script:**

```bash
rm -f /opt/crd-info.txt
```

#### Variation 1.2: Live Patching a Custom Resource

**1. CKAD Style Question:**
A custom resource named `weekly-cleanup` (of type `CronTab`) is already running. The security team requires the image to be updated. Live-edit the resource to change its `spec.image` to `"backup-runner:v3"`.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  name: weekly-cleanup
spec:
  cronSpec: "0 0 * * 0"
  image: "backup-runner:v1"
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get crontab weekly-cleanup -o jsonpath='{.spec.image}')" == "backup-runner:v3" ] && echo "✅ CR successfully patched" || echo "❌ CR patching failed"
```

<details>
  
4. Solution:

```bash
# You can use standard kubectl imperative commands on Custom Resources!
kubectl patch crontab weekly-cleanup -p '{"spec":{"image":"backup-runner:v3"}}' --type=merge

# Alternatively, just edit it live:
# kubectl edit crontab weekly-cleanup
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete crontab weekly-cleanup
kubectl delete crd crontabs.stable.example.com
```

-----

### Task 2: Querying and Filtering Custom Resources (Highly Repeating)

Instead of asking you to create a Custom Resource, they will populate the cluster with dozens of them and test your ability to query and filter these non-standard objects just like you would native Pods.

#### Main Task 2: Field Filtering

**1. CKAD Style Question:**
A CRD for a resource named `Database` (API group: `db.k8s.local/v1`) is installed. Several `Database` custom resources have been deployed across all namespaces.
Find the `Database` resource that has its `spec.engine` explicitly set to `postgres`. Write the exact name of that specific custom resource to `/opt/postgres-db-name.txt`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
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
echo "--- Testing Main Task 2 ---"
[ "$(cat /opt/postgres-db-name.txt | tr -d '[:space:]')" == "analytics-db" ] && echo "✅ Correct Database name written to file" || echo "❌ Incorrect Database name"
```

<details>
  
4. Solution:

```bash
# Method A (Manual Inspection):
kubectl get databases -A -o yaml | grep -B 5 -A 2 "engine: postgres"
# Visually spot 'analytics-db' and write it:
echo "analytics-db" > /opt/postgres-db-name.txt

# Method B (JSONPath filter - faster):
kubectl get databases -A -o jsonpath='{range .items[?(@.spec.engine=="postgres")]}{.metadata.name}{"\n"}{end}' > /opt/postgres-db-name.txt
```

</details>

**5. Clean-up Script:**

```bash
rm -f /opt/postgres-db-name.txt
# Leave CRs for variations
```

#### Variation 2.1: Custom Columns Output

**1. CKAD Style Question:**
You need a clean report of all databases. Retrieve all `Database` resources in the `default` namespace. Format the output using `custom-columns` to show only the resource name (Header: `NAME`) and its storage capacity (Header: `SIZE`). Save the output to `/opt/db-report.txt`.

**2. Setup Script:**
*(Relies on Setup Script from Main Task 2)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
grep -q "NAME.*SIZE" /opt/db-report.txt && echo "✅ Headers are correct" || echo "❌ Headers missing or incorrect"
grep -q "user-db.*50Gi" /opt/db-report.txt && echo "✅ Data row correctly formatted" || echo "❌ Data row missing or incorrect"
```

<details>
  
4. Solution:

```bash
# Use custom-columns to extract fields from the CRD's JSON/YAML structure
kubectl get databases -n default -o custom-columns=NAME:.metadata.name,SIZE:.spec.storage > /opt/db-report.txt
```

</details>

**5. Clean-up Script:**

```bash
rm -f /opt/db-report.txt
```

#### Variation 2.2: Label Filtering & Bulk Deletion

**1. CKAD Style Question:**
Several `Database` resources were tagged with `tier=cache`. This tier is being deprecated. Find all `Database` resources across all namespaces with the label `tier=cache` and delete them using a single command.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: db.k8s.local/v1
kind: Database
metadata:
  name: redis-cache
  namespace: default
  labels:
    tier: cache
spec:
  engine: redis
  storage: 10Gi
---
apiVersion: db.k8s.local/v1
kind: Database
metadata:
  name: memcached-cache
  namespace: kube-public
  labels:
    tier: cache
spec:
  engine: memcached
  storage: 5Gi
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.2 ---"
kubectl get databases -A -l tier=cache 2>&1 | grep -q "No resources found" && echo "✅ Cache databases successfully deleted" || echo "❌ Cache databases still exist"
kubectl get database user-db -n default >/dev/null 2>&1 && echo "✅ Other databases remain untouched" || echo "❌ FAILED: You deleted the wrong databases"
```

<details>
  
4. Solution:

```bash
# You can use standard label selectors (-l) to perform bulk actions on Custom Resources
kubectl delete databases -A -l tier=cache
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete database --all -A
kubectl delete crd databases.db.k8s.local
```

-----

### Task 3: Schema Validation and Structural Troubleshooting (The Boss Fight)

Because CRDs introduce non-standard YAML structures, the exam tests your ability to read the API server's rejection messages and fix malformed Custom Resources so they comply with the strict OpenAPI schema rules.

#### Main Task 3: Fixing a Schema Validation Error

**1. CKAD Style Question:**
A CRD named `MessageQueue` (`messagequeues.messaging.local`) is installed. It requires the `spec.size` field to be an integer.
You are provided a broken YAML file at `/opt/broken-mq.yaml`. When you try to apply it, it fails validation. Fix the YAML file so it conforms to the schema and apply it to the `default` namespace.

**2. Setup Script:**

```bash
sudo mkdir -p /opt && sudo chmod 777 /opt
# Install CRD that enforces integer typing
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: messagequeues.messaging.local
spec:
  group: messaging.local
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
                size:
                  type: integer    # Enforces integer
  scope: Namespaced
  names:
    plural: messagequeues
    singular: messagequeue
    kind: MessageQueue
EOF
sleep 2

# Provide the broken YAML (size is a string)
cat <<EOF > /opt/broken-mq.yaml
apiVersion: messaging.local/v1
kind: MessageQueue
metadata:
  name: prod-queue
spec:
  size: "5"
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(kubectl get messagequeue prod-queue -o jsonpath='{.spec.size}')" == "5" ] && echo "✅ MessageQueue successfully fixed and applied" || echo "❌ MessageQueue failed validation or missing"
```

<details>
  
4. Solution:

```bash
# 1. Attempt to apply the file to see the specific error:
# kubectl apply -f /opt/broken-mq.yaml
# Error: ... Invalid value: "string": spec.size in body must be of type integer: "string"

# 2. Fix the file by removing the quotes around the number
vi /opt/broken-mq.yaml
```

```yaml
apiVersion: messaging.local/v1
kind: MessageQueue
metadata:
  name: prod-queue
spec:
  size: 5      # Removed the quotes so it is parsed as an integer
```

```bash
# 3. Apply the fixed file
kubectl apply -f /opt/broken-mq.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete messagequeue prod-queue
rm -f /opt/broken-mq.yaml
# Leave CRD for variations
```

#### Variation 3.1: Handling Cluster-Scoped Custom Resources

**1. CKAD Style Question:**
A CRD named `GlobalSetting` (`globalsettings.config.local`) is installed. It is uniquely configured as a **Cluster-scoped** resource (it does not belong to any namespace).
Create a `GlobalSetting` named `security-baseline` with `spec.level: high`. Ensure the YAML structure correctly reflects its cluster-wide scope.

**2. Setup Script:**

```bash
# Install Cluster-scoped CRD
cat <<EOF | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: globalsettings.config.local
spec:
  group: config.local
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
                level:
                  type: string
  scope: Cluster        # This is Cluster scoped!
  names:
    plural: globalsettings
    singular: globalsetting
    kind: GlobalSetting
EOF
sleep 2
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get globalsetting security-baseline -o jsonpath='{.spec.level}')" == "high" ] && echo "✅ Cluster-scoped CR successfully created" || echo "❌ CR creation failed"
```

<details>
  
4. Solution:

```bash
# Because it is Cluster-scoped, you MUST NOT include a 'namespace' field in the metadata block.
vi global.yaml
```

```yaml
apiVersion: config.local/v1
kind: GlobalSetting
metadata:
  name: security-baseline
  # NO namespace field here!
spec:
  level: high
```

```bash
kubectl apply -f global.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete globalsetting security-baseline
kubectl delete crd globalsettings.config.local
rm -f global.yaml
```

#### Variation 3.2: Deep JSONPath on Custom Objects

**1. CKAD Style Question:**
You need to extract a highly nested value from a Custom Resource. A `MessageQueue` named `audit-queue` is running.
Use JSONPath to extract **only** the value of `spec.config.retention.days` from this specific resource, and save the number to `/opt/retention-days.txt`.

**2. Setup Script:**

```bash
# We dynamically patch the CRD to accept arbitrary JSON for this test
kubectl patch crd messagequeues.messaging.local --type=merge -p '{"spec":{"versions":[{"name":"v1","schema":{"openAPIV3Schema":{"x-kubernetes-preserve-unknown-fields":true}}}]}}'
sleep 2
cat <<EOF | kubectl apply -f -
apiVersion: messaging.local/v1
kind: MessageQueue
metadata:
  name: audit-queue
spec:
  size: 10
  config:
    retention:
      days: 30
      archival: true
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.2 ---"
[ "$(cat /opt/retention-days.txt | tr -d '[:space:]')" == "30" ] && echo "✅ Deep JSONPath extraction successful" || echo "❌ Extraction failed or incorrect value"
```

<details>
  
4. Solution:

```bash
# You must navigate the specific schema of the Custom Resource
kubectl get messagequeue audit-queue -o jsonpath='{.spec.config.retention.days}' > /opt/retention-days.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete messagequeue audit-queue
kubectl delete crd messagequeues.messaging.local
rm -f /opt/retention-days.txt
```
### The CRD Strategy

If you understand that CRDs behave exactly like native Kubernetes objects once installed—meaning they respond to `get`, `describe`, `explain`, and `apply -f` in the exact same way—you have mastered this topic.

