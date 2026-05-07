# 🐳 Docker-Zero-to-Production — Dockerfile Reference Collection

> **By Saroj Behera | [sarojops.cloud](https://sarojops.cloud)**  
> Hands-on Dockerfile collection covering the most industry-used base images with production-grade patterns.

---

## 📁 Repository Structure

```
dockerfiles/
├── 01_node18/            → Node.js 18 LTS (multi-stage, non-root user)
├── 02_ubuntu/            → Ubuntu 22.04 (DevOps tooling, AWS CLI)
├── 03_httpd/             → Apache HTTPD 2.4 (static sites, SSL)
├── 04_mysql/             → MySQL 8.0 (init scripts, volumes)
├── 05_mariadb/           → MariaDB 10.11 LTS (WordPress stack)
├── 06_nginx/             → Nginx 1.25 (reverse proxy, React build)
├── 07_python_flask/      → Python 3.11 + Flask (Gunicorn, non-root)
└── 08_java_springboot/   → Java 17 + Spring Boot (Maven multi-stage)
```

---

## 🔑 High-Impact Dockerfile Commands — Master Reference

| Command | Purpose | Example |
|---|---|---|
| `FROM` | Base image — every Dockerfile starts here | `FROM node:18-alpine` |
| `LABEL` | Metadata: maintainer, version, description | `LABEL maintainer="saroj@sarojops.cloud"` |
| `RUN` | Execute shell commands during build | `RUN apt-get update && apt-get install -y curl` |
| `COPY` | Copy files from host → container | `COPY package.json ./` |
| `ADD` | Like COPY but supports URLs and tar auto-extract | `ADD app.tar.gz /app/` |
| `WORKDIR` | Set working directory (creates if not exists) | `WORKDIR /app` |
| `ENV` | Set runtime environment variables | `ENV NODE_ENV=production` |
| `ARG` | Build-time variables (not available at runtime) | `ARG BUILD_DATE` |
| `EXPOSE` | Document which port the app listens on | `EXPOSE 3000` |
| `VOLUME` | Mount point for persistent/external storage | `VOLUME ["/var/lib/mysql"]` |
| `USER` | Switch to non-root user (security best practice) | `USER appuser` |
| `HEALTHCHECK` | Docker health monitoring config | `HEALTHCHECK CMD curl -f http://localhost/` |
| `CMD` | Default command (overridable by `docker run`) | `CMD ["node", "server.js"]` |
| `ENTRYPOINT` | Fixed entrypoint (CMD becomes its arguments) | `ENTRYPOINT ["java", "-jar"]` |
| `ONBUILD` | Trigger for child images (used in base images) | `ONBUILD COPY . /app` |

---

## 🚀 CMD vs ENTRYPOINT — The Most Confused Pair

```dockerfile
# Shell form (spawns /bin/sh -c — DON'T use in prod, misses SIGTERM)
CMD node server.js

# Exec form ✅ (recommended — handles signals correctly)
CMD ["node", "server.js"]

# ENTRYPOINT + CMD together
ENTRYPOINT ["java", "-jar"]
CMD ["app.jar"]
# Result: java -jar app.jar
# Override at runtime: docker run myimage other.jar → java -jar other.jar
```

---

## 🏗️ Multi-Stage Build Pattern (Reduces Image Size 60-80%)

```dockerfile
# Stage 1: Build
FROM node:18 AS builder
WORKDIR /app
COPY . .
RUN npm ci && npm run build

# Stage 2: Production (only runtime artifacts copied)
FROM node:18-alpine AS production
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]
```

---

## 🔒 Security Best Practices (All Dockerfiles Here Follow These)

1. **Non-root user** — `adduser` + `USER nonroot`
2. **Alpine/slim images** — smaller attack surface
3. **No secrets in ENV** — use Docker secrets / AWS Secrets Manager
4. **Pinned versions** — `FROM node:18.19.0` not `FROM node:latest`
5. **Layer caching** — `COPY package.json` before `COPY . .`
6. **.dockerignore** — exclude `node_modules`, `.git`, `.env`

---

## 📦 Quick Build & Run Commands

```bash
# Build with tag
docker build -t myapp:v1 .

# Build with ARG
docker build --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) -t myapp:v1 .

# Run with port mapping
docker run -d -p 3000:3000 --name myapp myapp:v1

# Run with environment override
docker run -d -e NODE_ENV=staging myapp:v1

# Run with volume mount
docker run -d -v $(pwd)/data:/var/lib/mysql mysql:v1

# Inspect running container
docker exec -it myapp /bin/sh

# View logs
docker logs -f myapp

# Check image layers and size
docker history myapp:v1
docker images myapp:v1
```

---

## 🏷️ .dockerignore (Always Include This!)

```
node_modules/
.git/
.env
.env.*
*.log
dist/
coverage/
.DS_Store
```

---

*Part of `Docker-Zero-to-Production` by Saroj Behera*