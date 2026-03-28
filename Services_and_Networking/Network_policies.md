## Network Policies

I want to be completely candid with you: **Network Policies are the #1 reason candidates fail the CKAD.** Why? Because unlike Deployments or Pods, **there is zero imperative support for Network Policies.** You cannot run a `kubectl create netpol` command. You *must* know exactly what to search for in the Kubernetes documentation (`kubernetes.io/docs`), copy the template, and aggressively edit the YAML. Furthermore, the indentation in these files dictates whether a rule is an "AND" condition or an "OR" condition—a trap that catches almost everyone.

We are going to lock this down with a massive 9-part matrix. We will cover Ingress (incoming), Egress (outgoing), and the notorious "AND vs. OR" Boss Fights. 

*(Exam Pro-Tip: Before you start this section on the exam, search "Network Policy" in the K8s docs and keep that tab open!)*

---

### Group 1: Ingress Mastery (Incoming Traffic)
By default, all Pods in Kubernetes can talk to all other Pods. A Network Policy acts as a firewall. The moment you apply a policy that targets a Pod, that Pod enters "Default Deny" mode for whatever traffic type you specified, and only explicitly whitelisted traffic is allowed.

#### Main Task 1: Intra-Namespace Ingress (The Bread & Butter)
**1. CKAD Style Question:**
A Pod named `db-server` exists in the `default` namespace with the label `app=db`. 
Create a NetworkPolicy named `db-allow-internal`. It must restrict incoming traffic to `db-server` on TCP port `3306`, allowing it *only* from Pods in the `default` namespace that possess the label `app=api`. 

**2. Setup Script:**
```bash
kubectl run db-server --image=nginx --labels=app=db --expose --port=3306
kubectl run api-server --image=busybox --labels=app=api -- sleep 3600
kubectl run rogue-server --image=busybox --labels=app=rogue -- sleep 3600
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get netpol db-allow-internal -o jsonpath='{.spec.ingress[0].from[0].podSelector.matchLabels.app}')" == "api" ] && echo "✅ Ingress from app=api allowed" || echo "❌ Ingress rule failed"
[ "$(kubectl get netpol db-allow-internal -o jsonpath='{.spec.ingress[0].ports[0].port}')" == "3306" ] && echo "✅ Port 3306 configured" || echo "❌ Port config failed"
```

<details>

**4. Solution:**
```bash
# You must write the YAML. Use the docs template!
vi netpol-internal.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-internal
  namespace: default
spec:
  podSelector:             # 1. TARGET: Which pods does this firewall apply to?
    matchLabels:
      app: db
  policyTypes:
  - Ingress                # 2. TYPE: We are filtering incoming traffic
  ingress:
  - from:                  # 3. RULE: Who is allowed in?
    - podSelector:
        matchLabels:
          app: api
    ports:                 # 4. PORT: On what port?
    - protocol: TCP
      port: 3306
```
```bash
kubectl apply -f netpol-internal.yaml
```

</details>

**5. Clean-up:** `kubectl delete netpol db-allow-internal; kubectl delete pod db-server api-server rogue-server; kubectl delete svc db-server; rm netpol-internal.yaml`

#### Variation 1.1: Inter-Namespace Ingress (The Cross-Boundary Rule)
**1. CKAD Style Question:**
Create a NetworkPolicy named `allow-external-ns` in the `default` namespace targeting Pods with `app=web`. 
Allow incoming traffic on port `80` *only* from Pods located in the `qa-env` namespace. 
*(Massive Exam Cheat Code: In modern Kubernetes, every namespace automatically gets a hidden label called `kubernetes.io/metadata.name: <namespace-name>`. You do not need to label the namespace manually!)*

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get netpol allow-external-ns -o jsonpath='{.spec.ingress[0].from[0].namespaceSelector.matchLabels.kubernetes\.io/metadata\.name}')" == "qa-env" ] && echo "✅ Correct namespace selected" || echo "❌ Namespace selector failed"
```

<details>

**4. Solution:**
```bash
vi netpol-ns.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-ns
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:      # Use namespaceSelector instead of podSelector
        matchLabels:
          kubernetes.io/metadata.name: qa-env
    ports:
    - protocol: TCP
      port: 80
```
```bash
kubectl apply -f netpol-ns.yaml
```

</details>

**5. Clean-up:** `kubectl delete netpol allow-external-ns; rm netpol-ns.yaml`

#### Variation 1.2: The Default Deny (Lockdown)
**1. CKAD Style Question:**
The security team demands that the `secure-zone` namespace be completely locked down. Create a NetworkPolicy named `default-deny-all` in the `secure-zone` namespace that denies **all** incoming traffic to **all** Pods in that namespace.

**2. Setup Script:** `kubectl create ns secure-zone`

**3. Testcase Script:**
```bash
[ "$(kubectl get netpol default-deny-all -n secure-zone -o jsonpath='{.spec.podSelector.matchLabels}')" == "" ] && echo "✅ Empty pod selector (targets ALL pods)" || echo "❌ Pod selector is not empty"
```

<details>

**4. Solution:**
```bash
vi deny-all.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: secure-zone
spec:
  podSelector: {}          # Empty braces mean "Target EVERY pod in this namespace"
  policyTypes:
  - Ingress                # By omitting the 'ingress:' rule block below this, NO traffic is whitelisted.
```
```bash
kubectl apply -f deny-all.yaml
```

</details>

**5. Clean-up:** `kubectl delete ns secure-zone; rm deny-all.yaml`

---

### Group 2: Egress Mastery (Outgoing Traffic)
Egress is trickier because if you block outgoing traffic, you often accidentally block DNS lookups, causing your application to crash because it can't resolve URLs.

#### Main Task 2: Strict CIDR Egress
**1. CKAD Style Question:**
A Pod named `data-miner` exists in the `default` namespace with `app=miner`. 
Create a NetworkPolicy named `miner-egress`. It must restrict outgoing traffic from this Pod so it can *only* reach the external IP block `10.0.0.0/24` on port `443`. Block all other outbound traffic.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get netpol miner-egress -o jsonpath='{.spec.egress[0].to[0].ipBlock.cidr}')" == "10.0.0.0/24" ] && echo "✅ Egress to CIDR allowed" || echo "❌ Egress rule failed"
```

<details>

**4. Solution:**
```bash
vi netpol-egress.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: miner-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: miner
  policyTypes:
  - Egress                 # 1. TYPE: Outgoing traffic
  egress:
  - to:                    # 2. RULE: Where is it allowed to go?
    - ipBlock:             # Use ipBlock for external IP ranges
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 443
```
```bash
kubectl apply -f netpol-egress.yaml
```

</details>

**5. Clean-up:** `kubectl delete netpol miner-egress; rm netpol-egress.yaml`

#### Variation 2.1: The DNS Escape Hatch
**1. CKAD Style Question:**
Update the `miner-egress` policy from the previous task. Add a second egress rule that allows the `data-miner` pod to communicate with the cluster's DNS server. (DNS operates on UDP port 53).

**2. Setup Script:** *(Relies on Main Task 2)*

**3. Testcase Script:**
```bash
[ "$(kubectl get netpol miner-egress -o jsonpath='{.spec.egress[1].ports[0].port}')" == "53" ] && echo "✅ DNS port 53 allowed" || echo "❌ DNS port missing"
```

<details>

**4. Solution:**
```bash
# Edit the existing policy
kubectl edit netpol miner-egress
```
*Add a second item under the `egress` array:*
```yaml
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 443
  - ports:                 # ADD THIS BLOCK. No 'to:' means "allowed to anywhere on this port"
    - protocol: UDP
      port: 53
```

</details>

**5. Clean-up:** `kubectl delete netpol miner-egress`

#### Variation 2.2: Bi-Directional Policy (Ingress + Egress)
**1. CKAD Style Question:**
Create a policy named `api-boundary` for pods with `app=api`. 
1. Allow Ingress on port 80 from pods with `app=frontend`.
2. Allow Egress on port 5432 to pods with `app=database`.

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
```bash
[ "$(kubectl get netpol api-boundary -o jsonpath='{.spec.policyTypes[0]}')" == "Ingress" ] && echo "✅ PolicyType Ingress set" || echo "❌"
[ "$(kubectl get netpol api-boundary -o jsonpath='{.spec.policyTypes[1]}')" == "Egress" ] && echo "✅ PolicyType Egress set" || echo "❌"
```

<details>

**4. Solution:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-boundary
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

</details>

**5. Clean-up:** `kubectl delete netpol api-boundary`

---

### Group 3: The Boss Fight (AND vs. OR Traps)
This is where people fail. A single hyphen (`-`) changes a rule from "Namespace AND Pod" to "Namespace OR Pod". 

#### Main Task 3: The "AND" Condition (Highly Tested)
**1. CKAD Style Question:**
Create a NetworkPolicy named `strict-allow` for pods with `app=secure`. 
Allow incoming traffic **ONLY IF** the traffic comes from the `analytics` namespace **AND** the pod sending the traffic has the label `role=scraper`. 

**2. Setup Script:** *(None required)*

**3. Testcase Script:**
*(Testing YAML arrays natively in bash is tricky, rely on visual inspection of the YAML structure).*

<details>

**4. Solution (Look closely at the hyphens!):**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: strict-allow
spec:
  podSelector:
    matchLabels:
      app: secure
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:                       # Hyphen starts the array item here
        matchLabels:
          kubernetes.io/metadata.name: analytics
      podSelector:                             # NO HYPHEN HERE! This links it to the namespace selector above it (AND condition).
        matchLabels:
          role: scraper
```

</details>

**5. Clean-up:** `kubectl delete netpol strict-allow`

You are absolutely right, and I appreciate you calling me out on that. I cut corners on those last two variations to highlight the YAML syntax, and that completely breaks the drill sheet format we’ve been building. If you are going to practice these in a terminal, you need the full setup and verification scripts, no matter how small the configuration change is. Consistency is exactly how you build muscle memory.

Let's do this right. Here are the fully fleshed-out, 5-component versions of those two critical Boss Fight variations.

---

### Variation 3.1: The "OR" Condition
The hyphen placement completely changes the logic of the firewall. Here is the full drill to practice the "OR" logic.

**1. CKAD Style Question:**
Create a NetworkPolicy named `or-allow` in the `default` namespace targeting Pods with the label `app=secure`. 
Allow incoming traffic on TCP port `8080` if it comes from **ANY** pod in the `analytics` namespace, **OR** if it comes from any pod with `role=scraper` in the `default` namespace.

**2. Setup Script:**
```bash
kubectl create ns analytics
kubectl run secure-pod --image=nginx --labels=app=secure --expose --port=8080
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(kubectl get netpol or-allow -o jsonpath='{.spec.ingress[0].from[0].namespaceSelector.matchLabels.kubernetes\.io/metadata\.name}')" == "analytics" ] && echo "✅ Condition 1 (Analytics NS) configured" || echo "❌ Condition 1 failed"
[ "$(kubectl get netpol or-allow -o jsonpath='{.spec.ingress[0].from[1].podSelector.matchLabels.role}')" == "scraper" ] && echo "✅ Condition 2 (Scraper Pod OR condition) configured" || echo "❌ Condition 2 failed (Check your hyphens!)"
```

<details>

**4. Solution:**
```bash
vi netpol-or.yaml
```
*Notice the two hyphens under `from:`. This creates two separate whitelist rules.*
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: or-allow
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: secure
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:                       # Hyphen 1 (Rule A)
        matchLabels:
          kubernetes.io/metadata.name: analytics
    - podSelector:                             # Hyphen 2 (Rule B). This creates the OR condition.
        matchLabels:
          role: scraper
    ports:
    - protocol: TCP
      port: 8080
```
```bash
kubectl apply -f netpol-or.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete netpol or-allow
kubectl delete svc secure-pod
kubectl delete pod secure-pod
kubectl delete ns analytics
rm netpol-or.yaml
```

---

### Variation 3.2: Multi-Port Allowances
The exam will frequently ask you to open multiple specific ports in a single rule block, which requires another careful array structure.

**1. CKAD Style Question:**
Create a NetworkPolicy named `multi-port-allow` in the `default` namespace targeting Pods with the label `app=backend`. 
Allow incoming traffic from Pods with the label `app=frontend` on both TCP port `80` **AND** TCP port `443`.

**2. Setup Script:**
```bash
kubectl run backend-pod --image=nginx --labels=app=backend
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 3.2 ---"
[ "$(kubectl get netpol multi-port-allow -o jsonpath='{.spec.ingress[0].ports[0].port}')" == "80" ] && echo "✅ Port 80 allowed" || echo "❌ Port 80 missing"
[ "$(kubectl get netpol multi-port-allow -o jsonpath='{.spec.ingress[0].ports[1].port}')" == "443" ] && echo "✅ Port 443 allowed" || echo "❌ Port 443 missing"
```

<details>

**4. Solution:**
```bash
vi netpol-ports.yaml
```
*Notice the array structure under the `ports:` block.*
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: multi-port-allow
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP     # Hyphen 1 (Port A)
      port: 80
    - protocol: TCP     # Hyphen 2 (Port B)
      port: 443
```
```bash
kubectl apply -f netpol-ports.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete netpol multi-port-allow
kubectl delete pod backend-pod
rm netpol-ports.yaml
```

---

If you can confidently navigate the difference between the `AND` and `OR` YAML structures, and you know how to lock down a namespace using the `{}` PodSelector, you will easily survive the most dangerous part of the CKAD.
