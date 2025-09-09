FROM --platform=$BUILDPLATFORM nvidia/cuda:12.8.1-base-ubuntu24.04

# Install Python and other dependencies
RUN apt-get update -y &&  \
    apt-get install -y python3.10 python3-venv espeak-ng espeak-ng-data git libsndfile1 curl ffmpeg g++ &&  \
    apt-get clean && rm -rf /var/lib/apt/lists/* && \
    mkdir -p /usr/share/espeak-ng-data &&  \
    ln -s /usr/lib/*/espeak-ng-data/* /usr/share/espeak-ng-data/ && \
    curl -LsSf https://astral.sh/uv/install.sh | sh && \
    mv /root/.local/bin/uv /usr/local/bin/ && \
    mv /root/.local/bin/uvx /usr/local/bin/ && \
    mkdir -p /app/api/src/models/v1_0 && \
    chgrp -R 0 /app && chmod -R g=u /app

WORKDIR /app

# Copy dependency files
COPY pyproject.toml ./pyproject.toml

# Install dependencies with GPU extras
RUN uv venv --python 3.10 && \
    uv sync --extra gpu --no-cache

# Copy project files including models
COPY api ./api
COPY web ./web
COPY docker/scripts/ ./
RUN chmod +x ./entrypoint.sh

# Re-apply group perms after COPY (since --chown was removed)
RUN chgrp -R 0 /app && chmod -R g=u /app

# Set all environment variables in one go
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app:/app/api \
    UV_LINK_MODE=copy \
    USE_GPU=true \
    PHONEMIZER_ESPEAK_PATH=/usr/bin \
    PHONEMIZER_ESPEAK_DATA=/usr/share/espeak-ng-data \
    ESPEAK_DATA_PATH=/usr/share/espeak-ng-data \
    DEVICE="gpu"

ENV DOWNLOAD_MODEL=true

# Download model if enabled
RUN if [ "$DOWNLOAD_MODEL" = "true" ]; then \
    python download_model.py --output api/src/models/v1_0; \
    fi

# Run FastAPI server through entrypoint.sh
CMD ["./entrypoint.sh"]
