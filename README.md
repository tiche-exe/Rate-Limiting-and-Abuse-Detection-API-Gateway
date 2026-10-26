# Rate Limiting and Abuse Detection API Gateway

A Spring Boot API gateway that sits in front of a simple notes service, enforcing API-key checks, rate limits and abuse detection, with a Grafana dashboard for monitoring.

## About this repository

This is my own version of a project I worked on as part of a team for CS335 (Project 3). The original team repository is here: https://github.com/ozdenur99/CS335-Project3

I took part in building the original project. This repository is a separate copy that I now maintain on my own, and any changes made here are my own work and are not part of the original team repository. The core design (gateway filter chain, rate-limiting algorithms, logging and metrics) comes from the team project, and credit for that shared work belongs to everyone who contributed to it.

### What I've changed or added

- _Add your own changes here, for example: "Rewrote the setup guide", "Added X feature", "Fixed Y"._

---

## Architecture

```
[ Client ]  ──►  [ API Gateway 1 :8080 ]  ──┐
                 [ API Gateway 2 :8082 ]  ──┼──►  [ Backend Service :8081 ]
                                            │
                         [ Redis :6379 ] ◄──┘   (shared rate-limit counters and blacklist)

[ Grafana :3000 ]  reads metrics from the gateways
[ Locust :8089 ]   generates load for testing
```

| Technology | Details |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.5.11 (Spring MVC) |
| Build tool | Maven Wrapper |
| Shared state | Redis |
| Monitoring | Grafana (with the Infinity data source plugin) |
| Load testing | Locust |
| Runtime | Docker Compose |

## Features

| Component | Behaviour |
|---|---|
| API key validation | Missing or invalid `X-API-Key` header returns `401 Unauthorized` |
| Rate limiting | Token bucket, fixed window and sliding window algorithms; exceeding the limit returns `429 Too Many Requests` |
| Abuse detection | Detects spikes and repeated failures; blocked clients get `403 Forbidden` |
| Request logging and metrics | Records timestamp, key, IP, path, decision, reason, algorithm and latency for every request |
| Bot detection | Flags IPs by request volume (LOW over 50, MEDIUM over 100, HIGH over 200) |
| Request forwarding | Valid requests are forwarded to the backend service |

Every request passes through the filter chain in this order: API key check, rate limit, abuse detection, logging, then forwarding to the backend.

---

## Running the project

Everything runs in Docker, so you don't need to install Java, Maven or Redis yourself.

### 1. Prerequisites

- **Git**
- **Docker Desktop**, installed and running
  - On Windows, Docker Desktop needs WSL 2. If it shows a WSL error on startup, open PowerShell as Administrator and run:
    ```powershell
    wsl --install --no-distribution
    wsl --update
    wsl --shutdown
    ```
    Then restart your PC and reopen Docker Desktop.

### 2. Clone the repository

```bash
git clone https://github.com/YOUR-USERNAME/YOUR-REPO.git
cd YOUR-REPO
```

### 3. Start everything

From the project root (the folder containing `docker-compose.yml`):

```bash
docker compose up --build
```

The first run takes a few minutes while the backend and gateways build. Leave this terminal open, because closing it stops the services.

This starts:

| Service | URL |
|---|---|
| Gateway 1 | http://localhost:8080 |
| Gateway 2 | http://localhost:8082 |
| Backend | http://localhost:8081 |
| Redis | localhost:6379 |
| Grafana | http://localhost:3000 |
| Locust | http://localhost:8089 |

### 4. Check it works

Open http://localhost:8080/metrics in a browser. If it loads, the gateway is running.

Send a test request (PowerShell):

```powershell
curl -Uri "http://localhost:8080/api/test123/notes" -Headers @{"X-API-Key"="dev-key-token"}
```

Mac or Linux:

```bash
curl http://localhost:8080/api/test123/notes -H "X-API-Key: dev-key-token"
```

You should get `[]` back. A bad key (for example `bad-key`) returns `401`.

### 5. Stop everything

Press `Ctrl+C` in the terminal running Compose, then run:

```bash
docker compose down
```

---

## Setting up the Grafana dashboard

Grafana runs at http://localhost:3000 with anonymous admin access, so no login is needed.

The dashboard (`dashboard.json`) refers to two data sources by fixed IDs: one for Gateway 1 and one for Gateway 2. A fresh Grafana doesn't have them, so create them first.

### 1. Create the two data sources

Run this in a new PowerShell window while the containers are running:

```powershell
$gw = @(
  @{ name="Gateway 1"; uid="dfk5lf85xepz4f"; url="http://gateway1:8080" },
  @{ name="Gateway 2"; uid="afk5lqu9tk3k0d"; url="http://gateway2:8080" }
)
foreach ($g in $gw) {
  $body = @{
    name = $g.name
    uid = $g.uid
    type = "yesoreyeram-infinity-datasource"
    access = "proxy"
    url = $g.url
    jsonData = @{ allowedHosts = @($g.url) }
  } | ConvertTo-Json -Depth 5
  Invoke-RestMethod -Method Post -Uri "http://localhost:3000/api/datasources" -ContentType "application/json" -Body $body
}
```

The URLs use the Docker service names (`gateway1`, `gateway2`) because Grafana runs inside Docker, where `localhost` would point at Grafana itself.

### 2. Import the dashboard

1. Go to http://localhost:3000/dashboard/import
2. Upload `dashboard.json` from the project root, then click **Load**.
3. Click **Import**.

### 3. Set the time range

The saved dashboard has an old fixed time range. At the top right, change it to **Last 15 minutes** and turn on auto-refresh.

---

## Generating traffic

The dashboard stays empty until requests go through the gateways.

### PowerShell script

This sends 120 requests to each gateway using a mix of valid and invalid keys:

```powershell
$keys = "dev-key-token","dev-key-fixed","dev-key-sliding","dev-key-business","bad-key"
foreach ($port in 8080, 8082) {
    1..120 | ForEach-Object {
        $k = $keys | Get-Random
        try {
            Invoke-WebRequest -Uri "http://localhost:$port/api/test123/notes" -Headers @{"X-API-Key"=$k} | Out-Null
        } catch {}
    }
}
```

It prints nothing when it works. Afterwards check http://localhost:8080/metrics and http://localhost:8082/metrics, then refresh Grafana.

### Locust

1. Open http://localhost:8089
2. Set the host to `http://gateway1:8080`, choose a number of users, and click **Start**.

---

## API keys

| Key | Algorithm |
|---|---|
| `dev-key-token` | Token bucket |
| `dev-key-fixed` | Fixed window |
| `dev-key-sliding` | Sliding window |
| `dev-key-business` | Token bucket (higher limit) |

These are local development keys only. All gateway requests must include an `X-API-Key` header.

## Backend endpoints

Accessed through the gateway, where `{guid}` is a user or tenant scope such as `test123`:

```
POST   /api/{guid}/notes        Create a note
GET    /api/{guid}/notes        List all notes
PUT    /api/{guid}/notes/{id}   Update a note
DELETE /api/{guid}/notes/{id}   Delete a note
```

## Metrics endpoints

No API key is needed for these.

| Endpoint | Description |
|---|---|
| `GET /metrics` | Totals, blocked, allowed and per-key counts |
| `GET /metrics/logs` | Last 100 requests with full details |
| `GET /metrics/logs/filter?...` | Filter by decision, reason, apiKey or algorithm |
| `GET /metrics/logs/export/json` | Download logs as JSON |
| `GET /metrics/logs/export/csv` | Download logs as CSV |
| `GET /metrics/suspicious` | IPs flagged as possible bots |
| `GET /metrics/suspicious/risk` | Flagged IPs grouped by HIGH, MEDIUM and LOW risk |
| `GET /metrics/latency?apiKey=...` | p50, p95 and p99 latency, status codes and risk score per client |
| `GET /metrics/risk` | Risk score for all tracked clients |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Docker Desktop shows a WSL error | Run the WSL commands in the prerequisites, then restart your PC |
| `docker compose` command fails | Make sure Docker Desktop is running and you're in the project root |
| Port already in use | Close whatever is using 3000, 6379, 8080, 8081, 8082 or 8089 |
| Grafana panels show "No data" | Check the time range is **Last 15 minutes**, that both data sources exist, and that you've sent traffic to both gateways |
| Grafana says a data source is not found | Run the data source script again |
| `/metrics` shows zero counts | Requests aren't reaching that gateway, so check the port (8080 or 8082) |

Metrics are held in memory per gateway and reset when the containers restart.

## Credits

Based on the CS335 Project 3 team repository: https://github.com/ozdenur99/CS335-Project3
