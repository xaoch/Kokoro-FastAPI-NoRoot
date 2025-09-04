FROM --platform=$BUILDPLATFORM nvidia/cuda:12.8.1-base-ubuntu24.04

# --- system deps ---
RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
        python3.10 python3-venv \
        espeak-ng espeak-ng-data \
        git libsndfile1 curl ffmpeg g++ \
        python-is-python3 \
    && apt-get clean && rm -rf /var/lib/apt/lists/* \
    && mkdir -p /usr/share/espeak-ng-data \
    && ln -s /usr/lib/*/espeak-ng-data/* /usr/share/espeak-ng-data/ \
    && curl -LsSf https://astral.sh/uv/install.sh | sh \
    && mv /root/.local/bin/uv /usr/local/bin/ \
    && mv /root/.local/bin/uvx /usr/local/bin/ \
    && useradd -m -u 1001 appuser

WORKDIR /app

# --- copy project BEFORE uv sync so api/src exists ---
COPY pyproject.toml ./pyproject.toml
# (README is referenced during build; include it to avoid warnings)
COPY README.md ./README.md
COPY api ./api
COPY web ./web
COPY docker/scripts/ ./
COPY download_model.py ./download_model.py

# --- create venv + install deps (installs your project too) ---
RUN uv venv --python 3.10 && \
    uv sync --extra gpu --no-cache

# entrypoint permissions
RUN chmod +x /app/entrypoint.sh

# --- OpenShift arbitrary-UID friendliness ---
RUN chgrp -R 0 /app && chmod -R g=u /app \
 && mkdir -p /home/appuser && chgrp -R 0 /home/appuser && chmod -R g=u /home/appuser

# --- runtime env ---
ENV PATH="/app/.venv/bin:${PATH}" \
    PYTHONUNBUFFERED=1 \
    PYTHONPATH=/app:/app/api \
    UV_LINK_MODE=copy \
    USE_GPU=true \
    PHONEMIZER_ESPEAK_PATH=/usr/bin \
    PHONEMIZER_ESPEAK_DATA=/usr/share/espeak-ng-data \
    ESPEAK_DATA_PATH=/usr/share/espeak-ng-data \
    DEVICE="gpu"

# (optional) download model at build; set to false if you want a smaller image
ARG DOWNLOAD_MODEL=true
ENV DOWNLOAD_MODEL=${DOWNLOAD_MODEL}
RUN if [ "${DOWNLOAD_MODEL}" = "true" ]; then \
      /app/.venv/bin/python /app/download_model.py --output /app/api/src/models/v1_0; \
    fi

# quick sanity check
RUN /app/.venv/bin/python -c "import loguru, fastapi; print('deps ok')"

USER 1001
CMD ["./entrypoint.sh"]
