FROM python:3.11-slim AS builder

WORKDIR /build

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml ./
COPY proto ./proto
COPY claw_vector_svc ./claw_vector_svc

# One virtualenv holds everything (torch included) and is copied whole into the
# runtime image. Installing torch globally and the rest with --prefix left the
# runtime without torch, and pip skipped dependencies it already saw globally
# (ModuleNotFoundError: typing_extensions at startup).
RUN python -m venv /opt/venv \
    && /opt/venv/bin/pip install --no-cache-dir --upgrade pip \
    && /opt/venv/bin/pip install --no-cache-dir "torch>=2.0" --index-url https://download.pytorch.org/whl/cpu \
    && /opt/venv/bin/pip install --no-cache-dir '.[inference]'

ARG MODEL_NAME="sentence-transformers/all-MiniLM-L6-v2"
ENV HF_HOME=/opt/hf
RUN /opt/venv/bin/python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('${MODEL_NAME}')"

FROM python:3.11-slim AS runtime

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/* \
    && useradd --uid 1001 --home-dir /root --no-create-home app \
    && mkdir -p /app \
    && chown -R 1001:1001 /app

COPY --from=builder /opt/venv /opt/venv
COPY --from=builder /opt/hf /opt/hf
COPY claw_vector_svc ./claw_vector_svc
COPY proto ./proto

ENV PATH=/opt/venv/bin:$PATH
ENV HF_HOME=/opt/hf
ENV HF_HUB_OFFLINE=1
ENV PYTHONUNBUFFERED=1
ENV HTTP_PORT=8080
ENV GRPC_PORT=50051

USER 1001

EXPOSE 50051 8080

HEALTHCHECK --interval=30s --timeout=5s --start-period=30s --retries=3 CMD curl -f http://localhost:8080/ready || exit 1

CMD ["uvicorn", "claw_vector_svc.main:app", "--host", "0.0.0.0", "--port", "8080"]
