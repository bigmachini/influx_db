# Production-Grade InfluxDB 2.x Deployment with Docker

## Table of Contents

- [Features](#-features)
- [Prerequisites](#-prerequisites)
- [Full Deployment Guide](#-full-deployment-guide)
    - [Initial Setup](#1-initial-setup)
    - [Certificate Setup](#2-certificate-setup)
    - [Service Deployment](#3-service-deployment)
- [Configuration Details](#-configuration-details)
    - [Environment Variables](#environment-variables)
    - [Performance Tuning](#performance-tuning)
    - [Security Settings](#security-settings)
- [Monitoring Setup](#-monitoring-setup)
- [Maintenance](#-maintenance)
    - [Daily Operations](#daily-operations)
    - [Backup Procedures](#backup-procedures)
    - [Upgrade Process](#upgrade-process)
- [Troubleshooting](#-troubleshooting)
- [License](#-license)

## 🌟 Features

- **Auto-scaling architecture** handles 2M+ metrics/day
- **End-to-end encryption** with automated Let's Encrypt certificates
- **Real-time monitoring** with Telegraf integration
- **Production-optimized** configuration out of the box

## 🛠 Prerequisites

- Linux server with:
    - Docker 20.10.23+
    - docker-compose 2.17+
    - 4 CPU cores / 8GB RAM minimum
    - SSD storage (50GB+ free space)
- Domain name with DNS A record pointing to your server
- Ports 80/443 accessible from the internet

## 🚀 Full Deployment Guide

### 1. Initial Setup

```bash
# Clone the repository
git clone https://github.com/yourorg/influxdb-prod.git
cd influxdb-prod

# Create environment file
cat > .env <<EOL
DOMAIN_NAME=metrics.yourdomain.com
CERTBOT_EMAIL=admin@yourdomain.com
INFLUXDB_ADMIN_PASSWORD=$(openssl rand -hex 16)
INFLUXDB_ADMIN_TOKEN=$(openssl rand -hex 32)
HOST_DATA_PATH=/mnt/ssd/influx_data
EOL

# Create required directories
mkdir -p {config,certbot/{www,conf},telegraf}
sudo mkdir -p /opt/volumes/influx_data && sudo chown -R 1000:1000 /opt/volumes/influx_data
```

### 2. Certificate Setup

```bash
# Create nginx configuration
cat > config/nginx.conf <<'EOL'
events { worker_connections 4096; }

http {
  server {
    listen 80;
    server_name $DOMAIN_NAME;
    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    location / { return 301 https://$host$request_uri; }
  }
}
EOL

# Start temporary nginx
docker-compose up -d nginx

# Test certificate issuance
docker-compose run --rm certbot certonly --webroot -w /var/www/certbot \
  --email $CERTBOT_EMAIL --agree-tos --no-eff-email \
  -d $DOMAIN_NAME --dry-run

# Confirm success then remove --dry-run
sed -i 's/--dry-run //' docker-compose.yml
```

### 3. Service Deployment

```bash
# Create final nginx config with SSL
cat > config/nginx.conf <<'EOL'
events { worker_connections 4096; }

http {
  server {
    listen 443 ssl http2;
    server_name $DOMAIN_NAME;
    
    ssl_certificate /etc/letsencrypt/live/$DOMAIN_NAME/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$DOMAIN_NAME/privkey.pem;
    ssl_protocols TLSv1.3;
    
    location / {
      proxy_pass http://influxdb:8086;
      proxy_set_header Host $host;
      limit_req zone=influx_limit burst=2000;
    }
  }
}
EOL

# Create Telegraf config
cat > telegraf/telegraf.conf <<'EOL'
[agent]
  interval = "10s"

[[inputs.influxdb]]
  urls = ["http://influxdb:8086/metrics"]
  token = "$INFLUX_TOKEN"

[[outputs.influxdb_v2]]
  urls = ["http://influxdb:8086"]
  token = "$INFLUX_TOKEN"
  organization = "prod_org"
  bucket = "monitoring"
EOL

# Start all services
docker-compose up -d --build
```

## 🔧 Configuration Details

### Environment Variables

| Variable                  | Description                                                | Example                   |
|---------------------------|------------------------------------------------------------|---------------------------|
| `DOMAIN_NAME`             | Your public domain name for HTTPS access                   | `metrics.yourcompany.com` |
| `INFLUXDB_ADMIN_PASSWORD` | Admin password for initial setup (automatically generated) | `$(openssl rand -hex 16)` |
| `HOST_DATA_PATH`          | Absolute path to SSD storage for time-series data          | `/mnt/ssd/influx_data`    |
| `INFLUXDB_ADMIN_TOKEN`    | API token for admin operations (automatically generated)   | `$(openssl rand -hex 32)` |
| `CERTBOT_EMAIL`           | Email for Let's Encrypt certificate expiry notices         | `admin@yourdomain.com`    |

**Usage Example:**

```bash
echo "DOMAIN_NAME=metrics.prod.example.com" >> .env
echo "HOST_DATA_PATH=/mnt/nvme/influx_data" >> .env
```

### Performance Tuning

```yaml
# In docker-compose.yml

influxdb:
environment:
  - INFLUXD_ENGINE_WAL_ENABLED=true
  - INFLUXD_QUERY_MEMORY_BYTES=2147483648
ulimits:
nofile: 262144
```

### Security Settings

```yaml
# In config/nginx.conf

add_header Strict-Transport-Security "max-age=63072000" always;
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
```

## 📊 Monitoring Setup

Access the dashboard at https://yourdomain.com:

Login with credentials from .env file

Create new dashboard with this Flux query:

```flux
from(bucket: "monitoring")
|> range(start: -1h)
|> filter(fn: (r) => r._measurement == "influxdb")
|> yield(name: "system_health")
```

## 🔄 Maintenance

### Daily Operations

```bash 
# Check service status
docker-compose ps

# View logs
docker-compose logs -f --tail=100
```

### Backup Procedures

```bash
# Daily backup script
docker exec influxdb_prod influx backup \
  /var/lib/influxdb2/backups/$(date +%Y-%m-%d) \
  -t $INFLUXDB_ADMIN_TOKEN

# Restore from backup
docker exec influxdb_prod influx restore \
  /var/lib/influxdb2/backups/2023-12-01 \
  -t $INFLUXDB_ADMIN_TOKEN
```

### Upgrade Process

```bash
# Pull new images
docker-compose pull

# Recreate services with zero downtime
docker-compose up -d --no-deps --build
```

## 🛠 Troubleshooting

### Certificate Renewal Failed

```bash
docker-compose run --rm certbot renew --force-renewal
```

### High Memory Usage

```bash
docker exec influxdb_prod influx query '
  from(bucket: "monitoring")
    |> range(start: -1h)
    |> filter(fn: (r) => r._measurement == "influxdb")
    |> filter(fn: (r) => r._field == "memory")
'
```

## 📜 License

MIT License

Copyright (c) [YEAR] [FULLNAME]

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

## Need Help?
Contact support@bigmachini.net for production support.
