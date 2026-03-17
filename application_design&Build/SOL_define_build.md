
### Scenario 1: The Secure Reverse Proxy
**The Most Efficient Approach:** Alpine Linux doesn't have the standard `useradd` command found in Debian/Ubuntu. The fastest way to create a user in an Alpine base image is `adduser -D <username>`. 

**`Dockerfile`:**
```dockerfile
FROM nginx:1.25-alpine
COPY custom-nginx.conf /etc/nginx/nginx.conf
RUN adduser -D proxyuser
USER proxyuser
EXPOSE 8080
```

**Terminal Commands:**
```bash
docker build -t secure-proxy:v1.0 .
```

### Scenario 2: The Multi-Stage Go API Archive
**The Most Efficient Approach:** When compiling Go binaries, you do not need the Go runtime in the final image. Copying the compiled binary into a virtually empty `alpine` (or even `scratch`) image drops the size from ~800MB to ~10MB.

**`Dockerfile`:**
```dockerfile
FROM golang:1.20 AS builder
WORKDIR /app
COPY main.go .
RUN go build -o api-server main.go

FROM alpine:latest
WORKDIR /root/
COPY --from=builder /app/api-server .
ENV DB_HOST=primary-db.local
CMD ["./api-server"]
```

**Terminal Commands:**
```bash
docker build -t go-api:optimized .
docker save -o /tmp/go-api-backup.tar go-api:optimized
```

### Scenario 3: The Flexible CLI Utility
**The Most Efficient Approach:** By combining `ENTRYPOINT` (the immutable executable) and `CMD` (the default, mutable arguments), you allow the user to easily override the target hostname at runtime without rewriting the entire python execution command. JSON array syntax `["..."]` is mandatory here to prevent unexpected shell wrapping.

**`Dockerfile`:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY network-test.py .
ENTRYPOINT ["python", "/app/network-test.py"]
CMD ["--target=localhost"]
```

**Terminal Commands:**
```bash
docker build -t net-tester:v2 .
docker run -d net-tester:v2
```

### Scenario 4: The Hardened Python Backend
**The Most Efficient Approach:** The industry standard for multi-stage Python builds is to create a virtual environment (`venv`) in the builder stage, install dependencies there, and simply copy the entire isolated `venv` folder to the final stage. Since `python:3.10-slim` is Debian-based, you use `useradd` instead of Alpine's `adduser`.

**`Dockerfile`:**
```dockerfile
FROM python:3.10 AS builder
WORKDIR /app
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN pip install -r requirements.txt

FROM python:3.10-slim
RUN useradd -u 1001 -m appuser
USER 1001
WORKDIR /app
COPY --from=builder /opt/venv /opt/venv
COPY app.py .
ENV PATH="/opt/venv/bin:$PATH"
ENV FLASK_ENV=production
CMD ["python", "app.py"]
```

**Terminal Commands:**
```bash
docker build -t backend-service:prod .
```

### Scenario 5: The Legacy App Migration & Export
**The Most Efficient Approach:** If a script lacks executable permissions on the host, the container will crash with a "Permission denied" error. The `RUN chmod +x` layer guarantees the script executes regardless of the host system's file state.

**`Dockerfile`:**
```dockerfile
FROM ubuntu:22.04
COPY start-batch.sh /usr/local/bin/start-batch.sh
RUN chmod +x /usr/local/bin/start-batch.sh
ENTRYPOINT ["/usr/local/bin/start-batch.sh"]
CMD ["daily-run"]
```

**Terminal Commands:**
```bash
docker build -t batch-processor:legacy .
docker save -o /tmp/batch-processor.tar batch-processor:legacy
```

### Scenario 6: The Air-Gapped Bridge
**The Most Efficient Approach:** Using `docker inspect -f '{{.Config.WorkingDir}}' node:18-alpine` returns a blank output, meaning the default working directory is simply the root path `/`. To imperatively generate the Pod YAML instantly, use `kubectl run` with the `--dry-run=client -o yaml` flags.

**Terminal Commands (Inspect, Build, Tag, Push):**
```bash
docker pull node:18-alpine
docker inspect -f '{{.Config.WorkingDir}}' node:18-alpine
docker build -t local-api:v1 .
docker tag local-api:v1 localhost:5000/local-api:v1
docker push localhost:5000/local-api:v1
```

**`Dockerfile`:**
```dockerfile
FROM node:18-alpine
COPY server.js /
CMD ["node", "server.js"]
```

**Terminal Command (Generate YAML):**
```bash
kubectl run bridge-pod --image=local-api:v1 --dry-run=client -o yaml > bridge-pod.yaml
```

**`bridge-pod.yaml` (After Manual Edit):**
You must open the file and inject `imagePullPolicy: Never` (or `IfNotPresent`) directly under the image name so Kubernetes doesn't try to pull your locally built image from Docker Hub.

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: bridge-pod
  name: bridge-pod
spec:
  containers:
  - image: local-api:v1
    imagePullPolicy: Never 
    name: bridge-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

***

Now that you have the complete patterns and solutions for container images locked in, would you like me to map out the scenarios for the **Application Deployment** domain (covering Deployments, Rolling Updates, and Helm) to continue building your structural framework?
