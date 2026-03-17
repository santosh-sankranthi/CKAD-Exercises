# 🚀 Vim Define Build and modify Container images.

## Scenario 1: The Secure Reverse Proxy

Combined Patterns: File Injection (COPY) + Non-Root Security + Port Exposure + Basic Build/Tag
The Setup: You have a local file named custom-nginx.conf in your current directory.
Your Tasks:

1. Create a Dockerfile using nginx:1.25-alpine as the base image.

2. Copy custom-nginx.conf to /etc/nginx/nginx.conf inside the image.

3. Ensure the container runs as a non-root user (create a user named proxyuser and set the USER instruction).

4. Document that the container listens on port 8080.

5. Build the image and tag it exactly as secure-proxy:v1.0.

<details>
  <summary><b>config and testcase checker</b></summary>
test:
  
```yaml
events {}
http {
    server {
        listen 8080;
        location / {
            return 200 'Secure Proxy Running\n';
        }
    }
}
```
config:

```yaml
#!/bin/bash
CLI="docker"
IMAGE="secure-proxy:v1.0"

echo "Checking $IMAGE..."
if ! $CLI image inspect $IMAGE >/dev/null 2>&1; then echo "❌ Image not found"; exit 1; fi

USER=$($CLI inspect -f '{{.Config.User}}' $IMAGE)
if [[ "$USER" == "" || "$USER" == "root" || "$USER" == "0" ]]; then echo "❌ User is still root"; else echo "✅ Non-root user set"; fi

PORTS=$($CLI inspect -f '{{json .Config.ExposedPorts}}' $IMAGE)
if [[ "$PORTS" == *"8080"* ]]; then echo "✅ Port 8080 exposed"; else echo "❌ Port 8080 not exposed"; fi
```
Cleanup: docker rmi secure-proxy:v1.0

</details>

---

## Scenario 2: The Flexible CLI Utility
Combined Patterns: Base Image Update + File Injection (COPY) + Entrypoint vs. CMD + Local Verification
The Setup: You are given an outdated Dockerfile using python:3.7. You have a script named network-test.py locally.
Your Tasks:

1. Update the base image to python:3.11-slim.

2. Copy network-test.py into the /app directory.

3. Configure the image so that the ENTRYPOINT always executes python /app/network-test.py.

4. Set the default CMD to pass the argument --target=localhost.

5. Build the image as net-tester:v2, then run it locally in detached mode to verify it executes without crashing.

6. Build the image as batch-processor:legacy, then save it to a tarball named /opt/archives/batch-processor.tar.

<details>
  <summary><b>config and testcase checker</b></summary>

Setup File (network-test.py):

```yaml
import sys, argparse
parser = argparse.ArgumentParser()
parser.add_argument('--target', default='unknown')
args = parser.parse_args()
print(f"Pinging {args.target}...")
```
Test Script (verify.sh):

```yaml
#!/bin/bash
CLI="docker"
IMAGE="net-tester:v2"

if ! $CLI image inspect $IMAGE >/dev/null 2>&1; then echo "❌ Image not found"; exit 1; fi

ENTRYPOINT=$($CLI inspect -f '{{json .Config.Entrypoint}}' $IMAGE)
if [[ "$ENTRYPOINT" == *"python"* && "$ENTRYPOINT" == *"network-test.py"* ]]; then echo "✅ Entrypoint configured correctly"; else echo "❌ Entrypoint incorrect: $ENTRYPOINT"; fi

CMD=$($CLI inspect -f '{{json .Config.Cmd}}' $IMAGE)
if [[ "$CMD" == *"--target=localhost"* ]]; then echo "✅ CMD configured correctly"; else echo "❌ CMD incorrect: $CMD"; fi
```

Cleanup: docker rm -f $($CLI ps -aq --filter ancestor=net-tester:v2) 2>/dev/null; docker rmi net-tester:v2

</details>
---

## Scenario 3: The Hardened Python Backend
Combined Patterns: Multi-Stage Optimization + Non-Root Security + Environment Variables + Basic Build/Tag
The Setup: You need to containerize a Python backend service while keeping the image size tiny and secure.
Your Tasks:

1. Use python:3.10 as a builder stage to install dependencies from a requirements.txt file into a specific virtual environment folder.

2. Use python:3.10-slim for the final stage. Copy only the virtual environment and the application code from the builder.

3. Create a user appuser (UID 1001) and ensure the container runs as this user.

4. Add the environment variable FLASK_ENV=production.

5. Build and tag the image as backend-service:prod.

<details>
  <summary><b>config and testcase checker</b></summary>
  
Create requirements.txt:
  
```yaml
Flask==3.0.0
```
app.py:

```yaml
print("App running")
```
Test Script (verify.sh):

```yaml
#!/bin/bash
CLI="docker"
IMAGE="backend-service:prod"

if ! $CLI image inspect $IMAGE >/dev/null 2>&1; then echo "❌ Image not found"; exit 1; fi

USER=$($CLI inspect -f '{{.Config.User}}' $IMAGE)
if [[ "$USER" == "1001" || "$USER" == "appuser" ]]; then echo "✅ User set correctly"; else echo "❌ User incorrect: $USER"; fi

ENV=$($CLI inspect -f '{{json .Config.Env}}' $IMAGE)
if [[ "$ENV" == *"FLASK_ENV=production"* ]]; then echo "✅ FLASK_ENV set"; else echo "❌ Env var missing"; fi
```

Cleanup: docker rmi backend-service:prod

</details>
---

## Scenario 4: The Legacy App Migration & Export
Combined Patterns: File Injection (COPY) + Entrypoint vs. CMD + Basic Build/Tag + OCI Tarball Export
The Setup: A legacy bash script start-batch.sh needs to be containerized and shipped to an offline node.
Your Tasks:

1. Create a Dockerfile using ubuntu:22.04.

2. Copy the start-batch.sh script to /usr/local/bin/ and ensure it has executable permissions (using a RUN chmod command).

3. Set the ENTRYPOINT to the script.

4. Set the CMD to pass the default argument daily-run.

<details>
  <summary><b>config and testcase checker</b></summary>
Setup File (start-batch.sh):
  
(Make sure to run chmod +x start-batch.sh locally first if you are going to copy it without modifying permissions in the Dockerfile!)
  
```yaml
#!/bin/bash
echo "Starting batch job with argument: $1"
```
Test Script (verify.sh):
(Note: I've changed the export path to /tmp/batch-processor.tar instead of /opt/ so you don't run into local sudo/permission issues while practicing).
```yaml
#!/bin/bash
CLI="docker"
IMAGE="batch-processor:legacy"
TAR_PATH="/tmp/batch-processor.tar"

if ! $CLI image inspect $IMAGE >/dev/null 2>&1; then echo "❌ Image not found"; exit 1; else echo "✅ Image built"; fi

CMD=$($CLI inspect -f '{{json .Config.Cmd}}' $IMAGE)
if [[ "$CMD" == *"daily-run"* ]]; then echo "✅ CMD set to daily-run"; else echo "❌ CMD missing or incorrect"; fi

if [ -f "$TAR_PATH" ]; then echo "✅ Tarball found at $TAR_PATH"; else echo "❌ Tarball not found"; fi
```

Cleanup: docker rmi batch-processor:legacy && rm -f /tmp/batch-processor.tar

</details>
--- 

## Scenario 5: The Multi-Stage Go API Archive
Combined Patterns: Multi-Stage Optimization + Environment Variables + Basic Build/Tag + OCI Tarball Export
The Setup: You have a main.go file that connects to a database.
Your Tasks:

1. Write a multi-stage Dockerfile. The first stage must use golang:1.20 to build the binary (name it api-server).

2. The second stage must use a minimal alpine:latest base image.

3. Inject an environment variable DB_HOST=primary-db.local into the final image.

4. Build and tag the image as go-api:optimized.

5. Export the finished image as a tarball named go-api-backup.tar to your /tmp/ directory.

<details>
  <summary><b>config and testcase checker</b></summary>
  
setup
  
```yaml
package main
import ("fmt"; "os")
func main() {
    fmt.Println("DB_HOST is:", os.Getenv("DB_HOST"))
}
```
test

```yaml
#!/bin/bash
CLI="docker"
IMAGE="go-api:optimized"
TAR_PATH="/tmp/go-api-backup.tar"

if ! $CLI image inspect $IMAGE >/dev/null 2>&1; then echo "❌ Image not found"; exit 1; else echo "✅ Image built"; fi

ENV_VARS=$($CLI inspect -f '{{json .Config.Env}}' $IMAGE)
if [[ "$ENV_VARS" == *"DB_HOST=primary-db.local"* ]]; then echo "✅ Env var DB_HOST found"; else echo "❌ Env var missing"; fi

if [ -f "$TAR_PATH" ]; then echo "✅ Tarball found at $TAR_PATH"; else echo "❌ Tarball not found"; fi
```

</details>



