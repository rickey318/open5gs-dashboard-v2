# Open5GS Enhanced Dashboard v2

A full-featured web dashboard for managing Open5GS 4G/5G core networks. Runs as a single Docker container alongside your existing Open5GS stack — no modifications to your existing setup required.

![Dashboard](https://img.shields.io/badge/Open5GS-Dashboard-00d4ff?style=flat-square)
![Docker](https://img.shields.io/badge/Docker-Required-blue?style=flat-square&logo=docker)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)

---

## Screenshots

| Dashboard | Services | Subscribers |
|---|---|---|
| Live stats, service health, activity log | All Open5GS containers with start/stop/restart | Full CRUD, bulk CSV import/export |

---

## Features

| Section | Description |
|---|---|
| **Dashboard** | Live stat cards, service health, activity log |
| **Services** | Real-time container status, restart/start/stop |
| **Subscribers** | Add, edit, delete, search, export CSV, bulk import |
| **APNs / DNNs** | APN overview and subscriber distribution |
| **eNBs / gNBs** | Connected base stations and UE counts |
| **Alarm Feed** | Live container log events, filterable by severity |
| **Resources** | CPU and memory usage per container |
| **IP Pool** | UE IP address assignments |
| **DNS Records** | IMS and EPC DNS zone viewer |
| **PyHSS** | IMS HSS subscriber, AUC, and roaming data |
| **User Management** | Create/delete users, reset passwords (admin) |
| **Audit Log** | Full action history with user, IP, timestamp (admin) |
| **Change Password** | Self-service password change |

---

## Requirements

- Docker and Docker Compose
- Running Open5GS stack with:
  - MongoDB container named `mongo` on network `docker_open5gs_default`
  - Prometheus container named `metrics` (for eNB/UE counts)
  - PyHSS container named `pyhss` (optional, for IMS data)
- Port `3080` available on the host

---

## Quick Install

### 1. Clone the repo

```bash
git clone https://github.com/rickey318/open5gs-dashboard-v2.git
```

### 2. Copy files into your Open5GS directory

```bash
cp -r open5gs-dashboard-v2/open5gs-dashboard ~/docker_open5gs/
cp open5gs-dashboard-v2/docker-compose.dashboard.yml ~/docker_open5gs/
```

### 3. Run the installer

```bash
bash ~/docker_open5gs/open5gs-dashboard/install.sh
```

### 4. Open in your browser

```
http://YOUR_SERVER_IP:3080
```

Default login: `admin` / `admin` — **change your password after first login**

---

## Manual Install (without the install script)

```bash
# 1. Copy files
cp -r open5gs-dashboard-v2/open5gs-dashboard ~/docker_open5gs/
cp open5gs-dashboard-v2/docker-compose.dashboard.yml ~/docker_open5gs/

# 2. Build and start
cd ~/docker_open5gs
docker compose -f docker-compose.dashboard.yml up -d --build

# 3. Create admin user (first time only)
docker exec -it mongo mongosh open5gs --eval "
const crypto = require('crypto');
const salt = crypto.randomBytes(16).toString('hex');
const hash = crypto.createHmac('sha256', salt).update('admin').digest('hex');
db.dashboard_users.insertOne({
  username: 'admin', hash, salt,
  role: 'admin', mustChangePassword: false,
  createdAt: new Date()
});
print('Admin user created — login: admin / admin');
"
```

---

## Configuration

Edit `docker-compose.dashboard.yml` before deploying:

```yaml
environment:
  MONGO_URI: "mongodb://mongo:27017/open5gs"   # Your MongoDB URI
  OPEN5GS_WEBUI_URL: "http://webui:9999"       # Open5GS WebUI URL
  SESSION_SECRET: "change-me-to-a-random-string" # Change this!
  PORT: "3001"
```

To serve on a different port, change `"3080:3080"` to `"YOUR_PORT:3080"`.

---

## Directory Structure

```
open5gs-dashboard/          ← copy this folder into ~/docker_open5gs/
├── backend/
│   ├── server.js           # Express API (auth, subscribers, containers, etc.)
│   └── package.json
├── frontend/
│   └── index.html          # Single-page dashboard UI
├── nginx/
│   └── default.conf        # Reverse proxy config
├── Dockerfile
├── supervisord.conf
└── install.sh              # Automated installer

docker-compose.dashboard.yml  ← copy this to ~/docker_open5gs/
README.md
```

---

## Common Commands

```bash
# Start
cd ~/docker_open5gs && docker compose -f docker-compose.dashboard.yml up -d

# Stop
cd ~/docker_open5gs && docker compose -f docker-compose.dashboard.yml down

# View logs
docker logs open5gs-dashboard -f --tail=50

# Rebuild after updating files
docker compose -f docker-compose.dashboard.yml up -d --build --force-recreate

# Reset admin password back to 'admin'
docker exec -it mongo mongosh open5gs --eval "
const crypto = require('crypto');
const salt = crypto.randomBytes(16).toString('hex');
const hash = crypto.createHmac('sha256', salt).update('admin').digest('hex');
db.dashboard_users.updateOne(
  {username:'admin'},
  {\$set:{hash,salt,mustChangePassword:false}}
);
print('Password reset to: admin');
"
```

---

## How It Works

```
Browser → nginx (port 3080) → static index.html
                             → /api/* proxy → Express (port 3001)
                                              ├── MongoDB (subscribers, users, audit)
                                              ├── Prometheus (eNB/UE metrics)
                                              ├── Docker socket (container status/control)
                                              └── PyHSS API (IMS data)
```

The dashboard container joins `docker_open5gs_default` network and communicates with other containers by name.

---

## Troubleshooting

**Container won't start**
```bash
docker logs open5gs-dashboard --tail=30
```

**Can't login / forgot password**
```bash
# Run the reset command in the Common Commands section above
```

**eNBs/gNBs shows 0 or N/A**
- Verify Prometheus is running: `docker ps | grep metrics`
- Prometheus must be accessible at `http://metrics:9090`

**PyHSS section shows error**
- Verify PyHSS is running: `docker ps | grep pyhss`
- PyHSS API must be accessible at `http://pyhss:8080`

**Sections show blank or fail to load**
- Check the browser console (F12) for errors
- Verify the Open5GS network exists: `docker network ls | grep open5gs`
- Verify MongoDB is running: `docker ps | grep mongo`

---

## Security

- **Change** the default `admin` password after first login
- **Change** `SESSION_SECRET` in docker-compose.dashboard.yml to a random string
- The dashboard runs over **HTTP** — for production use, put it behind an HTTPS reverse proxy (nginx, Caddy)
- The Docker socket mount gives the dashboard control over containers — limit network access appropriately

---

## License

MIT — free to use, modify, and distribute.

---

## Acknowledgements

Built for [Open5GS](https://open5gs.org/) — the open source 5G and LTE mobile packet core network implementation.
