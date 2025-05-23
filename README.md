# PyGeoAPI Installation Guide for Ubuntu Server 22.04

This guide provides step-by-step instructions for installing pygeoapi with Flask on a fresh Ubuntu Server 22.04 system.

## Prerequisites

- Fresh Ubuntu Server 22.04 installation
- Root or sudo access
- Internet connection

## Step 1: System Update and Basic Dependencies

```bash
# Update the system
sudo apt update && sudo apt upgrade -y

# Install essential build tools and system dependencies
sudo apt install -y \
    software-properties-common \
    apt-transport-https \
    ca-certificates \
    curl \
    wget \
    gnupg \
    lsb-release \
    build-essential \
    git \
    unzip
```

## Step 2: Install Python and Development Tools

```bash
# Install Python 3.10+ and development headers
sudo apt install -y \
    python3 \
    python3-dev \
    python3-pip \
    python3-venv \
    python3-setuptools \
    python3-wheel

# Verify Python installation
python3 --version
pip3 --version
```

## Step 3: Install Geospatial System Libraries

```bash
# Install GDAL and geospatial libraries
sudo apt install -y \
    gdal-bin \
    libgdal-dev \
    libproj-dev \
    libgeos-dev \
    libspatialite-dev \
    libsqlite3-mod-spatialite \
    proj-bin \
    proj-data

# Install additional geospatial tools
sudo apt install -y \
    libffi-dev \
    libssl-dev \
    libxml2-dev \
    libxslt1-dev \
    libyaml-dev \
    zlib1g-dev
```

## Step 4: Create System User and Directory Structure

```bash
# Create a dedicated user for pygeoapi
sudo useradd -r -m -s /bin/bash pygeoapi

# Create directory structure
sudo mkdir -p /opt/pygeoapi/{config,data,logs,static}
sudo chown -R pygeoapi:pygeoapi /opt/pygeoapi

# Switch to pygeoapi user
sudo su - pygeoapi
```

## Step 5: Set Up Python Virtual Environment

```bash
# Create virtual environment
cd /opt/pygeoapi
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Upgrade pip and basic tools
pip install --upgrade pip setuptools wheel
```

## Step 6: Install PyGeoAPI and Dependencies

```bash
# Install pygeoapi with Flask support
pip install pygeoapi[flask]

# Install additional recommended packages
pip install \
    gunicorn \
    psycopg2-binary \
    pymongo \
    pyyaml \
    requests \
    click

# Verify installation
pygeoapi --version
```

## Step 7: Create Configuration Files

### Main Configuration File

```bash
# Create main config directory
mkdir -p /opt/pygeoapi/config

# Create a basic configuration file
cat > /opt/pygeoapi/config/pygeoapi-config.yml << 'EOF'
server:
    bind:
        host: 0.0.0.0
        port: 5000
    url: http://localhost:5000
    mimetype: application/json; charset=UTF-8
    encoding: utf-8
    gzip: false
    languages:
        - en-US
    cors: true
    pretty_print: true
    limit: 10
    map:
        url: https://tile.openstreetmap.org/{z}/{x}/{y}.png
        attribution: '&copy; <a href="https://openstreetmap.org/copyright">OpenStreetMap contributors</a>'
    templates:
        path: /opt/pygeoapi/venv/lib/python3.10/site-packages/pygeoapi/templates
        static: /opt/pygeoapi/static

logging:
    level: INFO
    logfile: /opt/pygeoapi/logs/pygeoapi.log

metadata:
    identification:
        title:
            en: PyGeoAPI Server
        description:
            en: PyGeoAPI provides an API to geospatial data
        keywords:
            en:
                - geospatial
                - data
                - api
        keywords_type: theme
        terms_of_service: https://creativecommons.org/licenses/by/4.0/
        url: https://example.org
    license:
        name: CC-BY 4.0 license
        url: https://creativecommons.org/licenses/by/4.0/
    provider:
        name: Organization Name
        url: https://example.org
    contact:
        name: Lastname, Firstname
        position: Position Title
        address: Mailing Address
        city: City
        stateorprovince: Administrative Area
        postalcode: Zip or Postal Code
        country: Country
        phone: +xx-xxx-xxx-xxxx
        fax: +xx-xxx-xxx-xxxx
        email: you@example.org
        url: Contact URL
        hours: Mo-Fr 08:00-17:00
        instructions: During hours of service. Off on weekends.
        role: pointOfContact

resources: {}
EOF
```

### Environment Configuration

```bash
# Create environment file
cat > /opt/pygeoapi/.env << 'EOF'
PYGEOAPI_CONFIG=/opt/pygeoapi/config/pygeoapi-config.yml
PYGEOAPI_OPENAPI=/opt/pygeoapi/config/pygeoapi-openapi.yml
PYTHONPATH=/opt/pygeoapi/venv/lib/python3.10/site-packages
EOF
```

## Step 8: Generate OpenAPI Documentation

```bash
# Activate virtual environment
source /opt/pygeoapi/venv/bin/activate

# Set environment variables
export PYGEOAPI_CONFIG=/opt/pygeoapi/config/pygeoapi-config.yml
export PYGEOAPI_OPENAPI=/opt/pygeoapi/config/pygeoapi-openapi.yml

# Generate OpenAPI document
pygeoapi openapi generate $PYGEOAPI_CONFIG --output-file $PYGEOAPI_OPENAPI
```

## Step 9: Create Systemd Service

```bash
# Exit from pygeoapi user
exit

# Create systemd service file
sudo tee /etc/systemd/system/pygeoapi.service << 'EOF'
[Unit]
Description=PyGeoAPI Server
After=network.target

[Service]
Type=notify
User=pygeoapi
Group=pygeoapi
WorkingDirectory=/opt/pygeoapi
Environment=PYGEOAPI_CONFIG=/opt/pygeoapi/config/pygeoapi-config.yml
Environment=PYGEOAPI_OPENAPI=/opt/pygeoapi/config/pygeoapi-openapi.yml
ExecStart=/opt/pygeoapi/venv/bin/gunicorn --bind 0.0.0.0:5000 --workers 4 --timeout 120 --keepalive 2 --max-requests 1000 --max-requests-jitter 100 pygeoapi.flask_app:APP
ExecReload=/bin/kill -s HUP $MAINPID
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=pygeoapi

[Install]
WantedBy=multi-user.target
EOF
```

## Step 10: Configure Log Rotation

```bash
# Create logrotate configuration
sudo tee /etc/logrotate.d/pygeoapi << 'EOF'
/opt/pygeoapi/logs/*.log {
    daily
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 0644 pygeoapi pygeoapi
    postrotate
        systemctl reload pygeoapi || true
    endscript
}
EOF
```

## Step 11: Set Up Nginx (Optional but Recommended)

```bash
# Install Nginx
sudo apt install -y nginx

# Create Nginx configuration
sudo tee /etc/nginx/sites-available/pygeoapi << 'EOF'
server {
    listen 80;
    server_name your-domain.com;  # Replace with your domain
    
    # Security headers
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";
    
    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;
    
    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
    }
    
    location /static/ {
        alias /opt/pygeoapi/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
EOF

# Enable the site
sudo ln -s /etc/nginx/sites-available/pygeoapi /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
```

## Step 12: Configure Firewall

```bash
# Install and configure UFW
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH
sudo ufw allow ssh

# Allow HTTP and HTTPS
sudo ufw allow 'Nginx Full'

# Enable firewall
sudo ufw --force enable
sudo ufw status verbose
```

## Step 13: Start and Enable Services

```bash
# Reload systemd
sudo systemctl daemon-reload

# Start and enable pygeoapi
sudo systemctl start pygeoapi
sudo systemctl enable pygeoapi

# Check service status
sudo systemctl status pygeoapi

# Check logs
sudo journalctl -u pygeoapi -f
```

## Step 14: Verification and Testing

```bash
# Test local connection
curl -s http://localhost:5000/ | head -20

# Test through Nginx (if configured)
curl -s http://localhost/ | head -20

# Check API endpoints
curl -s http://localhost:5000/openapi | jq '.info'
curl -s http://localhost:5000/conformance | jq '.conformsTo'
curl -s http://localhost:5000/collections | jq '.collections'
```

## Step 15: Monitoring and Maintenance

### Create monitoring script

```bash
sudo tee /opt/pygeoapi/scripts/health-check.sh << 'EOF'
#!/bin/bash

# Health check script for pygeoapi
LOG_FILE="/opt/pygeoapi/logs/health-check.log"
ENDPOINT="http://localhost:5000/"

# Function to log with timestamp
log_message() {
    echo "$(date -u '+%Y-%m-%dT%H:%M:%S.%3NZ') - $1" >> "$LOG_FILE"
}

# Check if service is running
if ! systemctl is-active --quiet pygeoapi; then
    log_message "ERROR: pygeoapi service is not running"
    systemctl start pygeoapi
    exit 1
fi

# Check HTTP response
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$ENDPOINT")
if [ "$HTTP_CODE" != "200" ]; then
    log_message "ERROR: HTTP response code $HTTP_CODE from $ENDPOINT"
    exit 1
fi

log_message "INFO: Health check passed"
exit 0
EOF

sudo chmod +x /opt/pygeoapi/scripts/health-check.sh
sudo chown pygeoapi:pygeoapi /opt/pygeoapi/scripts/health-check.sh
```

### Add cron job for monitoring

```bash
# Add cron job for health checks every 5 minutes
sudo crontab -u pygeoapi -e
# Add this line:
# */5 * * * * /opt/pygeoapi/scripts/health-check.sh
```

## Step 16: Security Hardening

```bash
# Set proper file permissions
sudo chmod 750 /opt/pygeoapi
sudo chmod 640 /opt/pygeoapi/config/*.yml
sudo chmod 755 /opt/pygeoapi/logs
sudo chmod 644 /opt/pygeoapi/logs/*.log

# Configure fail2ban (optional)
sudo apt install -y fail2ban

sudo tee /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3

[sshd]
enabled = true

[nginx-http-auth]
enabled = true

[nginx-limit-req]
enabled = true
EOF

sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

## Troubleshooting

### Common Issues and Solutions

1. **Service fails to start:**
   ```bash
   sudo journalctl -u pygeoapi -n 50
   sudo systemctl status pygeoapi -l
   ```

2. **Permission issues:**
   ```bash
   sudo chown -R pygeoapi:pygeoapi /opt/pygeoapi
   sudo chmod -R 755 /opt/pygeoapi
   ```

3. **Python module issues:**
   ```bash
   sudo su - pygeoapi
   source /opt/pygeoapi/venv/bin/activate
   pip list
   pip install --upgrade pygeoapi
   ```

4. **Configuration validation:**
   ```bash
   pygeoapi config validate /opt/pygeoapi/config/pygeoapi-config.yml
   ```

## Next Steps

1. **Add data sources:** Configure your geospatial data sources in the `resources` section of the configuration file
2. **SSL/TLS:** Set up Let's Encrypt certificates for HTTPS
3. **Database integration:** Configure PostgreSQL with PostGIS or MongoDB for data storage
4. **Backup strategy:** Implement regular backups of configuration and data
5. **Monitoring:** Set up comprehensive monitoring with tools like Prometheus and Grafana

## Configuration File Locations

- Main config: `/opt/pygeoapi/config/pygeoapi-config.yml`
- OpenAPI spec: `/opt/pygeoapi/config/pygeoapi-openapi.yml`
- Service file: `/etc/systemd/system/pygeoapi.service`
- Nginx config: `/etc/nginx/sites-available/pygeoapi`
- Log files: `/opt/pygeoapi/logs/`

Your pygeoapi installation is now complete and running with Flask on Ubuntu Server 22.04!
