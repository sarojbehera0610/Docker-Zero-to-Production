# ============================================================
# Dockerfile: MariaDB 10.11 LTS — MySQL-Compatible Database
# Base Image: mariadb:10.11
# Use case: Drop-in MySQL replacement, WordPress, Drupal,
#           open-source stacks (LAMP), better performance at scale
# ============================================================

FROM mariadb:10.11

LABEL maintainer="saroj@sarojops.cloud"
LABEL description="MariaDB 10.11 LTS database server"

# MariaDB uses same ENV variable names as MySQL (compatible!)
ENV MARIADB_ROOT_PASSWORD=RootSecure@456
ENV MARIADB_DATABASE=wordpress_db
ENV MARIADB_USER=wp_user
ENV MARIADB_PASSWORD=WpSecure@456
ENV MARIADB_AUTO_UPGRADE=1
# MARIADB_AUTO_UPGRADE: auto-upgrades system tables on version change

# Custom MariaDB server configuration
COPY ./config/mariadb.cnf /etc/mysql/conf.d/

# Initialization SQL scripts (same as MySQL — runs on first start)
COPY ./init/ /docker-entrypoint-initdb.d/

# Character set configuration (important for multilingual apps)
RUN echo '[mysqld]' >> /etc/mysql/conf.d/charset.cnf \
    && echo 'character-set-server = utf8mb4' >> /etc/mysql/conf.d/charset.cnf \
    && echo 'collation-server = utf8mb4_unicode_ci' >> /etc/mysql/conf.d/charset.cnf

# Persist database data with a named volume
VOLUME ["/var/lib/mysql"]

# Expose MariaDB port
EXPOSE 3306

# Health check using mariadb-admin (MariaDB-specific binary)
HEALTHCHECK --interval=20s --timeout=10s --start-period=45s --retries=5 \
    CMD mariadb-admin ping -h localhost -u root -p${MARIADB_ROOT_PASSWORD} || exit 1

CMD ["mariadbd"]