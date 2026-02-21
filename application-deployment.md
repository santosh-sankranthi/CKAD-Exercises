# CKAD Practice Questions: Application Deployment

## Overview
This section focuses on the Application Deployment domain of the Certified Kubernetes Application Developer (CKAD) exam. It covers strategies, tools, and techniques for deploying and managing applications in Kubernetes clusters. Understanding deployment strategies, rolling updates, and tools like Helm and Kustomize is crucial for success.

---

## Topics Covered
- Kubernetes deployment strategies (e.g., blue/green, canary)
- Rolling updates and Deployment configurations
- Using Helm for application deployment
- kustomize.

---

## Questions

### 1. Implement a blue/green deployment strategy - imp

**Scenario**:
- Deploy two versions of the same application named `app-blue` (version 1.0) and `app-green` (version 2.0).
- Traffic should initially flow to the `app-blue` deployment.
- Switch traffic to `app-green` by updating the Service selector.

#### Solution:

<details>
<summary>Declarative YAML Configuration</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
      - name: app-container
        image: my-app:1.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
      - name: app-container
        image: my-app:2.0
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
    version: blue
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

#### Steps to Apply:
1. Save the YAML file and apply it:
   ```bash
   kubectl apply -f blue-green-deployment.yaml
   ```
2. Switch traffic to the `app-green` deployment by updating the Service selector:
   ```bash
   kubectl patch service my-app-service -p '{"spec":{"selector":{"app":"my-app","version":"green"}}}'
   ```
</details>

---

### 2. Implement a canary deployment strategy

**Scenario**:
- Deploy a new version of an application named `app-canary`.
- Initially, only 10% of traffic should flow to the new version.
- Gradually increase traffic to 100% if the new version is stable.

#### Solution:

<details>
<summary>Declarative YAML Configuration</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: my-app
      version: stable
  template:
    metadata:
      labels:
        app: my-app
        version: stable
    spec:
      containers:
      - name: app-container
        image: my-app:1.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: canary
  template:
    metadata:
      labels:
        app: my-app
        version: canary
    spec:
      containers:
      - name: app-container
        image: my-app:2.0
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
```

#### Steps to Apply:
1. Deploy the stable and canary versions:
   ```bash
   kubectl apply -f canary-deployment.yaml
   ```
2. Gradually scale up the canary deployment and scale down the stable deployment:
   ```bash
   kubectl scale deployment app-canary --replicas=5
   kubectl scale deployment app-stable --replicas=5
   ```
</details>
---

### 3. Configure a rolling update for an application

**Scenario**:
- Update the image of an existing deployment named `rolling-update-app` from `nginx:1.19` to `nginx:1.21`.
- Ensure there is no downtime during the update.

#### Solution:

<details>
<summary>Declarative YAML Configuration</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rolling-app
  template:
    metadata:
      labels:
        app: rolling-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 1
          maxSurge: 1
```

#### Steps to Apply:
1. Save and apply the YAML file:
   ```bash
   kubectl apply -f rolling-update.yaml
   ```
2. Verify the rollout:
   ```bash
   kubectl rollout status deployment/rolling-update-app
   ```
</details>

---

### 4. Deploy an application using Helm

**Scenario**:
- Use Helm to deploy an application named `helm-app`.
- Configure the Helm chart to use `nginx:latest` and 3 replicas.

#### Solution:

<details>
<summary>Helm Command</summary>

```bash
helm create helm-app
```
Edit the `values.yaml` file:
```yaml
replicaCount: 3
image:
  repository: nginx
  tag: latest
  pullPolicy: IfNotPresent
```
Deploy the Helm chart:
```bash
helm install helm-app ./helm-app
```

#### Steps to Verify:
1. Verify the Helm release:
   ```bash
   helm list
   ```
2. Check the pods:
   ```bash
   kubectl get pods -l app.kubernetes.io/name=helm-app
   ```
</details>

---

### 5. Rollback a failed deployment

**Scenario**:
- Rollback a deployment named `test-deploy` to its previous stable version after a failed update.

#### Solution:

<details>
<summary>Command</summary>

Rollback the deployment:
```bash
kubectl rollout undo deployment/test-deploy
```

#### Steps to Verify:
1. Check the rollout history:
   ```bash
   kubectl rollout history deployment/test-deploy
   ```
2. Ensure the pods are running the previous stable version:
   ```bash
   kubectl get pods
   ```
</details>

### 6. Perform a staged rollout with incremental updates

**Scenario**:
- Deploy an application named `staged-rollout-app` with an initial 25% traffic allocation to the new version.
- Incrementally roll out the new version while ensuring no downtime.

<details>
<summary>Declarative YAML Configuration</summary>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: staged-rollout-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: staged-rollout
  template:
    metadata:
      labels:
        app: staged-rollout
    spec:
      containers:
      - name: app-container
        image: staged-rollout:1.0
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 1
          maxSurge: 1
```

#### Steps to Apply

Save the YAML file and deploy the initial version:

```bash
kubectl apply -f staged-rollout.yaml
```

Incrementally update the deployment to the new version:

```bash
kubectl set image deployment/staged-rollout-app app-container=staged-rollout:1.1
```

Monitor the rollout status:

```bash
kubectl rollout status deployment/staged-rollout-app
```

</details>

---

### 7. Deploy an application using Helm with custom values

**Scenario**:
- Use Helm to deploy an application named `custom-helm-app`.
- Override the default values for replica count (5) and image version (`nginx:1.20`).

<details>
<summary>Command</summary>


#### Create and customize the Helm chart

```bash
helm create custom-helm-app
```

Edit the `values.yaml` file:

```yaml
replicaCount: 5
image:
  repository: nginx
  tag: 1.20
  pullPolicy: IfNotPresent
```

Deploy the Helm chart with the custom values:

```bash
helm install custom-helm-app ./custom-helm-app
```

#### Steps to Verify

1. List Helm releases:
   ```bash
   helm list
   ```
2. Check the pods created by the Helm deployment:
   ```bash
   kubectl get pods -l app.kubernetes.io/name=custom-helm-app
   ```

</details>

---

### 8. Rollback a Helm release to a previous revision

**Scenario**:
- A Helm release named `example-app` was updated to a faulty version.
- Rollback the release to the previous stable revision.

<details>
<summary>Command</summary>

#### Rollback the Helm release

```bash
helm rollback example-app 1
```

#### Steps to Verify

1. Check the Helm release history:
   ```bash
   helm history example-app
   ```
2. Confirm the pods reflect the rollback:
   ```bash
   kubectl get pods -l app.kubernetes.io/name=example-app
   ```

</details>

---

### 9. Perform a rolling update for a StatefulSet

**Scenario**:
- Update the image of an existing StatefulSet named `stateful-app` from `redis:6.2` to `redis:7.0`.
- Ensure each pod is updated sequentially.

<details>
<summary>Declarative YAML Configuration</summary>

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
      - name: redis
        image: redis:7.0
```

#### Steps to Apply

Save the YAML file and update the StatefulSet:

```bash
kubectl apply -f stateful-update.yaml
```

Monitor the pod updates sequentially:

```bash
kubectl rollout status statefulset/stateful-app
```

</details>

---

### 10. Deploy a workload using Helm with external secrets

**Scenario**:
- Deploy an application named `secret-helm-app` using Helm.
- Use external Secrets for database credentials.

<details>
<summary>External Secret YAML Configuration</summary>

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  namespace: default
stringData:
  username: admin
  password: securepassword
```

Deploy the Secret:

```bash
kubectl apply -f external-secret.yaml
```

#### Helm Chart Configuration

Customize `values.yaml` in the Helm chart:

```yaml
environment:
  DB_USER: {{ .Values.secrets.username }}
  DB_PASSWORD: {{ .Values.secrets.password }}
```

Deploy the Helm chart:

```bash
helm install secret-helm-app ./helm-chart
```

#### Steps to Verify

1. Ensure the Secret is mounted correctly:
   ```bash
   kubectl describe pod -l app.kubernetes.io/name=secret-helm-app
   ```
2. Verify the application is running:
   ```bash
   kubectl get pods
   ```

</details>

---

### 11. Kustomize.

**Description**:
- kustomize is a tool inbuilt into kubernetes that helps developers manage k8s config files in a better way and in a less error prone way. It is less complicated than Helm, it uses YAML, and is pretty straight forward to learn and grasp.



**Scenario**:

**TRASFORMERS**
 1. The Naming Transformers (namePrefix & nameSuffix)
    Description: Automatically modifies the metadata.name of all resources listed in the kustomization.yaml.
Exam Pattern to Recognize: "You have a folder of YAML files. Deploy them, but ensure all resource names start with qa- or end with -v2."


<details>
<summary>Declarative YAML Configuration</summary>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml

namePrefix: qa-
nameSuffix: -v2
```

The Effect: A Deployment originally named backend-api becomes qa-backend-api-v2. A Service named db-svc becomes qa-db-svc-v2.

</details>

2. The Metadata Transformers (commonLabels & commonAnnotations)
   Description: Injects key-value pairs into the metadata of all resources. commonLabels is incredibly powerful because it also updates the spec.selector in Services and Deployments to ensure they still route traffic correctly.
Exam Pattern to Recognize: "Ensure all resources in the /opt/k8s/app directory are labeled with env: staging and annotated with release: alpha."

<details>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml

commonLabels:
  env: staging
  tier: backend

commonAnnotations:
  release: alpha
  contact: dev-team
```
The Effect: Every resource gets these labels and annotations. The Service selector will automatically be updated to look for Pods with env: staging and tier: backend.

</details>

3. The Environment Isolation Transformer (namespace)
Description: Overrides or sets the metadata.namespace for all resources, regardless of what is hardcoded in their individual YAML files.
Exam Pattern to Recognize: "Deploy the resources found in ./base into the security-core namespace without editing the base YAML files."

<details>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - rbac.yaml

namespace: security-core
```
The Effect: Even if deployment.yaml explicitly says namespace: default, Kustomize forces it into security-core.

</details>

4. The Image Tag Transformer (images)
Description: Modifies the container image name, tag, or digest defined in a Deployment/Pod without touching the source YAML.
Exam Pattern to Recognize: "Update the image used in the Kustomize deployment from nginx:1.14 to nginx:1.24.0."


<details>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml

images:
  - name: nginx              # The original image name in the deployment.yaml
    newName: nginx           # Optional: Change the image name entirely (e.g., to a private registry)
    newTag: 1.24.0           # The new tag to apply
```
The Effect: The resulting Pod spec will pull nginx:1.24.0 instead of whatever was originally written in the Deployment file.

</details>

5. The Generators (configMapGenerator & secretGenerator)
Description: Creates ConfigMaps or Secrets dynamically and automatically appends a unique hash to their names (e.g., app-config-8f7d6g). Kustomize then intelligently updates any Deployment referencing that ConfigMap/Secret to use the new hashed name, forcing a Pod restart if the config changes.
Exam Pattern to Recognize: "Create a ConfigMap named app-config from the literal value PORT=8080 and ensure it is managed via Kustomize."

<details>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml

configMapGenerator:
  - name: app-config
    literals:
      - PORT=8080
      - MODE=production
  - name: file-config
    files:
      - application.properties  # Pulls from a file in the same directory

secretGenerator:
  - name: db-credentials
    literals:
      - password=supersecret
```
The Effect: Creates the app-config, file-config, and db-credentials resources and seamlessly injects their generated hashed names into the deployment.yaml volume mounts or environment variables.

</details>

---




## Notes and Tips
1. **Deployment Strategies:**
   - Blue/green deployments require two separate environments. Use `Service` objects to switch traffic between versions.
   - Canary deployments can be implemented using `nginx-ingress` or `service mesh` solutions like Istio.

2. **Rolling Updates:**
   - Use `kubectl rollout status` to monitor updates.
   - Always test updates in a non-production environment before applying them.

3. **Helm Tips:**
   - Use `helm lint` to validate your Helm charts before deployment.
   - Always version-control your `values.yaml` file.
---

## Resources
1. [Kubernetes Official Documentation](https://kubernetes.io/docs/)
2. [Helm Official Documentation](https://helm.sh/docs/)
3. [CKAD Exam Curriculum](https://github.com/cncf/curriculum/blob/master/CKAD_Curriculum_v1.31.pdf)
---
