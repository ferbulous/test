FROM mcr.microsoft.com/devcontainers/base:debian

# Install basics
RUN apt-get update && apt-get install -y \
    curl \
    sudo \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install Tailscale
RUN curl -fsSL https://tailscale.com/install.sh | sh
