# FROM nvidia/cuda:12.8.0-devel-ubuntu22.04
FROM registry.cloud.rt.nyu.edu/sw77/cuda:12.6.3-ubuntu-22.04-20250103

ENV DEBIAN_FRONTEND=noninteractive
ENV TORCH_CUDA_ARCH_LIST="7.5;8.0;8.6+PTX"

# ---- OpenShift-friendly cache locations (prevents writes to /.cache) ----
ENV XDG_CACHE_HOME=/tmp/.cache
ENV UV_CACHE_DIR=/tmp/uv-cache
ENV PIP_CACHE_DIR=/tmp/pip-cache

WORKDIR /app

# Install system dependencies, including libglib2.0-0 needed by cv2
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3 python3-dev python3-pip python3-venv python3-distutils \
        build-essential cmake ninja-build git curl \
        libglib2.0-0 libsm6 libxext6 libxrender1 libgl1 \
    && ln -s /usr/bin/python3 /usr/bin/python \
    && ln -sf /usr/bin/pip3 /usr/bin/pip \
    && python3 -m pip install --upgrade pip setuptools wheel \
    && rm -rf /var/lib/apt/lists/*

# ---- Permissions for random UID on OpenShift ----
# Make /app and /tmp writable for arbitrary UIDs (via group 0)
RUN mkdir -p /tmp/.cache /tmp/uv-cache /tmp/pip-cache /app \
 && chgrp -R 0 /app /tmp \
 && chmod -R g=u /app /tmp

# Copy code
COPY . .

# Install Python dependencies
RUN python3 -m pip install --no-cache-dir numpy
RUN python3 -m pip install --no-cache-dir torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
RUN python3 -m pip install --no-cache-dir opencv-python-headless

# Editable install
RUN python3 -m pip install --no-cache-dir -e .

# Install FastAPI & Uvicorn
RUN python3 -m pip install --no-cache-dir fastapi uvicorn python-multipart

WORKDIR /app/server
EXPOSE 4040

# NOTE: --reload is typically not used in production containers.
# CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "4040"]
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "4040"]
