# syntax=docker/dockerfile:1.7

# Build stage: compile wheels so build toolchain does not land in runtime image
ARG PYTHON_IMAGE=python:3.12.7-alpine
FROM ${PYTHON_IMAGE} AS builder

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

WORKDIR /app

# Install build toolchain only in builder image
RUN apk add --no-cache build-base

# Copy only dependency manifest for better layer caching
COPY requirements.txt .

# Build wheels with a cached pip directory (BuildKit)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip wheel --no-cache-dir --wheel-dir=/wheels -r requirements.txt

# Runtime stage
FROM ${PYTHON_IMAGE} AS runtime

# Re-declare ARGs per stage
ARG APP_VERSION=0.1.0

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1 \
    APP_VERSION=${APP_VERSION}

# Create non-root user/group with stable UID/GID
RUN addgroup -S app && adduser -S -G app -u 10001 app

WORKDIR /app

# Install deps from prebuilt wheels, no compiler or index access
COPY --from=builder /wheels /wheels
COPY requirements.txt .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels

# Copy app code with least-privilege ownership
COPY --chown=app:app . .

EXPOSE 5001

USER 10001

# Liveness check; change path if your health endpoint differs
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import sys,urllib.request; urllib.request.urlopen('http://127.0.0.1:5001/healthz').read()" || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "5001"]