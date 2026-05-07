# ============================================================
# Dockerfile: Node.js 18 (LTS) — Production-Ready Express App
# Base Image: node:18-alpine (lightweight, ~50MB vs ~300MB full)
# Use case: REST APIs, microservices, Next.js, React SSR
# ============================================================

# --- Stage 1: Build Stage ---
FROM node:18-alpine AS builder

# Set maintainer label (good practice for image metadata)
LABEL maintainer="saroj@sarojops.cloud"
LABEL version="1.0"
LABEL description="Node.js 18 production app"

# Install OS-level dependencies (e.g., for native npm modules)
RUN apk add --no-cache \
    python3 \
    make \
    g++

# Set working directory inside container
WORKDIR /app

# Copy package files FIRST (leverages Docker layer caching)
# If package.json doesn't change, npm install layer is reused
COPY package.json package-lock.json ./

# Install only production dependencies
RUN npm ci --only=production

# Copy rest of the source code
COPY . .

# --- Stage 2: Production Stage (Multi-stage build) ---
FROM node:18-alpine AS production

# Set NODE_ENV for optimized runtime behavior
ENV NODE_ENV=production
ENV PORT=3000

# Create a non-root user for security (never run as root!)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy built artifacts from builder stage only
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app .

# Change ownership to non-root user
RUN chown -R appuser:appgroup /app

# Switch to non-root user
USER appuser

# Expose the application port
EXPOSE 3000

# Health check: Docker will auto-restart if unhealthy
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Use exec form (not shell form) — handles signals correctly (SIGTERM)
CMD ["node", "server.js"]