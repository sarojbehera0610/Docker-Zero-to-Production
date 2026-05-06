# 06 — Dockerized 3-Tier Student Registration App

A fully containerized 3-tier web application deployed on AWS EC2 using Docker.  
Built with a **React/HTML Frontend**, **Spring Boot Backend**, and **MariaDB Database** — each running in its own Docker container.

---

## 🏗️ Architecture

```
Browser
   │
   ▼
┌─────────────────────┐
│  Frontend Container  │  Apache HTTP Server · Port 80
│  (frontend:latest)   │
└─────────┬───────────┘
          │ HTTP API calls → :8080
          ▼
┌─────────────────────┐
│  Backend Container   │  Spring Boot · Port 8080
│  (backend:latest)    │
└─────────┬───────────┘
          │ JDBC → :3306
          ▼
┌─────────────────────┐
│  Database Container  │  MariaDB · Port 3306
│  (mariadb:latest)    │
└─────────────────────┘
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | HTML, CSS, JavaScript — served via Apache |
| Backend | Java Spring Boot 3.3.5 |
| Database | MariaDB |
| Containerization | Docker |
| Cloud | AWS EC2 (Ubuntu) |
| Source Code | [EASY_CRUD by abhilashmwaghmare](https://github.com/abhilashmwaghmare/EASY_CRUD) |

---

## 📋 Prerequisites

- AWS EC2 instance (Ubuntu) — `t2.micro` or higher
- Security Group inbound rules: **Port 22, 80, 8080** open
- Git installed on EC2

---

## 🚀 Step-by-Step Deployment

### Step 1 — Launch EC2 & Install Docker

```bash
# Update packages
sudo apt update -y

# Install Docker
sudo apt install docker.io -y

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group (optional)
sudo usermod -aG docker ubuntu

# Verify
docker --version
```

---

### Step 2 — Run MariaDB Container

```bash
docker run -d \
  --name mariadb_database \
  -e MYSQL_ROOT_PASSWORD=saroj1234 \
  -e MYSQL_DATABASE=student_db \
  -p 3306:3306 \
  mariadb:latest
```

Verify it is running:
```bash
docker ps
```

---

### Step 3 — Install MySQL Client & Create Database

```bash
# Install mysql client
sudo apt install mysql-client -y

# Get MariaDB container IP
docker inspect mariadb_database | grep IPAddress

# Access the database
mysql -h 172.17.0.2 -u root -psaroj1234

# Inside MySQL shell
SHOW DATABASES;
USE student_db;
SHOW TABLES;
EXIT;
```

---

### Step 4 — Clone the Repository

```bash
git clone https://github.com/abhilashmwaghmare/EASY_CRUD.git
cd EASY_CRUD
```

---

### Step 5 — Configure & Build Backend

```bash
cd backend

# Update application.properties
vim src/main/resources/application.properties
```

Set the following values:

```properties
server.port=8080

spring.datasource.url=jdbc:mariadb://<MARIADB_CONTAINER_IP>:3306/student_db?sslMode=trust
spring.datasource.username=root
spring.datasource.password=saroj1234

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

> ⚠️ Replace `<MARIADB_CONTAINER_IP>` with the actual IP from Step 3.  
> Get it with: `docker inspect mariadb_database | grep IPAddress`

Update the Dockerfile to skip tests and limit memory:

```bash
vim Dockerfile
```

```dockerfile
FROM maven:3.8.3-openjdk-17
COPY . /opt/
WORKDIR /opt
ENV MAVEN_OPTS="-Xmx512m -XX:MaxMetaspaceSize=256m"
RUN mvn clean package -DskipTests
WORKDIR target/
EXPOSE 8080
ENTRYPOINT ["java","-jar"]
CMD ["student-registration-backend-0.0.1-SNAPSHOT.jar"]
```

Build and run the backend container:

```bash
docker build -t backend:latest .

docker run -d \
  --name backend_cont \
  -p 8080:8080 \
  backend:latest
```

Verify backend started correctly:

```bash
docker logs backend_cont | grep "Tomcat started"
```

---

### Step 6 — Configure & Build Frontend

```bash
cd ../frontend

# Update .env file with your EC2 public IP
vim .env
```

```env
REACT_APP_API_URL=http://<EC2_PUBLIC_IP>:8080
```

> Replace `<EC2_PUBLIC_IP>` with your EC2 public IP e.g. `13.200.254.178`

Update Dockerfile if needed:

```bash
vim Dockerfile
```

Build and run the frontend container:

```bash
docker build -t frontend:latest .

docker run -d \
  --name frontend_cont \
  -p 80:80 \
  frontend:latest
```

---

### Step 7 — Verify All Containers Are Running

```bash
docker ps
```

Expected output:

```
CONTAINER ID   IMAGE              STATUS        PORTS
xxxxxxxxxxxx   frontend:latest    Up X minutes  0.0.0.0:80->80/tcp
xxxxxxxxxxxx   backend:latest     Up X minutes  0.0.0.0:8080->8080/tcp
xxxxxxxxxxxx   mariadb:latest     Up X minutes  0.0.0.0:3306->3306/tcp
```

---

### Step 8 — Test the Application

Open browser and go to:
```
http://<EC2_PUBLIC_IP>
```

Fill in the registration form and click **Register**.

Verify data saved in MariaDB:

```bash
mysql -h 172.17.0.2 -u root -psaroj1234

USE student_db;
SELECT * FROM user;
```

---

## 🐛 Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Exit code 137 | OOM — Maven out of memory | Add `ENV MAVEN_OPTS="-Xmx512m"` + `-DskipTests` in Dockerfile |
| Exit code 1 | Maven tests fail — DB unreachable during build | Add `-DskipTests` to `mvn clean package` |
| `ERR_CONNECTION_REFUSED` on :8080 | Backend container not running | Check `docker logs backend_cont` |
| Data not saving | Wrong DB IP in `application.properties` | Use `docker inspect mariadb_database \| grep IPAddress` and update |
| `No such file or directory` | Wrong path in docker exec | Use `docker exec <container> find / -name "*.html"` to locate files |

---

## 📸 Screenshots

> Add screenshots of:
> - Running containers (`docker ps`)
> - Application UI
> - Data saved in MariaDB (`SELECT * FROM user`)

---

## 🔑 Key Learnings

- How to containerize a multi-tier application with Docker
- Difference between `server.port` (app port) and `datasource` port (DB port)
- Why container IPs can change on restart — and how to handle it
- Debugging Docker containers using `docker logs` and `docker exec`
- Maven build optimization with `-DskipTests` and `MAVEN_OPTS`
- How Docker default bridge network works

---

## 👨‍💻 Author

**Saroj Behera**  
DevOps Intern @ Greamio Technologies  
[GitHub](https://github.com/sarojbehera0610) · [LinkedIn](https://www.linkedin.com/in/saroj-behera-7bb84b204/) · [Portfolio](https://sarojops.cloud)