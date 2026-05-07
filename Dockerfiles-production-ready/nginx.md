# ============================================================
# Dockerfile: Nginx — Reverse Proxy / Static Server
# Base Image: nginx:1.25-alpine
# Use case: Static website hosting, reverse proxy for Node/Python,
#           load balancer, SSL termination, React/Vue/Angular builds
# ============================================================

# Stage 1: Build React/Angular/Vue app
FROM node:18-alpine AS frontend-build

WORKDIR /build
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
# Output: /build/dist or /build/build

# Stage 2: Serve with Nginx
FROM nginx:1.25-alpine

LABEL maintainer="saroj@sarojops.cloud"
LABEL description="Nginx reverse proxy and static file server"

# Remove default Nginx config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom Nginx configuration
COPY ./nginx/nginx.conf /etc/nginx/nginx.conf
COPY ./nginx/default.conf /etc/nginx/conf.d/default.conf

# Copy built frontend from Stage 1
COPY --from=frontend-build /build/dist /usr/share/nginx/html

# OR copy static files directly (without build stage)
# COPY ./public/ /usr/share/nginx/html/

# Copy SSL certificates (for HTTPS with self-signed cert for dev)
# COPY ./ssl/nginx.crt /etc/nginx/ssl/nginx.crt
# COPY ./ssl/nginx.key /etc/nginx/ssl/nginx.key

# Create nginx cache directory
RUN mkdir -p /var/cache/nginx

# Expose HTTP and HTTPS
EXPOSE 80
EXPOSE 443

# Health check for Nginx
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost/ || exit 1

# Nginx runs in foreground by default (daemon off directive in config)
CMD ["nginx", "-g", "daemon off;"]