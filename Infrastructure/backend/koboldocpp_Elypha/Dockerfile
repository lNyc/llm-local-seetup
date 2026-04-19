# ──────────────────────────────────────────────────────────
# hybrid-cpp — pre-built standalone binary (no Python, no .so)
# koboldcpp-linux-x64 bundles everything including CUDA
# ──────────────────────────────────────────────────────────
FROM nvidia/cuda:12.6.3-runtime-ubuntu24.04

ENV PATH="/usr/local/cuda/bin:${PATH}"
ENV LD_LIBRARY_PATH="/usr/local/cuda/lib64:${LD_LIBRARY_PATH}"

RUN apt-get update && apt-get install -y \
    curl \
    libgomp1 \
    libcurl4 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

ARG RELEASE_TAG=v1.111.2.0
ARG RELEASE_BASE=https://github.com/Elypha/hybrid-cpp/releases/download/${RELEASE_TAG}

RUN curl -fL "${RELEASE_BASE}/koboldcpp-linux-x64" -o koboldcpp && \
    chmod +x koboldcpp

EXPOSE 5001

ENTRYPOINT ["/app/koboldcpp"]