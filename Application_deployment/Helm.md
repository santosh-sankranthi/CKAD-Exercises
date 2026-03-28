## Helm

The CKAD does not test your ability to build Helm charts from scratch. It tests your ability to act as a **Consumer**: finding charts, customizing their values, deploying them to specific namespaces, and managing their lifecycle (upgrades/rollbacks).

---

### Group 1: Repositories and Baseline Installation
Before you can deploy, you have to know how to connect your cluster to external chart repositories and install them cleanly.

#### Main Task 1: The Standard Repo & Install
**1. CKAD Style Question:**
Add the `bitnami` Helm repository (`https://charts.bitnami.com/bitnami`). Update your local repository cache. Then, use Helm to install the `bitnami/nginx` chart into the `default` namespace. Name the release `frontend-web`.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
helm repo list | grep -q "bitnami" && echo "✅ Repo added" || echo "❌ Repo missing"
[ "$(helm list -qf "^frontend-web$")" == "frontend-web" ] && echo "✅ Helm release deployed" || echo "❌ Helm release missing"
```

<details>

**4. Solution:**
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install frontend-web bitnami/nginx
```

</details>

**5. Clean-up:** `helm uninstall frontend-web`

#### Variation 1.1: The Specific Version Target
**1. CKAD Style Question:**
You need to install an older, certified version of a chart. Search the `bitnami` repository for the `apache` chart. Install exactly version `9.2.7` of the `bitnami/apache` chart. Name the release `legacy-web`.

**2. Setup Script:** `helm repo add bitnami https://charts.bitnami.com/bitnami; helm repo update`

**3. Testcase Script:**
```bash
[ "$(helm list -f "^legacy-web$" -o jsonpath='{.[0].chart}')" == "apache-9.2.7" ] && echo "✅ Correct chart version installed" || echo "❌ Version mismatch"
```

<details>

**4. Solution:**
```bash
# Optional: search to find available versions
# helm search repo bitnami/apache --versions
helm install legacy-web bitnami/apache --version 9.2.7
```

</details>

**5. Clean-up:** `helm uninstall legacy-web`

#### Variation 1.2: Namespace Isolation
**1. CKAD Style Question:**
Create a new namespace named `cache-env`. Install the `bitnami/redis` chart exclusively into this namespace. Name the release `fast-cache`. Ensure Helm creates the namespace if it doesn't already exist.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(helm list -n cache-env -qf "^fast-cache$")" == "fast-cache" ] && echo "✅ Release exists in correct namespace" || echo "❌ Release missing in namespace"
```

<details>

**4. Solution:**
```bash
# The --create-namespace flag is a massive time-saver on the exam!
helm install fast-cache bitnami/redis --namespace cache-env --create-namespace
```

</details>
  
**5. Clean-up:** `helm uninstall fast-cache -n cache-env; kubectl delete ns cache-env`

---

### Group 2: Value Customization (The Heavyweight)
This is the most highly tested Helm skill. You must know how to override the default configuration of a chart using the command line or a values file.

#### Main Task 2: The Multi-Set Override
**1. CKAD Style Question:**
Install the `bitnami/mariadb` chart. Name the release `user-db`. During installation, you must override two default values using the command line: 
1. Set the root password (`auth.rootPassword`) to `secret123`.
2. Set the default database name (`auth.database`) to `appdata`.

**2. Setup Script:** *(Relies on bitnami repo being present)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
helm get values user-db | grep -q "rootPassword: secret123" && echo "✅ Root password overridden" || echo "❌ Password override failed"
helm get values user-db | grep -q "database: appdata" && echo "✅ Database name overridden" || echo "❌ DB override failed"
```

<details>

**4. Solution:**
```bash
# Use comma separation for multiple --set values to keep it on one line
helm install user-db bitnami/mariadb --set auth.rootPassword=secret123,auth.database=appdata
```

</details>

**5. Clean-up:** `helm uninstall user-db; kubectl delete pvc -l app.kubernetes.io/instance=user-db`

#### Variation 2.1: The Custom `values.yaml` File
**1. CKAD Style Question:**
Sometimes there are too many values to use `--set`. 
1. Extract the default values of the `bitnami/nginx` chart into a file named `custom-values.yaml`.
2. Edit the file to change `replicaCount` to `3`.
3. Install the chart as `big-web` using this custom file.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get deploy big-web-nginx -o jsonpath='{.spec.replicas}')" == "3" ] && echo "✅ Chart installed with custom values file" || echo "❌ Replica count failed"
```

<details>

**4. Solution:**
```bash
# 1. Inspect the chart and pipe the default values to a file
helm show values bitnami/nginx > custom-values.yaml

# 2. Edit the file (change replicaCount: 1 to 3)
vi custom-values.yaml

# 3. Install using the -f flag
helm install big-web bitnami/nginx -f custom-values.yaml
```

</details>

**5. Clean-up:** `helm uninstall big-web; rm custom-values.yaml`

#### Variation 2.2: The Live Upgrade Mutation
**1. CKAD Style Question:**
A Helm release named `active-web` (using `bitnami/nginx`) is currently running. The developers need the service exposed. Upgrade the existing release to change its `service.type` to `NodePort`, without uninstalling the application.

**2. Setup Script:** `helm install active-web bitnami/nginx`

**3. Testcase Script:**
```bash
[ "$(kubectl get svc active-web-nginx -o jsonpath='{.spec.type}')" == "NodePort" ] && echo "✅ Upgraded to NodePort" || echo "❌ Service type is incorrect"
```

<details>

**4. Solution:**
```bash
helm upgrade active-web bitnami/nginx --set service.type=NodePort
```

</details>
  
**5. Clean-up:** `helm uninstall active-web`

---

### Group 3: History, Time Travel, and Dry-Runs
When a chart upgrade breaks the cluster, you must know how to rewind time using Helm's built-in history tracking.

#### Main Task 3: The Broken Upgrade & Rollback
**1. CKAD Style Question:**
A release named `stable-app` (using `bitnami/nginx`) was just upgraded, but the new configuration broke the routing. 
1. Check the Helm history for `stable-app`.
2. Roll the release back to Revision `1` (its original state).

**2. Setup Script:** ```bash
helm install stable-app bitnami/nginx
helm upgrade stable-app bitnami/nginx --set image.tag=broken-tag
sleep 2```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(helm history stable-app | tail -n 1 | awk '{print $4}')" == "rollback" ] && echo "✅ Rollback executed" || echo "❌ Rollback missing from history"
```

<details>

**4. Solution:**
```bash
# 1. View the history to identify the revision numbers
helm history stable-app

# 2. Roll back to revision 1
helm rollback stable-app 1
```

</details>
  
**5. Clean-up:** `helm uninstall stable-app`

#### Variation 3.1: The Dry-Run Template (Troubleshooting)
**1. CKAD Style Question:**
You are about to install `bitnami/nginx` as a release named `test-web`, but you want to see the exact Kubernetes YAML manifests it *would* generate before actually applying it to the cluster. Generate the templates and save them to `/opt/helm-output.yaml`.

**2. Setup Script:** `sudo mkdir -p /opt; sudo chmod 777 /opt`

**3. Testcase Script:**
```bash
grep -q "kind: Deployment" /opt/helm-output.yaml && echo "✅ Templates successfully generated to file" || echo "❌ Template file missing or invalid"
```

<details>

**4. Solution:**
```bash
# The 'template' command renders the chart locally without hitting the API server
helm template test-web bitnami/nginx > /opt/helm-output.yaml
```

</details>

**5. Clean-up:** `rm -f /opt/helm-output.yaml`

#### Variation 3.2: Complete Uninstall and Verify
**1. CKAD Style Question:**
A release named `doomed-app` is running in the `default` namespace. Completely uninstall the release and ensure no artifacts are left behind.

**2. Setup Script:** `helm install doomed-app bitnami/nginx`

**3. Testcase Script:**
```bash
helm status doomed-app 2>&1 | grep -q "not found" && echo "✅ Release uninstalled" || echo "❌ Release still exists"
```

<details>

**4. Solution:**
```bash
helm uninstall doomed-app
```

</details>

**5. Clean-up:** *(Already handled by the solution!)*

---

By mastering these 9 commands (`repo add`, `install`, `install --version`, `install --namespace`, `install --set`, `show values`, `upgrade`, `history`, and `rollback`), you have exhausted the entire Helm feature set testable on the CKAD.

