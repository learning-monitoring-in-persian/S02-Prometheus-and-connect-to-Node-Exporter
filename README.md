[English](README.md) | [فارسی](README-persian.md)

# Set up Prometheus

> [!NOTE]
> If you plan to install **Prometheus** on a machine that already has **Docker**, or if you want to run tools like **Grafana** alongside it, I recommend using the **Docker** version of Prometheus. This keeps your setup more flexible And it makes the maintenance of Prometheus will be easier in the future.
>
> However, if the machine will only run Prometheus and doesn’t have Docker, it’s better to avoid extra overhead and install the Prometheus binary directly.

## Install the Prometheus binary

To set up the Prometheus binary, run the commands below:

```bash
# Important Note
# If your system architecture is not amd64 the command below will not work for you.
# For example if it is arm64, replace all `amd64` with `arm64` in the commands below:

VER="$(curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest | grep -m1 '"tag_name"' | cut -d'"' -f4)"
TAR="prometheus-${VER#v}.linux-amd64.tar.gz"

wget "https://github.com/prometheus/prometheus/releases/download/$VER/$TAR" \
  && tar -xvf "$TAR" \
  && cd "prometheus-${VER#v}.linux-amd64"

./prometheus
```

After that, Prometheus will be available on port **9090**. You can access the UI at:

- `http://{IP_ADDRESS}:9090`

> [!IMPORTANT]
> If you run Prometheus on a server, the Prometheus port must be accessible to view the UI.
> If you have an active firewall, allow the port using **ufw**, **iptables**, or **nftables**.

> [!NOTE]
> To change the listening port (example: from 9090 to 9091), run Prometheus with:
>
> `./prometheus --web.listen-address=":9091"`
>
> For a full list of options:
>
> `./prometheus --help`

To run Prometheus using a config file:

```bash
./prometheus --config.file=prometheus.yml
```

> [!NOTE]
> The `prometheus.yml` file is the main configuration file for Prometheus. You can customize it according to your needs based on the official [Prometheus Configuration Documentation](https://prometheus.io/docs/prometheus/latest/configuration/configuration/).

If your machine restarts, the process will stop. To run Prometheus as a service, follow the steps below.

## Run Prometheus as a systemd service (recommended)

### 1) Create user & directories

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

### 2) Install binaries

```bash
sudo cp prometheus promtool /usr/local/bin/
sudo chown root:root /usr/local/bin/prometheus /usr/local/bin/promtool
```

### 3) Install the config file

```bash
sudo cp prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus
```

> [!NOTE]
> If you want the built-in web console templates, copy these folders too:
>
> ```bash
> sudo cp -r consoles console_libraries /etc/prometheus/
> sudo chown -R prometheus:prometheus /etc/prometheus
> ```

### 4) Create the systemd unit

Create `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Finally, enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

Prometheus UI:

- `http://{IP_ADDRESS}:9090`

## Change Prometheus listening port

To change the UI port (example: from 9090 to 9091), use the option below:

```bash
./prometheus --web.listen-address=":9091"
```

For systemd, add it to `ExecStart`:

```ini
--web.listen-address=":9091"
```

## Set up Prometheus with Docker Compose

Example `docker-compose.yml`:

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.9.1
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - ./prometheus/targets:/etc/prometheus/targets:ro
      - prometheus_data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

volumes:
  prometheus_data:
```

### Suggested folder layout

```text
.
├─ docker-compose.yml
└─ prometheus/
   ├─ prometheus.yml
   ├─ rules/
   │  └─ node_alerts.yml
   └─ targets/
```

### Docker Prometheus config (scrape Node Exporter)

`./prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["prometheus:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["{NODE_IP}:9100"]
```

Then start:

```bash
docker compose up -d
```

Prometheus UI:

- `http://{IP_ADDRESS}:9090`

## Configure Prometheus to scrape Node Exporter (port 9100)

Below is a simple `prometheus.yml` that scrapes:

- Prometheus itself (`:9090`)
- Node Exporter (`:9100`)

> [!IMPORTANT]
> Replace `{NODE_IP}` with your server IP or hostname (or use `localhost` if Node Exporter runs on the same machine).

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["{NODE_IP}:9100"]
```

## Validate your config

With binary install (in the Prometheus directory):

```bash
./promtool check config prometheus.yml
```

With systemd install:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

## Troubleshooting

- **Can’t reach `{IP_ADDRESS}:9090`**
  - Ensure Prometheus is running and the port is open in your firewall / cloud firewall rules.
- **Node Exporter target is DOWN**
  - Check Node Exporter is running and reachable on `:9100`:
    - `curl http://{NODE_IP}:9100/metrics`
  - Ensure port `9100/tcp` is allowed if scraping from another machine.

## Add alert rules (Node down, High CPU, High Memory)

Create a rules file, e.g. `/etc/prometheus/rules/node_alerts.yml` (binary install) or `./prometheus/rules/node_alerts.yml` (Docker Compose example):

```yaml
groups:
  - name: node-alerts
    rules:
      - alert: NodeDown
        expr: up{job="node"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Node Exporter is down (instance={{ $labels.instance }})"
          description: "Prometheus cannot scrape Node Exporter for more than 2 minutes."

      - alert: HighCPUUsage
        expr: (100 - (avg by (instance) (rate(node_cpu_seconds_total{job="node",mode="idle"}[5m])) * 100)) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage (instance={{ $labels.instance }})"
          description: "CPU usage > 85% for 5 minutes."

      - alert: HighMemoryUsage
        expr: ((1 - (node_memory_MemAvailable_bytes{job="node"} / node_memory_MemTotal_bytes{job="node"})) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage (instance={{ $labels.instance }})"
          description: "Memory usage > 80% for 5 minutes."
```

> [!NOTE]
> Prometheus evaluates rules locally, but to **send alerts** (email/Slack/etc.) you typically configure **Alertmanager**.
> Without Alertmanager, you can still see alerts under **Status -> Alerts** in the Prometheus UI.

---

> [!NOTE]
> I will talk about setting up **Alertmanager** in another repo in the future :)
