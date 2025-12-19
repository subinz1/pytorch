# RHEL 9.6 PyTorch CI/CD Changes Documentation

This document details all changes made to enable PyTorch builds and tests on RHEL 9.6 self-hosted runners.

## Table of Contents
1. [Overview](#overview)
2. [Infrastructure Changes](#infrastructure-changes)
3. [Docker Image Changes](#docker-image-changes)
4. [Workflow Changes](#workflow-changes)
5. [Build Script Changes](#build-script-changes)
6. [Key Issues and Solutions](#key-issues-and-solutions)
7. [Testing](#testing)

---

## Overview

**Goal**: Enable PyTorch CI/CD pipeline to run on RHEL 9.6 self-hosted runners without AWS dependencies.

**Key Requirements**:
- Use self-hosted runners with label `test-runner-git-109-vpc`
- Run builds in RHEL 9.6 containers using Podman
- Remove all AWS dependencies (S3, ECR)
- Use local storage for Docker images and artifacts
- Support CUDA 12.8 with NVIDIA H200 GPUs

---

## Infrastructure Changes

### 1. Self-Hosted Runner Configuration

**Runner Label**: `test-runner-git-109-vpc`

**Key Characteristics**:
- RHEL 9 host system
- Podman for container runtime (emulating Docker CLI)
- NVIDIA H200 GPUs with driver 580.82.07
- Persistent storage at `/mnt/podman_storage/vpcuser/`
- No AWS credentials or access

### 2. Storage Configuration

**Persistent Storage Locations**:
```bash
/mnt/podman_storage/vpcuser/docker-images/  # Docker image tar files
/mnt/podman_storage/vpcuser/github-tester/  # Workspace
```

**Podman Configuration**:
```bash
# ~/.config/containers/registries.conf
short-name-mode = "permissive"
```

**Temporary Directory**:
```bash
export TMPDIR=$HOME/tmp
```

---

## Docker Image Changes

### File: `.ci/docker/rhel9/Dockerfile`

#### Base Image Version
```dockerfile
# Changed from 9.4 to 9.6
ARG RHEL_OS_VERSION=9.6
FROM docker.io/redhat/ubi9:${RHEL_OS_VERSION}
```

#### Simplified to Single-Stage Build
- Removed multi-stage build complexity
- All dependencies installed in single stage
- CUDA 12.8 support only (removed CPU variant)

#### Conda Environment Setup
```dockerfile
ARG CONDA_ENV="pytorch_build"

# Install Miniforge (lighter than Anaconda)
RUN curl -fsSL -o ~/miniforge.sh \
    https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh && \
    bash ~/miniforge.sh -b -p /opt/conda && \
    rm ~/miniforge.sh

# Create pytorch_build environment
RUN . /opt/conda/etc/profile.d/conda.sh && \
    conda create -y -n ${CONDA_ENV} python=${PY_VER}

# Install all PyTorch CI requirements
RUN . /opt/conda/etc/profile.d/conda.sh && \
    conda activate ${CONDA_ENV} && \
    conda install -y sccache && \
    curl -fsSL https://raw.githubusercontent.com/pytorch/pytorch/main/.ci/docker/requirements-ci.txt \
         -o /tmp/requirements-ci.txt && \
    pip install -r /tmp/requirements-ci.txt && \
    rm /tmp/requirements-ci.txt
```

#### PATH Configuration
```dockerfile
# Ensure conda environment is in PATH
ENV PATH=/usr/local/cuda/bin:/opt/conda/envs/${CONDA_ENV}/bin:/opt/conda/bin:$PATH
```

#### Auto-activate Conda Environment
```dockerfile
RUN echo "conda activate ${CONDA_ENV}" >> ~/.bashrc
```

#### No Jenkins User
- RHEL images run as root (no jenkins user)
- Removed all jenkins user setup and sudo commands

---

## Workflow Changes

### 1. New Workflow: `.github/workflows/rhel-build-test.yml`

**Purpose**: Main orchestration workflow for RHEL builds and tests

**Key Features**:
- Triggers on pushes to `runner_test` branch
- Three jobs: image build → PyTorch build → PyTorch test
- Uses self-hosted runner: `test-runner-git-109-vpc`
- No AWS permissions or dependencies

**Structure**:
```yaml
jobs:
  build-docker-image:
    uses: ./.github/workflows/build-rhel9-images.yml

  rhel-9_6-py3_12-gcc11-build:
    needs: build-docker-image
    uses: ./.github/workflows/_linux-build.yml
    with:
      runner: "test-runner-git-109-vpc"
      build-environment: linux-rhel-9.6-py3.12-gcc11
      docker-image-name: ci-image:pytorch-rhel-9.6-py3.12-gcc11

  rhel-9_6-py3_12-gcc11-test:
    needs: rhel-9_6-py3_12-gcc11-build
    uses: ./.github/workflows/_linux-test.yml
```

### 2. Image Build Workflow: `.github/workflows/build-rhel9-images.yml`

**Changes**:

#### Image Detection Fix
```yaml
# Find the tmp.* tagged image created by build.sh
# Podman prefixes images with localhost/
NEW_IMAGE=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep 'tmp\.' | head -1)
if [ -z "$NEW_IMAGE" ]; then
  echo "Error: Could not find tmp.* image after build"
  docker images
  exit 1
fi
```

#### Image Export to Persistent Storage
```yaml
# Export the image to tar file
docker save -o /mnt/podman_storage/vpcuser/docker-images/rhel9-py3.12-gcc11.tar \
  localhost/ci-image:pytorch-rhel-9.6-py3.12-gcc11

# Create a .ready marker file
touch /mnt/podman_storage/vpcuser/docker-images/rhel9-py3.12-gcc11.tar.ready
```

### 3. Build Workflow: `.github/workflows/_linux-build.yml`

**Changes**:

#### Skip AWS Authentication
```yaml
- name: configure aws credentials
  if: inputs.build-environment != 'linux-rhel-9.6-py3.12-gcc11'
```

#### Import Docker Image from Local Storage
```yaml
- name: Build
  run: |
    if [[ ${BUILD_ENVIRONMENT} == *"rhel"* ]]; then
      IMAGE_TAR_PATH="/mnt/podman_storage/vpcuser/docker-images/rhel9-py3.12-gcc11.tar"
      READY_MARKER="${IMAGE_TAR_PATH}.ready"

      # Remove any existing old images to avoid conflicts
      docker rmi localhost/ci-image:pytorch-rhel-9.6-py3.12-gcc11 2>/dev/null || true
      docker rmi ci-image:pytorch-rhel-9.6-py3.12-gcc11 2>/dev/null || true

      # Wait for the .ready marker file
      for i in {1..12}; do
        if [ -f "${READY_MARKER}" ]; then
          echo "Ready marker found, proceeding with import..."
          break
        fi
        echo "Waiting for Docker image to be ready... (attempt $i/12)"
        sleep 5
      done

      # Import the image
      if [ -f "${IMAGE_TAR_PATH}" ]; then
        docker load -i "${IMAGE_TAR_PATH}"
      fi
    fi
```

#### Activate Conda Environment in Docker Exec
```yaml
# Use bash login shell and activate conda environment
if [[ ${BUILD_ENVIRONMENT} == *"rhel"* ]]; then
  docker exec -t "${container_name}" bash -l -c \
    '. /opt/conda/etc/profile.d/conda.sh && conda activate pytorch_build && .ci/pytorch/build.sh'
else
  docker exec -t "${container_name}" sh -c '.ci/pytorch/build.sh'
fi
```

#### Limit MAX_JOBS for RHEL
```yaml
if [[ ${BUILD_ENVIRONMENT} == *"rhel"* ]]; then
  # Limit to 4 jobs to avoid "Resource temporarily unavailable" errors
  MAX_JOBS_VALUE=4
else
  MAX_JOBS_VALUE="$(nproc --ignore=2)"
fi
```

#### Skip Setup Linux Action
```yaml
- name: Setup Linux
  if: inputs.build-environment != 'linux-rhel-9.6-py3.12-gcc11'
```

#### Skip Teardown Linux or Add skip-wait-ssh
```yaml
- name: Teardown Linux
  uses: pytorch/test-infra/.github/actions/teardown-linux@main
  if: always() && inputs.build-environment != 'linux-rhel-9.6-py3.12-gcc11'
  with:
    skip-wait-ssh: "true"
```

#### Skip AWS S3 Uploads
```yaml
- name: Store PyTorch Build Artifacts on S3
  if: inputs.build-environment != 'linux-rhel-9.6-py3.12-gcc11'
```

#### Use GitHub Actions Artifacts Instead
```yaml
- name: Upload build artifacts
  uses: actions/upload-artifact@v4
  if: inputs.build-environment == 'linux-rhel-9.6-py3.12-gcc11'
```

### 4. Test Workflow: `.github/workflows/_linux-test.yml`

**Changes**:

#### Skip AWS Authentication
```yaml
- name: configure aws credentials
  if: !contains(inputs.build-environment, 'rhel')
```

#### Import Docker Image from Local Storage
```yaml
- name: Import RHEL Docker image from local storage
  if: contains(inputs.build-environment, 'rhel')
  shell: bash
  run: |
    IMAGE_TAR_PATH="/mnt/podman_storage/vpcuser/docker-images/rhel9-py3.12-gcc11.tar"
    READY_MARKER="${IMAGE_TAR_PATH}.ready"

    # Wait for the .ready marker file (max 60 seconds)
    for i in {1..12}; do
      if [ -f "${READY_MARKER}" ]; then
        echo "Ready marker found, proceeding with import..."
        break
      fi
      sleep 5
    done

    if [ -f "${IMAGE_TAR_PATH}" ]; then
      docker load -i "${IMAGE_TAR_PATH}"
    fi
```

#### Skip NVIDIA Driver Installation or Continue on Error
```yaml
- name: Install nvidia driver, nvidia-docker runtime, set GPU_FLAG
  if: ${{ !contains(matrix.runner, 'b200') }}
  continue-on-error: true
```

**Reason**: NVIDIA driver is already installed on RHEL runners. The setup-nvidia action fails with "Unknown distribution centos9" but the driver is present and working.

#### Skip Teardown SSH Wait
```yaml
- name: Teardown Linux
  uses: pytorch/test-infra/.github/actions/teardown-linux@main
  with:
    skip-wait-ssh: "true"
```

**Reason**: Eliminates 2-hour wait for SSH sessions to drain.

#### Use GitHub Actions Artifacts
```yaml
- name: Download build artifacts
  uses: actions/download-artifact@v4
  if: contains(inputs.build-environment, 'rhel')
```

---

## Build Script Changes

### File: `.ci/pytorch/common-build.sh`

**Sccache Configuration for Local Disk Cache**:

```bash
if which sccache > /dev/null; then
    # For RHEL builds, use local disk cache instead of S3
    if [[ "$BUILD_ENVIRONMENT" == *rhel* ]]; then
      echo "Configuring sccache for local disk cache (RHEL build)"
      export SCCACHE_DIR=/tmp/sccache
      mkdir -p /tmp/sccache
      unset SCCACHE_BUCKET
      unset SCCACHE_REGION
    fi
fi
```

### File: `.ci/pytorch/build.sh`

**Skip Jenkins User Operations for RHEL**:

```bash
# RHEL images don't have jenkins user, skip chown
if [[ "$BUILD_ENVIRONMENT" != *"rhel"* ]]; then
    sudo chown -R jenkins /var/lib/jenkins/workspace
fi
```

**Skip Cleanup for RHEL**:

```bash
cleanup_workspace() {
    # RHEL images don't have jenkins user, skip cleanup
    if [[ "$BUILD_ENVIRONMENT" == *"rhel"* ]]; then
      return
    fi
    sudo chown -R "$WORKSPACE_ORIGINAL_OWNER_ID" /var/lib/jenkins/workspace
}
```

---

## Key Issues and Solutions

### Issue 1: Conda Environment Not Found
**Error**: `bash: line 1: /opt/conda/etc/profile.d/conda.sh: No such file or directory`

**Root Cause**: Docker exec wasn't activating the conda environment.

**Solution**:
```bash
docker exec -t "${container_name}" bash -l -c \
  '. /opt/conda/etc/profile.d/conda.sh && conda activate pytorch_build && .ci/pytorch/build.sh'
```

### Issue 2: Old Cached Docker Images
**Error**: Images from 13 months ago being used instead of fresh builds.

**Root Cause**:
1. Base image version was 9.4 instead of 9.6
2. Docker image detection logic failed
3. Old cached images not removed before import

**Solution**:
1. Changed Dockerfile to use RHEL 9.6: `ARG RHEL_OS_VERSION=9.6`
2. Fixed image detection to match `localhost/tmp.*` pattern
3. Remove old images before importing: `docker rmi localhost/ci-image:* 2>/dev/null || true`

### Issue 3: Resource Exhaustion
**Error**: `OS can't spawn worker thread: Resource temporarily unavailable (os error 11)`

**Root Cause**: Too many parallel compilation jobs exhausted system thread limits.

**Solution**: Limit MAX_JOBS to 4 for RHEL builds.

### Issue 4: Sccache AWS Errors
**Error**: `sccache: error: Operation not permitted (os error 1)`

**Root Cause**: Sccache trying to access S3 without credentials.

**Solution**: Configure sccache to use local disk cache:
```bash
export SCCACHE_DIR=/tmp/sccache
unset SCCACHE_BUCKET
unset SCCACHE_REGION
```

### Issue 5: NVIDIA Driver Installation Failure
**Error**: `ERROR: Unknown distribution centos9`

**Root Cause**: The setup-nvidia action doesn't recognize RHEL 9 (which identifies as centos9).

**Solution**: Allow the step to continue on error since the driver is already installed:
```yaml
continue-on-error: true
```

### Issue 6: 2-Hour Teardown Delays
**Error**: Teardown step waiting 2 hours for SSH sessions.

**Root Cause**: Default teardown-linux action waits for SSH sessions to drain.

**Solution**: Skip SSH wait:
```yaml
with:
  skip-wait-ssh: "true"
```

### Issue 7: Docker Image Detection Failed
**Error**: NEW_IMAGE variable empty after build.

**Root Cause**: Grep pattern `^tmp\.` didn't match `localhost/tmp.*` images created by podman.

**Solution**: Simplified grep to match both patterns:
```bash
NEW_IMAGE=$(docker images --format '{{.Repository}}:{{.Tag}}' | grep 'tmp\.' | head -1)
```

### Issue 8: Shared Memory Size Conflict
**Error**: `Error: invalid config provided: cannot set shmsize when running in the {host } IPC Namespace`

**Root Cause**: The test workflow was using both `--ipc=host` and `--shm-size=2g` options, which are incompatible in Docker/Podman. When `--ipc=host` is set, shared memory is managed by the host, not the container.

**Solution**: Set `SHM_OPTS` to empty for RHEL builds in `_linux-test.yml`:
```yaml
elif [[ ${BUILD_ENVIRONMENT} == *"rhel"* ]]; then
  # RHEL images don't have jenkins user setup, run as root
  # Cannot use --shm-size with --ipc=host, so leave SHM_OPTS empty
  SHM_OPTS=
  JENKINS_USER=
  DOCKER_SHELL_CMD=
  USED_IMAGE="${DOCKER_IMAGE}"
```

### Issue 9: Docker/Podman Service Detection Failed
**Error**: `Failed to start docker.service: Interactive authentication required.`

**Root Cause**: The setup-linux action was checking for podman.service (systemd service), which doesn't exist for rootless podman. The check fell through to try starting docker daemon without sudo.

**Solution**: Reorganized detection logic in `.github/actions/setup-linux/action.yml`:
```bash
# First check if docker command is already available (covers rootless podman)
if command -v docker &> /dev/null && docker ps &> /dev/null; then
    echo "Docker/Podman is already running and accessible";
# Check if podman service exists (RHEL/Fedora systems with systemd podman)
elif systemctl list-unit-files | grep -q podman.service; then
    if systemctl is-active --quiet podman; then
        echo "Podman service is running...";
    else
        echo "Starting podman service..." && systemctl --user start podman;
    fi
# Otherwise check for docker service
elif systemctl is-active --quiet docker; then
    echo "Docker daemon is running...";
else
    echo "Starting docker daemon..." && sudo systemctl start docker;
fi
```

### Issue 10: S3 Upload Without AWS Credentials
**Error**: `Missing credentials in config, if using AWS_CONFIG_FILE, set AWS_SDK_LOAD_CONFIG=1`

**Root Cause**: The upload-test-artifacts action was uploading to S3 by default, which requires AWS credentials not available on self-hosted RHEL runners.

**Solution**: Added `use-gha: "1"` parameter to test workflow in `rhel-build-test.yml`:
```yaml
rhel-9_6-py3_12-gcc11-test:
  name: rhel-9.6-py3.12-gcc11
  uses: ./.github/workflows/_linux-test.yml
  needs: rhel-9_6-py3_12-gcc11-build
  with:
    build-environment: linux-rhel-9.6-py3.12-gcc11
    docker-image: ${{ needs.rhel-9_6-py3_12-gcc11-build.outputs.docker-image }}
    test-matrix: ${{ needs.rhel-9_6-py3_12-gcc11-build.outputs.test-matrix }}
    use-gha: "1"  # Use GitHub Actions artifacts instead of S3
  secrets: inherit
```

### Issue 11: Python Setup Failing on Self-Hosted Runner
**Error**: `The version '3.10' with architecture 'x64' was not found for this operating system.`

**Root Cause**: The upload-utilization-stats action tried to install Python 3.10 using setup-python, which doesn't work on self-hosted RHEL runners (they don't have pre-installed Python versions like GitHub-hosted runners).

**Solution**: Skip upload-utilization-stats step for RHEL in `_linux-test.yml`:
```yaml
- name: Upload utilization stats
  if: ${{ always() && steps.test.conclusion && steps.test.conclusion != 'skipped' && !inputs.disable-monitor && inputs.build-environment != 'linux-s390x-binary-manywheel' && !contains(inputs.build-environment, 'rhel') }}
  continue-on-error: true
  uses: ./.github/actions/upload-utilization-stats
```

### Issue 12: Workspace Permission Changes Failing in Tests
**Error**: `sudo chown -R 0 /var/lib/jenkins/workspace` failing with exit code 1

**Root Cause**: The `.ci/pytorch/test.sh` script was trying to change workspace ownership to jenkins user and back, but:
1. RHEL Docker images don't have a jenkins user (they run as root)
2. sudo chown operations fail in the container context
3. These operations are unnecessary since RHEL containers already run as root

**Solution**: Added RHEL check to skip workspace permission changes in `test.sh`:
```bash
# Do not change workspace permissions for ROCm, s390x, and RHEL CI jobs
# as it can leave workspace with bad permissions for cancelled jobs
# RHEL images don't have jenkins user and run as root, so skip chown operations
if [[ "$BUILD_ENVIRONMENT" != *rocm* && "$BUILD_ENVIRONMENT" != *s390x* && "$BUILD_ENVIRONMENT" != *rhel* && -d /var/lib/jenkins/workspace ]]; then
  # Workaround for dind-rootless userid mapping
  WORKSPACE_ORIGINAL_OWNER_ID=$(stat -c '%u' "/var/lib/jenkins/workspace")
  cleanup_workspace() {
    echo "sudo may print the following warning message that can be ignored. The chown command will still run."
    echo "    sudo: setrlimit(RLIMIT_STACK): Operation not permitted"
    echo "For more details refer to https://github.com/sudo-project/sudo/issues/42"
    sudo chown -R "$WORKSPACE_ORIGINAL_OWNER_ID" /var/lib/jenkins/workspace
  }
  trap_add cleanup_workspace EXIT
  sudo chown -R jenkins /var/lib/jenkins/workspace
  git config --global --add safe.directory /var/lib/jenkins/workspace
fi
```

### Issue 13: CUDA Kernel Architecture Mismatch
**Error**: `CUDA error: no kernel image is available for execution on the device`

**Root Cause**: PyTorch was built without specifying CUDA architecture for the H200 GPU (compute capability 9.0). The default CUDA arch list "5.2" is too old for Hopper architecture GPUs.

**Solution**: Added `cuda-arch-list: "9.0"` parameter to build workflow in `rhel-build-test.yml`:
```yaml
rhel-9_6-py3_12-gcc11-build:
  needs: build-docker-image
  name: rhel-9.6-py3.12-gcc11
  uses: ./.github/workflows/_linux-build.yml
  with:
    runner_prefix: ""
    runner: "test-runner-git-109-vpc"
    build-environment: linux-rhel-9.6-py3.12-gcc11
    docker-image-name: ci-image:pytorch-rhel-9.6-py3.12-gcc11
    cuda-arch-list: "9.0"  # H200 GPU requires compute capability 9.0
    test-matrix: |
      { include: [
        { config: "default", shard: 1, num_shards: 3, runner: "test-runner-git-109-vpc" },
        { config: "default", shard: 2, num_shards: 3, runner: "test-runner-git-109-vpc" },
        { config: "default", shard: 3, num_shards: 3, runner: "test-runner-git-109-vpc" },
      ]}
  secrets: inherit
```

### Issue 14: Triton Not Installed
**Error**: `torch._inductor.exc.TritonMissing: Cannot find a working triton installation`

**Root Cause**: The test `dynamo/test_aot_compile.py::TestAOTCompile::test_aot_compile_with_aoti` failed because Triton (a Python library for GPU programming) was not installed in the RHEL Docker image. Triton is required by PyTorch's inductor for certain optimizations and AOT compilation.

**Solution**: Added Triton installation to `.ci/docker/rhel9/Dockerfile`:
```dockerfile
RUN . /opt/conda/etc/profile.d/conda.sh && \
    conda activate ${CONDA_ENV} && \
    conda install -y sccache && \
    curl -fsSL https://raw.githubusercontent.com/pytorch/pytorch/main/.ci/docker/requirements-ci.txt -o /tmp/requirements-ci.txt && \
    pip install -r /tmp/requirements-ci.txt && \
    rm /tmp/requirements-ci.txt && \
    pip install triton
```

---

## Testing

### Test Workflow: `.github/workflows/test-rhel-runner.yml`

Simple workflow to verify runner connectivity:

```yaml
name: test-rhel-runner

on:
  workflow_dispatch:
  push:
    branches:
      - runner_test

jobs:
  Test-RHEL-Runner:
    runs-on: test-runner-git-109-vpc
    steps:
      - name: Verify runner
        run: |
          echo "RHEL runner is working!"
          uname -a
          cat /etc/os-release
```

### Manual Testing

1. **Trigger image build**:
   ```bash
   git push origin runner_test
   ```

2. **Monitor workflows**:
   - Navigate to Actions tab
   - Check "RHEL 9.6 Build and Test" workflow
   - Verify all three jobs complete successfully

3. **Verify artifacts**:
   - Check `/mnt/podman_storage/vpcuser/docker-images/rhel9-py3.12-gcc11.tar`
   - Verify `.ready` marker file exists
   - Check image timestamp matches recent build

---

## Summary of Changes by Category

### Dockerfile Changes
- Changed base image from RHEL 9.4 to 9.6
- Simplified from multi-stage to single-stage build
- Added Miniforge/conda setup with `pytorch_build` environment
- Installed all PyTorch CI requirements from upstream
- Added Triton installation for inductor support
- Removed jenkins user setup (run as root)
- Fixed PATH to include conda environment
- Added auto-activation of conda environment in bashrc

### Workflow Changes
- Created new `rhel-build-test.yml` orchestration workflow with:
  - `use-gha: "1"` parameter for GitHub Actions artifacts
  - `cuda-arch-list: "9.0"` for H200 GPU support
- Modified `build-rhel9-images.yml` for proper image tagging and export
- Modified `_linux-build.yml` to:
  - Skip AWS authentication
  - Import images from local storage
  - Activate conda environment in docker exec
  - Limit MAX_JOBS to 4
  - Skip Setup Linux action
  - Skip/modify Teardown Linux
  - Use GitHub Actions artifacts instead of S3
- Modified `_linux-test.yml` to:
  - Skip AWS authentication
  - Import images from local storage
  - Set SHM_OPTS to empty (avoid --shm-size with --ipc=host conflict)
  - Allow NVIDIA driver install errors
  - Skip teardown SSH wait
  - Skip upload-utilization-stats step (requires Python setup)
  - Skip pytest-cache-upload step (requires S3)
  - Use GitHub Actions artifacts via use-gha parameter
- Modified `.github/actions/setup-linux/action.yml` to:
  - Check if docker command works before trying to start services
  - Support rootless podman detection
  - Use sudo when starting docker daemon

### Build Script Changes
- Configure sccache for local disk cache in `common-build.sh`
- Skip jenkins user operations for RHEL in `build.sh`
- Skip workspace cleanup for RHEL in `build.sh`
- Skip workspace permission changes for RHEL in `test.sh`

### Infrastructure
- Self-hosted runner with label `test-runner-git-109-vpc`
- Podman as container runtime
- Persistent storage at `/mnt/podman_storage/vpcuser/`
- No AWS dependencies

---

## Maintenance Notes

### Updating the Base Image Version
To update to a newer RHEL version:
```dockerfile
# In .ci/docker/rhel9/Dockerfile
ARG RHEL_OS_VERSION=9.7  # Change version here
```

### Updating CUDA Version
```dockerfile
# In .ci/docker/rhel9/Dockerfile
ARG CUDA_VERSION=12-9    # Update CUDA version
ARG CUDNN_VERSION=12.9   # Update cuDNN version
```

Then update the workflow:
```yaml
# In .github/workflows/build-rhel9-images.yml
matrix:
  tag: [cuda12.9]  # Update tag
```

### Cleaning Up Old Images
```bash
# On the runner
rm -f /mnt/podman_storage/vpcuser/docker-images/*.tar
rm -f /mnt/podman_storage/vpcuser/docker-images/*.ready
podman system prune -af
```

---

## Branch Information

**Branch**: `runner_test`
**Based on**: PyTorch `main` branch
**Fork**: `subinz1/pytorch`
**Upstream**: `pytorch/pytorch`

To sync with upstream:
```bash
git fetch upstream
git merge upstream/main
git push origin runner_test
```

---

## Contact

For questions or issues related to RHEL builds, please check:
- Workflow runs: https://github.com/subinz1/pytorch/actions
- This documentation: `RHEL_CHANGES.md`
- Runner logs on the self-hosted machine
