# ============================================================
# Dockerfile: Ubuntu 22.04 LTS — Custom Tooling / Shell Scripts
# Base Image: ubuntu:22.04
# Use case: CI/CD runners, custom tooling, shell script automation,
#           sysadmin tasks, DevOps utilities
# ============================================================

FROM ubuntu:22.04

LABEL maintainer="saroj@sarojops.cloud"
LABEL description="Ubuntu 22.04 DevOps utility container"

# Prevent interactive prompts during apt installs (critical for Docker!)
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Kolkata

# Update package index and install tools in ONE RUN layer
# (Combining RUN commands reduces image layers and size)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    wget \
    vim \
    git \
    unzip \
    net-tools \
    iputils-ping \
    dnsutils \
    htop \
    tree \
    jq \
    ca-certificates \
    gnupg \
    lsb-release \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
# ^^^ Always clean apt cache to reduce image size

# Install AWS CLI v2
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && ./aws/install \
    && rm -rf awscliv2.zip aws/

# Set working directory
WORKDIR /workspace

# Copy shell scripts into container
COPY scripts/ ./scripts/

# Make scripts executable
RUN chmod +x ./scripts/*.sh

# Create a non-root user
RUN useradd -ms /bin/bash devops

# Set environment variables
ENV APP_ENV=development
ENV LOG_LEVEL=info

# Switch to non-root user
USER devops

# ARG: Build-time variable (different from ENV which is runtime)
ARG BUILD_DATE
LABEL build-date=$BUILD_DATE

# Default shell command (keeps container running for exec)
CMD ["/bin/bash"]