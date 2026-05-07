# ============================================================
# Dockerfile: MySQL 8.0 — Production Database
# Base Image: mysql:8.0
# Use case: Relational database, backend data storage,
#           microservices DB layer, init with seed data
# ============================================================

FROM mysql:8.0

LABEL maintainer="saroj@sarojops.cloud"
LABEL description="MySQL 8.0 production database"

# -------------------------------------------------------
# ENV variables = MySQL configuration (critical!)
# These are used by the official MySQL entrypoint script
# -------------------------------------------------------
ENV MYSQL_ROOT_PASSWORD=StrongRootPass@123
ENV MYSQL_DATABASE=appdb
ENV MYSQL_USER=appuser
ENV MYSQL_PASSWORD=AppUserPass@123
# TIP: In production, use Docker secrets or AWS Secrets Manager
#      instead of hardcoding passwords here!

# Custom MySQL configuration file
# Override default config: increase connections, set charset, etc.
COPY ./config/my.cnf /etc/mysql/conf.d/custom.cnf

# Auto-initialization: Any .sql or .sh files in this directory
# are executed automatically on FIRST container startup
COPY ./init-scripts/ /docker-entrypoint-initdb.d/
# Example: 01_schema.sql, 02_seed_data.sql

# Create directory for custom data persistence
# (Usually handled by Docker volumes, not COPY)
VOLUME ["/var/lib/mysql"]

# Expose MySQL default port
EXPOSE 3306

# Health check: ensures MySQL is ready to accept connections
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=5 \
    CMD mysqladmin ping -h localhost -u root -p${MYSQL_ROOT_PASSWORD} || exit 1

# Default CMD from base image runs mysqld — no need to override
# CMD ["mysqld"]