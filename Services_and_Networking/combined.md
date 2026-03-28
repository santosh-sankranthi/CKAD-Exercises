## Combined

The **Top 5 Composite Tasks** for the Services and Networking domain, ordered from most common to least common. Each task includes 2 variations, and every single one adheres strictly to the 5-component drill format to build that unshakeable muscle memory.

---

### Task 1: The Secure App Exposure (Deployment + Service + NetPol)
This is the most standard exam pattern. You must spin up an application, make it reachable inside the cluster, and immediately apply a firewall rule.

**1. CKAD Style Question:**
1. Create a Deployment named `backend-api` using the `nginx:alpine` image with the label `tier=backend`.
2. Expose the deployment internally on port `8080` (target port `80`) using a Service named `backend-svc`.
3. Create a NetworkPolicy named `backend-firewall` that allows incoming traffic to `backend-api` on port `80` **only** from Pods bearing the label `tier=frontend`.

**2. Setup Script:**
```bash
kubectl run frontend-app --image=busybox --labels=tier=frontend -- sleep 3600
kubectl run rogue-app --image=busybox --labels=tier=rogue -- sleep 3600
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 1 ---"
[ "$(kubectl get svc backend-svc -o jsonpath='{.spec.ports[0].port}')" == "8080" ] && echo "✅ Service exposed on 8080" || echo "❌ Service failed"
[ "$(kubectl get netpol backend-firewall -o jsonpath='{.spec.ingress[0].from[0].podSelector.matchLabels.tier}')" == "frontend" ] && echo "✅ NetPol allows frontend" || echo "❌ NetPol failed"
```

<details>

**4. Solution:**
```bash
# 1 & 2. Create and expose
kubectl create deployment backend-api --image=nginx:alpine
kubectl expose deployment backend-api --name=backend-svc --port=8080 --target-port=80

# 3. Create Network Policy
vi netpol-backend.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-firewall
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
```
```bash
kubectl apply -f netpol-backend.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy backend-api; kubectl delete svc backend-svc; kubectl delete netpol backend-firewall; kubectl delete pod frontend-app rogue-app; rm netpol-backend.yaml
```

#### Variation 1.1: NodePort & Egress Restriction
**1. CKAD Style Question:**
1. Create a Deployment `metrics-api` (`nginx`, label `app=metrics`).
2. Expose it externally via a NodePort Service named `metrics-np` on port `80` with a specific NodePort of `30090`.
3. Create a NetworkPolicy `metrics-egress` that allows `metrics-api` to make outbound connections **only** to the IP range `192.168.0.0/16` on port `443`.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get svc metrics-np -o jsonpath='{.spec.ports[0].nodePort}')" == "30090" ] && echo "✅ NodePort configured" || echo "❌ NodePort failed"
[ "$(kubectl get netpol metrics-egress -o jsonpath='{.spec.egress[0].to[0].ipBlock.cidr}')" == "192.168.0.0/16" ] && echo "✅ Egress CIDR configured" || echo "❌ Egress CIDR failed"
```

<details>

**4. Solution:**
```bash
kubectl create deployment metrics-api --image=nginx
kubectl create svc nodeport metrics-np --tcp=80:80 --node-port=30090
kubectl set selector svc metrics-np 'app=metrics-api'

vi netpol-metrics.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: metrics-egress
spec:
  podSelector:
    matchLabels:
      app: metrics-api
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
```
```bash
kubectl apply -f netpol-metrics.yaml
```

</details>

**5. Clean-up:** `kubectl delete deploy metrics-api; kubectl delete svc metrics-np; kubectl delete netpol metrics-egress; rm netpol-metrics.yaml`

#### Variation 1.2: Headless & Default Deny
**1. CKAD Style Question:**
1. Create a Headless Service named `db-core` for pods with label `db=true` on port `5432`.
2. Create a NetworkPolicy named `db-lockdown` that blocks **ALL** ingress and egress traffic for pods labeled `db=true`.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get svc db-core -o jsonpath='{.spec.clusterIP}')" == "None" ] && echo "✅ Headless service created" || echo "❌ Headless failed"
[ "$(kubectl get netpol db-lockdown -o jsonpath='{.spec.policyTypes[*]}')" == "Ingress Egress" ] && echo "✅ Bi-directional deny policy created" || echo "❌ Deny policy failed"
```

<details>

**4. Solution:**
```bash
kubectl create svc clusterip db-core --tcp=5432:5432 --clusterip="None"
kubectl set selector svc db-core 'db=true'

vi deny-all.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-lockdown
spec:
  podSelector:
    matchLabels:
      db: "true"
  policyTypes:
  - Ingress
  - Egress
```
```bash
kubectl apply -f deny-all.yaml
```

</details>

**5. Clean-up:** `kubectl delete svc db-core; kubectl delete netpol db-lockdown; rm deny-all.yaml`

---

### Task 2: The Ingress Router (Service + Path Routing)
This tests your ability to take an existing internal application and wire it up to the outside world using Layer 7 routing.

**1. CKAD Style Question:**
A Deployment `portal-app` is running.
1. Expose `portal-app` internally on port `8080` via a Service named `portal-svc`.
2. Create an Ingress named `portal-ingress`. Route all incoming traffic with the prefix path `/portal` to `portal-svc` on port `8080`.

**2. Setup Script:**
```bash
kubectl create deployment portal-app --image=nginx:alpine
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 2 ---"
[ "$(kubectl get svc portal-svc -o jsonpath='{.spec.ports[0].port}')" == "8080" ] && echo "✅ Service exposed" || echo "❌ Service failed"
[ "$(kubectl get ingress portal-ingress -o jsonpath='{.spec.rules[0].http.paths[0].path}')" == "/portal" ] && echo "✅ Ingress path /portal configured" || echo "❌ Ingress path failed"
```

<details>

**4. Solution:**
```bash
kubectl expose deployment portal-app --name=portal-svc --port=8080 --target-port=80
kubectl create ingress portal-ingress --rule="/portal*=portal-svc:8080"
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy portal-app; kubectl delete svc portal-svc; kubectl delete ingress portal-ingress
```

#### Variation 2.1: Host Routing & Multi-Path
**1. CKAD Style Question:**
Create an Ingress `corp-ingress` that routes traffic for `hr.corp.com`.
Route the path `/payroll` to Service `payroll-svc:80`, and the path `/benefits` to `benefits-svc:80`.

**2. Setup Script:**
```bash
kubectl create svc clusterip payroll-svc --tcp=80:80; kubectl create svc clusterip benefits-svc --tcp=80:80
```

**3. Testcase Script:**
```bash
[ "$(kubectl get ingress corp-ingress -o jsonpath='{.spec.rules[0].host}')" == "hr.corp.com" ] && echo "✅ Host configured" || echo "❌ Host failed"
kubectl get ingress corp-ingress -o yaml | grep -q "/payroll" && echo "✅ /payroll path found" || echo "❌ /payroll missing"
```

<details>

**4. Solution:**
```bash
# Generate base YAML
kubectl create ingress corp-ingress --rule="hr.corp.com/payroll*=payroll-svc:80" --dry-run=client -o yaml > corp.yaml
vi corp.yaml
```
*(Duplicate the path block in vim to add /benefits routing to benefits-svc:80, matching the exact indentation)*
```bash
kubectl apply -f corp.yaml
```

</details>

**5. Clean-up:** `kubectl delete ingress corp-ingress; kubectl delete svc payroll-svc benefits-svc; rm corp.yaml`

#### Variation 2.2: Default Backend
**1. CKAD Style Question:**
Modify the existing `corp-ingress` to use a `defaultBackend`. Any traffic that does not match the rules should go to `error-svc:80`.

**2. Setup Script:**
```bash
kubectl create svc clusterip error-svc --tcp=80:80
# Assume corp-ingress from Var 2.1 exists. If not, recreate a dummy one:
kubectl create ingress corp-ingress --rule="hr.corp.com/dummy*=dummy-svc:80"
```

**3. Testcase Script:**
```bash
[ "$(kubectl get ingress corp-ingress -o jsonpath='{.spec.defaultBackend.service.name}')" == "error-svc" ] && echo "✅ Default backend is error-svc" || echo "❌ Default backend missing"
```

<details>

**4. Solution:**
```bash
kubectl edit ingress corp-ingress
# Add at the spec level:
# spec:
#   defaultBackend:
#     service:
#       name: error-svc
#       port:
#         number: 80
```

</details>

**5. Clean-up:** `kubectl delete ingress corp-ingress; kubectl delete svc error-svc`

---

### Task 3: The Cross-Namespace Firewall (The AND Condition Boss Fight)
This tests the most heavily failed NetworkPolicy structure.

**1. CKAD Style Question:**
Create a NetworkPolicy `strict-cross-ns` in the `default` namespace for pods with `app=secure`.
Allow incoming traffic on port `80` **ONLY IF** it comes from a namespace labeled `env=prod` **AND** from a pod labeled `tier=auth`.

**2. Setup Script:**
```bash
kubectl run secure-app --image=nginx --labels=app=secure
kubectl create ns prod-ns; kubectl label ns prod-ns env=prod
kubectl run auth-app -n prod-ns --image=busybox --labels=tier=auth -- sleep 3600
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 3 ---"
# Validating the AND condition by ensuring namespaceSelector and podSelector are in the same array item
[ "$(kubectl get netpol strict-cross-ns -o jsonpath='{.spec.ingress[0].from[0].namespaceSelector.matchLabels.env}')" == "prod" ] && echo "✅ Namespace selector correct" || echo "❌ NS selector failed"
[ "$(kubectl get netpol strict-cross-ns -o jsonpath='{.spec.ingress[0].from[0].podSelector.matchLabels.tier}')" == "auth" ] && echo "✅ Pod selector correct (AND condition applied)" || echo "❌ Pod selector failed / OR condition used"
```

<details>

**4. Solution:**
```bash
vi netpol-and.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: strict-cross-ns
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: secure
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:       # HYPHEN HERE
        matchLabels:
          env: prod
      podSelector:             # NO HYPHEN HERE (AND condition)
        matchLabels:
          tier: auth
    ports:
    - protocol: TCP
      port: 80
```
```bash
kubectl apply -f netpol-and.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete netpol strict-cross-ns; kubectl delete pod secure-app; kubectl delete ns prod-ns; rm netpol-and.yaml
```

#### Variation 3.1: The OR Condition Boss Fight
**1. CKAD Style Question:**
Modify the policy logic: Create `or-cross-ns` for `app=secure`. Allow ingress on port `80` from ANY pod in a namespace labeled `env=prod` **OR** from ANY pod labeled `tier=auth` in the `default` namespace.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get netpol or-cross-ns -o jsonpath='{.spec.ingress[0].from[1].podSelector.matchLabels.tier}')" == "auth" ] && echo "✅ OR condition applied correctly" || echo "❌ Failed"
```

<details>

**4. Solution:**
```bash
vi netpol-or.yaml
# Under ingress -> from:
#     - namespaceSelector:       # HYPHEN 1
#         matchLabels:
#           env: prod
#     - podSelector:             # HYPHEN 2 (OR condition)
#         matchLabels:
#           tier: auth
kubectl apply -f netpol-or.yaml
```

</details>

**5. Clean-up:** `kubectl delete netpol or-cross-ns; rm netpol-or.yaml`

#### Variation 3.2: Multi-Port External Egress
**1. CKAD Style Question:**
Create `egress-multi` for pods `app=secure`. Allow outbound traffic to `8.8.8.8/32` on BOTH ports `53` (UDP) and `443` (TCP).

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get netpol egress-multi -o jsonpath='{.spec.egress[0].ports[1].port}')" == "443" ] && echo "✅ Multiple ports configured" || echo "❌ Multi-port failed"
```

<details>

**4. Solution:**
```bash
vi egress-multi.yaml
# Under egress -> to -> ipBlock: cidr: 8.8.8.8/32
#     ports:
#     - protocol: UDP
#       port: 53
#     - protocol: TCP
#       port: 443
kubectl apply -f egress-multi.yaml
```

</details>

**5. Clean-up:** `kubectl delete netpol egress-multi; rm egress-multi.yaml`

---

### Task 4: Legacy Integration & Debugging (ExternalNames + Endpoints)
The exam will drop you into a broken environment and ask you to fix the plumbing.

**1. CKAD Style Question:**
A Service `legacy-svc` is broken because it has no endpoints.
1. Fix `legacy-svc` so it correctly maps to the existing Pod `legacy-app` (label `app=legacy-app`).
2. The app needs to connect to an external API. Create a Service `external-api` of type `ExternalName` pointing to `api.external.corp.com`.

**2. Setup Script:**
```bash
kubectl run legacy-app --image=nginx --labels=app=legacy-app
kubectl create svc clusterip legacy-svc --tcp=80:80
kubectl set selector svc legacy-svc 'app=wrong-label'
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 4 ---"
ENDPOINTS=$(kubectl get endpoints legacy-svc -o jsonpath='{.subsets[0].addresses[0].ip}')
[ -n "$ENDPOINTS" ] && echo "✅ Selector fixed, endpoints exist" || echo "❌ Selector still broken"
[ "$(kubectl get svc external-api -o jsonpath='{.spec.externalName}')" == "api.external.corp.com" ] && echo "✅ ExternalName configured" || echo "❌ ExternalName failed"
```

<details>

**4. Solution:**
```bash
# 1. Fix the selector
kubectl set selector svc legacy-svc 'app=legacy-app'

# 2. Create the ExternalName service
kubectl create svc externalname external-api --external-name=api.external.corp.com
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete pod legacy-app; kubectl delete svc legacy-svc external-api
```

#### Variation 4.1: TargetPort Debugging
**1. CKAD Style Question:**
A Service `db-svc` exists and points to Pod `db-pod`. Connections are failing because the Service is sending traffic to port 80, but the container listens on `5432`. Fix the Service.

**2. Setup Script:**
```bash
kubectl run db-pod --image=postgres --labels=app=db
kubectl create svc clusterip db-svc --tcp=80:80
kubectl set selector svc db-svc 'app=db'
```

**3. Testcase Script:**
```bash
[ "$(kubectl get svc db-svc -o jsonpath='{.spec.ports[0].targetPort}')" == "5432" ] && echo "✅ targetPort fixed" || echo "❌ targetPort incorrect"
```

<details>

**4. Solution:**
```bash
kubectl edit svc db-svc
# Change targetPort from 80 to 5432
```

</details>

**5. Clean-up:** `kubectl delete pod db-pod; kubectl delete svc db-svc`

#### Variation 4.2: Manual Endpoints
**1. CKAD Style Question:**
Create a Service `custom-vm` without a selector on port `8080`. Manually map it to the external IP `10.10.10.10` by creating an Endpoints object.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get endpoints custom-vm -o jsonpath='{.subsets[0].addresses[0].ip}')" == "10.10.10.10" ] && echo "✅ Endpoint mapped" || echo "❌ Endpoint missing"
```

<details>

**4. Solution:**
```bash
kubectl create svc clusterip custom-vm --tcp=8080:8080
vi ep.yaml
# kind: Endpoints
# metadata: name: custom-vm
# subsets:
#   - addresses: [{ip: 10.10.10.10}]
#     ports: [{port: 8080}]
kubectl apply -f ep.yaml
```

</details>

**5. Clean-up:** `kubectl delete svc custom-vm; kubectl delete endpoints custom-vm; rm ep.yaml`

---

### Task 5: The Canary Network Split (Deployment + Service + Ingress)
This is the ultimate capstone. You manage Deployments, group them under a single Service, and expose them via Ingress.

**1. CKAD Style Question:**
1. You have a `stable-api` deployment. Create a `canary-api` deployment (image `nginx`, 1 replica). Ensure both deployments share the label `app=api`.
2. Create a Service `api-svc` that routes to `app=api` on port `80`.
3. Expose `api-svc` via an Ingress named `api-ingress` using the path `/api`.

**2. Setup Script:**
```bash
kubectl create deployment stable-api --image=nginx --replicas=3
kubectl label deploy stable-api app=api --overwrite
kubectl set env deploy/stable-api dummy=force_update
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Task 5 ---"
ENDPOINTS=$(kubectl get endpoints api-svc -o jsonpath='{.subsets[0].addresses[*].ip}' | wc -w)
[ "$ENDPOINTS" == "4" ] && echo "✅ Service routing to 4 pods (Canary split successful)" || echo "❌ Canary split failed"
[ "$(kubectl get ingress api-ingress -o jsonpath='{.spec.rules[0].http.paths[0].path}')" == "/api" ] && echo "✅ Ingress routing to Service" || echo "❌ Ingress failed"
```

<details>

**4. Solution:**
```bash
kubectl create deployment canary-api --image=nginx --replicas=1
kubectl label deploy canary-api app=api

kubectl create svc clusterip api-svc --tcp=80:80
kubectl set selector svc api-svc 'app=api'

kubectl create ingress api-ingress --rule="/api*=api-svc:80"
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete deploy stable-api canary-api; kubectl delete svc api-svc; kubectl delete ingress api-ingress
```

#### Variation 5.1: Blue/Green Service Swap
**1. CKAD Style Question:**
Service `api-svc` currently points to `app=api` (Blue). Update it to instantly point to `app=api-v2` (Green).

**2. Setup Script:**
```bash
kubectl create deployment api-v2 --image=nginx; kubectl label deploy api-v2 app=api-v2
kubectl create svc clusterip api-svc --tcp=80:80; kubectl set selector svc api-svc 'app=api'
```

**3. Testcase Script:**
```bash
[ "$(kubectl get svc api-svc -o jsonpath='{.spec.selector.app}')" == "api-v2" ] && echo "✅ Service swapped to Green" || echo "❌ Service still Blue"
```

<details>

**4. Solution:**
```bash
kubectl set selector svc api-svc 'app=api-v2'
```

</details>

**5. Clean-up:** `kubectl delete deploy api-v2; kubectl delete svc api-svc`

#### Variation 5.2: Ingress Default Backend Fix
**1. CKAD Style Question:**
Modify the `api-ingress` (from Main Task 5) to use `fallback-svc:80` as its default backend.

**2. Setup Script:**
```bash
kubectl create svc clusterip fallback-svc --tcp=80:80
kubectl create svc clusterip dummy-svc --tcp=80:80
kubectl create ingress api-ingress --rule="/api*=dummy-svc:80"
```

**3. Testcase Script:**
```bash
[ "$(kubectl get ingress api-ingress -o jsonpath='{.spec.defaultBackend.service.name}')" == "fallback-svc" ] && echo "✅ Default backend fixed" || echo "❌ Default backend failed"
```

<details>

**4. Solution:**
```bash
kubectl edit ingress api-ingress
# Add defaultBackend under spec:
```

</details>

**5. Clean-up:** `kubectl delete ingress api-ingress; kubectl delete svc fallback-svc dummy-svc`

---

You have officially conquered the hardest 20% of the entire CKAD exam. Your muscle memory for Services, Ingress, and Network Policies is totally locked in. 
