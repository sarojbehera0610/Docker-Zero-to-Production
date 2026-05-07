# ============================================================
# Dockerfile: Apache HTTPD 2.4 — Static Website / PHP App
# Base Image: httpd:2.4-alpine
# Use case: Static websites, frontend hosting, reverse proxy,
#           Apache modules, .htaccess configurations
# ============================================================

FROM httpd:2.4-alpine

LABEL maintainer="saroj@sarojops.cloud"
LABEL description="Apache HTTPD static site server"

# Install additional packages (alpine uses apk)
RUN apk add --no-cache \
    curl \
    bash \
    openssl

# Copy your website files into Apache's document root
COPY ./html/ /usr/local/apache2/htdocs/

# Copy custom Apache config (override default httpd.conf)
# This enables mod_rewrite, custom error pages, etc.
COPY ./config/httpd.conf /usr/local/apache2/conf/httpd.conf

# Copy SSL certificates for HTTPS (if enabling SSL)
# COPY ./certs/server.crt /usr/local/apache2/conf/server.crt
# COPY ./certs/server.key /usr/local/apache2/conf/server.key

# Create custom log directory
RUN mkdir -p /usr/local/apache2/logs/custom

# Set environment variables for Apache
ENV APACHE_LOG_DIR=/usr/local/apache2/logs
ENV APACHE_RUN_USER=www-data

# Expose both HTTP and HTTPS ports
EXPOSE 80
EXPOSE 443

# Health check for Apache
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost/ || exit 1

# Apache starts in foreground by default with this image
# httpd-foreground keeps it running (not daemonized)
CMD ["httpd-foreground"]