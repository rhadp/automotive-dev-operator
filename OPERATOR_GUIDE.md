# Automotive Dev Operator - Complete Guide

## Overview

The Automotive Dev Operator is a Kubernetes operator designed to automate the building, management, and distribution of automotive operating system images. It provides a cloud-native solution for building automotive OS images using the automotive-image-builder tool, with an integrated web UI and CLI for easy management.

## What Does This Operator Do?

The Automotive Dev Operator manages three primary custom resources:

1. **ImageBuild**: Triggers and manages automotive OS image builds using Tekton pipelines
2. **Image**: Catalogs and tracks built images in a registry
3. **OperatorConfig**: Configures the operator's components (WebUI, Build API, and OS build tasks)

### Key Features

- Automated OS image building using Tekton pipelines
- Web UI for visual management of builds and images
- RESTful Build API for programmatic access
- CLI tool (caib) for command-line interactions
- Support for multiple architectures (x86_64, aarch64, etc.)
- Artifact serving with automatic expiry
- Registry integration for publishing images
- OpenShift and vanilla Kubernetes support

## Architecture

The operator consists of several components:

### Core Components

1. **Controller Manager**: Reconciles the three custom resources (ImageBuild, Image, OperatorConfig)
2. **Build API Server**: REST API for managing builds and images
3. **Web UI**: React-based interface for interactive management
4. **Tekton Tasks**: Containerized build tasks that execute the actual image builds

### Build Workflow

```
User creates ImageBuild CR
    ↓
Operator creates PVC for workspace
    ↓
Operator creates Tekton TaskRun
    ↓
Build task runs automotive-image-builder in a pod
    ↓
Artifact is stored in PVC
    ↓
(Optional) Artifact is served via HTTP
    ↓
(Optional) Artifact is published to registry
    ↓
(Optional) Image CR is created to catalog the build
```

## Installation

### Prerequisites

- Kubernetes v1.11.3+ cluster
- kubectl configured to access the cluster
- Tekton Pipelines installed on the cluster
- (Optional) OpenShift for enhanced features like Routes
- go version v1.22.0+ (for building from source)
- docker version 17.03+ (for building from source)

### Option 1: Install from GitHub Release (Recommended)

Download and apply the pinned installer manifest:

```bash
TAG=v0.0.10  # Replace with desired version
curl -L -o install-$TAG.yaml \
  https://github.com/centos-automotive-suite/automotive-dev-operator/releases/download/$TAG/install-$TAG.yaml

kubectl apply -f install-$TAG.yaml
```

### Option 2: Build and Deploy from Source

1. Clone the repository:
```bash
git clone https://github.com/centos-automotive-suite/automotive-dev-operator
cd automotive-dev-operator
```

2. Build and push the operator image:
```bash
export IMG=<your-registry>/automotive-dev-operator:tag
make docker-build docker-push IMG=$IMG
```

3. Install CRDs:
```bash
make install
```

4. Deploy the operator:
```bash
make deploy IMG=$IMG
```

### Verify Installation

Check that all deployments are running:

```bash
kubectl -n automotive-dev-operator-system get deployments
kubectl -n automotive-dev-operator-system rollout status deploy/ado-controller-manager
```

## Configuration

### OperatorConfig Custom Resource

After installing the operator, create an `OperatorConfig` to configure its components:

```yaml
apiVersion: automotive.sdv.cloud.redhat.com/v1alpha1
kind: OperatorConfig
metadata:
  name: automotive-dev
  namespace: automotive-dev-operator-system
spec:
  # Enable/disable the Web UI
  webUI: true

  # OS Builds configuration
  osBuilds:
    # Enable Tekton tasks for OS builds
    enabled: true

    # Size for persistent volume claims (default: 8Gi)
    pvcSize: "8Gi"

    # How long to serve artifacts before cleanup (default: 24 hours)
    serveExpiryHours: 24

    # Optional: Use memory-backed volumes for faster builds
    # useMemoryVolumes: false
    # memoryVolumeSize: "2Gi"

    # Optional: Runtime class for build pods (e.g., kata containers)
    # runtimeClassName: "kata"
```

Apply the configuration:

```bash
kubectl apply -f config/samples/automotive_v1_operatorconfig.yaml
```

Check the operator components:

```bash
kubectl -n automotive-dev-operator-system get deploy
# Expected: ado-controller-manager, ado-build-api, ado-webui
```

## Usage

### Building an OS Image

#### Step 1: Create a Manifest ConfigMap

Create a ConfigMap containing your image build manifest (MPP format):

```bash
kubectl create configmap mpp --from-file=manifest.mpp
```

#### Step 2: Create an ImageBuild Custom Resource

```yaml
apiVersion: automotive.sdv.cloud.redhat.com/v1alpha1
kind: ImageBuild
metadata:
  name: my-automotive-build
spec:
  # Target architecture
  architecture: "amd64"  # or "arm64", "aarch64", "x86_64"

  # Distribution to build
  distro: "cs9"  # CentOS Stream 9

  # Build target
  target: "qemu"

  # Build mode
  mode: "image"

  # Export format
  exportFormat: "qcow2"  # or "image" for raw disk image

  # Automotive Image Builder container image
  automotiveImageBuilder: "quay.io/centos-sig-automotive/automotive-image-builder:1.0.0"

  # ConfigMap containing the manifest
  manifestConfigMap: mpp

  # Serve the artifact via HTTP after build
  serveArtifact: true

  # Expose a route to the artifact (OpenShift only)
  exposeRoute: true

  # Hours before artifact expires (default: 24)
  serveExpiryHours: 48

  # Optional: Storage class for the workspace PVC
  # storageClass: "lvms-vg1"

  # Optional: Runtime class for the build pod
  # runtimeClassName: "kata"

  # Optional: Compression for artifacts (default: gzip)
  compression: "gzip"  # or "lz4"

  # Optional: Environment variables secret for private registries
  # envSecretRef: "registry-credentials"

  # Optional: Publish to a registry after build
  # publishers:
  #   registry:
  #     repositoryUrl: "quay.io/myorg/automotive-image:latest"
  #     secret: "registry-credentials"
```

Apply the ImageBuild:

```bash
kubectl apply -f imagebuild.yaml
```

#### Step 3: Monitor the Build

Check build status:

```bash
kubectl get imagebuild my-automotive-build
kubectl describe imagebuild my-automotive-build
```

Watch the build progress:

```bash
# View the TaskRun
kubectl get taskruns -l automotive.sdv.cloud.redhat.com/imagebuild-name=my-automotive-build

# View build logs
kubectl logs -f <taskrun-pod-name>
```

#### Step 4: Access the Built Artifact

Once the build completes, if `serveArtifact: true` was set:

```bash
# Get the artifact URL (on OpenShift with exposeRoute: true)
kubectl get imagebuild my-automotive-build -o jsonpath='{.status.artifactURL}'

# Download the artifact
curl -O <artifact-url>/<filename>
```

### Managing Images (Image Catalog)

After building an image, you can catalog it using the `Image` custom resource:

```yaml
apiVersion: automotive.sdv.cloud.redhat.com/v1alpha1
kind: Image
metadata:
  name: autosd-qemu-v1
spec:
  distro: "autosd"
  target: "qemu"
  architecture: "x86_64"
  exportFormat: "qcow2"
  mode: "image"
  version: "1.0.0"
  description: "AutoSD QEMU image for development"

  tags:
    - "autosd"
    - "qemu"
    - "development"

  # Location where the image is stored
  location:
    type: "registry"
    registry:
      url: "quay.io/myorg/autosd-qemu:1.0.0"
      digest: "sha256:abcd1234..."
      secretRef: "registry-credentials"

  # Size information
  size:
    compressedBytes: 1073741824    # 1GB
    uncompressedBytes: 5368709120  # 5GB
    virtualBytes: 21474836480      # 20GB (virtual disk size)

  # Metadata
  metadata:
    createdBy: "automotive-dev-operator"
    buildDate: "2025-01-15T10:30:00Z"
    sourceImageBuild: "my-automotive-build"
    labels:
      "automotive.io/distro": "autosd"
      "automotive.io/arch": "x86_64"
    annotations:
      "automotive.io/purpose": "Development testing"
```

The Image controller will:
- Verify the image location is accessible
- Track image availability
- Maintain access statistics
- Periodically re-verify accessibility

Check image status:

```bash
kubectl get images
kubectl describe image autosd-qemu-v1
```

## Using the CLI (caib)

### Installation

Download the CLI from a release:

```bash
TAG=v0.0.11
ARCH=linux-amd64  # or linux-arm64

curl -L -o caib-$TAG-$ARCH \
  https://github.com/centos-automotive-suite/automotive-dev-operator/releases/download/$TAG/caib-$TAG-$ARCH

sudo install -m 0755 caib-$TAG-$ARCH /usr/local/bin/caib

# Verify installation
caib --version
```

### Configuration

Point the CLI to your Build API:

```bash
export CAIB_SERVER="https://build-api.YOUR_DOMAIN"
```

Or pass `--server` with each command.

### CLI Usage Examples

See `cmd/caib/README.md` for detailed CLI documentation.

## Using the Web UI

1. Get the Web UI URL:

```bash
# On OpenShift
kubectl get route -n automotive-dev-operator-system ado-webui -o jsonpath='{.spec.host}'

# On Kubernetes with Ingress
kubectl get ingress -n automotive-dev-operator-system ado-webui -o jsonpath='{.spec.rules[0].host}'
```

2. Open the URL in your browser

3. The Web UI provides:
   - Visual dashboard of all builds
   - Image catalog browser
   - Build creation wizard
   - Real-time build status updates
   - Artifact download links

## Advanced Configuration

### Using Private Registries

Create a secret with registry credentials:

```bash
kubectl create secret generic registry-credentials \
  --from-literal=REGISTRY_USERNAME=myuser \
  --from-literal=REGISTRY_PASSWORD=mypassword
```

Reference it in your ImageBuild:

```yaml
spec:
  envSecretRef: "registry-credentials"
  publishers:
    registry:
      repositoryUrl: "private.registry.io/myorg/image:tag"
      secret: "registry-credentials"
```

### Using Memory-Backed Volumes

For faster builds, configure memory-backed volumes in OperatorConfig:

```yaml
spec:
  osBuilds:
    useMemoryVolumes: true
    memoryVolumeSize: "2Gi"
```

### Using Alternative Runtimes

Configure Kata Containers or other runtimes:

```yaml
spec:
  osBuilds:
    runtimeClassName: "kata"
```

Or per-build:

```yaml
spec:
  runtimeClassName: "kata"
```

### File Upload Server

For builds that reference local files in the manifest:

```yaml
spec:
  inputFilesServer: true
```

This creates an upload pod where you can copy files before the build starts:

```bash
# Copy files to the upload pod
kubectl cp myfiles/ <upload-pod-name>:/workspace/shared/

# Mark uploads complete
kubectl annotate imagebuild my-build \
  automotive.sdv.cloud.redhat.com/uploads-complete=true
```

## Custom Resource Definitions Reference

### ImageBuild

**Spec Fields:**
- `architecture`: Target architecture (required)
- `distro`: Distribution name (required)
- `target`: Build target, e.g., "qemu" (required)
- `mode`: Build mode, "image" or "package" (required)
- `exportFormat`: Output format, e.g., "qcow2", "image" (required)
- `automotiveImageBuilder`: Container image for the builder (required)
- `manifestConfigMap`: ConfigMap name containing the manifest (required)
- `compression`: Compression algorithm, "gzip" or "lz4" (default: gzip)
- `serveArtifact`: Whether to serve the artifact (default: false)
- `exposeRoute`: Whether to create a Route (OpenShift) (default: false)
- `serveExpiryHours`: Hours before artifact cleanup (default: 24)
- `storageClass`: Storage class for workspace PVC (optional)
- `runtimeClassName`: Runtime class for build pod (optional)
- `envSecretRef`: Secret with environment variables (optional)
- `inputFilesServer`: Enable file upload server (default: false)
- `publishers`: Registry publishing configuration (optional)

**Status Fields:**
- `phase`: Current phase (Building, Completed, Failed, Uploading)
- `message`: Human-readable status message
- `taskRunName`: Name of the associated Tekton TaskRun
- `pvcName`: Name of the workspace PVC
- `artifactFileName`: Name of the built artifact file
- `artifactPath`: Path to the artifact in the PVC
- `artifactURL`: Public URL for downloading the artifact
- `startTime`: When the build started
- `completionTime`: When the build finished

### Image

**Spec Fields:**
- `distro`: Distribution name (required)
- `target`: Build target (required)
- `architecture`: Target architecture (required)
- `exportFormat`: Output format (required)
- `location`: Storage location configuration (required)
  - `type`: Location type, currently only "registry" supported
  - `registry`: Registry configuration
    - `url`: Full image URL (required)
    - `digest`: Content digest (optional)
    - `secretRef`: Credentials secret (optional)
- `mode`: Build mode (optional)
- `version`: Image version (optional)
- `description`: Human-readable description (optional)
- `tags`: List of tags for categorization (optional)
- `size`: Size information (optional)
  - `compressedBytes`: Compressed size
  - `uncompressedBytes`: Uncompressed size
  - `virtualBytes`: Virtual disk size
- `metadata`: Additional metadata (optional)
  - `createdBy`: Creator identifier
  - `buildDate`: Build timestamp
  - `sourceImageBuild`: Source ImageBuild name
  - `labels`: Key-value labels
  - `annotations`: Key-value annotations

**Status Fields:**
- `phase`: Current phase (Available, Unavailable, Verifying)
- `message`: Status message
- `conditions`: Standard Kubernetes conditions
- `lastVerified`: Last verification timestamp
- `lastAccessed`: Last access timestamp
- `accessCount`: Number of accesses

### OperatorConfig

**Spec Fields:**
- `webUI`: Enable Web UI (default: true)
- `osBuilds`: OS builds configuration (optional)
  - `enabled`: Enable Tekton tasks (default: true)
  - `pvcSize`: PVC size for builds (default: "8Gi")
  - `serveExpiryHours`: Artifact expiry in hours (default: 24)
  - `useMemoryVolumes`: Use memory-backed volumes (default: false)
  - `memoryVolumeSize`: Memory volume size (required if useMemoryVolumes is true)
  - `runtimeClassName`: Runtime class for build pods (optional)

**Status Fields:**
- `phase`: Current phase (Ready, Reconciling, Failed)
- `message`: Status message
- `webUIDeployed`: Whether Web UI is deployed
- `osBuildsDeployed`: Whether OS builds tasks are deployed

## Troubleshooting

### Build Fails

1. Check the TaskRun logs:
```bash
kubectl logs -f <taskrun-pod-name>
```

2. Check ImageBuild status:
```bash
kubectl describe imagebuild <name>
```

3. Verify the manifest ConfigMap exists:
```bash
kubectl get configmap <manifest-configmap-name>
```

### Web UI Not Accessible

1. Check deployments:
```bash
kubectl get deploy -n automotive-dev-operator-system
```

2. Check Routes (OpenShift) or Ingress (Kubernetes):
```bash
kubectl get route -n automotive-dev-operator-system
kubectl get ingress -n automotive-dev-operator-system
```

3. Check logs:
```bash
kubectl logs -n automotive-dev-operator-system deploy/ado-webui
kubectl logs -n automotive-dev-operator-system deploy/ado-build-api
```

### PVC Issues

Check PVC status:
```bash
kubectl get pvc
kubectl describe pvc <pvc-name>
```

Ensure your cluster has a default storage class or specify `storageClass` in ImageBuild spec.

### Tekton Not Installed

Install Tekton Pipelines:
```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

## Uninstallation

### Delete Sample Resources

```bash
kubectl delete -k config/samples/
```

### Uninstall CRDs

```bash
make uninstall
```

### Undeploy Operator

```bash
make undeploy
```

Or if installed from release:

```bash
kubectl delete -f install-$TAG.yaml
```

## Development

See `DEVELOPMENT.md` for instructions on running the build server and UI locally for development.

## Contributing

Contributions are welcome! Please submit issues and pull requests to the GitHub repository.

## License

Licensed under the Apache License, Version 2.0. See LICENSE file for details.

## Additional Resources

- [Kubebuilder Documentation](https://book.kubebuilder.io/)
- [Tekton Documentation](https://tekton.dev/docs/)
- [Automotive Image Builder](https://github.com/centos-automotive-suite/automotive-image-builder)
- [Project Repository](https://github.com/centos-automotive-suite/automotive-dev-operator)
