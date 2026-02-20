# CKAD Practice: Services and Networking

## Overview
The Services and Networking topic in the CKAD (Certified Kubernetes Application Developer) exam focuses on understanding and managing connectivity within Kubernetes clusters. This includes creating and managing network policies, exposing applications using services, and configuring ingress rules for external access.

## Topics Covered
- Demonstrate basic understanding of NetworkPolicies
- Provide and troubleshoot access to applications via services
- Use Ingress rules to expose applications

##Topic based briefing.

### 1. Network Policies
<details>
a NetworkPolicy is essentially a firewall for your pods. By default, Kubernetes operates on a "flat network" principle—meaning every pod can talk to every other pod across the entire cluster.

While that’s great for getting things running quickly, it’s a bit of a security nightmare. NetworkPolicies allow you to implement Least Privilege networking.

1. How It Works: The "Default Deny"
Think of a NetworkPolicy as an allow-list. Once a pod is selected by a policy, it will reject any connection that isn't explicitly permitted.

Isolation: Pods are "non-isolated" by default (all traffic allowed).

Selection: Once you apply a policy to a pod, it becomes "isolated," and only the rules you write can let traffic in or out.

2. Key Components of a Policy

When you write a NetworkPolicy (in YAML), you define four main things:

podSelector: Which pods does this rule apply to? (e.g., app: database)

policyTypes: Does this apply to Ingress (incoming), Egress (outgoing), or both?

from/to: Which sources/destinations are allowed?

podSelector: Specific pods in the same namespace.

namespaceSelector: Any pod within a specific namespace.

ipBlock: Specific IP ranges (CIDR).

ports: Which TCP/UDP ports are open? (e.g., 80, 443, 5432).

3. Services:
   
Load balancers:

these are not k8s native 


  
</details>

## Practice Questions

### 1. Create a Default Deny-All NetworkPolicy

**Scenario**:
- Create a NetworkPolicy named `deny-all` that denies all ingress and egress traffic for Pods in the `default` namespace.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f deny-all.yaml
   ```
2. Verify the policy:
   ```bash
   kubectl describe networkpolicy deny-all
   ```

</details>

---

### 2. Allow Ingress Traffic from Specific Pods

**Scenario**:
- Create a NetworkPolicy named `allow-frontend` to allow traffic to Pods labeled `app=backend` only from Pods labeled `role=frontend`.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f allow-frontend.yaml
   ```
2. Test connectivity from a frontend Pod to a backend Pod.

</details>

---

### 3. Restrict Egress Traffic to Specific IPs

**Scenario**:
- Create a NetworkPolicy named `restrict-egress` to restrict Pods labeled `app=web` to only communicate with the IP range `192.168.1.0/24`.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.1.0/24
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f restrict-egress.yaml
   ```
2. Verify egress rules:
   ```bash
   kubectl describe networkpolicy restrict-egress
   ```

</details>

---

### 4. Allow Ingress Traffic on Specific Ports

**Scenario**:
- Create a NetworkPolicy named `allow-port` to allow traffic to Pods labeled `app=database` only on port 3306 (MySQL).

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector: {}
      ports:
      - protocol: TCP
        port: 3306
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f allow-port.yaml
   ```
2. Test connectivity to the database Pod on port 3306.

</details>

---

### 5. Combine Ingress and Egress Rules

**Scenario**:
- Create a NetworkPolicy named `combined-policy` to allow ingress traffic from `role=frontend` Pods and egress traffic to `192.168.2.0/24`.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: combined-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.2.0/24
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f combined-policy.yaml
   ```
2. Test ingress and egress connectivity for the backend Pods.

</details>

---

### 6. Test Default Allow Behavior

**Scenario**:
- Deploy Pods in a namespace without any NetworkPolicy.
- Verify that all traffic is allowed by default.

<details>
<summary>Details</summary>

#### Steps to Test

1. Deploy two Pods:
   ```bash
   kubectl run pod1 --image=busybox --command -- sleep 3600
   kubectl run pod2 --image=busybox --command -- sleep 3600
   ```
2. Test connectivity:
   ```bash
   kubectl exec pod1 -- ping pod2
   ```

</details>

---

### 7. Deny All Egress Traffic

**Scenario**:
- Create a NetworkPolicy named `deny-egress` that blocks all egress traffic for Pods in the `prod` namespace.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress: []
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f deny-egress.yaml
   ```
2. Test egress connectivity from any Pod in the `prod` namespace.

</details>

---

### 8. Allow Egress to a Specific Namespace

**Scenario**:
- Create a NetworkPolicy named `namespace-egress` that allows Pods in `dev` to communicate with Pods in `prod`.

<details>
<summary>Details</summary>

#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-egress
  namespace: dev
spec:
  podSelector:
    matchLabels:
      app: frontend
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          environment: prod
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f namespace-egress.yaml
   ```
2. Test connectivity between `dev` and `prod` namespaces.

</details>

---

### 9. Isolate a Namespace with a Default Deny Policy

**Scenario**:
- Apply a default deny-all NetworkPolicy to isolate all Pods in the `staging` namespace.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-namespace
  namespace: staging
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f isolate-namespace.yaml
   ```
2. Verify that traffic is denied.

</details>

---

### 10. Allow DNS Traffic for Specific Pods

**Scenario**:
- Allow Pods labeled `app=web` to access DNS servers on port 53.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: web
  egress:
  - to:
    - ipBlock:
        cidr: 8.8.8.8/32
      ports:
      - protocol: UDP
        port: 53
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f allow-dns.yaml
   ```
2. Verify DNS access for the web Pods.

</details>

---

### 11. Expose a Deployment via a ClusterIP Service

**Scenario**:
- Create a Deployment named `my-app` with 3 replicas.
- Expose it internally using a ClusterIP Service named `my-app-service`.

<details>
<summary>Details</summary>

#### Declarative YAML Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo hello; sleep 5; done"]
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f my-app-service.yaml
   ```
2. Test the Service:
   ```bash
   kubectl exec -it <pod-name> -- curl my-app-service
   ```

</details>

---

### 12. Expose a Deployment Externally Using a NodePort Service

**Scenario**:
- Create a Deployment named `external-app`.
- Expose it externally using a NodePort Service on port 30007.

<details>
<summary>Details</summary>

#### Declarative YAML Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-app
  labels:
    app: external-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: external-app
  template:
    metadata:
      labels:
        app: external-app
    spec:
      containers:
      - name: app
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo hello; sleep 5; done"]
---
apiVersion: v1
kind: Service
metadata:
  name: external-app-service
spec:
  selector:
    app: external-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    nodePort: 30007
  type: NodePort
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f external-app-service.yaml
   ```
2. Test the Service externally:
   ```bash
   curl <node-ip>:30007
   ```

</details>

---

### 13. Configure a LoadBalancer Service for a Deployment

**Scenario**:
- Deploy an application named `loadbalanced-app`.
- Expose it using a LoadBalancer Service.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: loadbalanced-app
  labels:
    app: loadbalanced-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: loadbalanced-app
  template:
    metadata:
      labels:
        app: loadbalanced-app
    spec:
      containers:
      - name: app
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: loadbalanced-service
spec:
  selector:
    app: loadbalanced-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f loadbalanced-service.yaml
   ```
2. Verify the external IP of the LoadBalancer:
   ```bash
   kubectl get svc loadbalanced-service
   ```
3. Test access to the application using the external IP.

</details>

---

### 14. Troubleshoot a Service Not Forwarding Traffic

**Scenario**:
- A Service named `troubleshoot-service` is not forwarding traffic to the backend Pods.
- Investigate and resolve the issue.

<details>
<summary>Details</summary>

#### Steps to Troubleshoot

1. Verify Service configuration:
   ```bash
   kubectl describe service troubleshoot-service
   ```
2. Check the endpoint mappings:
   ```bash
   kubectl get endpoints troubleshoot-service
   ```
3. Ensure the backend Pods are running and labeled correctly:
   ```bash
   kubectl get pods -l app=<label>
   ```
4. Correct any misconfigurations and test again.

</details>

---

### 15. Use ExternalName Service to Alias External Resources

**Scenario**:
- Create an ExternalName Service named `external-service` to alias `example.com`.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: example.com
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f external-service.yaml
   ```
2. Test the alias:
   ```bash
   kubectl exec -it <pod-name> -- curl external-service
   ```

</details>

---

### 16. Configure a Headless Service for StatefulSets

**Scenario**:
- Create a StatefulSet named `stateful-app`.
- Use a headless Service to provide direct access to each Pod.

<details>
<summary>Details</summary>

#### Declarative YAML Configuration

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-app
spec:
  serviceName: "stateful-service"
  replicas: 3
  selector:
    matchLabels:
      app: stateful
  template:
    metadata:
      labels:
        app: stateful
    spec:
      containers:
      - name: app
        image: busybox
        command: ["/bin/sh", "-c", "while true; do echo hello; sleep 5; done"]
---
apiVersion: v1
kind: Service
metadata:
  name: stateful-service
spec:
  clusterIP: None
  selector:
    app: stateful
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f stateful-service.yaml
   ```
2. Verify the DNS entries for the headless Service:
   ```bash
   kubectl exec -it <pod-name> -- nslookup stateful-service
   ```

</details>

---

### 17. Configure Session Affinity in a Service

**Scenario**:
- Create a Service named `session-service` with session affinity enabled.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: session-service
spec:
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  sessionAffinity: ClientIP
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f session-service.yaml
   ```
2. Test session affinity:
   ```bash
   kubectl exec -it <pod-name> -- curl session-service
   ```

</details>

---

### 18. Implement a Service with Health Checks

**Scenario**:
- Create a Deployment named `health-app`.
- Expose it using a Service with a health check configured.

<details>
<summary>Details</summary>

#### Declarative YAML Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-app
  labels:
    app: health-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: health-app
  template:
    metadata:
      labels:
        app: health-app
    spec:
      containers:
      - name: app
        image: nginx
        readinessProbe:
          httpGet:
            path: /
            port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: health-service
spec:
  selector:
    app: health-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

#### Steps to Apply

1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f health-service.yaml
   ```
2. Verify Pod readiness:
   ```bash
   kubectl get pods -l app=health-app
   ```

</details>

---

### 19. Expose a Single Application Using Ingress

**Scenario**:
- Deploy an application named `simple-app`.
- Expose it using an Ingress resource with the hostname `simple-app.local`.

<details>
<summary>Details</summary>

#### Declarative YAML Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: simple-app
  labels:
    app: simple-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: simple-app
  template:
    metadata:
      labels:
        app: simple-app
    spec:
      containers:
      - name: app
        image: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: simple-app-service
spec:
  selector:
    app: simple-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-app-ingress
spec:
  rules:
  - host: simple-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: simple-app-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f simple-app-ingress.yaml
   ```
2. Test the Ingress:
   ```bash
   curl -H "Host: simple-app.local" <ingress-controller-ip>
   ```

</details>

---

### 20. Configure TLS for an Ingress Resource

**Scenario**:
- Secure an application exposed through Ingress with TLS.
- Use a Secret named `tls-secret` for the certificate and key.

<details>
<summary>Details</summary>

#### Declarative YAML Configuration

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: default
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-app-ingress
spec:
  tls:
  - hosts:
    - tls-app.local
    secretName: tls-secret
  rules:
  - host: tls-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tls-app-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f tls-app-ingress.yaml
   ```
2. Test HTTPS access:
   ```bash
   curl -k https://tls-app.local --resolve tls-app.local:<ingress-controller-ip>
   ```

</details>

---

### 21. Expose Multiple Applications Using Host-Based Rules

**Scenario**:
- Expose two applications (`app1` and `app2`) through a single Ingress.
- Use host-based rules to route traffic.

<details>
<summary>Details</summary>

#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-host-ingress
spec:
  rules:
  - host: app1.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f multi-host-ingress.yaml
   ```
2. Test each application:
   ```bash
   curl -H "Host: app1.local" <ingress-controller-ip>
   curl -H "Host: app2.local" <ingress-controller-ip>
   ```

</details>

---

### 22. Implement Path-Based Routing in Ingress

**Scenario**:
- Route `/app1` to `app1-service` and `/app2` to `app2-service` using an Ingress resource.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-ingress
spec:
  rules:
  - host: path-app.local
    http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f path-routing-ingress.yaml
   ```
2. Test each path:
   ```bash
   curl -H "Host: path-app.local" <ingress-controller-ip>/app1
   curl -H "Host: path-app.local" <ingress-controller-ip>/app2
   ```

</details>

---

### 23. Debug an Ingress Resource Not Routing Traffic

**Scenario**:
- An Ingress resource named `debug-ingress` is not routing traffic to the backend.
- Debug and resolve the issue.

<details>
<summary>Details</summary>


#### Steps to Debug

1. Verify Ingress configuration:
   ```bash
   kubectl describe ingress debug-ingress
   ```
2. Check Service endpoints:
   ```bash
   kubectl get endpoints <service-name>
   ```
3. Ensure backend Pods are running and labeled correctly:
   ```bash
   kubectl get pods -l <label-selector>
   ```
4. Correct any misconfigurations and test again.

</details>

---

### 24. Use Default Backend for Unmatched Routes

**Scenario**:
- Configure an Ingress resource with a default backend to handle unmatched requests.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: default-backend-ingress
spec:
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - host: example.local
    http:
      paths:
      - path: /app
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f default-backend-ingress.yaml
   ```
2. Test default backend routing:
   ```bash
   curl -H "Host: example.local" <ingress-controller-ip>/unmatched-path
   ```

</details>

---

### 25. Configure a Rewrite Rule in an Ingress

**Scenario**:
- Use annotations to rewrite `/old-path` to `/new-path` for an application.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /new-path
spec:
  rules:
  - host: rewrite-app.local
    http:
      paths:
      - path: /old-path
        pathType: Prefix
        backend:
          service:
            name: rewrite-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f rewrite-ingress.yaml
   ```
2. Test the rewrite rule:
   ```bash
   curl -H "Host: rewrite-app.local" <ingress-controller-ip>/old-path
   ```

</details>

---

### 26. Configure CORS Using Ingress Annotations

**Scenario**:
- Add CORS headers to an Ingress resource to allow cross-origin requests.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cors-ingress
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "PUT, GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization"
spec:
  rules:
  - host: cors-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: cors-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f cors-ingress.yaml
   ```
2. Test CORS configuration:
   ```bash
   curl -H "Origin: http://example.com" -H "Access-Control-Request-Method: GET" -i http://cors-app.local
   ```

</details>

---

### 27. Implement Multi-Tenant Hosting with Ingress

**Scenario**:
- Host two applications (`tenant1-app` and `tenant2-app`) on the same Ingress with different subdomains: `tenant1.example.com` and `tenant2.example.com`.

<details>
<summary>Details</summary>

#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-tenant-ingress
spec:
  rules:
  - host: tenant1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tenant1-service
            port:
              number: 80
  - host: tenant2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tenant2-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f multi-tenant-ingress.yaml
   ```
2. Test each subdomain:
   ```bash
   curl -H "Host: tenant1.example.com" <ingress-controller-ip>
   curl -H "Host: tenant2.example.com" <ingress-controller-ip>
   ```

</details>

---

### 28. Add Rate Limiting to an Ingress

**Scenario**:
- Configure rate limiting for an application using Ingress annotations.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "5"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "2"
spec:
  rules:
  - host: rate-limited-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rate-limited-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f rate-limited-ingress.yaml
   ```
2. Test rate limiting:
   ```bash
   ab -n 100 -c 10 -H "Host: rate-limited-app.local" http://<ingress-controller-ip>/
   ```

</details>

---

### 29. Use Ingress Annotations for Canary Deployments

**Scenario**:
- Split traffic between two versions (`v1` and `v2`) of an application using Ingress annotations.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  rules:
  - host: canary-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: stable-service
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress-v2
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
spec:
  rules:
  - host: canary-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: canary-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f canary-ingress.yaml
   ```
2. Test traffic distribution:
   ```bash
   curl -H "Host: canary-app.local" <ingress-controller-ip>
   ```

</details>

---
### 29.1 rewrite annotaion: 
Your Ingress is receiving traffic on a specific path (like /billing), but your backend application is hardcoded to serve traffic from its own root directory (/). If you don't rewrite the path, the Ingress forwards the full /billing path to the app, the app says "I don't have a /billing page," and throws a 404 error.

**Scenario**:
- The Static Drop (Simple Rewrite).

This is the most straightforward variation. The prompt will ask you to strip away a single, static path completely.

How it’s asked: "Create an Ingress that routes traffic from /auth to the auth-service on port 80. The auth-service expects all traffic to arrive at its root path (/)."

The Strategy: You capture the incoming path and tell the Ingress to replace the entire thing with a single forward slash.
<details>
   
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: static-rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 80
```
</details>

- The Regex Capture (Dynamic Rewrite)


This is the modern, more complex variation, and it is highly likely to appear. It happens when you need to strip the base path but keep everything that comes after it.

How it’s asked: "Configure the Ingress to route /api/v1/... to the backend-api service. Ensure that any sub-paths are preserved. For example, a request to /api/v1/users/123 must be forwarded to the service as /users/123."

The Strategy: You need to use Regular Expressions (Regex) to "capture" the second half of the URL and pass it to the rewrite target.

The YAML Implementation:
To do this successfully, you must add a second annotation to enable regex, and use capture groups ($1, $2, etc.) in your rewrite target.

<details>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dynamic-rewrite-ingress
  annotations:
    # 1. Enable Regex matching
    nginx.ingress.kubernetes.io/use-regex: "true"
    # 2. Rewrite using the second capture group ($2)
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      # The Regex path: Match /api/v1, then capture the trailing slash ($1), then capture the rest ($2)
      - path: /api/v1(/|$)(.*)
        pathType: ImplementationSpecific # Note: Prefix often works, but ImplementationSpecific is safer for regex
        backend:
          service:
            name: backend-api
            port:
              number: 80
```
   
Result: A user visits yourdomain.com/api/v1/users/123 -> The app receives a request for /users/123.

</details>

The Exam "Gotcha" Checklist
When writing these in a high-pressure environment, double-check these three things:

Did I spell the annotation right? (nginx.ingress.kubernetes.io/rewrite-target).

Does my path match my regex? If you use $2 in the target, you must have two sets of parentheses () in your path.

Did I include use-regex: "true"? If you are doing Pattern 2, it will silently fail without this.

---

### 29.2. SSL Redirect (ssl-redirect)

description: By default, if you configure a TLS (Transport Layer Security) section in your Ingress manifest, the NGINX controller automatically assumes you want everything secure. If a user tries to access your site via plain HTTP, NGINX will automatically slap them with a 308 Permanent Redirect and send them to the HTTPS version.

In the real world, this is exactly what you want. But in an exam environment, it can cause headaches.



**Scenario**:
- diable HTTPS redirect

<details>


How it's asked in the exam
Usually, exam environments don't have valid public SSL certificates set up. The grading script or your own curl tests might fail if the Ingress forces a redirect to an HTTPS endpoint that doesn't have a trusted certificate.

The Scenario: You will be asked to expose a secure application but explicitly told to "ensure the application is also accessible via HTTP" or "disable HTTPS redirection."

The Solution: You need to explicitly turn off that default behavior.


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: no-redirect-ingress
  annotations:
    # This disables the automatic HTTP -> HTTPS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - secure-app.com
    secretName: my-tls-secret
  rules:
  - host: secure-app.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: secure-service
            port:
              number: 80
```


```yaml

```


</details>

- scenario: force secure / redirect to HTTPS
<details>

This is the second variation, and it trips people up because it involves external infrastructure (like AWS or GCP load balancers).

Sometimes, the SSL certificate isn't handled by the Kubernetes Ingress itself; it's handled by a cloud load balancer sitting in front of the cluster. Because the Ingress YAML doesn't have a tls: section, NGINX won't automatically redirect HTTP traffic to HTTPS.

How it's asked: "The cluster is behind an external load balancer that terminates SSL. Expose the payment-service via Ingress. Ensure that any direct HTTP requests hitting the Ingress are forcibly redirected to HTTPS."

The Strategy: Since NGINX doesn't see a tls: block, it assumes HTTP is fine. You have to explicitly force it to redirect.

```yaml
nginx.ingress.kubernetes.io/ssl-redirect: "true"
```
</details>

Crucial Reminder: Whether it is "true" or "false", it must be in quotes in your YAML, or it will throw a syntax error!

---

### 29.3. Sticky sessions in ingress.

Normally, a Kubernetes Service load-balances requests round-robin across all available Pods. But if you have a stateful application—like a user interacting with a specific ML model loaded into memory on a KubeRay worker node—you need that user's subsequent requests to hit the exact same Pod every time.

To solve this, NGINX Ingress uses cookies to "stick" a client to a specific backend Pod.

Variation 1: The Basic "Stick" (Default Settings)

<details>

This is the simplest form. The exam simply asks you to enable session affinity without giving you any specific constraints about the cookie itself.

How it’s asked: "Configure the Ingress stateful-ingress so that a user's session is consistently routed to the same backend pod."

The Strategy: You just need to turn on cookie-based affinity. NGINX will automatically create a cookie (named route by default) and handle the hashing.


```yaml
nginx.ingress.kubernetes.io/affinity: "cookie"
```
</details>

Variation 2: The Custom Cookie (Named Session)

<details>

This is highly likely to appear. The grading script will specifically run a curl command to check if a cookie with a specific name is being returned in the HTTP headers.

How it’s asked: "Update the Ingress to use session affinity. Ensure the Ingress uses a cookie named CKAD_SESSION to track the routing."

The Strategy: You must enable affinity AND pass a second annotation to override the default cookie name.

```yaml
nginx.ingress.kubernetes.io/affinity: "cookie"
nginx.ingress.kubernetes.io/session-cookie-name: "CKAD_SESSION"
```
</details>

Variation 3: The Timed Session (Expiration/Max Age)


<details>
This is the "hard mode" variation. They will ask you to ensure the sticky session expires after a certain amount of time.

How it’s asked: "Enable session affinity for the webapp-ingress. The session cookie must be named APP_STICKY and must expire after 2 hours (7200 seconds)."

The Strategy: You need the affinity flag, the custom name flag, and a third flag to define the max age in seconds.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: stateful-ingress
  annotations:
    # 1. Turn it on
    nginx.ingress.kubernetes.io/affinity: "cookie"
    # 2. Name the cookie
    nginx.ingress.kubernetes.io/session-cookie-name: "APP_STICKY"
    # 3. Set the lifespan (in seconds)
    nginx.ingress.kubernetes.io/session-cookie-max-age: "7200"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: stateful-service
            port:
              number: 80
```

</details>

The Exam "Gotcha" Checklist
Always stringify values: Notice how "cookie", "APP_STICKY", and "7200" are all in quotation marks. If you write max-age: 7200 as an integer, the YAML will fail to apply.

Don't memorize them, know how to search: If you forget session-cookie-max-age during the test, go to the NGINX Ingress Annotations page we talked about earlier and Ctrl+F for "cookie". All the variations are grouped together on that page.

---


### 30. Redirect HTTP to HTTPS in Ingress

**Scenario**:
- Use Ingress annotations to enforce HTTPS by redirecting HTTP traffic.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: https-redirect-ingress
  annotations:
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - https-app.local
    secretName: tls-secret
  rules:
  - host: https-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: https-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f https-redirect-ingress.yaml
   ```
2. Test HTTPS redirection:
   ```bash
   curl http://https-app.local --resolve https-app.local:<ingress-controller-ip>
   ```

</details>

---

### 31. Use Wildcard Hostnames in Ingress Rules

**Scenario**:
- Configure a single Ingress to handle requests for multiple subdomains using a wildcard hostname.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-ingress
spec:
  rules:
  - host: "*.example.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: wildcard-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f wildcard-ingress.yaml
   ```
2. Test with subdomains:
   ```bash
   curl -H "Host: app1.example.com" <ingress-controller-ip>
   curl -H "Host: app2.example.com" <ingress-controller-ip>
   ```

</details>

---

### 32. Configure Backend Timeouts in Ingress

**Scenario**:
- Set timeout for backend responses in an Ingress resource.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: timeout-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
spec:
  rules:
  - host: timeout-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: timeout-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f timeout-ingress.yaml
   ```
2. Test timeout behavior:
   ```bash
   curl -H "Host: timeout-app.local" <ingress-controller-ip>
   ```

</details>

---

### 33. Configure Custom Error Pages for Ingress

**Scenario**:
- Use Ingress annotations to display custom error pages for HTTP 404 errors.

<details>
<summary>Details</summary>


#### Declarative YAML Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: error-page-ingress
  annotations:
    nginx.ingress.kubernetes.io/custom-http-errors: "404"
    nginx.ingress.kubernetes.io/server-snippet: |
      error_page 404 /custom_404.html;
      location = /custom_404.html {
        internal;
        root /usr/share/nginx/html;
      }
spec:
  rules:
  - host: error-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: error-service
            port:
              number: 80
```

#### Steps to Apply and Verify

1. Save and apply the YAML configuration:
   ```bash
   kubectl apply -f error-page-ingress.yaml
   ```
2. Test the custom error page:
   ```bash
   curl -H "Host: error-app.local" <ingress-controller-ip>/nonexistent-path
   ```

</details>

---

## Notes and Tips
- **NetworkPolicies**:
  - NetworkPolicies are designed to control pod-level network communication. They are namespace-scoped resources.
  - Understand the default behavior (allow all traffic if no policy exists) and how to create rules to allow/deny ingress and egress traffic.
  
- **Services**:
  - Understand the types of Kubernetes Services (ClusterIP, NodePort, LoadBalancer, and ExternalName) and their use cases.
  - Be familiar with `kubectl` commands for troubleshooting services, such as checking endpoints with `kubectl describe`.

- **Ingress**:
  - Ingress rules provide HTTP and HTTPS routes to services.
  - Know how to configure `Ingress` objects with different paths, hostnames, and TLS configurations.

## Resources
- [Kubernetes Official Documentation on NetworkPolicies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes Official Documentation on Services](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes Official Documentation on Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes Networking Tutorials](https://kubernetes.io/docs/tutorials/services/)
