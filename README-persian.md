[فارسی](README-persian.md) | [English](README.md)

# راه‌اندازی Prometheus

> ### نکته
>
> اگر قصد دارید **Prometheus** را روی سیستمی نصب کنید که از قبل **Docker** دارد، یا می‌خواهید ابزارهایی مثل **Grafana** را هم کنار آن اجرا کنید، پیشنهاد ‌می‌کنم از نسخه‌ی **Docker**‌ی Prometheus استفاده کنید تا سرویس‌تان منعطف‌تر باشد و در آینده هم نگهداری (maintenance) آن برای شما ساده‌تر شود.
>
> اما اگر این ماشین فقط قرار است Prometheus را اجرا کند و Docker هم ندارد، بهتر است برای جلوگیری از سربار اضافی، نسخه‌ی باینری را مستقیم نصب کنید.

## نصب نسخه‌ی باینری Prometheus

برای راه‌اندازی نسخه‌ی باینری Prometheus، دستورات زیر را اجرا کنید:

```bash
# نکته‌ مهم
# اگر معماری سیستمی که Prometheus قرار است روی آن نصب شود amd64 نیست دستورات زیر به درستی برای شما کار نخواهند کرد.
# برای مثال اگر معماری شما arm64 است باید تمامی  `amd64` ها را با `arm64` در کامند‌های زیر جایگزین کنید:

VER="$(curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest | grep -m1 '"tag_name"' | cut -d'"' -f4)"
TAR="prometheus-${VER#v}.linux-amd64.tar.gz"

wget "https://github.com/prometheus/prometheus/releases/download/$VER/$TAR" \
  && tar -xvf "$TAR" \
  && cd "prometheus-${VER#v}.linux-amd64"

./prometheus
```

بعد از اجرا، Prometheus روی پورت **9090** در دسترس است. برای دسترسی به پنل وب:

- `http://{IP_ADDRESS}:9090`

> ### نکته مهم
>
> اگر Prometheus را روی یک سرور اجرا می‌کنید، پورت Prometheus باید برای دسترسی به پنل وب قابل دسترسی باشد.
> اگر فایروال فعال دارید، باید پورت را با **ufw** یا **iptables** یا **nftables** باز کنید.

> ### نکته
>
> برای تغییر پورت (مثلا از 9090 به 9091)، Prometheus را با این فلگ اجرا کنید:
>
> `./prometheus --web.listen-address=":9091"`
>
> برای دیدن همه گزینه‌هایی که میتوانید تنظیم کنید هم دستور پایین را اجرا کنید:
>
> `./prometheus --help`

برای اجرای Prometheus با فایل کانفیگ:

```bash
./prometheus --config.file=prometheus.yml
```

> ### نکته
>
> فایل `prometheus.yml` فایل اصلی کانفیگ پرومتئوس است. می‌توانید طبق توضیحات [داکیومنت رسمی کانفیگ پرومتئوس](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) این فایل را کاستومایز کنید.

اگر ماشین ری‌استارت شود، اجرای دستی Prometheus متوقف می‌شود. برای اینکه به شکل سرویس اجرا شود، مراحل زیر را انجام دهید.

## راه‌اندازی Prometheus به‌صورت سرویس systemd (پیشنهادی)

### ۱) ساخت کاربر و دایرکتوری‌ها

```bash
sudo useradd --no-create-home --shell /usr/sbin/nologin prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

### ۲) انتقال باینری‌ها

```bash
sudo cp prometheus promtool /usr/local/bin/
sudo chown root:root /usr/local/bin/prometheus /usr/local/bin/promtool
```

### ۳) انتقال فایل کانفیگ

```bash
sudo cp prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus
```

> ### نکته
>
> اگر قالب‌های پیش‌فرض پنل وب را می‌خواهید، این پوشه‌ها را هم کپی کنید:
>
> ```bash
> sudo cp -r consoles console_libraries /etc/prometheus/
> sudo chown -R prometheus:prometheus /etc/prometheus
> ```

### ۴) ساخت فایل سرویس

فایل `/etc/systemd/system/prometheus.service` را با محتوای زیر بسازید:

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

در نهایت سرویس را فعال و اجرا کنید:

```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
```

دسترسی به پنل وب:

- `http://{IP_ADDRESS}:9090`

## تغییر پورت اجرایی Prometheus

برای تغییر پورت (مثلا از 9090 به 9091)، از فلگ زیر استفاده کنید:

```bash
./prometheus --web.listen-address=":9091"
```

در حالت systemd باید آن را به خط `ExecStart` اضافه کنید:

```ini
--web.listen-address=":9091"
```

## راه‌اندازی Prometheus با Docker Compose

یک فایل `docker-compose.yml` به شکل زیر بسازید:

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

### ساختار پیشنهادی پوشه‌ها برای داکر

```text
.
├─ docker-compose.yml
└─ prometheus/
   ├─ prometheus.yml
   ├─ rules/
   │  └─ node_alerts.yml
   └─ targets/
```

### کانفیگ Prometheus در داکر (برای scrape کردن Node Exporter)

محتوای فایل `./prometheus/prometheus.yml`:

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

سپس کانتینر را اجرا کنید:

```bash
docker compose up -d
```

پنل وب:

- `http://{IP_ADDRESS}:9090`

## کانفیگ Prometheus برای scrape کردن Node Exporter (پورت 9100)

در زیر یک `prometheus.yml` ساده آمده است که موارد زیر را scrape می‌کند:

- خود Prometheus (پورت `9090`)
- ابزار Node Exporter (پورت `9100`)

> ### نکته مهم
>
> به جای `{NODE_IP}` آی‌پی سرور یا hostname خود را قرار دهید (یا اگر Node Exporter روی همین ماشین است از `localhost` استفاده کنید).

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

## اعتبارسنجی کانفیگ

در نصب باینری (در پوشه Prometheus):

```bash
./promtool check config prometheus.yml
```

در نصب systemd:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

## عیب‌یابی (Troubleshooting)

- **نمی‌توانید به `{IP_ADDRESS}:9090` متصل شوید**
  - مطمئن شوید که Prometheus در حال اجراست و پورت آن در فایروال ماشین / کلاد باز است.
- **تارگت Node Exporter در حالت DOWN است**
  - چک کنید که Node Exporter در حال اجراست و روی پورت `:9100` قابل دسترسی است:
    - `curl http://{NODE_IP}:9100/metrics`
  - اگر از ماشین دیگری scrape می‌کنید، مطمئن شوید پورت `9100/tcp` مجاز است.

## اضافه کردن Ruleهای هشدار (Node down, High CPU, High Memory)

یک فایل rule بسازید، مثلاً `/etc/prometheus/rules/node_alerts.yml` (برای نصب باینری) یا `./prometheus/rules/node_alerts.yml` (برای داکر کمپوز):

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

> ### نکته
>
> ابزار Prometheus قوانین را به صورت محلی (local) بررسی می‌کند، اما برای **ارسال هشدار** (ایمیل/Slack و غیره) معمولاً باید **Alertmanager** را پیکربندی کنید.
> بدون Alertmanager هم می‌توانید هشدارها را در بخش **Status -> Alerts** در پنل وب Prometheus ببینید.

---

> ### نکته
>
> در آینده در مورد راه‌اندازی و استفاده از **Alertmanager** در یک ریپازیتوری دیگر صحبت خواهم کرد :)
