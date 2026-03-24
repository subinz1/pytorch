# RHEL 9.6 Docker image for PyTorch CI with simplified single-stage build

# GLOBAL ARGS
ARG RHEL_OS_VERSION=9.6
ARG OS_TYPE=x86_64
ARG PY_VER=3.12
ARG CUDA_VERSION=12-8
ARG CUDNN_VERSION=12.8
ARG CONDA_ENV="pytorch_build"
# This flag has to be set to true if there is no active subscription
ARG ENABLE_SUBSCRIPTION=false
# Enable LAPACK/OpenBLAS support
ARG ENABLE_LAPACK=true

# BASE IMAGE
FROM docker.io/redhat/ubi9:${RHEL_OS_VERSION}

# Re-declare build args so they're available after FROM
ARG ENABLE_SUBSCRIPTION
ARG OS_TYPE
ARG PY_VER
ARG CUDA_VERSION
ARG CUDNN_VERSION
ARG CONDA_ENV
ARG ENABLE_LAPACK

# Set locale
ENV LC_ALL=en_US.UTF-8
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US.UTF-8

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

# Update packages and install dependencies in one layer
RUN dnf -y update && \
    dnf -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm && \
    dnf -y install glibc-langpack-en && \
    dnf install -y --allowerasing \
    wget \
    jq \
    vim \
    git \
    gdb \
    unzip \
    which \
    libffi-devel \
    openssl-devel \
    gcc \
    gcc-c++ \
    make \
    ninja-build \
    lld \
    openblas \
    openblas-devel \
    findutils \
    sudo \
    curl \
    perl \
    util-linux \
    xz \
    bzip2 \
    patch \
    zlib-devel \
    yum-utils \
    autoconf \
    automake \
    gcc-gfortran \
    python3.12 \
    python3.12-pip \
    python3.12-devel && \
    dnf clean all

# Set up Python 3.12 as default
RUN ln -sf /usr/bin/python3.12 /usr/bin/python3 && \
    ln -sf /usr/bin/python3.12 /usr/bin/python && \
    python3.12 -m pip install --upgrade pip

# Install CMake 3.27+ via pip (PyTorch requires >= 3.27)
# Also install setuptools and wheel
RUN python3.12 -m pip install 'cmake>=3.27' setuptools wheel

# Configure git to trust all directories
RUN git config --global --add safe.directory '*'

# Install Miniforge and setup conda environment in one layer
RUN wget --no-check-certificate https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh && \
    chmod +x Miniforge3-Linux-x86_64.sh && \
    bash Miniforge3-Linux-x86_64.sh -b -p /opt/conda && \
    rm Miniforge3-Linux-x86_64.sh && \
    /opt/conda/bin/conda update -y conda && \
    /opt/conda/bin/conda init && \
    if [ -n "${PY_VER}" ]; then \
        /opt/conda/bin/conda create -n ${CONDA_ENV} python=${PY_VER} -y; \
    else \
        /opt/conda/bin/conda create -n ${CONDA_ENV} -y; \
    fi

# Set PATH for conda
ENV PATH=/opt/conda/bin:${PATH}

# Install CUDA, cuDNN, and NCCL
RUN . /opt/conda/etc/profile.d/conda.sh && \
    conda activate ${CONDA_ENV} && \
    rm -rf /var/cache/dnf/* && \
    rm -f /etc/yum.repos.d/cuda*.repo && \
    dnf config-manager --add-repo https://developer.download.nvidia.com/compute/cuda/repos/rhel9/x86_64/cuda-rhel9.repo && \
    dnf clean all && \
    dnf makecache && \
    dnf -y install cuda-toolkit-${CUDA_VERSION} && \
    conda install -y nvidia::cudnn cuda-version=${CUDNN_VERSION}

# Build and install NCCL from source (matching upstream CI pinned version)
RUN git clone -b v2.29.7-1 --depth 1 https://github.com/NVIDIA/nccl.git && \
    cd nccl && \
    make -j$(nproc) src.build CUDA_HOME=/usr/local/cuda && \
    cp -a build/include/* /usr/local/cuda/include/ && \
    cp -a build/lib/* /usr/local/cuda/lib64/ && \
    cd .. && \
    rm -rf nccl && \
    ldconfig

# Use system NCCL instead of building from third_party/nccl
ENV USE_SYSTEM_NCCL=1

# Install sccache and PyTorch build dependencies via conda/pip
# Download and install all requirements from upstream PyTorch requirements-ci.txt
RUN . /opt/conda/etc/profile.d/conda.sh && \
    conda activate ${CONDA_ENV} && \
    conda install -y sccache && \
    curl -fsSL https://raw.githubusercontent.com/pytorch/pytorch/main/.ci/docker/requirements-ci.txt -o /tmp/requirements-ci.txt && \
    pip install -r /tmp/requirements-ci.txt && \
    rm /tmp/requirements-ci.txt && \
    pip install triton

# Set OpenBLAS paths (only effective when LAPACK is enabled)
ENV OpenBLAS_HOME=/usr/lib64:${OpenBLAS_HOME}

# Adding CUDA and library to the paths
# Use the pytorch_build conda environment bin directory
ENV PATH=/usr/local/cuda/bin:/opt/conda/envs/${CONDA_ENV}/bin:/opt/conda/bin:$PATH
ENV LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
ENV CUDA_HOME=/usr/local/cuda

# Setting flags and changing the linker to lld from default ld
ENV CMAKE_LINKER="lld"
ENV LDFLAGS="-fuse-ld=lld"

# Set compiler flags for proper float16/half-precision support
# -mf16c enables F16C instruction set for half-precision conversions on x86-64
# -mavx2 enables AVX2 which is required for efficient FP16 operations
ENV CFLAGS="-mf16c -mavx2"
ENV CXXFLAGS="-mf16c -mavx2"

# Set default conda environment for interactive use
ENV CONDA_DEFAULT_ENV=${CONDA_ENV}
RUN echo "conda activate ${CONDA_ENV}" >> ~/.bashrc

# Set permissions for conda
RUN touch /.condarc && \
    chmod o+rw /.condarc && \
    chmod -R o+rw /opt/conda

WORKDIR /pytorch
