# ---- Base ----
FROM --platform=$BUILDPLATFORM nvidia/cuda:12.8.1-base-ubuntu24.04

# System deps (keep as root)
RUN apt-get update -y && \
    apt-get install -y \
        python3.10 python3-venv \
        espeak-ng espeak-ng-data \
        git libsndfile1 curl ffmpeg g++ \
        # optional convenience: `python` -> `python3`
        python-is-python3 \
    && apt-get clean && rm -rf /var/lib/apt/lists/* \
    && mkdir -p /usr/share/espeak-ng-data \
    && ln -s /usr/lib/*/espeak-ng-data/* /usr/share/espeak-ng-data/ \
    && curl -LsSf https://astral.sh/uv/install.sh | sh \
    && mv /root/.local/bin/uv /usr/local/bin/ \
    && mv /root/.local/bin/uvx /usr/local/bin/ \
    && useradd -m -u 1001 appuser

# App directory (stay root while we set it up)
WORKDIR /app

# ---- Python deps via uv ----
# Copy only dependency metadata first for layer caching
COPY pyproject.toml ./pyproject.toml

# Create venv and install deps (GPU extra)
RUN uv venv --python 3.10 && \
    uv sync --extra gpu --no-cache

# ---- Copy project files ----
# Copy code and scripts (still as root); include entrypoint + downloader
COPY api ./api
COPY web ./web
COPY docker/scripts/ ./
COPY download_model.py ./download_model.py

# Ensure entrypoint is executable
RUN chmod +x /app/entrypoint.sh

# ---- OpenShift arbitrary-UID friendliness ----
# Make /app group-owned by root (0) and mirror user perms to group
# so any random UID in group 0 (how OCP runs) can read/exec the venv.
RUN chgrp -R 0 /app && chmod -R g=u /app

# (Optional but nice) also prepare the appuser home in case it's used at runtime
RUN mkdir -p /home/appuser && chgrp -R 0 /home/appuser && chmod -R g=u /home/appuser

# ---- Runtime env ----
ENV PATH="/app/.venv/bin:${PATH}" \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app:/app/api \
    UV_LINK_MODE=copy \
    USE_GPU=true \
    PHONEMIZER_ESPEAK_PATH=/usr/bin \
    PHONEMIZER_ESPEAK_DATA=/usr/share/espeak-ng-data \
    ESPEAK_DATA_PATH=/usr/share/espeak-ng-data \
    DEVICE="gpu"

# ---- (Build-time) Model download (if enabled) ----
ARG DOWNLOAD_MODEL=true
ENV DOWNLOAD_MODEL=${DOWNLOAD_MODEL}

# Use the venv's interpreter explicitly to avoid ModuleNotFoundError (e.g., loguru)
RUN if [ "${DOWNLOAD_MODEL}" = "true" ]; then \
      /app/.venv/bin/python /app/download_model.py --output /app/api/src/models/v1_0; \
    fi

# (Optional) sanity check to fail early if critical deps are missing
RUN /app/.venv/bin/python -c "import loguru, fastapi; print('deps ok')"

# Drop privileges AFTER perms are fixed
USER 1001

# ---- Start ----
CMD ["./entrypoint.sh"]
