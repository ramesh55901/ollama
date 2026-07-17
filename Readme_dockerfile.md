# Ollama CPU (UBI9) - Build and Local Helm Deployment

## Overview

This document describes the work completed to:

- Create a **CPU-only `Dockerfile.cpu`** based on the upstream Ollama Dockerfile.
- Migrate the runtime image to **Red Hat UBI9** while preserving the original build logic.
- Build and validate the CPU image locally using Podman.
- Deploy the image to a local Kind Kubernetes cluster using Helm.
- Verify the deployment and Ollama API.

---

# 1. Dockerfile.cpu Changes

## Objective

Create a CPU-only Dockerfile with minimal changes from the upstream project while migrating to UBI9.

## Changes Performed

- Created a dedicated `Dockerfile.cpu`.
- Preserved the original multi-stage build structure.
- Removed GPU-specific stages:
  - CUDA
  - ROCm
  - Vulkan
  - MLX
- Kept only the CPU build pipeline.
- Migrated runtime image to **Red Hat UBI9**.
- Installed only the required runtime dependencies.
- Embedded the Ollama version during the Go build using linker flags.
- Preserved upstream build logic as much as possible.

---

## Build Image

```bash
podman build \
  -f Dockerfile.cpu \
  --platform linux/amd64 \
  --build-arg VERSION=v0.32.0 \
  -t localhost/ollama:v0.32.0 .
```

Verify:

```bash
podman images
```

Expected:

```
localhost/ollama   v0.32.0
```

---

# 2. Validate the Image Locally

Run the container:

```bash
podman run -d \
  --name ollama-v032 \
  -p 11434:11434 \
  localhost/ollama:v0.32.0
```

Verify:

```bash
curl http://localhost:11434/api/version
```

Expected:

```json
{"version":"v0.32.0"}
```

---

# 3. Create Local Kubernetes Cluster

Create Kind cluster:

```bash
kind create cluster --name ollama
```

Verify:

```bash
kubectl get nodes
```

Expected:

```
NAME                     STATUS   ROLES           VERSION
ollama-control-plane     Ready    control-plane
```

---

# 4. Load Local Image into Kind

Export image:

```bash
podman save -o ollama-v0.32.0.tar localhost/ollama:v0.32.0
```

Load into Kind:

```bash
kind load image-archive ollama-v0.32.0.tar --name ollama
```

---

# 5. Create Helm Chart

Generate a Helm chart:

```bash
helm create ollama
```

---

# 6. Configure Helm Values

Updated `values.yaml` with the following changes.

## Image

```yaml
image:
  repository: localhost/ollama
  tag: "v0.32.0"
  pullPolicy: Never
```

## Service

```yaml
service:
  type: ClusterIP
  port: 11434
```

## Health Probes

```yaml
livenessProbe:
  httpGet:
    path: /api/version
    port: http

readinessProbe:
  httpGet:
    path: /api/version
    port: http
```

## Resources

```yaml
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
```

---

# 7. Validate Helm Chart

Lint:

```bash
helm lint .
```

Render manifests:

```bash
helm template ollama .
```

---

# 8. Deploy Using Helm

Install:

```bash
helm install ollama .
```

Verify release:

```bash
helm list
```

---

# 9. Verify Kubernetes Resources

Pods:

```bash
kubectl get pods
```

Services:

```bash
kubectl get svc
```

Expected service:

```
ollama   ClusterIP   11434/TCP
```

---

# 10. Access the Application

Port-forward:

```bash
kubectl port-forward svc/ollama 11434:11434
```

---

# 11. Validate Deployment

Verify version:

```bash
curl http://localhost:11434/api/version
```

Expected:

```json
{"version":"v0.32.0"}
```

Verify available models:

```bash
curl http://localhost:11434/api/tags
```

Expected:

```json
{"models":[]}
```

---

# Summary

Successfully completed the following:

- Created a dedicated CPU-only `Dockerfile.cpu`.
- Migrated the runtime image to Red Hat UBI9.
- Preserved the upstream multi-stage build logic with minimal modifications.
- Embedded the application version during the build.
- Built and validated the image locally using Podman.
- Created a local Kind Kubernetes cluster.
- Loaded the local image into the Kind cluster.
- Created and configured a Helm chart for deployment.
- Successfully deployed the application using Helm.
- Verified the deployment using Kubernetes and the Ollama REST API.

---

# Notes

- The deployment uses the locally built image (`localhost/ollama:v0.32.0`).
- `image.pullPolicy` is set to `Never` since the image is loaded directly into the Kind cluster.
- Ollama listens on port **11434**.
- Health checks use the `/api/version` endpoint.
- This setup is intended for local development and validation before publishing the image to a container registry.