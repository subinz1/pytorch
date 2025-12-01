# RHEL 9.6 Docker Image Build

This directory contains the Dockerfile and build scripts for creating RHEL 9.6-based Docker images for PyTorch CI.

## RHEL Subscription (Optional)

The build supports optional RHEL subscription registration for accessing additional repositories beyond UBI repos.

### Setup Subscription Credentials

If you need to use RHEL subscription:

1. **Create credential files in your home directory:**

```bash
mkdir -p ~/.rhsm
echo "YOUR_ORG_ID" > ~/.rhsm/rhsm-org
echo "YOUR_ACTIVATION_KEY" > ~/.rhsm/rhsm-activationkey
chmod 600 ~/.rhsm/rhsm-*
```

2. **The build script will automatically detect these files** and enable subscription during the Docker build.

### Custom Credential Locations

You can use custom paths by setting environment variables before running the build:

```bash
export RHSM_ORG_FILE=/path/to/your/org-file
export RHSM_KEY_FILE=/path/to/your/activation-key-file
```

### Without Subscription

If the credential files are not found, the build will proceed using only UBI (Universal Base Image) repositories, which provide most common packages without requiring a subscription.

## Building the Image

```bash
# Build with CUDA 12.8
./build.sh rhel9-builder:cuda12.8

# Build CPU-only version
./build.sh rhel9-builder:cpu
```

## Supported CUDA Versions

- CUDA 12.6
- CUDA 12.8
- CUDA 12.9
- CUDA 13.0
- CPU (no CUDA)

## Environment Variables

- `RHSM_ORG_FILE` - Path to file containing RHEL organization ID (default: `~/.rhsm/rhsm-org`)
- `RHSM_KEY_FILE` - Path to file containing RHEL activation key (default: `~/.rhsm/rhsm-activationkey`)
- `DOCKER_BUILDKIT` - Enable Docker BuildKit (automatically set to 1)
