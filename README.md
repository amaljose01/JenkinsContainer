# Jenkins + Kubernetes CI Setup on Mac (Docker Desktop)

This comprehensive guide explains step-by-step how to set up Jenkins on Docker Desktop, create Git-managed CI images, run pipelines in Kubernetes pods, and safely manage Docker images on Mac. By the end of this guide, you'll have a fully functional CI/CD environment running locally on your macOS machine using Docker Desktop's built-in Kubernetes cluster.

---

## Table of Contents

1. [Docker Desktop Setup (Mac)](#docker-desktop-setup-mac)
2. [Prerequisites](#prerequisites)
3. [Install Jenkins with Kubernetes plugin](#install-jenkins)
4. [Create Git-managed CI images](#create-ci-images)
5. [Build CI images locally](#build-ci-images-locally)
6. [Create Jenkins pipeline using fixed images](#jenkins-pipeline)
7. [Run Python-only container](#python-only-container)
8. [List Docker images](#list-docker-images)
9. [Clean up unused images safely](#cleanup-unused-images)

---

## Docker Desktop Setup (Mac)

This project assumes Docker Desktop is installed and configured correctly on macOS. Docker Desktop provides both the Docker engine and a local Kubernetes cluster, which are essential for running Jenkins and your CI pipelines.

---

### 1. Install Docker Desktop

Docker Desktop is the foundation for this entire setup. It includes Docker Engine (the containerization platform) and Kubernetes (the orchestration platform).

**Steps:**

1. Go to the official Docker website:
   https://www.docker.com/products/docker-desktop/

2. Download **Docker Desktop for Mac**
   - If you have an **Apple Silicon Mac** (M1, M2, M3, etc.), download the Apple Silicon version
   - If you have an **Intel Mac**, download the Intel version

3. Install Docker Desktop by dragging the downloaded application to the **Applications** folder

4. Open **Applications** and double-click **Docker.app** to start Docker Desktop

5. You'll see the Docker menu icon (whale icon) in your macOS menu bar once it's running

---

### 2. Verify Docker Is Running

Once Docker Desktop is open and running, verify the installation by checking the Docker version.

**Open Terminal and run:**

```bash
docker version
```

You should see both **Client** and **Server** information, confirming Docker is installed and running correctly.

**Example output:**
```
Client: Docker Engine - Community
 Version:           24.0.6
 API version:       1.43
 Go version:        go1.20.12

Server: Docker Engine - Community
 Engine:
  Version:          24.0.6
  API version:      1.43
```

---

### 3. Enable Kubernetes in Docker Desktop

Docker Desktop includes a built-in Kubernetes cluster that runs on your local machine. By default, it may not be enabled, so you need to activate it.

**Steps:**

1. Click the **Docker menu icon** (whale) in the macOS menu bar
2. Select **Settings** (or **Preferences** on older versions)
3. Navigate to the **Kubernetes** tab on the left sidebar
4. Check the box **Enable Kubernetes**
5. Click **Apply & Restart**
6. Wait 2-3 minutes while Docker Desktop restarts and initializes Kubernetes

Once complete, you'll see a green checkmark and "Kubernetes is running" message.

---

### 4. Install kubectl (Kubernetes CLI)

`kubectl` is the command-line tool for managing Kubernetes clusters. It's used to interact with your local Kubernetes cluster.

**Check if kubectl is already installed:**

```bash
kubectl version --client
```

**If not installed, install using Homebrew:**

```bash
brew install kubectl
```

**Verify the installation:**

```bash
kubectl get nodes
```

You should see output showing the `docker-desktop` node:

```
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   5m    v1.27.1
```

---

### 5. (Optional) Install Helm

**Helm** is a package manager for Kubernetes. While optional for this guide, it's useful for installing complex applications on Kubernetes more easily.

**Install Helm using Homebrew:**

```bash
brew install helm
```

**Verify the installation:**

```bash
helm version
```

Expected output:
```
version.BuildInfo{Version:"v3.12.0", GitCommit:"c9f554d75773799f72ceef38c51342dc1d8110b5", ...}
```

---

### 6. Verify Kubernetes Is Working

Test that Kubernetes is properly configured and running:

```bash
kubectl get pods -A
```

This command lists all pods (containers) running in all Kubernetes namespaces. You should see system pods for Kubernetes itself.

**Example output:**
```
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE
kube-system   coredns-5d78c0869d-xyz                   1/1     Running   0          2m
kube-system   etcd-docker-desktop                      1/1     Running   0          2m
kube-system   kube-apiserver-docker-desktop            1/1     Running   0          2m
kube-system   kube-controller-manager-docker-desktop   1/1     Running   0          2m
```

---

### 7. Important Notes

**What Docker Desktop provides:**

- **Docker Engine**: The containerization platform that runs your containers
- **Kubernetes**: A single-node local cluster for orchestrating containers
- **Image Availability**: Docker images built locally are automatically available to Kubernetes (no external registry required)

**Key benefits for local development:**

- No need for a Docker Hub account or external container registry
- Images built with `docker build` are immediately available to Kubernetes
- Perfect for testing CI/CD pipelines locally before deploying to production
- Isolated environment that doesn't affect your system

---

## Prerequisites

Before proceeding with Jenkins installation, ensure you have completed all Docker Desktop setup steps above. You'll also need:

- **macOS machine** (Intel or Apple Silicon)
- **Docker Desktop** installed and running (with Kubernetes enabled)
- **kubectl** installed and verified
- **Git** installed on your system (check with `git --version`)
- **Basic terminal knowledge** (comfortable with bash/zsh commands)
- **Homebrew** installed (optional, but recommended for package management)

If you don't have Git installed, install it via Homebrew:

```bash
brew install git
```

---

## Install Jenkins

Jenkins is an open-source automation server that orchestrates complex build, test, and deployment pipelines. In this setup, Jenkins runs in a Docker container and can spawn pods on your local Kubernetes cluster to execute build steps.

---

### Step 1: Run Jenkins Container

Start a Jenkins container with persistent storage and port mappings:

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:2.528.3-jdk21
```

**Explanation of flags:**

- `-d`: Run in detached mode (background)
- `--name jenkins`: Give the container a readable name
- `-p 8080:8080`: Map container port 8080 (Jenkins web UI) to localhost:8080
- `-p 50000:50000`: Map port 50000 for Jenkins agent communication
- `-v jenkins_home:/var/jenkins_home`: Create a Docker volume to persist Jenkins data (configs, plugins, build history)
- `jenkins/jenkins:2.528.3-jdk21`: Specific Jenkins image version with Java 21

**Verify Jenkins is running:**

```bash
docker ps | grep jenkins
```

---

### Step 2: Access Jenkins

Open your web browser and navigate to:

```
http://localhost:8080
```

On first access, you'll see a "Unlock Jenkins" screen. To get the initial admin password:

```bash
docker logs jenkins | grep -A 5 "Please use the following password to proceed to installation"
```

Or:

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Copy the password (a long string of characters) and paste it into the Jenkins unlock page.

---

### Step 3: Install Required Plugins

After unlocking Jenkins, you'll be prompted to install plugins. Select **Install suggested plugins**, then additionally install:

1. **Kubernetes plugin** - Allows Jenkins to spawn pods on Kubernetes for build execution
2. **Git plugin** - For checking out code from Git repositories
3. **Pipeline plugin** - For creating declarative and scripted pipelines

**To manually install plugins:**

1. Click **Manage Jenkins** → **Manage Plugins**
2. Go to the **Available** tab
3. Search for "Kubernetes", "Git", and "Pipeline"
4. Check the boxes and click **Install without restart**

---

### Step 4: Configure Kubernetes Cloud

This step tells Jenkins where and how to connect to your Kubernetes cluster.

**Steps:**

1. Click **Manage Jenkins** → **Configure System**
2. Scroll down to find **Cloud** section (or **Kubernetes**)
3. Click **New cloud** and select **Kubernetes**
4. Set the following:
   - **Kubernetes URL**: Leave as `https://kubernetes.default` (Docker Desktop default)
   - **Kubernetes Namespace**: `default`
   - **Jenkins URL**: `http://jenkins:8080` (use the container name as hostname within Docker network)
   - **Jenkins Tunnel**: `jenkins:50000`
   - **Credentials**: None required (Docker Desktop shares local cluster)

5. Click **Test Connection** to verify
6. Click **Save**

**Why this works:**

- Docker Desktop's Kubernetes cluster automatically includes a DNS entry for `jenkins` when running in the same Docker network
- This allows pods spawned by Kubernetes to communicate back to the Jenkins container

---

## Create Git-managed CI Images

Instead of manually installing tools in containers, we'll create reusable Dockerfile images and manage them in Git. This approach ensures:

- **Reproducibility**: Same environment every time
- **Version control**: Track changes to your build environment
- **Fast builds**: No need to install tools on every build
- **Easy maintenance**: Update tools in one place, rebuild images

---

### Directory Structure

Create the following structure in your project root:

```
ci-images/
├── python/
│   ├── Dockerfile
│   └── README.md
├── node/
│   └── Dockerfile
└── build-images.sh
```

**Create the directory:**

```bash
mkdir -p ci-images/python ci-images/node
```

---

### Python Dockerfile

Create `ci-images/python/Dockerfile` with pre-installed Python libraries commonly used in CI/CD:

```dockerfile
FROM python:3.11-slim

# Install common data science and utility libraries
RUN pip install --no-cache-dir \
    requests \
    pandas \
    numpy

# Set working directory for pipeline executions
WORKDIR /workspace
```

**What this does:**

- `FROM python:3.11-slim`: Starts with a lightweight Python 3.11 image
- `RUN pip install`: Pre-installs popular Python packages
- `--no-cache-dir`: Reduces image size by not caching pip downloads
- `WORKDIR /workspace`: Sets the directory where Jenkins will execute commands

---

### Node Dockerfile

Create `ci-images/node/Dockerfile` for Node.js-based projects:

```dockerfile
FROM node:20-slim

# Set working directory for pipeline executions
WORKDIR /workspace
```

**What this does:**

- Uses Node.js 20 LTS (slim image)
- Sets up `/workspace` as the execution directory

---

### Build Script

Create `ci-images/build-images.sh` to build all images at once:

```bash
#!/bin/bash
set -e

echo "Building Python image..."
docker build -t ci/python-runner:1.0 ./python

echo "Building Node image..."
docker build -t ci/node-runner:1.0 ./node

echo "All images built successfully"
```

**Make it executable:**

```bash
chmod +x ci-images/build-images.sh
```

**What this script does:**

- `set -e`: Exit immediately if any command fails
- `-t ci/python-runner:1.0`: Tags the image with a repository name and version
- Builds all images in sequence
- `1.0`: Version tag (update when you change the Dockerfile)

---

## Build CI Images Locally

Now that you've created the Dockerfiles and build script, build the images on your local machine.

---

### Step 1: Run the Build Script

Navigate to the project root and execute the build script:

```bash
cd ci-images
./build-images.sh
```

**What happens:**

1. Docker reads the `python/Dockerfile`
2. Downloads the `python:3.11-slim` base image
3. Runs the pip install command
4. Tags the result as `ci/python-runner:1.0`
5. Repeats for Node.js

**First build takes 2-3 minutes** (downloading base images). Subsequent builds are faster due to Docker's layer caching.

---

### Step 2: Verify Images Were Built

```bash
docker images | grep ci/
```

**Expected output:**

```
ci/python-runner   1.0       abcd1234efgh   2 minutes ago   985MB
ci/node-runner     1.0       ijkl5678mnop   1 minute ago    432MB
```

If you don't see these images, check the build script output for errors.

---

### Step 3: Commit to Git

Version control ensures you can rebuild your environment anytime:

```bash
git init
git add ci-images/
git commit -m "Add CI images for Python and Node"
```

**Why commit these files:**

- The `Dockerfile`s are small text files (perfect for Git)
- Teammates can clone and rebuild the same images
- You can track changes to build environments
- No need to commit the built images themselves (they're large binary files)

**Optional: Update .gitignore**

If you want to ensure docker images aren't accidentally committed, create or update `.gitignore`:

```bash
echo "*.tar" >> .gitignore
echo "*.tar.gz" >> .gitignore
git add .gitignore
git commit -m "Add .gitignore"
```

---

## Create Jenkins Pipeline Using Fixed Images

A Jenkins Pipeline is a collection of tasks (stages) that run in sequence to build, test, and deploy your code. The Kubernetes plugin allows each stage to run in a separate container (pod) on your Kubernetes cluster.

**Benefits:**

- Different stages can use different container images
- Containers are created on-demand for each pipeline run
- No dependency pollution between builds
- Automatic cleanup after builds

---

### Multi-Container Pipeline

This pipeline example uses both Python and Node.js containers:

```groovy
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: python
    image: ci/python-runner:1.0
    command: ['cat']
    tty: true

  - name: node
    image: ci/node-runner:1.0
    command: ['cat']
    tty: true
"""
    }
  }

  stages {
    stage('Build') {
      steps {
        container('node') {
          sh 'npm install'
        }
        container('python') {
          sh 'python scripts/run.py'
        }
      }
    }
  }
}
```

**How to create this pipeline in Jenkins:**

1. Click **New Item** on the Jenkins home page
2. Enter a job name (e.g., "Multi-Container Build")
3. Select **Pipeline**
4. Click **OK**
5. In the **Pipeline** section, select **Pipeline script**
6. Paste the above Groovy code into the script area
7. Click **Save**
8. Click **Build Now** to run the pipeline

**What each part does:**

- **`agent { kubernetes { ... } }`**: Tells Jenkins to run this pipeline in Kubernetes pods
- **`apiVersion: v1` and `kind: Pod`**: Kubernetes pod configuration
- **`containers:`**: List of containers available during the build
- **`image: ci/python-runner:1.0`**: Use the Python image you built earlier
- **`command: ['cat']` and `tty: true`**: Keep the container running so Jenkins can execute commands
- **`container('node')`**: Run the next steps inside the Node container
- **`sh 'npm install'`**: Execute a shell command inside the container

**Container naming rules:**

- Container names in the YAML must match the names used in `container('name')` calls
- Container names are lowercase and alphanumeric
- A pod can have multiple containers, all running simultaneously

---

### View Pipeline Execution

Once you click **Build Now**:

1. Jenkins creates a Kubernetes pod with both containers
2. Runs npm install in the Node container
3. Runs your Python script in the Python container
4. Cleans up the pod when done

**Monitor the build:**

1. Click the build number (e.g., #1) in the **Build History**
2. Click **Console Output** to see real-time logs
3. Watch as containers start, commands execute, and output is captured

---

## Run Python-Only Container

If your project only requires Python (no Node.js or other tools), you can simplify the pipeline to use a single container.

---

### Simple Python Pipeline

```groovy
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: python
    image: ci/python-runner:1.0
    command: ['cat']
    tty: true
"""
    }
  }

  stages {
    stage('Run Python Script') {
      steps {
        container('python') {
          sh 'python scripts/run.py'
        }
      }
    }
  }
}
```

**Key differences:**

- Only one container in the pod spec
- Only one `stage` that runs Python
- No Node.js or other dependencies
- Faster startup time (smaller pod)
- Lighter memory footprint

---

### Adding Additional Stages

You can expand this pipeline to include multiple stages:

```groovy
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: python
    image: ci/python-runner:1.0
    command: ['cat']
    tty: true
"""
    }
  }

  stages {
    stage('Setup') {
      steps {
        container('python') {
          sh 'pip install -r requirements.txt'
        }
      }
    }

    stage('Test') {
      steps {
        container('python') {
          sh 'python -m pytest tests/'
        }
      }
    }

    stage('Run') {
      steps {
        container('python') {
          sh 'python scripts/run.py'
        }
      }
    }
  }
}
```

**How this pipeline works:**

1. **Setup stage**: Installs Python dependencies from `requirements.txt`
2. **Test stage**: Runs unit tests using pytest
3. **Run stage**: Executes your main Python script

Each stage runs sequentially. If any stage fails, the pipeline stops and reports the failure.

---

### Important Notes on Container Configuration

**Why `command: ['cat']` and `tty: true`?**

- By default, containers execute a command and exit
- Jenkins needs to "exec" into the container to run commands
- `command: ['cat']` starts the container with the `cat` command (which does nothing)
- `tty: true` allocates a pseudo-terminal
- Together, they keep the container running until Jenkins finishes

**Container naming rules:**

- Container names in the YAML must match the names used in `container('name')` calls
- Container names are lowercase and alphanumeric
- A pod can have multiple containers, all running simultaneously

---

## List Docker Images

Managing Docker images is important for understanding what's installed on your system and troubleshooting issues.

---

### View All Images

List all Docker images on your system:

```bash
docker images
```

**Example output:**

```
REPOSITORY                      TAG           IMAGE ID      CREATED        SIZE
ci/python-runner                1.0           abcd1234efgh  2 hours ago    985MB
ci/node-runner                  1.0           ijkl5678mnop  1 hour ago     432MB
jenkins/jenkins                 2.528.3-jdk21 mnop9012qrst  3 days ago     842MB
python                          3.11-slim     qrst3456uvwx  1 week ago     156MB
node                            20-slim       uvwx7890yzab  1 week ago     172MB
```

**Understanding the output:**

- **REPOSITORY**: Image name (namespace/name format)
- **TAG**: Version or label (1.0, latest, etc.)
- **IMAGE ID**: Unique identifier (first 12 characters shown)
- **CREATED**: When the image was built
- **SIZE**: Disk space used by the image

---

### Filter CI Images Only

Show only your custom CI images:

```bash
docker images | grep ci/
```

**Output:**

```
ci/python-runner   1.0       abcd1234efgh   2 hours ago   985MB
ci/node-runner     1.0       ijkl5678mnop   1 hour ago    432MB
```

---

### Show Image Details

Get detailed information about a specific image:

```bash
docker inspect ci/python-runner:1.0
```

This shows:

- Layer structure
- Environment variables
- Exposed ports
- Command that runs
- And much more

---

## Clean Up Unused Images Safely

Over time, Docker downloads base images, creates new images, and stores old versions. This can consume significant disk space. However, you must be careful not to delete images that are in use.

---

### Step 1: List Running Containers

First, see which images are actively in use:

```bash
docker ps
```

Shows only **running** containers. These images cannot be deleted.

**List all containers (including stopped ones):**

```bash
docker ps -a
```

Stopped containers may still use images, so be careful.

---

### Step 2: List Unused System Images

Show all images **except** your custom CI images:

```bash
docker images --format "{{.Repository}}:{{.Tag}}" | grep -v '^ci/'
```

**What this does:**

- `docker images --format "{{.Repository}}:{{.Tag}}"`: List images in format name:tag
- `grep -v '^ci/'`: Filter out (remove) images starting with `ci/`

**Example output:**

```
jenkins/jenkins:2.528.3-jdk21
python:3.11-slim
node:20-slim
docker/desktop-volumes:0.1.0
docker/desktop-proxy:0.1.0
registry.k8s.io/coredns:1.9.3
registry.k8s.io/kube-apiserver:v1.27.0
```

---

### Step 3: Review and Remove Unused Images

**Identify images you no longer need:**

Look at the list above and identify which ones you can safely remove. Usually:

- **Safe to remove**: Old versions of base images (python:3.10-slim if you use 3.11-slim)
- **NOT safe to remove**: See the safe list below

**Remove a single image:**

```bash
docker rmi jenkins/jenkins:2.400.0-jdk17
```

Replace with the image tag you want to remove.

**Force remove an image:**

```bash
docker rmi -f jenkins/jenkins:2.400.0-jdk17
```

Use `-f` only if the normal command fails (image in use).

**Remove multiple images at once:**

```bash
docker rmi python:3.10-slim node:18-slim
```

---

### Step 4: Keep These System Images

**Do NOT delete these images** - they're essential for Docker Desktop and Kubernetes:

```
docker/desktop-*             # Docker Desktop system images
jenkins/jenkins:*            # Jenkins (if you use Jenkins)
registry.k8s.io/*            # Kubernetes system images
<image used in docker ps>    # Any image from running containers
```

**Verify before deleting:**

Before running `docker rmi`, check if the image appears in:

```bash
docker ps -a
```

If it does, keep it (it's in use or was recently used).

---

### Step 5: Verify Cleanup

After removing images, verify they're gone:

```bash
docker images
```

**Expected result:**

Your custom CI images (`ci/python-runner`, `ci/node-runner`) should remain along with essential system images.

---

### Automated Cleanup (Advanced)

Docker has a prune command to clean up unused resources:

```bash
docker image prune
```

This removes all dangling images (layers not used by any image).

**For aggressive cleanup (removes all unused images):**

```bash
docker image prune -a
```

**WARNING**: This removes all images not currently used by a container. Use with caution.

---

## Complete Workflow Summary

Here's a high-level overview of the entire workflow:

---

### 1. Environment Setup

- Install Docker Desktop
- Enable Kubernetes
- Install kubectl and Helm (optional)

### 2. Jenkins Setup

- Start Jenkins container
- Unlock and configure Jenkins
- Install plugins (Kubernetes, Git, Pipeline)
- Configure Kubernetes cloud connection

### 3. CI Images

- Create Dockerfiles for Python, Node, etc.
- Build images locally
- Commit to Git for version control

### 4. Create Pipelines

- Use Kubernetes pods in Jenkins pipelines
- Reference your custom images
- Run builds and tests in containers

### 5. Maintenance

- Monitor image sizes
- Safely remove unused images
- Update Dockerfiles as needed
- Rebuild images
- Commit changes to Git

---

## Troubleshooting Common Issues

### Issue: Jenkins won't start

**Check if port 8080 is in use:**

```bash
lsof -i :8080
```

**Kill the process using port 8080:**

```bash
kill -9 <PID>
```

Then restart Jenkins.

### Issue: Kubernetes not enabled

**Verify status:**

```bash
kubectl get nodes
```

If it fails, enable Kubernetes in Docker Desktop Settings → Kubernetes → Enable Kubernetes.

### Issue: Pipeline fails to start

**Check Jenkins logs:**

```bash
docker logs jenkins
```

**Verify Kubernetes connectivity:**

```bash
kubectl cluster-info
```

### Issue: Container image not found

**Verify image exists:**

```bash
docker images | grep ci/
```

**If missing, rebuild:**

```bash
cd ci-images
./build-images.sh
```

---

## Next Steps

Now that you have Jenkins, Kubernetes, and custom CI images running:

1. **Create more complex pipelines** with multiple stages, tests, and deployments
2. **Add more tools** by creating additional Dockerfiles (Go, Java, Ruby, etc.)
3. **Integrate with Git** by creating Jenkinsfiles in your repository
4. **Set up webhooks** to trigger builds on Git push events
5. **Explore advanced features** like parameterized builds, credentials management, and artifact archiving

---

## Resources

- [Docker Documentation](https://docs.docker.com/)
- [Jenkins Documentation](https://www.jenkins.io/doc/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)
- [Kubernetes Plugin for Jenkins](https://plugins.jenkins.io/kubernetes/)
