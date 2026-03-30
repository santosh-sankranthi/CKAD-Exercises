
## Define, Build and Modify Container Images

### Task 1: The Core Assembly (Most Repeating)

You will be given an application file (like a Python script or an HTML file) and asked to write a `Dockerfile` from scratch, build it, and tag it correctly.

#### Main Task 1: Building from Scratch

**1. CKAD Style Question:**
You have been provided a web directory at `/opt/webapp` containing an `index.html` file.

1.  Write a `Dockerfile` inside this directory.
2.  Use `nginx:alpine` as the base image.
3.  Copy the `index.html` file into the container's `/usr/share/nginx/html/` directory.
4.  Expose port `80`.
5.  Build the image and tag it exactly as `custom-nginx:v1.0`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt/webapp && sudo chmod 777 /opt/webapp
echo "<h1>CKAD Practice</h1>" > /opt/webapp/index.html
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 1 ---"
docker image inspect custom-nginx:v1.0 >/dev/null 2>&1 && echo "✅ Image custom-nginx:v1.0 successfully built and tagged" || echo "❌ Image build or tag failed"
[ "$(docker inspect -f '{{.Config.ExposedPorts}}' custom-nginx:v1.0 | grep '80/tcp')" ] && echo "✅ Port 80 exposed correctly" || echo "❌ Port 80 not exposed"
docker run --rm custom-nginx:v1.0 cat /usr/share/nginx/html/index.html | grep -q "CKAD" && echo "✅ File successfully copied into image" || echo "❌ File copy failed"
```

<details>

4. Solution:

```bash
# 1. Navigate to the directory
cd /opt/webapp

# 2. Create the Dockerfile
vi Dockerfile
```

```dockerfile
# 3. Write the required instructions
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
```

```bash
# 4. Build the image (Don't forget the '.' at the end to specify the current directory!)
docker build -t custom-nginx:v1.0 .
```

</details>

**5. Clean-up Script:**

```bash
docker rmi custom-nginx:v1.0 --force
rm -rf /opt/webapp
```

#### Variation 1.1: Modifying and Securing an Existing Image

**1. CKAD Style Question:**
A flawed `Dockerfile` exists at `/opt/secure-app/Dockerfile`.
Modify the `Dockerfile` to meet the following new requirements:

1.  Change the base image from `ubuntu:latest` to `ubuntu:22.04`.
2.  Add an environment variable named `APP_ENV` with the value `production`.
3.  Add a user named `appuser` and ensure the container runs as this non-root user (using the `USER` instruction).
4.  Rebuild the image and tag it as `secure-app:v2`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt/secure-app && sudo chmod 777 /opt/secure-app
cat <<EOF > /opt/secure-app/Dockerfile
FROM ubuntu:latest
RUN apt-get update && apt-get install -y curl
CMD ["sleep", "3600"]
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.1 ---"
docker image inspect secure-app:v2 >/dev/null 2>&1 && echo "✅ Image built successfully" || echo "❌ Image build failed"
[ "$(docker inspect -f '{{.Config.User}}' secure-app:v2)" == "appuser" ] && echo "✅ USER instruction configured correctly" || echo "❌ USER instruction failed"
[ "$(docker inspect -f '{{.Config.Env}}' secure-app:v2 | grep 'APP_ENV=production')" ] && echo "✅ ENV instruction configured correctly" || echo "❌ ENV instruction failed"
```

<details>
  
4. Solution:

```bash
cd /opt/secure-app
vi Dockerfile
```

*Modify the file to look like this:*

```dockerfile
FROM ubuntu:22.04                                       # Changed tag
ENV APP_ENV=production                                  # Added ENV
RUN apt-get update && apt-get install -y curl
RUN useradd -ms /bin/bash appuser                       # Create the user
USER appuser                                            # Set the user
CMD ["sleep", "3600"]
```

```bash
docker build -t secure-app:v2 .
```

</details>

**5. Clean-up Script:**

```bash
docker rmi secure-app:v2 ubuntu:22.04 ubuntu:latest --force
rm -rf /opt/secure-app
```

#### Variation 1.2: The CMD vs ENTRYPOINT Override

**1. CKAD Style Question:**
A `Dockerfile` at `/opt/entry-app/Dockerfile` uses `CMD ["ping", "127.0.0.1"]`.
This is a bad practice for an executable container. Modify the `Dockerfile` so that `ping` is hardcoded as the `ENTRYPOINT`, and `127.0.0.1` is passed as the default `CMD`.
Build the image and tag it as `ping-app:v1`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt/entry-app && sudo chmod 777 /opt/entry-app
cat <<EOF > /opt/entry-app/Dockerfile
FROM busybox
CMD ["ping", "127.0.0.1"]
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 1.2 ---"
[ "$(docker inspect -f '{{.Config.Entrypoint}}' ping-app:v1)" == "[ping]" ] && echo "✅ ENTRYPOINT configured correctly" || echo "❌ ENTRYPOINT missing or incorrect"
[ "$(docker inspect -f '{{.Config.Cmd}}' ping-app:v1)" == "[127.0.0.1]" ] && echo "✅ CMD configured correctly" || echo "❌ CMD missing or incorrect"
```

<details>

4. Solution:

```bash
cd /opt/entry-app
vi Dockerfile
```

*Change the `CMD` instruction to an `ENTRYPOINT` + `CMD` pattern:*

```dockerfile
FROM busybox
ENTRYPOINT ["ping"]
CMD ["127.0.0.1"]
```

```bash
docker build -t ping-app:v1 .
```

</details>

**5. Clean-up Script:**

```bash
docker rmi ping-app:v1 busybox --force
rm -rf /opt/entry-app
```

-----

### Task 2: Advanced Image Management (Multi-Stage & Exporting)

The exam will test if you understand how to keep images lightweight (which speeds up Pod scheduling) and how to handle images in air-gapped environments without pushing to Docker Hub.

#### Main Task 2: Multi-Stage Builds (Optimization)

**1. CKAD Style Question:**
A `Dockerfile` at `/opt/heavy-build/Dockerfile` simulates compiling an application, resulting in a large image.
Modify the `Dockerfile` to use a **Multi-Stage Build**.

  * Stage 1 should be named `builder` and compile the app.
  * Stage 2 should use `alpine:latest` as the base image, copy the compiled binary (`/build/myapp`) from the `builder` stage into `/app/myapp`, and execute it.
    Build the final optimized image and tag it as `tiny-app:v1`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt/heavy-build && sudo chmod 777 /opt/heavy-build
# We are using 'dd' to simulate a heavy compiler environment and a resulting binary
cat <<EOF > /opt/heavy-build/Dockerfile
FROM ubuntu:latest
RUN mkdir /build && echo "Simulating compiled binary" > /build/myapp
CMD ["cat", "/build/myapp"]
EOF
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 2 ---"
docker image inspect tiny-app:v1 >/dev/null 2>&1 && echo "✅ Image built successfully" || echo "❌ Image build failed"
[ "$(docker inspect -f '{{.Config.Image}}' tiny-app:v1 | grep -i 'alpine')" ] || [ "$(docker run --rm tiny-app:v1 cat /etc/os-release | grep -i 'alpine')" ] && echo "✅ Base image is Alpine (Optimization successful)" || echo "❌ Base image is not Alpine"
docker run --rm tiny-app:v1 cat /app/myapp | grep -q "Simulating" && echo "✅ Binary successfully copied from builder stage" || echo "❌ COPY --from failed"
```

<details>

4. Solution:

```bash
cd /opt/heavy-build
vi Dockerfile
```

*Rewrite the Dockerfile using two `FROM` instructions. The critical instruction is `COPY --from=builder`:*

```dockerfile
# Stage 1: The Builder
FROM ubuntu:latest AS builder
RUN mkdir /build && echo "Simulating compiled binary" > /build/myapp

# Stage 2: The Lightweight Runtime
FROM alpine:latest
COPY --from=builder /build/myapp /app/myapp
CMD ["cat", "/app/myapp"]
```

```bash
docker build -t tiny-app:v1 .
```

</details>

**5. Clean-up Script:**

```bash
docker rmi tiny-app:v1 ubuntu:latest alpine:latest --force
rm -rf /opt/heavy-build
```

#### Variation 2.1: Exporting an Image as a Tarball

**1. CKAD Style Question:**
You have built an image named `offline-app:v1`. The target Kubernetes cluster does not have internet access or a private registry.
Save the `offline-app:v1` image as a tar archive to `/opt/offline-app.tar` so it can be manually transferred to the cluster nodes.

**2. Setup Script:**

```bash
docker pull busybox:latest
docker tag busybox:latest offline-app:v1
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 2.1 ---"
[ -f /opt/offline-app.tar ] && echo "✅ Tarball successfully created" || echo "❌ Tarball missing"
tar -tf /opt/offline-app.tar | grep -q "manifest.json" && echo "✅ File is a valid Docker archive" || echo "❌ File is not a valid image archive"
```

<details>

4. Solution:

```bash
# Use the docker save command. (Do NOT use docker export, which is for containers, not images).
docker save offline-app:v1 -o /opt/offline-app.tar

# Alternatively, you can use:
# docker save offline-app:v1 > /opt/offline-app.tar
```

</details>

**5. Clean-up Script:**

```bash
rm -f /opt/offline-app.tar
docker rmi offline-app:v1 busybox:latest --force
```

-----


### Task 3: Build Context and Advanced Instructions (The Edge Cases)

The exam will test your ability to optimize the build process itself, ensuring that unnecessary files aren't sent to the container engine, and that you know the shortcuts for extracting archives.

#### Main Task 3: Context Optimization (`.dockerignore`)

**1. CKAD Style Question:**
You are given a directory at `/opt/heavy-context` containing a web application (`app.js`) and a massive 5GB local log file (`debug.log`).
Write a `Dockerfile` using `node:alpine` that copies the entire directory contents (`COPY . /app`) into the image.
However, you must ensure that the `debug.log` file is **strictly excluded** from the build context so it does not bloat the image or slow down the build process. Build and tag the image as `fast-node:v1`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt/heavy-context && sudo chmod 777 /opt/heavy-context
echo "console.log('App running');" > /opt/heavy-context/app.js
# Simulating a massive log file
echo "MASSIVE ERROR LOGS" > /opt/heavy-context/debug.log
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Main Task 3 ---"
[ -f /opt/heavy-context/.dockerignore ] && grep -q "debug.log" /opt/heavy-context/.dockerignore && echo "✅ .dockerignore correctly configured" || echo "❌ .dockerignore missing or incorrect"
docker run --rm fast-node:v1 ls /app/debug.log 2>&1 | grep -q "No such file" && echo "✅ debug.log successfully excluded from final image" || echo "❌ FAILED: debug.log leaked into the image"
```

<details>

4. Solution:

```bash
cd /opt/heavy-context

# 1. You MUST create a .dockerignore file BEFORE building
echo "debug.log" > .dockerignore

# 2. Write the Dockerfile
vi Dockerfile
```

```dockerfile
FROM node:alpine
COPY . /app
CMD ["node", "/app/app.js"]
```

```bash
# 3. Build the image
docker build -t fast-node:v1 .
```

</details>

**5. Clean-up Script:**

```bash
docker rmi fast-node:v1 node:alpine --force
rm -rf /opt/heavy-context
```

#### Variation 3.1: Auto-Extraction and Working Directories (`ADD` vs `COPY`)

**1. CKAD Style Question:**
A tarball named `assets.tar.gz` exists in `/opt/archive-app`.
Write a `Dockerfile` using `nginx:alpine` that achieves the following:

1.  Sets the default working directory inside the container to `/usr/share/nginx/html`.
2.  Automatically extracts the contents of `assets.tar.gz` directly into this working directory during the build process. *(Hint: Do not use `COPY` and then `tar -xvf`. Use the instruction that does this natively).*
    Build and tag the image as `archive-web:v1`.

**2. Setup Script:**

```bash
sudo mkdir -p /opt/archive-app && sudo chmod 777 /opt/archive-app
cd /opt/archive-app
echo "<h1>Extracted Content</h1>" > index.html
tar -czvf assets.tar.gz index.html
rm index.html
```

**3. Testcase Script:**

```bash
#!/bin/bash
echo "--- Testing Variation 3.1 ---"
[ "$(docker inspect -f '{{.Config.WorkingDir}}' archive-web:v1)" == "/usr/share/nginx/html" ] && echo "✅ WORKDIR correctly configured" || echo "❌ WORKDIR failed"
docker run --rm archive-web:v1 cat /usr/share/nginx/html/index.html | grep -q "Extracted" && echo "✅ Tarball successfully auto-extracted via ADD" || echo "❌ Auto-extraction failed (Did you use COPY instead of ADD?)"
```

<details>

4. Solution:

```bash
cd /opt/archive-app
vi Dockerfile
```

*You must use `WORKDIR` to set the path, and `ADD` (not `COPY`) to trigger the automatic tarball extraction feature of Docker:*

```dockerfile
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
# ADD automatically unpacks compressed local tarballs into the destination
ADD assets.tar.gz .
```

```bash
docker build -t archive-web:v1 .
```

</details>

**5. Clean-up Script:**

```bash
docker rmi archive-web:v1 --force
rm -rf /opt/archive-app
```

-----

### The 100% Verdict

By covering the structural build process (`FROM`, `COPY`, `EXPOSE`), live overrides (`ENV`, `USER`, `ENTRYPOINT` vs `CMD`), multi-stage architecture optimization (`COPY --from`), and node-level exporting (`docker save`), you have thoroughly mapped the boundaries of the "Define, build and modify container images" objective.

Would you like to move directly into the next piece of Application Design and Build: **Jobs and CronJobs**?
