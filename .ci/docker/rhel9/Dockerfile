# RHEL 9.6 Docker image for PyTorch CI
ARG CUDA_VERSION=12.8
ARG BASE_TARGET=cuda${CUDA_VERSION}
# This flag has to be set to true if there is an active subscription
ARG ENABLE_SUBSCRIPTION=false

FROM docker.io/redhat/ubi9:9.6 as base

# Pass build args to the base stage
ARG ENABLE_SUBSCRIPTION

ENV LC_ALL en_US.UTF-8
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US.UTF-8

# Register the system with the entitlement server if subscription is enabled
# Expects /run/secrets/rhsm-org and /run/secrets/rhsm-activationkey to be mounted
RUN --mount=type=secret,id=rhsm-org \
    --mount=type=secret,id=rhsm-activationkey \
    if [ "${ENABLE_SUBSCRIPTION}" = "true" ] && [ -f /run/secrets/rhsm-org ] && [ -f /run/secrets/rhsm-activationkey ]; then \
        echo "Registering the system with entitlement server" && \
        subscription-manager register \
            --activationkey=$(cat /run/secrets/rhsm-activationkey) \
            --org=$(cat /run/secrets/rhsm-org); \
    fi

# Install EPEL and development tools
RUN dnf -y update && \
    dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm && \
    dnf -y install glibc-langpack-en

# Install build tools and dependencies
# Use --allowerasing to resolve curl/curl-minimal conflict
RUN dnf install -y --allowerasing \
    sudo \
    wget \
    curl \
    perl \
    util-linux \
    xz \
    bzip2 \
    git \
    patch \
    which \
    zlib-devel \
    openssl-devel \
    yum-utils \
    autoconf \
    automake \
    make \
    gcc \
    gcc-c++ \
    gcc-gfortran \
    gdb \
    python3.12 \
    python3.12-pip \
    python3.12-devel \
    ninja-build \
    lld \
    openblas \
    openblas-devel \
    findutils \
    jq \
    vim \
    unzip \
    libffi-devel && \
    dnf clean all

# Set up Python 3.12 as default
RUN ln -sf /usr/bin/python3.12 /usr/bin/python3 && \
    ln -sf /usr/bin/python3.12 /usr/bin/python && \
    python3.12 -m pip install --upgrade pip

# Install cmake via pip
RUN python3.12 -m pip install cmake==3.18.4 && \
    ln -s /usr/local/bin/cmake /usr/bin/cmake3

# Configure git to trust all directories
RUN git config --global --add safe.directory '*'

# Clean up any existing CUDA installations
RUN rm -rf /usr/local/cuda-*

FROM base as openssl
ADD ./common/install_openssl.sh install_openssl.sh
RUN bash ./install_openssl.sh && rm install_openssl.sh

FROM base as patchelf
# Install patchelf
ADD ./common/install_patchelf.sh install_patchelf.sh
RUN bash ./install_patchelf.sh && rm install_patchelf.sh && cp $(which patchelf) /patchelf

FROM base as conda
# Install Anaconda
ADD ./common/install_conda_docker.sh install_conda.sh
RUN bash ./install_conda.sh && rm install_conda.sh

# Install CUDA
FROM base as cuda
ARG CUDA_VERSION=12.8
RUN rm -rf /usr/local/cuda-*
ADD ./common/install_cuda.sh install_cuda.sh
COPY ./common/install_nccl.sh install_nccl.sh
COPY ./ci_commit_pins/nccl-cu* /ci_commit_pins/
COPY ./common/install_cusparselt.sh install_cusparselt.sh
ENV CUDA_HOME=/usr/local/cuda-${CUDA_VERSION}
# Preserve CUDA_VERSION for the builds
ENV CUDA_VERSION=${CUDA_VERSION}
# Make things in our path by default
ENV PATH=/usr/local/cuda-${CUDA_VERSION}/bin:$PATH

FROM cuda as cuda12.8
RUN bash ./install_cuda.sh 12.8
ENV DESIRED_CUDA=12.8
ENV CUDA_VERSION=12.8
ENV CUDA_HOME=/usr/local/cuda-12.8
ENV PATH=/usr/local/cuda-12.8/bin:$PATH
ENV LD_LIBRARY_PATH=/usr/local/cuda-12.8/lib64:$LD_LIBRARY_PATH

# Install MNIST test data
FROM base as mnist
ADD ./common/install_mnist.sh install_mnist.sh
RUN bash ./install_mnist.sh

# Final step
FROM ${BASE_TARGET} as final

# Re-declare CUDA_VERSION ARG to inherit from build-time argument
ARG CUDA_VERSION

COPY --from=openssl            /opt/openssl           /opt/openssl
COPY --from=patchelf           /patchelf              /usr/local/bin/patchelf
COPY --from=conda              /opt/conda             /opt/conda

# Add jni.h for java host build.
COPY ./common/install_jni.sh install_jni.sh
COPY ./java/jni.h jni.h
RUN bash ./install_jni.sh && rm install_jni.sh

# Set PATH with CUDA before conda to ensure nvcc is found
ENV PATH /usr/local/cuda-${CUDA_VERSION}/bin:/opt/conda/bin:$PATH
ENV CUDA_HOME /usr/local/cuda-${CUDA_VERSION}
ENV LD_LIBRARY_PATH /usr/local/cuda-${CUDA_VERSION}/lib64:$LD_LIBRARY_PATH
COPY --from=mnist  /usr/local/mnist /usr/local/mnist
# Don't remove CUDA - we need it! Comment out: RUN rm -rf /usr/local/cuda
RUN chmod o+rw /usr/local
RUN touch /.condarc && \
    chmod o+rw /.condarc && \
    chmod -R o+rw /opt/conda

# Set OpenBLAS paths
ENV OpenBLAS_HOME=/usr/lib64:${OpenBLAS_HOME}

# Set linker to lld for improved build performance
ENV CMAKE_LINKER="lld"
ENV LDFLAGS="-fuse-ld=lld"
