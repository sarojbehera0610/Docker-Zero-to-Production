# 🌍 WanderNest — Tour & Travel Registration Platform

A production-grade 3-Tier web application built and deployed using Docker & Docker Compose on AWS EC2.
This project demonstrates containerization of a full-stack application with a modern tech stack.

---

## 🧱 Architecture Overview

```
Browser
  │
  ▼
[ Frontend ] — Caddy Web Server (Port 80)
  │
  ▼
[ Backend ] — Python Flask + Gunicorn WSGI (Port 5000)
  │
  ▼
[ Database ] — PostgreSQL 16 (Port 5432 - Internal Only)
```

---

## 🛠️ Tech Stack

| Tier | Tool | Image | Port |
|---|---|---|---|
| Frontend | Caddy Web Server | `caddy:2-alpine` | 80 |
| Backend | Python Flask + Gunicorn | `python:3.11-slim` | 5000 |
| Database | PostgreSQL 16 | `postgres:16-alpine` | 5432 (internal) |
| Orchestration | Docker Compose | — | — |
| Infrastructure | AWS EC2 (Ubuntu) | — | — |

> **Why these tools?**
> - **Caddy** replaces nginx/httpd — modern, simpler config, auto-HTTPS capable
> - **Gunicorn** is a production WSGI server — replaces basic `flask run`
> - **PostgreSQL** replaces MySQL/MariaDB — industry standard relational database
> - **Docker Compose** orchestrates all 3 tiers with a single command

---

## 📁 Project Structure

```
wandernest/
├── .env                        # Environment variables (credentials)
├── docker-compose.yml          # Orchestrates all 3 services
├── frontend/
│   ├── Dockerfile              # Caddy image build
│   ├── Caddyfile               # Caddy server config
│   └── index.html              # Frontend UI (HTML + CSS + JS)
└── backend/
    ├── Dockerfile              # Python Flask image build
    ├── app.py                  # Flask REST API
    ├── requirements.txt        # Python dependencies
    └── init.sql                # DB schema + seed data (auto runs on first start)
```

---

## ⚙️ Features

- Browse available tours with destination, duration and price
- Register for a tour with name, email, phone and travel date
- View all recent registrations in a table
- REST API with `/health`, `/tours`, `/register`, `/registrations` endpoints
- PostgreSQL data persisted using Docker named volume
- Auto DB initialization using `init.sql` on first container start
- Health check on DB container — backend waits for DB to be ready before starting
- Environment variables managed via `.env` file

---

## 🚀 Setup & Deployment — Step by Step

### Prerequisites

- AWS EC2 instance running Ubuntu
- Ports 22, 80, 5000 open in Security Group Inbound Rules

---

### Step 1 — Install Docker

```bash
sudo apt update -y
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update -y
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```

Verify:

```bash
docker --version
docker compose version
```

> **Note:** If you are using root user, skip `sudo usermod -aG docker $USER`. Root already has full permissions. If you are using non-root user like ubuntu or ec2-user, run the below to avoid typing sudo every time:
> ```bash
> sudo usermod -aG docker $USER
> newgrp docker
> ```

---

### Step 2 — Install Docker Compose (Manual Method)

```bash
DOCKER_CONFIG=${DOCKER_CONFIG:-$HOME/.docker}
mkdir -p $DOCKER_CONFIG/cli-plugins
curl -SL https://github.com/docker/compose/releases/download/v2.27.1/docker-compose-linux-x86_64 -o $DOCKER_CONFIG/cli-plugins/docker-compose
chmod +x $DOCKER_CONFIG/cli-plugins/docker-compose
docker compose version
```

> **Note:** Always use v2.x releases. v5.x does not exist for Docker Compose.

---

### Step 3 — Create Project Structure

```bash
mkdir -p wandernest/frontend wandernest/backend
cd wandernest
```

---

### Step 4 — Create All Project Files

> **vim quick reference:**
> - Press `i` to enter insert mode and start typing
> - Press `Esc` then `:wq` and Enter to save and exit
> - Press `Esc` then `:q!` and Enter to exit without saving
> - Press `Esc` then `gg` then `dG` to delete all content in file

---

#### `.env`

```bash
vim .env
```

```
POSTGRES_DB=wandernest
POSTGRES_USER=wanderuser
POSTGRES_PASSWORD=Wander2024
DATABASE_URL=postgresql://wanderuser:Wander2024@db:5432/wandernest
FLASK_ENV=production
```

> **Important:** `.env` file goes in root of project alongside `docker-compose.yml`. Docker Compose reads it automatically. You never hardcode credentials in yaml or code files. The `${VARIABLE}` syntax in docker-compose.yml reads values from `.env` automatically at runtime.

> **Important:** Do not use special characters like `@` inside the password. The `@` symbol is used as a separator in the database URL. Having two `@` signs breaks the URL parser. See Problems & Solutions section for details.

---

#### `docker-compose.yml`

```bash
vim docker-compose.yml
```

```yaml
version: "3.9"

services:

  db:
    image: postgres:16-alpine
    container_name: wandernest-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pg_data:/var/lib/postgresql/data
      - ./backend/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - backend-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: wandernest-backend
    restart: unless-stopped
    environment:
      DATABASE_URL: ${DATABASE_URL}
      FLASK_ENV: ${FLASK_ENV}
    ports:
      - "5000:5000"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - backend-net

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: wandernest-frontend
    restart: unless-stopped
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - frontend-net

networks:
  backend-net:
    driver: bridge
  frontend-net:
    driver: bridge

volumes:
  pg_data:
    driver: local
```

---

#### `backend/init.sql`

```bash
vim backend/init.sql
```

```sql
CREATE TABLE IF NOT EXISTS tours (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    destination VARCHAR(100) NOT NULL,
    duration VARCHAR(50),
    price DECIMAL(10,2),
    description TEXT
);

CREATE TABLE IF NOT EXISTS registrations (
    id SERIAL PRIMARY KEY,
    full_name VARCHAR(100) NOT NULL,
    email VARCHAR(150) NOT NULL,
    phone VARCHAR(20),
    tour_id INTEGER REFERENCES tours(id),
    travel_date DATE,
    registered_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO tours (name, destination, duration, price, description) VALUES
('Himalayan Adventure', 'Manali, Himachal Pradesh', '7 Days / 6 Nights', 15999.00, 'Snow peaks, river rafting, and mountain treks.'),
('Kerala Backwaters', 'Alleppey, Kerala', '5 Days / 4 Nights', 12499.00, 'Houseboat stay, backwater cruise, Ayurvedic spa.'),
('Rajasthan Royal Tour', 'Jaipur, Jodhpur, Jaisalmer', '8 Days / 7 Nights', 18999.00, 'Forts, palaces, desert safari, camel ride.'),
('Goa Beach Escape', 'North and South Goa', '4 Days / 3 Nights', 9999.00, 'Beaches, water sports, seafood, nightlife.'),
('Andaman Islands', 'Port Blair, Havelock', '6 Days / 5 Nights', 22999.00, 'Coral reefs, scuba diving, pristine beaches.');
```

> This file runs automatically when PostgreSQL container starts for the first time. It creates tables and inserts 5 tour records. You do not need to run this manually.

---

#### `backend/requirements.txt`

```bash
vim backend/requirements.txt
```

```
flask==3.0.3
psycopg2-binary==2.9.9
flask-cors==4.0.1
gunicorn==22.0.0
python-dotenv==1.0.1
```

---

#### `backend/app.py`

```bash
vim backend/app.py
```

```python
from flask import Flask, request, jsonify
from flask_cors import CORS
import psycopg2
import os

app = Flask(__name__)
CORS(app)

DATABASE_URL = os.environ.get("DATABASE_URL")

def get_db():
    conn = psycopg2.connect(DATABASE_URL)
    return conn

@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "WanderNest Backend Running"})

@app.route("/tours", methods=["GET"])
def get_tours():
    conn = get_db()
    cur = conn.cursor()
    cur.execute("SELECT id, name, destination, duration, price, description FROM tours")
    rows = cur.fetchall()
    cur.close()
    conn.close()
    tours = []
    for r in rows:
        tours.append({
            "id": r[0], "name": r[1], "destination": r[2],
            "duration": r[3], "price": float(r[4]), "description": r[5]
        })
    return jsonify(tours)

@app.route("/register", methods=["POST"])
def register():
    data = request.get_json()
    full_name = data.get("full_name")
    email = data.get("email")
    phone = data.get("phone")
    tour_id = data.get("tour_id")
    travel_date = data.get("travel_date")
    if not all([full_name, email, tour_id, travel_date]):
        return jsonify({"error": "Missing required fields"}), 400
    conn = get_db()
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO registrations (full_name, email, phone, tour_id, travel_date) VALUES (%s, %s, %s, %s, %s) RETURNING id",
        (full_name, email, phone, tour_id, travel_date)
    )
    reg_id = cur.fetchone()[0]
    conn.commit()
    cur.close()
    conn.close()
    return jsonify({"message": "Registration successful", "registration_id": reg_id}), 201

@app.route("/registrations", methods=["GET"])
def get_registrations():
    conn = get_db()
    cur = conn.cursor()
    cur.execute("""
        SELECT r.id, r.full_name, r.email, r.phone, t.name, r.travel_date, r.registered_at
        FROM registrations r
        JOIN tours t ON r.tour_id = t.id
        ORDER BY r.registered_at DESC
    """)
    rows = cur.fetchall()
    cur.close()
    conn.close()
    result = []
    for r in rows:
        result.append({
            "id": r[0], "full_name": r[1], "email": r[2], "phone": r[3],
            "tour_name": r[4], "travel_date": str(r[5]), "registered_at": str(r[6])
        })
    return jsonify(result)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

---

#### `backend/Dockerfile`

```bash
vim backend/Dockerfile
```

```dockerfile
# Backend Dockerfile - Python Flask + Gunicorn
# Alternative backend images:
#   python:3.11-alpine  -> smaller size but needs gcc for psycopg2
#   python:3.12-slim    -> newer Python version

FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

# Gunicorn: production WSGI server
# workers=3 formula: (2 x CPU cores) + 1
# Alternative: use "flask run --host=0.0.0.0" instead but not recommended for production
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "3", "--timeout", "60", "app:app"]
```

---

#### `frontend/index.html`

```bash
vim frontend/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>WanderNest - Explore. Register. Travel.</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Segoe UI', sans-serif; background: #0f0f1a; color: #e0e0e0; }
    header { background: linear-gradient(135deg, #1a1a2e, #16213e); padding: 20px 40px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #2a2a4a; }
    header h1 { color: #00d4ff; font-size: 1.8rem; letter-spacing: 2px; }
    header span { color: #aaa; font-size: 0.9rem; }
    .hero { background: linear-gradient(135deg, #0f3460, #533483); padding: 60px 40px; text-align: center; }
    .hero h2 { font-size: 2.5rem; color: #fff; margin-bottom: 10px; }
    .hero p { color: #ccc; font-size: 1.1rem; }
    .container { max-width: 1100px; margin: 0 auto; padding: 40px 20px; }
    h3 { color: #00d4ff; margin-bottom: 20px; font-size: 1.4rem; border-bottom: 1px solid #2a2a4a; padding-bottom: 10px; }
    .tours-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 20px; margin-bottom: 50px; }
    .tour-card { background: #1a1a2e; border: 1px solid #2a2a4a; border-radius: 12px; padding: 20px; cursor: pointer; transition: all 0.3s; }
    .tour-card:hover { border-color: #00d4ff; transform: translateY(-4px); }
    .tour-card.selected { border-color: #00d4ff; background: #0f2a3a; }
    .tour-card h4 { color: #fff; margin-bottom: 6px; }
    .tour-card .dest { color: #00d4ff; font-size: 0.85rem; margin-bottom: 6px; }
    .tour-card .price { color: #f0a500; font-weight: bold; margin-bottom: 8px; }
    .tour-card p { color: #aaa; font-size: 0.85rem; }
    .badge { display: inline-block; background: #533483; color: #fff; font-size: 0.75rem; padding: 2px 8px; border-radius: 20px; margin-bottom: 8px; }
    form { background: #1a1a2e; border: 1px solid #2a2a4a; border-radius: 12px; padding: 30px; max-width: 600px; }
    form label { display: block; color: #aaa; font-size: 0.85rem; margin-bottom: 4px; margin-top: 14px; }
    form input, form select { width: 100%; padding: 10px 14px; background: #0f0f1a; border: 1px solid #2a2a4a; border-radius: 8px; color: #e0e0e0; font-size: 0.95rem; }
    form input:focus, form select:focus { outline: none; border-color: #00d4ff; }
    button[type="submit"] { margin-top: 20px; width: 100%; padding: 12px; background: linear-gradient(135deg, #00d4ff, #533483); color: #fff; border: none; border-radius: 8px; font-size: 1rem; cursor: pointer; letter-spacing: 1px; }
    button[type="submit"]:hover { opacity: 0.9; }
    #message { margin-top: 15px; padding: 12px; border-radius: 8px; display: none; }
    .success { background: #0d3321; color: #00e676; border: 1px solid #00e676; }
    .error { background: #3d0000; color: #ff5252; border: 1px solid #ff5252; }
    table { width: 100%; border-collapse: collapse; margin-top: 20px; }
    th { background: #16213e; color: #00d4ff; padding: 12px; text-align: left; }
    td { padding: 10px 12px; border-bottom: 1px solid #2a2a4a; color: #ccc; font-size: 0.88rem; }
    tr:hover td { background: #1a1a2e; }
    #loading { color: #aaa; font-style: italic; }
  </style>
</head>
<body>
<header>
  <h1>🌍 WanderNest</h1>
  <span>Explore. Register. Travel.</span>
</header>
<div class="hero">
  <h2>Find Your Next Adventure</h2>
  <p>Browse curated tours and register your spot in seconds.</p>
</div>
<div class="container">
  <h3>Available Tours</h3>
  <div class="tours-grid" id="tours-grid"><p id="loading">Loading tours...</p></div>

  <h3>Register for a Tour</h3>
  <form id="reg-form">
    <label>Full Name</label>
    <input type="text" id="full_name" placeholder="Your Full Name" required />
    <label>Email</label>
    <input type="email" id="email" placeholder="you@example.com" required />
    <label>Phone</label>
    <input type="text" id="phone" placeholder="+91 9876543210" />
    <label>Select Tour</label>
    <select id="tour_id" required><option value="">-- Choose a Tour --</option></select>
    <label>Travel Date</label>
    <input type="date" id="travel_date" required />
    <button type="submit">Confirm Registration</button>
    <div id="message"></div>
  </form>

  <br/><br/>
  <h3>Recent Registrations</h3>
  <table id="reg-table">
    <thead><tr><th>Name</th><th>Email</th><th>Tour</th><th>Travel Date</th><th>Registered At</th></tr></thead>
    <tbody id="reg-body"></tbody>
  </table>
</div>

<script>
  const API = "http://" + window.location.hostname + ":5000";
  let tours = [];

  async function loadTours() {
    try {
      const res = await fetch(API + "/tours");
      tours = await res.json();
      const grid = document.getElementById("tours-grid");
      const sel = document.getElementById("tour_id");
      grid.innerHTML = "";
      tours.forEach(t => {
        grid.innerHTML += `<div class="tour-card" onclick="selectTour(${t.id})">
          <h4>${t.name}</h4>
          <div class="dest">📍 ${t.destination}</div>
          <span class="badge">⏱ ${t.duration}</span>
          <div class="price">₹${t.price.toLocaleString()}</div>
          <p>${t.description}</p>
        </div>`;
        sel.innerHTML += `<option value="${t.id}">${t.name} — ₹${t.price.toLocaleString()}</option>`;
      });
    } catch(e) { document.getElementById("loading").textContent = "Failed to load tours. Backend may be starting..."; }
  }

  function selectTour(id) {
    document.querySelectorAll(".tour-card").forEach(c => c.classList.remove("selected"));
    event.currentTarget.classList.add("selected");
    document.getElementById("tour_id").value = id;
  }

  async function loadRegistrations() {
    try {
      const res = await fetch(API + "/registrations");
      const regs = await res.json();
      const tbody = document.getElementById("reg-body");
      tbody.innerHTML = "";
      regs.forEach(r => {
        tbody.innerHTML += `<tr><td>${r.full_name}</td><td>${r.email}</td><td>${r.tour_name}</td><td>${r.travel_date}</td><td>${r.registered_at.slice(0,19)}</td></tr>`;
      });
    } catch(e) {}
  }

  document.getElementById("reg-form").addEventListener("submit", async (e) => {
    e.preventDefault();
    const msg = document.getElementById("message");
    const payload = {
      full_name: document.getElementById("full_name").value,
      email: document.getElementById("email").value,
      phone: document.getElementById("phone").value,
      tour_id: parseInt(document.getElementById("tour_id").value),
      travel_date: document.getElementById("travel_date").value
    };
    try {
      const res = await fetch(API + "/register", {
        method: "POST", headers: {"Content-Type": "application/json"}, body: JSON.stringify(payload)
      });
      const data = await res.json();
      if (res.ok) {
        msg.className = "success"; msg.style.display = "block";
        msg.textContent = "✅ " + data.message + " (ID: " + data.registration_id + ")";
        e.target.reset(); loadRegistrations();
      } else {
        msg.className = "error"; msg.style.display = "block";
        msg.textContent = "❌ " + data.error;
      }
    } catch(err) {
      msg.className = "error"; msg.style.display = "block";
      msg.textContent = "❌ Could not connect to backend.";
    }
  });

  loadTours();
  loadRegistrations();
  setInterval(loadRegistrations, 15000);
</script>
</body>
</html>
```

---

#### `frontend/Caddyfile`

```bash
vim frontend/Caddyfile
```

```
# Caddy replaces nginx/httpd
# Alternative: nginx -> place files at /usr/share/nginx/html/ and use nginx.conf
# Alternative: httpd -> place files at /usr/local/apache2/htdocs/ and use httpd.conf

:80 {
    root * /srv
    file_server
    encode gzip
    try_files {path} /index.html
}
```

---

#### `frontend/Dockerfile`

```bash
vim frontend/Dockerfile
```

```dockerfile
# Frontend Dockerfile - Caddy Web Server
# Alternative images:
#   nginx:alpine      -> place files at /usr/share/nginx/html/
#   httpd:2.4-alpine  -> place files at /usr/local/apache2/htdocs/

FROM caddy:2-alpine

# Caddy serves static files from /srv by default
COPY index.html /srv/index.html
COPY Caddyfile /etc/caddy/Caddyfile

EXPOSE 80

CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile", "--adapter", "caddyfile"]
```

---

### Step 5 — Verify All Files Are Created

```bash
find . -type f
```

Expected output:

```
./.env
./docker-compose.yml
./backend/init.sql
./backend/requirements.txt
./backend/app.py
./backend/Dockerfile
./frontend/index.html
./frontend/Caddyfile
./frontend/Dockerfile
```

---

### Step 6 — Open EC2 Security Group Ports

Go to AWS Console → EC2 → Security Groups → your instance → Inbound Rules → Add:

| Port | Protocol | Source |
|---|---|---|
| 22 | TCP | Your IP |
| 80 | TCP | 0.0.0.0/0 |
| 5000 | TCP | 0.0.0.0/0 |

---

### Step 7 — Build and Start All Containers

```bash
docker compose up --build -d
```

What this does:
- `--build` builds Docker images from your Dockerfiles
- `up` creates and starts all containers
- `-d` runs in detached/background mode

Docker Compose starts containers in this order automatically:
1. `db` starts first — PostgreSQL boots, runs `init.sql`, creates tables and inserts tour data
2. `backend` starts after db healthcheck passes — Gunicorn starts Flask API on port 5000
3. `frontend` starts last — Caddy serves UI on port 80

---

### Step 8 — Verify Everything is Running

```bash
docker compose ps
```

All 3 containers should show running status.

```bash
docker compose logs db
docker compose logs backend
docker compose logs frontend
```

---

### Step 9 — Verify Database

```bash
docker exec -it wandernest-db psql -U wanderuser -d wandernest
```

Inside PostgreSQL shell:

```sql
-- show all tables
\dt

-- check tour data inserted from init.sql
SELECT * FROM tours;

-- check registrations (empty initially)
SELECT * FROM registrations;

-- exit
\q
```

> **Command explanation:**
> - `docker exec -it` — enter a running container interactively
> - `psql` — PostgreSQL command line client
> - `-U wanderuser` — `-U` means User, not password
> - `-d wandernest` — `-d` means Database name, not password
> - Password is not passed in this command — PostgreSQL uses trust auth locally inside container

---

### Step 10 — Access the Application

```
http://<your-ec2-public-ip>              # WanderNest UI
http://<your-ec2-public-ip>:5000/health  # Backend health check
http://<your-ec2-public-ip>:5000/tours   # Tours API JSON response
```

---

## 🔧 Useful Commands

```bash
# Stop all containers
docker compose down

# Stop and delete database volume (use when credentials or schema change)
docker compose down -v

# Rebuild only backend after code change
docker compose up --build backend -d

# View live logs of all containers
docker compose logs -f

# View logs of specific container
docker compose logs backend

# Enter backend container
docker exec -it wandernest-backend bash

# Enter database container
docker exec -it wandernest-db psql -U wanderuser -d wandernest

# List all Docker images
docker images

# List all containers including stopped ones
docker ps -a
```

---

## 🌐 API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | `/health` | Backend health check |
| GET | `/tours` | Get all available tours |
| POST | `/register` | Register for a tour |
| GET | `/registrations` | Get all registrations |

---

## ❌ Problems Faced & ✅ Solutions

### Problem 1 — `docker-compose-plugin` package not found

```
Error: Unable to locate package docker-compose-plugin
```

**Cause:** Ubuntu's default apt repository does not have Docker's official packages.

**Solution:** Add Docker's official repository first then install.

```bash
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update -y
sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y
```

---

### Problem 2 — Tours not loading, 500 Internal Server Error

**Symptom:** Frontend showed "Failed to load tours. Backend may be starting..." and `/tours` endpoint returned 500 Internal Server Error.

**Cause:** `DATABASE_URL` in `.env` had `@` symbol inside the password (`Wander@2024`). PostgreSQL connection URL uses `@` as a separator between credentials and hostname. Having two `@` signs confused the URL parser and it read `2024@db` as the hostname instead of `db`.

```
# Wrong — two @ signs break the URL parser
DATABASE_URL=postgresql://wanderuser:Wander@2024@db:5432/wandernest
# psycopg2 reads hostname as "2024@db" which does not exist

# Correct — only one @ sign between password and hostname
DATABASE_URL=postgresql://wanderuser:Wander2024@db:5432/wandernest
# psycopg2 correctly reads hostname as "db"
```

**Exact error in logs:**

```
psycopg2.OperationalError: could not translate host name "2024@db" to address: Name or service not known
```

**Solution:** Remove special characters like `@` from passwords used inside connection URLs. After fixing `.env`, run:

```bash
docker compose down -v
docker compose up --build -d
```

> `-v` flag is important — it removes the old PostgreSQL data volume so the database reinitializes with the correct password. Without `-v` the old password stays in the volume and login still fails.

---

### Problem 3 — Port confusion between services

**Question:** Why is PostgreSQL port 5432 in DATABASE_URL when Flask runs on 5000?

**Answer:** These are ports of two completely different services running independently.

```
5432  ->  PostgreSQL database engine default port (internal only, not exposed to host)
5000  ->  Flask/Gunicorn application port (exposed to host)
80    ->  Caddy web server port (exposed to host)
```

PostgreSQL port 5432 is never exposed to the host machine. Backend container talks to DB container on port 5432 internally inside the Docker network. The browser never touches port 5432 directly.

---

## 📝 Key Concepts Learned

- Docker Compose `depends_on` with `condition: service_healthy` ensures proper startup order
- `.env` file is the single source of truth for all credentials — never hardcode in yaml or code
- `${VARIABLE}` syntax in docker-compose.yml reads values from `.env` automatically
- PostgreSQL auto-runs any `.sql` file placed in `/docker-entrypoint-initdb.d/` on first container start
- Named volumes (`pg_data`) persist database data across container restarts
- `docker compose down -v` wipes volumes — always use when credentials or schema change
- Special characters like `@` in passwords break connection URL parsing
- `-U` flag in psql means User, `-d` flag means Database — neither is for password
- Root user on EC2 does not need `usermod` or `newgrp` — only needed for non-root users

---

## 👨‍💻 Author

**Saroj Behera**
DevOps Intern — Greamio Technologies Pvt. Ltd.
- GitHub: [github.com/sarojbehera0610](https://github.com/sarojbehera0610)
- LinkedIn: [linkedin.com/in/saroj-behera-7bb84b204](https://linkedin.com/in/saroj-behera-7bb84b204)
- Portfolio: [sarojops.cloud](https://sarojops.cloud)