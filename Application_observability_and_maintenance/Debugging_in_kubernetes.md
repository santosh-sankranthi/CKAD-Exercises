## Debugging in Kuberenetes

**"Debugging in Kubernetes"** tests your ability to interact with running applications, bypass broken startup commands, and inject tools into minimalist containers (like "distroless" images) that do not have their own shell.

This topic relies heavily on three core commands: `kubectl exec` (for healthy pods with shells), `kubectl debug` (for broken or shell-less pods), and `kubectl run --rm` (for standalone network testing).


-----

### Task 1: Live Shell Execution (`kubectl exec`)

The most basic debugging step is executing a command inside a running container. The exam will ask you to read a file, change a configuration, or test internal network connectivity from the perspective of the application.

#### Main Task 1: The Single Command Execution

**1. CKAD Style Question:**
A Pod named `app-server` is running.
Without opening an interactive terminal session, execute a command directly inside the container to read the contents of the `/etc/hostname` file. Save the output of this command to a file on your local machine at `/opt/app-hostname.txt`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt
sudo chmod 777 /opt
kubectl run app-server --image=nginx
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ -s /opt/app-hostname.txt ] && echo "✅ File created" || echo "❌ File missing"
grep -q "app-server" /opt/app-hostname.txt && echo "✅ Correct hostname successfully extracted" || echo "❌ Hostname missing or incorrect"
```

<details>

4. Solution:

```bash
# You do NOT need the -it flag if you just want to run a single command and exit!
# The double dash (--) separates kubectl arguments from container arguments.
kubectl exec app-server -- cat /etc/hostname > /opt/app-hostname.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod app-server --force
rm -f /opt/app-hostname.txt
```

#### Variation 1.1: Multi-Container Execution

**1. CKAD Style Question:**
A Pod named `web-combo` contains two containers: `nginx-app` and `helper-sidecar`.
Execute a command specifically inside the `helper-sidecar` container to print its environment variables (`env`), and save the output to `/opt/sidecar-env.txt`.

**2. Setup Script:**

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: web-combo
spec:
  containers:
  - name: nginx-app
    image: nginx:alpine
  - name: helper-sidecar
    image: busybox
    command: ["sleep", "3600"]
    env:
    - name: DEBUG_MODE
      value: "true"
EOF
sleep 4
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
grep -q "DEBUG_MODE=true" /opt/sidecar-env.txt && echo "✅ Correct container targeted and env extracted" || echo "❌ Env missing or wrong container targeted"
```

<details>
  
\<summary\>\<b\>4. Solution:\</b\> \<i\>(Click to expand)\</i\>\</summary\>

```bash
# Use the -c flag to specify which container to execute the command in.
kubectl exec web-combo -c helper-sidecar -- env > /opt/sidecar-env.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod web-combo --force
rm -f /opt/sidecar-env.txt
```

#### Variation 1.2: Interactive Troubleshooting

**1. CKAD Style Question:**
A Pod named `db-tester` is running. You need to drop into an interactive shell inside this pod to write a configuration file manually.
Connect to the pod interactively, and use `echo` to write the string `status=healthy` into a file located at `/tmp/health.conf` *inside the container*.

**2. Setup Script:**

```bash
kubectl run db-tester --image=busybox -- sleep 3600
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
kubectl exec db-tester -- cat /tmp/health.conf 2>/dev/null | grep -q "status=healthy" && echo "✅ File successfully created inside the container" || echo "❌ File missing or incorrect inside container"
```

<details>
  
\<summary\>\<b\>4. Solution:\</b\> \<i\>(Click to expand)\</i\>\</summary\>

```bash
# 1. Use -it to get an interactive terminal, and specify 'sh' as the command
kubectl exec -it db-tester -- sh

# 2. You are now inside the container! Run the command:
echo "status=healthy" > /tmp/health.conf

# 3. Type 'exit' to return to your host terminal.
exit
```

*(Alternatively, you can do this as a one-liner without going interactive: `kubectl exec db-tester -- sh -c 'echo "status=healthy" > /tmp/health.conf'`)*

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod db-tester --force
```

-----

### Task 2: Ephemeral Containers (`kubectl debug`)

Modern security practices dictate that production containers should not contain a shell (`sh` or `bash`) or debugging tools (`curl`, `wget`). If an application crashes in a "distroless" image, `kubectl exec` will fail. You must use `kubectl debug` to inject a temporary "ephemeral container" with the tools you need into the running Pod.

#### Main Task 2: Injecting an Ephemeral Container

**1. CKAD Style Question:**
A Pod named `locked-app` is running, but its container lacks a shell.
Inject an ephemeral container into the `locked-app` Pod using the `busybox` image.

**2. Setup Script:**

```bash
# We simulate a locked app by running an image and pretending we can't exec into it.
kubectl run locked-app --image=nginx
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get pod locked-app -o jsonpath='{.spec.ephemeralContainers[0].image}')" == "busybox" ] && echo "✅ Ephemeral container successfully injected into running pod" || echo "❌ Ephemeral container missing"
```

<details>
  
\<summary\>\<b\>4. Solution:\</b\> \<i\>(Click to expand)\</i\>\</summary\>

```bash
# This command adds a temporary busybox container into the running pod's namespace.
# It drops you into an interactive shell. When you type 'exit', the shell closes, but the container remains in the pod spec.
kubectl debug locked-app -it --image=busybox -- sh

# Type 'exit' immediately if you just need to satisfy the exam requirement.
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod locked-app --force
```

#### Variation 2.1: Pod Copying (Bypassing CrashLoopBackOff)

**1. CKAD Style Question:**
A Pod named `crashing-batch` keeps restarting before you can read its logs because its startup command is invalid. You cannot inject an ephemeral container into a dead pod\!
Use the `kubectl debug` command to create a **copy** of this pod named `crashing-batch-debug`. For the copy, override the failing container's command to just drop you into an interactive shell (`sh`) so you can poke around the filesystem.

**2. Setup Script:**

```bash
# This pod immediately exits with an error
kubectl run crashing-batch --image=busybox --command -- false
sleep 3
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get pod crashing-batch-debug -o jsonpath='{.spec.containers[0].command[0]}')" == "sh" ] && echo "✅ Pod successfully copied and command overridden" || echo "❌ Pod copy failed or command not overridden"
```

<details>
  
\<summary\>\<b\>4. Solution:\</b\> \<i\>(Click to expand)\</i\>\</summary\>

```bash
# The --copy-to flag duplicates the pod structure but gives it a new name.
# It requires you to specify the container name you are copying, and what command you want it to run instead.
kubectl debug crashing-batch -it --copy-to=crashing-batch-debug --container=crashing-batch -- sh

# Type 'exit' to leave the shell once it opens. The copy pod will remain in your cluster.
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod crashing-batch crashing-batch-debug --force
```

#### Variation 2.2: Node-Level Debugging

**1. CKAD Style Question:**
You suspect the node running the cluster is having disk space issues.
Find the name of the active node in your cluster. Use the CLI to create a privileged debugging session on that specific node using the `busybox` image. Save the name of the newly created debug pod to `/opt/debug-node-pod.txt`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.2 ---"
POD_NAME=$(cat /opt/debug-node-pod.txt | tr -d '[:space:]')
[ -n "$POD_NAME" ] && kubectl get pod $POD_NAME | grep -q "node-debugger" && echo "✅ Privileged node debugger pod successfully identified" || echo "❌ Node debugger missing or name incorrect"
```

<details>
  
<summary\>\<b\>4. Solution:\</b\> \<i\>(Click to expand)\</i\>\</summary>

```bash
# 1. Get the name of your active node
kubectl get nodes

# 2. Start a debug session on the node itself (using the name you just found, e.g., 'minikube' or 'kind-control-plane')
# This creates a pod named 'node-debugger-<node-name>-<random>' in the host's IPC, Network, and PID namespaces!
kubectl debug node/<your-node-name> -it --image=busybox

# 3. Type 'exit' to leave the shell.

# 4. Find the name of the debug pod that was left behind
kubectl get pods

# 5. Save the name to the file
echo "node-debugger-<your-node-name>-xxxx" > /opt/debug-node-pod.txt
```

</details>

**5. Clean-up Script:**

```bash
kubectl delete pod $(cat /opt/debug-node-pod.txt | tr -d '[:space:]') --force
rm -f /opt/debug-node-pod.txt
```

-----

### Task 3: The Standalone Network Tester (Temporary Pods)

Instead of debugging an existing broken application, the exam will ask you to verify that a Service's internal DNS routing is actually functioning. The fastest way to do this without leaving garbage behind in the cluster is spinning up a self-deleting pod.

#### Main Task 3: Temporary DNS Resolution Test

**1. CKAD Style Question:**
You need to test if the internal Kubernetes DNS server is resolving the `kubernetes.default` service.
Spin up a temporary, interactive pod named `dns-tester` using the `busybox:1.28` image. Run the command `nslookup kubernetes.default` from inside this pod, save the output to `/opt/dns.txt` on your local host, and ensure the pod is automatically deleted when the command finishes.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
grep -q "Name:" /opt/dns.txt && grep -q "Address 1:" /opt/dns.txt && echo "✅ DNS resolution output successfully captured locally" || echo "❌ DNS output missing or incorrect"
kubectl get pod dns-tester 2>&1 | grep -q "NotFound" && echo "✅ Pod correctly deleted itself (--rm flag used)" || echo "❌ Pod failed to delete itself"
```

<details>

<summary\>\<b\>4. Solution:\</b\> \<i\>(Click to expand)\</i\>\<summary\>

```bash
# Combine run with --rm (remove after execution), -it (interactive), and --restart=Never.
# Adding a command after '--' runs the command instead of the default entrypoint.
# The stdout from the pod is redirected to a file on your LOCAL machine.
kubectl run dns-tester --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default > /opt/dns.txt
```

</details>

**5. Clean-up Script:**

```bash
rm -f /opt/dns.txt
```

-----

If you drill these scripts in your terminal until the imperative commands and YAML modifications are entirely second nature, there is no trap or variation the examiners can build that you have not already dismantled.
