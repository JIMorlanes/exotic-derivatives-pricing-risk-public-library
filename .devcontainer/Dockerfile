FROM mcr.microsoft.com/devcontainers/base:3.0.10-resolute

USER root
RUN apt-get update \
    && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
        libstdc++6 \
        libgcc-s1 \
    && rm -rf /var/lib/apt/lists/*

USER vscode
