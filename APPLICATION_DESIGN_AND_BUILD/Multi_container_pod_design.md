
## Understand Multi-Container Pod Design

### Task 1: The Sidecar Pattern (Most Repeating)

A Sidecar is a helper container that enhances the main container without changing its code. The most common CKAD scenario is a main application writing logs to a file, and a sidecar container reading that file to forward or expose it.

#### Main Task 1: The Shared Volume Log Reader

**1. CKAD Style Question:**
Create a Pod named `fluent-logger` in the `default` namespace.

  * Create a shared `emptyDir` volume named `log-volume`.
  * **Main Container:** Name it `app-container`, use the `busybox` image. It must mount `log-volume` to `/var/log` and run the command: `/bin/sh, -c, while true; do echo "User logged in" >> /var/log/auth.log; sleep 2; done`
  * **Sidecar Container:** Name it `sidecar-container`, use the `busybox` image. It must mount `log-volume` to `/var/log` and run the command: `/bin/sh, -c, tail -f /var/log/auth.log`

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get pod fluent-logger -o jsonpath='{.spec.volumes[0].name}')" == "log-volume" ] && echo "✅ Shared volume created" || echo "❌ Shared volume missing"
[ "$(kubectl get pod fluent-logger -o jsonpath='{.spec.containers[0].volumeMounts[0].mountPath}')" == "/var/log" ] && echo "✅ Main container mounted correctly" || echo "❌ Main container mount failed"
[ "$(kubectl get pod fluent-logger -o jsonpath='{.spec.containers[1].volumeMounts[0].mountPath}')" == "/var/log" ] && echo "✅ Sidecar container mounted correctly" || echo "❌ Sidecar container mount failed"
kubectl logs fluent-logger -c sidecar-container | grep -q "User logged in" && echo "✅ Sidecar is successfully reading the main container's logs!" || echo "❌ Sidecar log extraction failed"
```

<details>

4. Solution:

```bash
# 1. Generate the base Pod YAML with the first container
kubectl run fluent-logger --image=busybox --dry-run=client -o yaml -- /bin/sh -c 'while true; do echo "User logged in" >> /var/log/auth.log; sleep 2; done' > sidecar.yaml

# 2. Edit the YAML to add the volume and the second container
vi sidecar.yaml
```

*You must manually add the `volumes` block and the second container under `containers`:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fluent-logger
spec:
  volumes:                              # 1. ADD SHARED VOLUME
  - name: log-volume
    emptyDir: {}
  containers:
  - name: app-container
    image: busybox
    command: ["/bin/sh", "-c", "while true; do echo \"User logged in\" >> /var/log/auth.log; sleep 2; done"]
    volumeMounts:                       # 2. MOUNT IN MAIN
    - name: log-volume
      mountPath: /var/log
  - name: sidecar-container             # 3. ADD SIDECAR CONTAINER
    image: busybox
    command: ["/bin/sh", "-c", "tail -f /var/log/auth.log"]
    volumeMounts:                       # 4. MOUNT IN SIDECAR
    - name: log-volume
      mountPath: /var/log
```

```bash
# 3. Apply the YAML
kubectl apply -f sidecar.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod fluent-logger
rm -f sidecar.yaml
```

#### Variation 1.1: Live Pod Mutation (Adding a Sidecar)

**1. CKAD Style Question:**
A Pod named `legacy-app` is currently running with a single container writing to an `emptyDir` volume at `/opt/data`.
You cannot edit the Pod YAML from scratch. Extract the running Pod's configuration, add a sidecar container named `metrics-exporter` (image: `busybox`, command: `sleep 3600`) that mounts the same volume to `/opt/metrics`, and replace the running pod.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: legacy-app
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: main-app
    image: busybox
    command: ["/bin/sh", "-c", "while true; do echo 'data' > /opt/data/metrics.txt; sleep 3600; done"]
    volumeMounts:
    - name: shared-data
      mountPath: /opt/data
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get pod legacy-app -o jsonpath='{.spec.containers[1].name}')" == "metrics-exporter" ] && echo "✅ Sidecar successfully injected into existing architecture" || echo "❌ Sidecar missing"
[ "$(kubectl get pod legacy-app -o jsonpath='{.spec.containers[1].volumeMounts[0].mountPath}')" == "/opt/metrics" ] && echo "✅ Sidecar mounted volume correctly" || echo "❌ Sidecar mount failed"
```

<details>

4. Solution:

```bash
# 1. Extract the running pod's YAML
kubectl get pod legacy-app -o yaml > mutate.yaml

# 2. Delete the running pod (Pods are immutable, you can't add containers live!)
kubectl delete pod legacy-app --force

# 3. Clean up the YAML and add the sidecar
vi mutate.yaml
```

*In vim, delete the cluster-specific metadata (resourceVersion, uid, status, etc.). Then add the second container:*

```yaml
  - name: metrics-exporter
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /opt/metrics
```

```bash
# 4. Re-apply the mutated YAML
kubectl apply -f mutate.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod legacy-app --force
rm -f mutate.yaml
```

-----

### Task 2: The Adapter Pattern (Highly Repeating)

An Adapter is a specialized sidecar that *transforms* the output of the main container so it conforms to a centralized standard (e.g., standardizing log formats for a monitoring tool).

#### Main Task 2: Data Transformation Pipeline

**1. CKAD Style Question:**
Create a Pod named `adapter-pod`.

  * **Main Container:** Name `app`, image `busybox`. Command: `/bin/sh, -c, while true; do echo "ERROR: connection lost" >> /var/log/app.log; sleep 2; done`. Mount an `emptyDir` volume named `data` to `/var/log`.
  * **Adapter Container:** Name `formatter`, image `busybox`. Mount the same volume to `/var/log`. It must run a command that tails `app.log`, replaces the word `ERROR` with `[CRITICAL]`, and outputs it to the console: `/bin/sh, -c, tail -f /var/log/app.log | sed 's/ERROR/[CRITICAL]/g'`

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
kubectl logs adapter-pod -c formatter | grep -q "\[CRITICAL\]: connection lost" && echo "✅ Adapter successfully transformed the main container's data!" || echo "❌ Adapter transformation failed"
```

<details>

4. Solution:

```bash
kubectl run adapter-pod --image=busybox --dry-run=client -o yaml -- /bin/sh -c 'while true; do echo "ERROR: connection lost" >> /var/log/app.log; sleep 2; done' > adapter.yaml
vi adapter.yaml
```

*Structure it identically to a sidecar, but focus on the adapter's transformation command:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter-pod
spec:
  volumes:
  - name: data
    emptyDir: {}
  containers:
  - name: app
    image: busybox
    command: ["/bin/sh", "-c", "while true; do echo \"ERROR: connection lost\" >> /var/log/app.log; sleep 2; done"]
    volumeMounts:
    - name: data
      mountPath: /var/log
  - name: formatter                 # ADAPTER CONTAINER
    image: busybox
    command: ["/bin/sh", "-c", "tail -f /var/log/app.log | sed 's/ERROR/[CRITICAL]/g'"]
    volumeMounts:
    - name: data
      mountPath: /var/log
```

```bash
kubectl apply -f adapter.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod adapter-pod
rm -f adapter.yaml
```

-----

### Task 3: The Ambassador Pattern (Localhost Networking)

An Ambassador acts as a network proxy for the main container. Because containers in a Pod share the same network namespace, the main container can just send traffic to `localhost`, and the Ambassador handles routing it to the outside world.

#### Main Task 3: The Localhost Proxy

**1. CKAD Style Question:**
Create a Pod named `network-proxy`.

  * **Ambassador Container:** Name `haproxy`, use the `nginx` image (which natively listens on port `80`).
  * **Main Container:** Name `app-client`, use the `busybox` image. Command: `/bin/sh, -c, while true; do wget -qO- http://localhost:80; sleep 5; done`
    *(Notice how the main container talks to the ambassador strictly over `localhost`\!)*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(kubectl get pod network-proxy -o jsonpath='{.spec.containers[0].image}')" == "nginx" ] && echo "✅ Ambassador container present" || echo "❌ Ambassador container missing"
kubectl logs network-proxy -c app-client | grep -q "Welcome to nginx" && echo "✅ Main container successfully reached ambassador over localhost!" || echo "❌ Localhost routing failed"
```

<details>

4. Solution:

```bash
kubectl run network-proxy --image=nginx --dry-run=client -o yaml > ambassador.yaml
vi ambassador.yaml
```

*Add the second container. Notice we DO NOT need to define any volumes for this pattern, as they communicate via the shared network\!*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-proxy
spec:
  containers:
  - name: haproxy                   # AMBASSADOR CONTAINER
    image: nginx
  - name: app-client                # MAIN CONTAINER
    image: busybox
    command: ["/bin/sh", "-c", "while true; do wget -qO- http://localhost:80; sleep 5; done"]
```

```bash
kubectl apply -f ambassador.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod network-proxy
rm -f ambassador.yaml
```

-----

### Task 4: InitContainers (The Data Pre-Loader)

While technically a lifecycle hook, `InitContainers` are heavily tested as a multi-container pattern. They run to completion *before* the main container starts, making them perfect for downloading configurations or pre-populating shared volumes.

#### Main Task 4: Pre-Populating a Shared Volume

**1. CKAD Style Question:**
Create a Pod named `web-prep`.

  * Mount an `emptyDir` volume named `html-dir` into both containers.
  * **InitContainer:** Name `downloader`, image `busybox`. Mount the volume to `/workdir`. Command: `/bin/sh, -c, echo "Bootstrapped Content" > /workdir/index.html`
  * **Main Container:** Name `web-server`, image `nginx`. Mount the volume to `/usr/share/nginx/html`.
    *(Because the InitContainer runs first, the Nginx server will serve the text created by the downloader\!)*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 4 ---"
[ "$(kubectl get pod web-prep -o jsonpath='{.spec.initContainers[0].name}')" == "downloader" ] && echo "✅ InitContainer successfully configured" || echo "❌ InitContainer missing"
POD_IP=$(kubectl get pod web-prep -o jsonpath='{.status.podIP}')
kubectl run curl-test --image=busybox --rm -it --restart=Never -- wget -qO- $POD_IP | grep -q "Bootstrapped Content" && echo "✅ Main container is successfully serving the InitContainer's data!" || echo "❌ Shared data bootstrap failed"
```

<details>

4. Solution:

```bash
kubectl run web-prep --image=nginx --dry-run=client -o yaml > init.yaml
vi init.yaml
```

*You must place the `initContainers` block at the same indentation level as `containers`:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-prep
spec:
  volumes:
  - name: html-dir
    emptyDir: {}
  initContainers:                       # ADD THIS ENTIRE BLOCK
  - name: downloader
    image: busybox
    command: ["/bin/sh", "-c", "echo 'Bootstrapped Content' > /workdir/index.html"]
    volumeMounts:
    - name: html-dir
      mountPath: /workdir
  containers:
  - image: nginx
    name: web-server
    volumeMounts:
    - name: html-dir
      mountPath: /usr/share/nginx/html
```

```bash
kubectl apply -f init.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod web-prep
rm -f init.yaml
```

#### Variation 4.1: The InitContainer Delay Trap

**1. CKAD Style Question:**
A Pod named `db-dependency` uses an InitContainer to wait for a database to become ready before starting the main app.
Modify the Pod so that the InitContainer (`wait-service`, image `busybox`) runs the command `sleep 10` before completing. The main container (`app`, image `nginx`) should start normally afterward.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 4.1 ---"
[ "$(kubectl get pod db-dependency -o jsonpath='{.spec.initContainers[0].command[1]}')" == "10" ] && echo "✅ InitContainer delay configured" || echo "❌ InitContainer delay failed"
```

<details>

4. Solution:

```bash
vi delay-init.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: db-dependency
spec:
  initContainers:
  - name: wait-service
    image: busybox
    command: ["sleep", "10"]
  containers:
  - name: app
    image: nginx
```

```bash
kubectl apply -f delay-init.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod db-dependency
rm -f delay-init.yaml
```

-----


#### Main Task 5: The Native Restart-Policy Trick (The modern standard)

**1. CKAD Style Question:**
Create a Pod named `native-sidecar-pod` using the `nginx:alpine` image as the main container (`main-app`).
Instead of adding a standard sidecar under the `containers` array, create a **Native Sidecar** using the `busybox` image running `sleep 3600`.
*(Hint: Native sidecars are defined inside the `initContainers` array but must have their `restartPolicy` explicitly set to `Always`).*

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 5 ---"
[ "$(kubectl get pod native-sidecar-pod -o jsonpath='{.spec.initContainers[0].restartPolicy}')" == "Always" ] && echo "✅ Native sidecar restartPolicy configured correctly" || echo "❌ Native sidecar failed (Is it an initContainer? Is the policy Always?)"
```

<details>

4. Solution:

```bash
kubectl run native-sidecar-pod --image=nginx:alpine --dry-run=client -o yaml > native.yaml
vi native.yaml
```

*You must place the sidecar in the `initContainers` block and add `restartPolicy: Always`. This tells Kubernetes to start it before the main container, but keep it running continuously in the background\!*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: native-sidecar-pod
spec:
  initContainers:                       # IT MUST BE AN INIT CONTAINER
  - name: helper-sidecar
    image: busybox
    command: ["sleep", "3600"]
    restartPolicy: Always               # THIS IS THE MAGIC LINE
  containers:
  - image: nginx:alpine
    name: main-app
```

```bash
kubectl apply -f native.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod native-sidecar-pod
rm -f native.yaml
```

-----

### Task 6: Shared Process Namespace (The Deep Debugger)

By default, containers in a Pod share the network and storage, but their processes are isolated. A sidecar cannot run `ps` and see what the main container is doing. The exam will test your ability to break this isolation for advanced debugging.

#### Main Task 6: Cross-Container Process Signaling

**1. CKAD Style Question:**
Create a Pod named `shared-pid-pod`.
Enable the **shared process namespace** feature for the entire Pod.

  * **Main Container:** Name `app`, image `nginx`.
  * **Sidecar Container:** Name `debugger`, image `busybox`, command `sleep 3600`.
    Verify that the `debugger` container can see the `nginx` processes running in the `app` container.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 6 ---"
[ "$(kubectl get pod shared-pid-pod -o jsonpath='{.spec.shareProcessNamespace}')" == "true" ] && echo "✅ Shared Process Namespace enabled" || echo "❌ Shared Process Namespace missing or false"
kubectl exec shared-pid-pod -c debugger -- ps | grep -q "nginx" && echo "✅ Success: The debugger sidecar can see the nginx processes!" || echo "❌ FAILED: Process isolation is still active"
```

<details>

4. Solution:

```bash
kubectl run shared-pid-pod --image=nginx --dry-run=client -o yaml > pid.yaml
vi pid.yaml
```

*Add the `shareProcessNamespace: true` directive at the Pod `spec` level, and add your second container:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-pid-pod
spec:
  shareProcessNamespace: true           # ADD THIS LINE
  containers:
  - image: nginx
    name: app
  - image: busybox                      # ADD SECOND CONTAINER
    name: debugger
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f pid.yaml
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod shared-pid-pod
rm -f pid.yaml
```

-----
### The 100% Verdict

By engineering scenarios for the **Sidecar** (shared volume readers), **Adapter** (shared volume transformers), **Ambassador** (shared network proxies), and **InitContainers** (bootstrapping prerequisites), you have mastered the precise design patterns tested on the CKAD.

The exam will test your ability to look at an empty YAML file, mentally map out the `volumes:`, `volumeMounts:`, and multiple `containers:`, and wire them together. If you can type out Task 1 and Task 4 from scratch without documentation, this subtopic is 100% secured.
