## Multi-Container design patterns

The "Multi-container Pod design patterns" objective is actually very concentrated. The exam doesn't test 20 different variations; it relentlessly tests **three core patterns** that revolve around one central concept: **Shared Volumes (`emptyDir`)**. If you understand how to make two containers talk to each other via a shared volume, you have mastered 100% of this topic.

---

### Task 1: The Classic Sidecar Pattern (Most Repeating)

The exam almost universally tests this by having a main application generate logs, while a secondary "sidecar" container ships or displays those logs using a shared `emptyDir` volume.

**1. CKAD Style Question:**
Create a Pod named `fluent-logger` in the `default` namespace.

* **Container 1:** Name it `app-container`, use the `busybox` image. It should run the command: `/bin/sh, -c, while true; do date >> /var/log/app.log; sleep 5; done`.
* **Container 2:** Name it `sidecar-container`, use the `busybox` image. It should run the command: `/bin/sh, -c, tail -f /var/log/app.log`.
* Both containers must mount an `emptyDir` volume named `log-volume` at the path `/var/log`.

**2. Setup Script:**
*(None required, uses default namespace)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get pod fluent-logger -o jsonpath='{.spec.containers[0].volumeMounts[0].name}')" == "log-volume" ] && echo "✅ Main container volume mounted" || echo "❌ Main container volume failed"
[ "$(kubectl get pod fluent-logger -o jsonpath='{.spec.containers[1].volumeMounts[0].name}')" == "log-volume" ] && echo "✅ Sidecar volume mounted" || echo "❌ Sidecar volume failed"
kubectl logs fluent-logger -c sidecar-container | grep -q "$(date +%Y | cut -c1-2)" && echo "✅ Sidecar is reading logs" || echo "❌ Sidecar log check failed (wait a few seconds and run again)"

```

<details>

**4. Solution:**

```bash
# Generate the base YAML with the first container
kubectl run fluent-logger --image=busybox --dry-run=client -o yaml --command -- /bin/sh -c "while true; do date >> /var/log/app.log; sleep 5; done" > sidecar.yaml

# Edit the YAML
vi sidecar.yaml

```

*Make these additions:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fluent-logger
spec:
  volumes:                         # 1. Add the emptyDir volume
  - name: log-volume
    emptyDir: {}
  containers:
  - name: app-container
    image: busybox
    command:
    - /bin/sh
    - -c
    - while true; do date >> /var/log/app.log; sleep 5; done
    volumeMounts:                  # 2. Mount it to container 1
    - name: log-volume
      mountPath: /var/log
  - name: sidecar-container        # 3. Add the second container
    image: busybox
    command: ["/bin/sh", "-c", "tail -f /var/log/app.log"]
    volumeMounts:                  # 4. Mount it to container 2
    - name: log-volume
      mountPath: /var/log

```

```bash
kubectl apply -f sidecar.yaml

```

</details>
  
**5. Clean-up Script:**

```bash
kubectl delete pod fluent-logger
rm sidecar.yaml

```

---

### Task 2: The Init Container Pattern (Highly Repeating)

This tests your knowledge of the Pod lifecycle. You must configure a container that runs to completion *before* the main application is allowed to start.

**1. CKAD Style Question:**
Create a Pod named `web-setup` in the `default` namespace.

* **Init Container:** Name it `setup-data`, use the `busybox` image. It must run the command `wget -O /work-dir/index.html http://neverssl.com/online`.
* **Main Container:** Name it `web-app`, use the `nginx:alpine` image.
* Provide an `emptyDir` volume named `html-dir`. Mount it to `/work-dir` in the init container, and to `/usr/share/nginx/html` in the main container.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get pod web-setup -o jsonpath='{.spec.initContainers[0].name}')" == "setup-data" ] && echo "✅ Init container exists" || echo "❌ Init container missing"
STATUS=$(kubectl get pod web-setup -o jsonpath='{.status.phase}')
if [ "$STATUS" == "Running" ]; then
    kubectl exec web-setup -c web-app -- grep -q "html" /usr/share/nginx/html/index.html && echo "✅ Data successfully seeded by init container" || echo "❌ Data missing in main container"
else
    echo "❌ Pod is not running yet. Current status: $STATUS"
fi

```

<details>

**4. Solution:**

```bash
# Generate the base YAML
kubectl run web-setup --image=nginx:alpine --dry-run=client -o yaml > init.yaml

# Edit the YAML
vi init.yaml

```

*Make these additions:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-setup
spec:
  volumes:                          # 1. Add volume
  - name: html-dir
    emptyDir: {}
  initContainers:                   # 2. Add the initContainers array
  - name: setup-data
    image: busybox
    command: ['sh', '-c', 'wget -O /work-dir/index.html http://neverssl.com/online']
    volumeMounts:
    - name: html-dir
      mountPath: /work-dir
  containers:
  - image: nginx:alpine
    name: web-app
    volumeMounts:                   # 3. Mount to main container
    - name: html-dir
      mountPath: /usr/share/nginx/html

```

```bash
kubectl apply -f init.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod web-setup
rm init.yaml

```

---

### Task 3: The Adapter/Transformer Pattern (Moderately Repeating)

Technically a variation of the sidecar pattern, but tested differently. Instead of just shipping logs, the secondary container *transforms* data written by the primary container.

**1. CKAD Style Question:**
Create a Pod named `data-adapter`.

* **Main Container:** Name it `generator`, using `busybox`. Command: `/bin/sh, -c, while true; do echo "12345" >> /data/raw.txt; sleep 5; done`.
* **Adapter Container:** Name it `transformer`, using `busybox`. Command: `/bin/sh, -c, tail -f /data/raw.txt | sed 's/12345/TRANSFORMED_DATA/g'`.
* Mount an `emptyDir` volume named `shared-data` to `/data` in both containers.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
POD_READY=$(kubectl get pod data-adapter -o jsonpath='{.status.containerStatuses[*].ready}')
if [[ "$POD_READY" == *"true true"* ]]; then
    echo "✅ Both containers are running"
    kubectl logs data-adapter -c transformer | grep -q "TRANSFORMED_DATA" && echo "✅ Data is being actively transformed" || echo "❌ Transformation failed"
else
    echo "❌ Pod containers are not fully ready."
fi

```

<details>

**4. Solution:**

```bash
# Generate base YAML
kubectl run data-adapter --image=busybox --dry-run=client -o yaml --command -- /bin/sh -c "while true; do echo '12345' >> /data/raw.txt; sleep 5; done" > adapter.yaml

# Edit the YAML
vi adapter.yaml

```

*Make these additions:*

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-adapter
spec:
  volumes:
  - name: shared-data
    emptyDir: {}
  containers:
  - name: generator
    image: busybox
    command:
    - /bin/sh
    - -c
    - while true; do echo "12345" >> /data/raw.txt; sleep 5; done
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: transformer            # Add the adapter container
    image: busybox
    command: ["/bin/sh", "-c", "tail -f /data/raw.txt | sed 's/12345/TRANSFORMED_DATA/g'"]
    volumeMounts:
    - name: shared-data
      mountPath: /data

```

```bash
kubectl apply -f adapter.yaml

```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod data-adapter
rm adapter.yaml

```

---

If you can confidently generate a basic Pod YAML, jump into `vim`, manually type out an `emptyDir` volume, and properly mount it across `containers` and `initContainers` arrays without syntax errors, you are 100% covered for this topic.

Would you like me to generate a similar drill sheet for the next major topic, **Pod Design (Labels, Annotations, and Node Selectors)**?
