# ============================================================
# Dockerfile: Python 3.11 + Flask — REST API / ML Application
# Base Image: python:3.11-slim (slim = no dev tools, smaller)
# Use case: Flask/FastAPI apps, ML model serving, data pipelines,
#           automation scripts, Django backends
# ============================================================

FROM python:3.11-slim AS base

LABEL maintainer="saroj@sarojops.cloud"
LABEL description="Python Flask REST API"

# Prevent Python from buffering stdout/stderr (see logs immediately)
ENV PYTHONUNBUFFERED=1
# Prevent Python from writing .pyc bytecode files
ENV PYTHONDONTWRITEBYTECODE=1
ENV PIP_NO_CACHE_DIR=1
ENV PIP_DISABLE_PIP_VERSION_CHECK=1

# Install OS dependencies for Python packages (e.g., psycopg2 needs libpq)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    curl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy requirements FIRST for layer caching optimization
COPY requirements.txt .

# Install Python dependencies
RUN pip install --upgrade pip && \
    pip install -r requirements.txt

# Copy application source code
COPY . .

# Create non-root user
RUN adduser --disabled-password --gecos '' flaskuser
RUN chown -R flaskuser:flaskuser /app
USER flaskuser

# Set Flask environment variables
ENV FLASK_APP=app.py
ENV FLASK_ENV=production
ENV PORT=5000

# Expose Flask port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Run with Gunicorn (production WSGI server, NOT flask dev server)
# 4 workers, bind to all interfaces
CMD ["gunicorn", "--workers=4", "--bind=0.0.0.0:5000", "--timeout=120", "app:app"]