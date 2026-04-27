# 🐳 Docker-Zero-to-Production

> A hands-on Docker learning repository covering containerization fundamentals, image management, registry workflows (DockerHub + AWS ECR), and production-style deployments — built on a live Linux server.

---

## 📌 About This Repository

This repo documents my end-to-end Docker journey — from running a first container to pushing production images to AWS ECR. Every command, concept, and project here was executed on a real Ubuntu server, not just local machines.

**Tech Stack:** Docker · DockerHub · AWS ECR · Nginx · Linux (Ubuntu) · AWS EC2

---

## 🗂️ Repository Structure

```
Docker-Zero-to-Production/
│
├── 01-core-commands/
│   └── commands-reference.md       # All essential Docker CLI commands
│
├── 02-nginx-on-docker/
│   ├── README.md                   # Step-by-step: expose Nginx via port mapping
│   └── notes.md                    # Port mapping (-p, -P) explained
│
├── 03-image-management/
│   ├── README.md                   # Build, tag, push, pull, save, load workflow
│   └── notes.md                    # docker save/load, rmi, prune explained
│
├── 04-dockerhub-workflow/
│   ├── README.md                   # Push image to DockerHub from EC2 server
│   └── notes.md                    # docker login, tag, push, pull from hub
│
├── 05-aws-ecr-workflow/
│   ├── README.md                   # Push image to AWS ECR (private registry)
│   └── notes.md                    # ECR auth, tag format, push steps
│
├── 06-container-ops/
│   └── commands-reference.md       # exec, cp, logs, inspect, stats, commit
│
├── 07-dockerfile/
│   ├── README.md                   # Dockerfile instructions and best practices
│   ├── Dockerfile.nginx-custom     # Custom Nginx image example
│   └── Dockerfile.node-app         # Node.js app containerization example
│
└── projects/
    ├── nginx-live-deployment/      # Nginx exposed to internet from EC2
    └── custom-image-to-ecr/        # Build → tag → push to AWS ECR
```

---

## ✅ Commands Covered

### Container Lifecycle
| Command | Purpose |
|---|---|
| `docker run -d` | Run container in detached mode |
| `docker start / stop` | Start or stop an existing container |
| `docker create` | Create container without starting it |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers (including stopped) |
| `docker rm -f` | Force remove a container |
| `docker logs` | View container stdout/stderr logs |
| `docker exec` | Run command inside a running container |
| `docker inspect` | Full JSON details of a container/image |
| `docker stats` | Live resource usage (CPU, RAM) |
| `docker cp` | Copy files between container and host |
| `docker commit` | Create a new image from a running container |

### Image Management
| Command | Purpose |
|---|---|
| `docker images` | List all local images |
| `docker pull` | Download image from registry |
| `docker tag` | Tag an image for a registry |
| `docker rmi` | Remove an image |
| `docker save -o` | Export image as a `.tar` archive |
| `docker load -i` | Import image from a `.tar` archive |
| `docker system prune` | Remove all unused containers, images, networks |

### Port Mapping
| Flag | Meaning |
|---|---|
| `-p 8080:80` | Map host port 8080 → container port 80 |
| `-P` | Auto-assign host ports for all exposed container ports |

---

## 🚀 Projects

### 1. Nginx Live on Internet via Port Mapping
**Location:** `projects/nginx-live-deployment/`

Deployed an Nginx container on an AWS EC2 instance and exposed it to the internet using Docker port mapping. Verified the live page from a browser using the public IP.

```bash
docker pull nginx
docker run -d -p 80:80 --name my-nginx nginx
# Open http://<EC2-PUBLIC-IP> in browser ✅
```

**Concepts:** `docker run`, `-p` flag, EC2 Security Group (inbound port 80), public IP access

---

### 2. Push Custom Image to DockerHub
**Location:** `04-dockerhub-workflow/`

Built a custom image on the EC2 server, tagged it with my DockerHub username, and pushed it to a public DockerHub repository.

```bash
docker commit <container-id> my-custom-nginx
docker tag my-custom-nginx sarojbehera/my-custom-nginx:v1
docker push sarojbehera/my-custom-nginx:v1
```

---

### 3. Push Image to AWS ECR (Private Registry)
**Location:** `05-aws-ecr-workflow/`

Created a private ECR repository in AWS, authenticated Docker with ECR using AWS CLI, tagged the image in ECR format, and pushed it.

```bash
# Authenticate
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin \
  <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com

# Tag and push
docker tag my-custom-nginx:v1 \
  <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/my-repo:v1

docker push <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/my-repo:v1
```

**AWS Services Used:** ECR, IAM (access keys), EC2

---

### 4. Dockerfile (In Progress 🔧)
**Location:** `07-dockerfile/`

Writing custom Dockerfiles to build images from scratch — starting with a custom Nginx page and a Node.js app containerization.

---

## 🧠 Key Concepts Summary

| Concept | Short Explanation |
|---|---|
| **Image vs Container** | Image = blueprint, Container = running instance |
| **Port Mapping** | Routes host traffic into the container |
| **DockerHub** | Public registry to share/pull images |
| **AWS ECR** | Private registry integrated with AWS IAM |
| **docker save/load** | Offline image transfer (no registry needed) |
| **docker commit** | Snapshot a modified running container as a new image |
| **docker prune** | Clean up disk space from unused Docker objects |
| **docker inspect** | Deep-dive JSON metadata on containers/images |
| **docker stats** | Real-time CPU/Memory usage per container |

---

## 🛠️ Environment

- **Server:** AWS EC2 (Ubuntu 22.04, t2.micro)
- **Docker Version:** 24.x
- **Region:** ap-south-1 (Mumbai)
- **Registry:** DockerHub + AWS ECR

---

## 📖 Next Steps

- [ ] Dockerfile deep dive (FROM, RUN, COPY, CMD, ENTRYPOINT, EXPOSE, ENV)
- [ ] Multi-stage builds
- [ ] Docker Compose (multi-container apps)
- [ ] Docker networking (bridge, host, overlay)
- [ ] Docker volumes (data persistence)
- [ ] Kubernetes (container orchestration)

---

## 👤 Author

**Saroj Behera** — DevOps Intern @ Greamio Technologies, Nagpur  
🔗 [Portfolio](https://sarojops.cloud) · [GitHub](https://github.com/sarojbehera0610) · [LinkedIn](https://linkedin.com/in/saroj-behera-7bb84b204)