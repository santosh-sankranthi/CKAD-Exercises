## Ingress

Because Ingress requires mapping URLs and Hostnames to Services, the YAML can get deeply nested and confusing. Fortunately, modern versions of `kubectl` have introduced powerful imperative commands for Ingress that can save you massive amounts of time on the exam. 

---

### Group 1: Path-Based Routing (The Most Repeating Scenario)
The most common question asks you to route traffic based on the URL path (e.g., `/app1` goes to Service A). 

#### Main Task 1: The Simple Prefix Route
**1. CKAD Style Question:**
Create an Ingress resource named `web-ingress` in the `default` namespace. 
It must route all traffic coming to the path `/home` (and any sub-paths under it, like `/home/images`) to an existing Service named `home-svc` on port `80`.

**2. Setup Script:**
```bash
kubectl create deployment home-app --image=nginx
kubectl expose deployment home-app --name=home-svc --port=80
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
[ "$(kubectl get ingress web-ingress -o jsonpath='{.spec.rules[0].http.paths[0].path}')" == "/home" ] && echo "✅ Path is /home" || echo "❌ Path incorrect"
[ "$(kubectl get ingress web-ingress -o jsonpath='{.spec.rules[0].http.paths[0].pathType}')" == "Prefix" ] && echo "✅ PathType is Prefix" || echo "❌ PathType incorrect"
[ "$(kubectl get ingress web-ingress -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.name}')" == "home-svc" ] && echo "✅ Backend Service is home-svc" || echo "❌ Backend incorrect"
```

<details>

**4. Solution:**
```bash
# The imperative command for Ingress is a massive time-saver. 
# The asterisk (*) at the end of the path automatically sets the PathType to 'Prefix'.
kubectl create ingress web-ingress --rule="/home*=home-svc:80"
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete ingress web-ingress
kubectl delete svc home-svc
kubectl delete deploy home-app
```

#### Variation 1.1: The "Exact" PathType
**1. CKAD Style Question:**
Create an Ingress resource named `auth-ingress`. 
It must route traffic to the Service `auth-svc` on port `8080`, but **only** if the path is exactly `/login`. (It should *not* route `/login/user`).

**2. Setup Script:**
```bash
kubectl create deployment auth-app --image=nginx
kubectl expose deployment auth-app --name=auth-svc --port=8080 --target-port=80
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
[ "$(kubectl get ingress auth-ingress -o jsonpath='{.spec.rules[0].http.paths[0].pathType}')" == "Exact" ] && echo "✅ PathType is Exact" || echo "❌ PathType is not Exact"
```

<details>

**4. Solution:**
```bash
# Omit the asterisk (*) to set the PathType to 'Exact'
kubectl create ingress auth-ingress --rule="/login=auth-svc:8080"
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete ingress auth-ingress
kubectl delete svc auth-svc
kubectl delete deploy auth-app
```

#### Variation 1.2: Multi-Path Routing (Fan-Out)
**1. CKAD Style Question:**
Create an Ingress named `media-ingress`. 
Configure it to fan out traffic based on the URL path:
* Traffic to `/video` (Prefix) must route to `video-svc` on port `80`.
* Traffic to `/audio` (Prefix) must route to `audio-svc` on port `80`.

**2. Setup Script:**
```bash
kubectl create svc clusterip video-svc --tcp=80:80
kubectl create svc clusterip audio-svc --tcp=80:80
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
PATHS=$(kubectl get ingress media-ingress -o jsonpath='{.spec.rules[0].http.paths[*].path}')
echo "$PATHS" | grep -q "/video" && echo "✅ /video path configured" || echo "❌ /video missing"
echo "$PATHS" | grep -q "/audio" && echo "✅ /audio path configured" || echo "❌ /audio missing"
```

<details>

**4. Solution:**
```bash
# You cannot easily do multiple paths in a single imperative command.
# Create the base YAML with one rule, then edit it.
kubectl create ingress media-ingress --rule="/video*=video-svc:80" --dry-run=client -o yaml > media.yaml
vi media.yaml
```
*Duplicate the path block in the YAML:*
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: media-ingress
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: video-svc
            port:
              number: 80
        path: /video
        pathType: Prefix
      - backend:                 # ADD FROM HERE
          service:
            name: audio-svc
            port:
              number: 80
        path: /audio
        pathType: Prefix         # TO HERE
```
```bash
kubectl apply -f media.yaml
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete ingress media-ingress
kubectl delete svc video-svc audio-svc
rm media.yaml
```

---

### Group 2: Host-Based Routing (Virtual Hosting)
Instead of relying on URL paths (`/app`), the exam will ask you to route traffic based on the domain name the user typed into their browser (`app.company.com`).

#### Main Task 2: The Specific Host Route
**1. CKAD Style Question:**
Create an Ingress named `host-ingress`. 
Route all traffic destined for the hostname `store.example.com` to the Service `store-svc` on port `80`.

**2. Setup Script:**
```bash
kubectl create svc clusterip store-svc --tcp=80:80
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
[ "$(kubectl get ingress host-ingress -o jsonpath='{.spec.rules[0].host}')" == "store.example.com" ] && echo "✅ Host is store.example.com" || echo "❌ Host incorrect"
[ "$(kubectl get ingress host-ingress -o jsonpath='{.spec.rules[0].http.paths[0].backend.service.name}')" == "store-svc" ] && echo "✅ Backend Service is store-svc" || echo "❌ Backend incorrect"
```

<details>

**4. Solution:**
```bash
# The imperative syntax is: --rule="host/path=service:port"
kubectl create ingress host-ingress --rule="store.example.com/=store-svc:80"
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete ingress host-ingress
kubectl delete svc store-svc
```

#### Variation 2.1: Host + Path Combination
**1. CKAD Style Question:**
Create an Ingress named `api-ingress`. 
It must intercept traffic for the hostname `api.corp.com`, but only route it if the path matches `/v1` (Prefix). Route this traffic to `api-v1-svc` on port `8080`.

**2. Setup Script:**
```bash
kubectl create svc clusterip api-v1-svc --tcp=8080:80
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ "$(kubectl get ingress api-ingress -o jsonpath='{.spec.rules[0].host}')" == "api.corp.com" ] && echo "✅ Host is api.corp.com" || echo "❌ Host incorrect"
[ "$(kubectl get ingress api-ingress -o jsonpath='{.spec.rules[0].http.paths[0].path}')" == "/v1" ] && echo "✅ Path is /v1" || echo "❌ Path incorrect"
```

<details>

**4. Solution:**
```bash
# Combine the host, the path, and the asterisk for Prefix
kubectl create ingress api-ingress --rule="api.corp.com/v1*=api-v1-svc:8080"
```

</details>

**5. Clean-up Script:**
```bash
kubectl delete ingress api-ingress
kubectl delete svc api-v1-svc
```

---

### Group 3: The Catch-All (Default Backend)
What happens if a user types a URL that doesn't match any of your rules? You need a default backend (usually a 404 error page).

#### Main Task 3: Configuring the Default Backend
**1. CKAD Style Question:**
An Ingress named `main-ingress` already exists, routing `/app` to `app-svc`.
Modify this existing Ingress to include a `defaultBackend`. Any traffic that does not match the `/app` rule should be routed to a Service named `error-page-svc` on port `80`.

**2. Setup Script:**
```bash
kubectl create svc clusterip app-svc --tcp=80:80
kubectl create svc clusterip error-page-svc --tcp=80:80
kubectl create ingress main-ingress --rule="/app*=app-svc:80"
```

**3. Testcase Script:**
```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ "$(kubectl get ingress main-ingress -o jsonpath='{.spec.defaultBackend.service.name}')" == "error-page-svc" ] && echo "✅ Default backend configured to error-page-svc" || echo "❌ Default backend missing or incorrect"
```

<details>

**4. Solution:**
```bash
# Since the ingress already exists, we must edit it live.
kubectl edit ingress main-ingress
```
*In vim, add the `defaultBackend` block directly under `spec:` (at the same indentation level as `rules:`):*
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  defaultBackend:            # ADD FROM HERE
    service:
      name: error-page-svc
      port:
        number: 80           # TO HERE
  rules:
  - http:
      paths:
      - backend:
# ... (rest of the file remains unchanged)
```
*(Save and exit)*

</details>

**5. Clean-up Script:**
```bash
kubectl delete ingress main-ingress
kubectl delete svc app-svc error-page-svc
```

---

By leveraging the `--rule` flag (`kubectl create ingress <name> --rule="host/path=svc:port"`), you can bypass writing 20 lines of highly nested YAML for almost every Ingress question on the exam. 
