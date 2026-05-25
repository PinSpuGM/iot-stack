# Bed Room IoT — Docker Environment

A four-service Docker Compose stack for collecting, processing, and storing IoT sensor data from ESP32 bedroom devices.

![Architecture Overview](overview.png)

---

## Docker Environment

All services run inside a shared bridge network named `docker`. Containers reach each other by **service name** (e.g. `mysql`, `emqx`) without needing IP addresses.

### Services

| Container | Image | Role |
|---|---|---|
| `emqx` | `emqx/emqx:6.2.0-gbc8c29b2` | MQTT broker — receives sensor data from IoT devices |
| `node-red` | `nodered/node-red:4.1.10-22-minimal` | Flow engine — subscribes to MQTT topics and writes to MySQL |
| `mysql` | `mysql:9.7.0` | Relational database — stores sensor readings |
| `phpmyadmin` | `phpmyadmin:5.2.3` | Web UI for MySQL |

### Persistent Volumes

| Volume | Service | Path inside container |
|---|---|---|
| `emqx_data` | emqx | `/opt/emqx/data` |
| `emqx_log` | emqx | `/opt/emqx/log` |
| `node_red_data` | node-red | `/data` |
| `mysql_data` | mysql | `/var/lib/mysql` |

Data survives container restarts and re-creation. To wipe all data: `docker compose down -v`.

### Start-up Order

```
emqx → node-red
mysql → phpmyadmin
```

---

## External Interfaces

Ports exposed from Docker to the host machine:

| Port (host) | Port (container) | Service | Protocol | Purpose |
|---|---|---|---|---|
| `1883` | `1883` | emqx | TCP / MQTT | Plain MQTT — IoT devices publish here |
| `8083` | `8083` | emqx | TCP / WS | MQTT over WebSocket |
| `8883` | `8883` | emqx | TCP / MQTTS | MQTT over SSL/TLS |
| `8084` | `8084` | emqx | TCP / WSS | MQTT over WebSocket SSL |
| `18083` | `18083` | emqx | HTTP | **EMQX Dashboard** → http://localhost:18083 |
| `1880` | `1880` | node-red | HTTP | **Node-RED Editor** → http://localhost:1880 |
| `3306` | `3306` | mysql | TCP | MySQL — direct DB access |
| `5050` | `80` | phpmyadmin | HTTP | **phpMyAdmin** → http://localhost:5050 |

### Web UIs

| Interface | URL | Default credentials |
|---|---|---|
| EMQX Dashboard | http://localhost:18083 | `admin` / `admin` |
| Node-RED Editor | http://localhost:1880 | — |
| phpMyAdmin | http://localhost:5050 | `admin` / `admin` |

> **Note:** MySQL default credentials — root password: `admin`, app user: `admin`/`admin`, database: `db`.

---

## Quick Start

```bash
# Start all services in the background
docker compose up -d

# Verify all containers are running
docker compose ps

# Tail logs
docker compose logs -f

# Stop (data volumes preserved)
docker compose down
```

---

## Data Flow

```
IoT Device (ESP32)
    │ publish → topic e.g. /home/bed/temp
    ▼ port 1883
  emqx (Broker)
    │ broker message to subscribers
    ▼ internal: emqx:1883
  node-red (Brain)
    │ parse payload → INSERT INTO db
    ▼ internal: mysql:3306
  mysql (Database)
    │
    ▼
  phpmyadmin (DB UI) — query & inspect data at http://localhost:5050
```
