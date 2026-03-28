## Provide and troubleshoot access to applications via services

The subtopic **"Provide and troubleshoot access to applications via services"** tests two distinct skills: your ability to expose Pods to the network (ClusterIP, NodePort, Headless, ExternalName), and your ability to debug *why* a Service isn't routing traffic properly (usually a mismatched selector or targetPort).

---

### Group 1: Internal Access & Discovery (ClusterIP & Headless)
This group focuses on routing traffic *inside* the cluster. By default, Services use `ClusterIP`.

#### Main Task 1: The Standard Internal Expose
**1. CKAD Style Question:**
Create a Deployment named `internal-api` using the `nginx:alpine` image.
Expose it internally using a Service named `api-svc`. The Service must listen on port `8080` and route traffic to the containers on port `80`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get svc api-svc -o jsonpath='{.spec.type}')" == "ClusterIP" ] && echo "✅ Service is ClusterIP" || echo "❌ Service type failed"
[ "$(kubectl get svc api-svc -o jsonpath='{.spec.ports[0].port}')" == "8080" ] && echo "✅ Port is 8080" || echo "❌ Port failed"
[ "$(kubectl get svc api-svc -o jsonpath='{.spec.ports[0].targetPort}')" == "80" ] && echo "✅ TargetPort is 80" || echo "❌ TargetPort failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the deployment
kubectl create deployment internal-api --image=nginx:alpine

# 2. Expose it imperatively. 
# --port is what the Service listens on. --target-port is what the container listens on.
kubectl expose deployment internal-api --name=api-svc --port=8080 --target-port=80
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy internal-api
kubectl delete svc api-svc
```

#### Variation 1.1: The Multi-Port Service
**1. CKAD Style Question:**
A Pod named `metrics-app` is running and labeled `app=metrics`. It listens on port `80` for web traffic and port `9090` for metrics. 
Create a ClusterIP Service named `multi-svc` that targets the `app=metrics` label and exposes BOTH port `80` (target `80`) and port `9090` (target `9090`).

**2. Setup Script:**
```bash
kubectl run metrics-app --image=nginx --labels=app=metrics
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get svc multi-svc -o jsonpath='{.spec.ports[0].port}')" == "80" ] && echo "✅ Port 80 exposed" || echo "❌ Port 80 missing"
[ "$(kubectl get svc multi-svc -o jsonpath='{.spec.ports[1].port}')" == "9090" ] && echo "✅ Port 9090 exposed" || echo "❌ Port 9090 missing"
```

<details>

**4. Solution:**
```bash
# You cannot easily do multi-port with 'kubectl expose'. Use 'create svc' and edit the selector.
kubectl create svc clusterip multi-svc --tcp=80:80 --tcp=9090:9090 --dry-run=client -o yaml > multi-svc.yaml

vi multi-svc.yaml
```
*Modify the `selector` to match the Pod:*
```yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-svc
spec:
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  - name: 9090-9090
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: metrics       # CHANGE THIS TO MATCH THE POD
  type: ClusterIP
```
```bash
kubectl apply -f multi-svc.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod metrics-app
kubectl delete svc multi-svc
rm multi-svc.yaml
```

#### Variation 1.2: The Headless Service (Discovery)
**1. CKAD Style Question:**
Stateful applications often need to bypass the Service load balancer and resolve directly to Pod IPs. 
Create a Headless Service named `db-headless` targeting pods with the label `tier=database` on port `3306`. (A Headless service has its ClusterIP explicitly set to `None`).

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(kubectl get svc db-headless -o jsonpath='{.spec.clusterIP}')" == "None" ] && echo "✅ Service is Headless (ClusterIP: None)" || echo "❌ Service is not Headless"
```

<details>

**4. Solution:**
```bash
# The imperative command has a specific flag for headless services!
kubectl create svc clusterip db-headless --tcp=3306:3306 --clusterip="None"

# Edit the generated service to match the requested label
kubectl set selector svc db-headless 'tier=database'
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete svc db-headless
```

---

### Group 2: External Access (NodePorts & ExternalNames)
This group tests your ability to bring traffic into the cluster from the outside world, or map cluster traffic to external legacy systems.

#### Main Task 2: The Specific NodePort Expose
**1. CKAD Style Question:**
Create a Deployment named `web-front` using the `nginx` image.
Expose it externally using a NodePort Service named `web-np`. The Service must listen on port `80`, target port `80`, and strictly use the NodePort `30080`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get svc web-np -o jsonpath='{.spec.type}')" == "NodePort" ] && echo "✅ Service is NodePort" || echo "❌ Service type failed"
[ "$(kubectl get svc web-np -o jsonpath='{.spec.ports[0].nodePort}')" == "30080" ] && echo "✅ Specific NodePort 30080 assigned" || echo "❌ NodePort assignment failed"
```

<details>

**4. Solution:**
```bash
kubectl create deployment web-front --image=nginx

# New versions of kubectl allow you to specify the nodeport imperatively
kubectl create svc nodeport web-np --tcp=80:80 --node-port=30080

# Link the service to the deployment
kubectl set selector svc web-np 'app=web-front'
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy web-front
kubectl delete svc web-np
```

#### Variation 2.1: Live Type Conversion
**1. CKAD Style Question:**
A Service named `backend-svc` is currently running as a `ClusterIP`. The networking team needs it exposed to external traffic immediately. Change the Service type to `NodePort` without deleting and recreating it.

**2. Setup Script:**
```bash
kubectl create deployment backend-app --image=nginx
kubectl expose deployment backend-app --name=backend-svc --port=80
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get svc backend-svc -o jsonpath='{.spec.type}')" == "NodePort" ] && echo "✅ Service converted to NodePort" || echo "❌ Service is still ClusterIP"
```

<details>

**4. Solution:**
```bash
# The fastest way to modify a live object is using 'kubectl patch'
kubectl patch svc backend-svc -p '{"spec": {"type": "NodePort"}}'

# Alternatively, you can use: kubectl edit svc backend-svc
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy backend-app
kubectl delete svc backend-svc
```

#### Variation 2.2: The ExternalName Mapping
**1. CKAD Style Question:**
Your application needs to connect to a legacy database located at `legacy-db.internal.corp.com`. 
Create a Service named `legacy-db` in the `default` namespace that acts as a CNAME record, routing any internal requests for `legacy-db` directly to `legacy-db.internal.corp.com`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.2 ---"
[ "$(kubectl get svc legacy-db -o jsonpath='{.spec.type}')" == "ExternalName" ] && echo "✅ Service is ExternalName" || echo "❌ Service type failed"
[ "$(kubectl get svc legacy-db -o jsonpath='{.spec.externalName}')" == "legacy-db.internal.corp.com" ] && echo "✅ ExternalName mapping is correct" || echo "❌ Mapping failed"
```

<details>

**4. Solution:**
```bash
# Imperative command for ExternalName
kubectl create svc externalname legacy-db --external-name=legacy-db.internal.corp.com
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete svc legacy-db
```

---

### Group 3: Troubleshooting Services (The Disconnects)
If a Service exists but traffic isn't flowing, it is almost always an Endpoints issue. The exam tests your ability to spot and fix these disconnections.

#### Main Task 3: The Mismatched Selector (Zero Endpoints)
**1. CKAD Style Question:**
A Deployment named `data-processor` with the label `app=processor` is running. A Service named `data-svc` exists, but applications cannot reach the Pods. 
Investigate the Service. You will find it has zero Endpoints. Fix the Service so it properly routes to the Pods.

**2. Setup Script:**
```bash
kubectl create deployment data-processor --image=nginx
kubectl label deploy data-processor app=processor --overwrite
kubectl set env deploy/data-processor app=processor # Force pod label update
kubectl create svc clusterip data-svc --tcp=80:80
# Intentionally break the selector
kubectl set selector svc data-svc 'app=wrong-label'
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
ENDPOINTS=$(kubectl get endpoints data-svc -o jsonpath='{.subsets[0].addresses[0].ip}')
[ -n "$ENDPOINTS" ] && echo "✅ Service successfully connected to Pod endpoints!" || echo "❌ Service still has 0 endpoints"
```

<details>

**4. Solution:**
```bash
# 1. Investigate the issue. You will see 'Endpoints: <none>'
kubectl describe svc data-svc

# 2. Check the Pod's labels
kubectl get pods --show-labels

# 3. Fix the Service selector to match the Pod's actual label (app=processor)
kubectl set selector svc data-svc 'app=processor'
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy data-processor
kubectl delete svc data-svc
```

#### Variation 3.1: The TargetPort Trap
**1. CKAD Style Question:**
A Service named `cache-svc` is successfully connected to a Pod named `cache-pod` (the Endpoints exist!). However, connections are refused. 
The Pod's container is actively listening on port `6379`, but the Service is routing traffic to the wrong port. Fix the Service to target port `6379`.

**2. Setup Script:**
```bash
kubectl run cache-pod --image=redis --labels=app=cache
kubectl create svc clusterip cache-svc --tcp=80:80
kubectl set selector svc cache-svc 'app=cache'
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get svc cache-svc -o jsonpath='{.spec.ports[0].targetPort}')" == "6379" ] && echo "✅ targetPort fixed to 6379" || echo "❌ targetPort is still incorrect"
```

<details>

**4. Solution:**
```bash
# You must edit the live YAML to change a targetPort
kubectl edit svc cache-svc
```
*In vim, scroll down to the ports section and modify `targetPort`:*
```yaml
  ports:
  - port: 80
    protocol: TCP
    targetPort: 6379     # Change this from 80 to 6379
```
*(Save and exit)*

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod cache-pod
kubectl delete svc cache-svc
```

#### Variation 3.2: The Ghost Endpoint (Manual Mapping)
**1. CKAD Style Question:**
Sometimes you need a Kubernetes Service to point to an IP address that lives *outside* the cluster (like an external VM at IP `192.168.1.50`), but without using an ExternalName URL.
Create a ClusterIP Service named `vm-bridge` on port `8080`. Since it has no selector, manually create an `Endpoints` object with the exact same name (`vm-bridge`) that maps port `8080` to the IP `192.168.1.50`.

**2. Setup Script:**
*(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.2 ---"
[ "$(kubectl get endpoints vm-bridge -o jsonpath='{.subsets[0].addresses[0].ip}')" == "192.168.1.50" ] && echo "✅ Endpoint manually mapped to external IP" || echo "❌ Endpoint mapping failed"
```

<details>

**4. Solution:**
```bash
# 1. Create the Service WITHOUT a selector (this prevents K8s from auto-managing the endpoint)
kubectl create svc clusterip vm-bridge --tcp=8080:8080

# 2. You must write the YAML for the Endpoints object
vi ep.yaml
```
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: vm-bridge       # MUST exactly match the Service name
subsets:
  - addresses:
      - ip: 192.168.1.50
    ports:
      - port: 8080
```
```bash
kubectl apply -f ep.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete svc vm-bridge
kubectl delete endpoints vm-bridge
rm ep.yaml
```

---

That gives you 100% mechanical coverage for the creation, modification, and troubleshooting of Kubernetes Services. 

