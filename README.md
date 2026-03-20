# Open5GS Enhanced Dashboard

A full-featured management dashboard for Open5GS, running as a separate Docker container alongside your existing stack.

## Features

| Feature | Details |
|---|---|
| **Subscriber Management** | Add, edit, delete subscribers with IMSI, MSISDN, K/OPc keys, AMF, slices |
| **APN / DNN Management** | View all APNs/DNNs derived from subscriber sessions |
| **Service Status** | Real-time status of all Open5GS processes (AMF, SMF, UPF, MME, HSS, etc.) |
| **eNB / gNB Count** | Connected base station count from MongoDB context collections |
| **UE Count** | Connected UE count from AMF/MME context |
| **Activity Log** | Live log of actions taken in the dashboard |
| **Auto-refresh** | Data refreshes every 30 seconds automatically |

---

## Prerequisites

- Docker and Docker Compose
- Open5GS already running (with MongoDB accessible on a Docker network)
- Your Open5GS MongoDB service name (default: `mongo`)
- Your Open5GS Docker network name

---

## Quick Start

### 1. Find your Open5GS Docker network

```bash
docker network ls
# Look for something like: open5gs_default, open5gs_net, etc.
docker inspect <your-open5gs-container> | grep NetworkMode
```

### 2. Clone / copy this dashboard folder

Place the `open5gs-dashboard/` folder next to your existing `docker-compose.yml`.

### 3. Configure the compose file

Edit `docker-compose.dashboard.yml`:

```yaml
environment:
  MONGO_URI: "mongodb://mongo:27017/open5gs"   # Your MongoDB service name
  OPEN5GS_WEBUI_URL: "http://open5gs-webui:3000"

networks:
  open5gs_default:
    external: true   # Change to your actual network name
```

### 4. Launch the dashboard

```bash
# Option A: Alongside existing compose
docker-compose -f docker-compose.yml -f docker-compose.dashboard.yml up -d

# Option B: Standalone
cd open5gs-dashboard
docker-compose -f docker-compose.dashboard.yml up -d --build
```

### 5. Open the dashboard

```
http://<your-server-ip>:3080
```

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `MONGO_URI` | `mongodb://localhost:27017/open5gs` | MongoDB connection string |
| `OPEN5GS_WEBUI_URL` | `http://open5gs-webui:3000` | URL of the original Open5GS WebUI |
| `PORT` | `3001` | Internal API port (not exposed externally) |

---

## Network Configuration

The dashboard container needs to be on the **same Docker network** as your MongoDB instance.

```bash
# Find your Open5GS network name
docker network ls

# Inspect to confirm MongoDB is on it
docker network inspect <network-name>
```

Then update `docker-compose.dashboard.yml`:
```yaml
networks:
  your_actual_network_name:   # <-- change this
    external: true
```

---

## eNB / gNB & UE Counts

The dashboard reads from these MongoDB collections if they exist:

- `ueContexts` — Connected UEs
- `gnbContexts` or `enbContexts` — Connected base stations  
- `smContexts` — Active PDU sessions

These are populated **only when Open5GS is running and nodes are actively connected**. If you see `N/A`, the collections don't exist yet — this is normal when no devices are attached.

For a more accurate live count, Open5GS 2.6+ exposes a Prometheus metrics endpoint. You can extend the backend `server.js` to scrape `http://amf:9090/metrics` for real-time UE counts.

---

## Ports

| Port | Service |
|---|---|
| `3080` | Dashboard UI (nginx) |
| `3001` | Internal API (not exposed, nginx proxies to it) |

To change the external port, edit `docker-compose.dashboard.yml`:
```yaml
ports:
  - "8080:3080"   # Access on port 8080 instead
```

---

## Rebuilding After Changes

```bash
docker-compose -f docker-compose.dashboard.yml up -d --build --force-recreate
```

---

## Troubleshooting

**Can't connect to MongoDB**
```bash
docker logs open5gs-dashboard
# Look for "MongoDB connection failed"

# Test connectivity from inside the container
docker exec -it open5gs-dashboard sh
apk add mongodb-tools
mongosh "mongodb://mongo:27017/open5gs"
```

**Services all showing "unknown"**

Process detection (`pgrep`) works when Open5GS processes are on the same host or container. If running in separate containers, processes won't be visible. Extend the `/api/services` endpoint to use HTTP health checks against each service's API port instead.

**Dashboard not accessible**
```bash
# Check the container is running
docker ps | grep open5gs-dashboard

# Check logs
docker logs -f open5gs-dashboard
```
