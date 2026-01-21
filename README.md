# Jenkins + Kubernetes CI Setup on Mac (Docker Desktop)

This guide explains step-by-step how to set up Jenkins on Docker Desktop, create Git-managed CI images, run pipelines in Kubernetes pods, and safely manage Docker images on Mac.

---

## Table of Contents

1. [Prerequisites](#prerequisites)  
2. [Install Jenkins with Kubernetes plugin](#install-jenkins)  
3. [Create Git-managed CI images](#create-ci-images)  
4. [Build CI images locally](#build-ci-images-locally)  
5. [Create Jenkins pipeline using fixed images](#jenkins-pipeline)  
6. [Run Python-only container](#python-only-container)  
7. [List Docker images](#list-docker-images)  
8. [Clean up unused images safely](#cleanup-unused-images)

---

## 1. Prerequisites

- Mac with Docker Desktop installed
- Kubernetes enabled in Docker Desktop
- Git installed
- Basic terminal knowledge

---

## 2. Install Jenkins

1. Run Jenkins container:

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:2.528.3-jdk21
```

2. Open Jenkins at http://localhost:8080

3. Install plugins:

- Kubernetes
- Git
- Pipeline

4. Configure Kubernetes cloud in Jenkins:

- Kubernetes URL: leave default (Docker Desktop)
- Jenkins URL: http://localhost:8080
- Credentials: None required (Docker Desktop shares local cluster)

---

## 3. Create Git-managed CI images

Directory structure:
```
ci-images/
├── python/
│   ├── Dockerfile
│   └── README.md
├── node/
│   └── Dockerfile
└── build-images.sh
```

**Python Dockerfile** (`ci-images/python/Dockerfile`):
```dockerfile
FROM python:3.11-slim

RUN pip install --no-cache-dir \
    requests \
    pandas \
    numpy

WORKDIR /workspace
```

**Node Dockerfile** (`ci-images/node/Dockerfile`):
```dockerfile
FROM node:20-slim

WORKDIR /workspace
```

**Build script** (`ci-images/build-images.sh`):
```bash
#!/bin/bash
set -e

echo "Building Python image..."
docker build -t ci/python-runner:1.0 ./python

echo "Building Node image..."
docker build -t ci/node-runner:1.0 ./node

echo "All images built successfully"
```

Make the script executable:
```bash
chmod +x build-images.sh
```

---

## 4. Build CI images locally

```bash
cd ci-images
./build-images.sh
```

Verify images:
```bash
docker images | grep ci/
```

Output should show:
```
ci/python-runner   1.0
ci/node-runner     1.0
```

Commit to Git:
```bash
git init
git add .
git commit -m "Add CI images for Python and Node"
```

---

## 5. Create Jenkins pipeline using fixed images

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

---

## 6. Python-only container

If you only need Python:

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

**Notes:**

- `command: ['cat']` and `tty: true` keep the container alive so Jenkins can exec into it.
- Container names in YAML must match `container('python')` references in pipeline.

---

## 7. List Docker images

```bash
docker images
```

Filter only CI images:
```bash
docker images | grep ci/
```

---

## 8. Clean up unused images safely

**Step 1 — List running containers**
```bash
docker ps
```
Images currently used by containers cannot be deleted.

**Step 2 — List all images except CI images**
```bash
docker images --format "{{.Repository}}:{{.Tag}}" | grep -v '^ci/'
```

**Step 3 — Remove safe unused images manually**
```bash
docker rmi <IMAGE_NAME>
```

Or force-remove (be careful with system images):
```bash
docker rmi -f <IMAGE_NAME>
```

**Recommended: Keep these system images**

- `docker/desktop-*`
- `jenkins/jenkins:*`
- `registry.k8s.io/*`
- Any image used by `docker ps` containers

**Step 4 — Verify cleanup**
```bash
docker images
```

Only Git-managed images (`ci/python-runner`, `ci/node-runner`) should remain.

---

## Summary

- Jenkins is running on Docker Desktop with Kubernetes plugin.
- CI images (Python, Node, etc.) are managed via Git.
- Pipelines reference fixed images, ensuring reproducible builds.
- Only unused Docker images are removed to keep environment clean.

This guide allows you to rebuild your environment at any time by following the steps above.
