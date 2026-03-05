# Harmony Enterprise Installation Guide

This guide provides instructions for installing and managing Harmony Enterprise, the self-hosted version of [Harmony](https://harmony.ac), using Docker Compose.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Updating](#updating)
- [Password resets](#password-resets)
- [Operations](#operations)
- [Troubleshooting](#troubleshooting)
- [Backup and Recovery](#backup-and-recovery)
- [Support](#support)

## Prerequisites

### System Requirements

- **Operating System**: Linux (Ubuntu 24.04 LTS recommended)
- **CPU**: 4+ cores recommended
- **Memory**: 8GB+ RAM recommended
- **Storage**: 20GB+ disk space
- **Network**: Internet connectivity for LLM API access (Anthropic, Google AI)

### Required Software

- **Git**: for cloning this repository
- **Docker Engine**: Version 28.0 or later
- **Docker Compose**: Version 2.34 or later

### Required keys

- **Harmony License Key**: You get this at purchase.
- **Harmony Container Registry login credentials**: You get this at purchase.
- **Anthropic API Key**: For using Anthropic's language models.
- **Google Generative AI API Key**: For using Google's language models.
- **SSL Certificate and private key for HTTPS**: Valid internally usable SSL certificate and private key for your server's domain name (e.g., `harmony.mycompany.intra`). Can be self-signed if the users' browsers are configured to trust it.

## Installation

### 1. Clone this repository

```bash
git clone https://github.com/harmony-ac/harmony-enterprise
cd harmony-enterprise
```

### 2. Configure environment

Copy the example environment file and edit it with your settings. See the file's comments for details.

```bash
# Copy the example environment file
cp .env.example .env

# Edit the configuration file
nano .env
```

### 3. Configure SSL certificates

Place your SSL certificate files in the project directory or update the `.env` file to point to their locations.

```bash
# Copy your certificates
cp /path/to/your/certificate.pem ./cert.pem
cp /path/to/your/private.pem ./key.pem
```

```bash
# Or update paths in .env
SSL_CERT_PATH=/path/to/your/certificate.pem
SSL_KEY_PATH=/path/to/your/private.pem
```

<details>
<summary>How to generate self-signed certificates</summary>
<em>If you don't have SSL certificates, you can generate a local CA and a server certificate signed by it. Browsers require these to be separate — a CA cert cannot be used directly as a server cert. You will need to add the CA certificate to the trusted certificates in the users' browsers or operating systems.</em>

```bash
COMPANY="My Company"
DOMAIN="harmony.mycompany.intra"

# Step 1: Generate CA key and self-signed CA certificate, valid for 10 years
# (enter a new passphrase when prompted, or add -noenc option to skip passphrase)
openssl req -x509 -newkey rsa:2048 -days 3650 \
  -keyout harmony-ca.key -out harmony-ca.crt \
  -subj "/C=US/ST=State/L=City/O=$COMPANY/OU=IT/CN=$COMPANY CA" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"

# Step 2: Generate server key and certificate signing request (CSR)
openssl req -newkey rsa:2048 -noenc \
  -keyout key.pem -out harmony.csr \
  -subj "/C=US/ST=State/L=City/O=$COMPANY/OU=IT/CN=$DOMAIN"

# Step 3: Sign the server certificate with the CA (enter CA key passphrase you added in Step 1)
openssl x509 -req -in harmony.csr -CA harmony-ca.crt -CAkey harmony-ca.key -CAcreateserial \
  -out cert.pem -days 365 -sha256 \
  -extfile <(printf "subjectAltName=DNS:$DOMAIN\nbasicConstraints=critical,CA:FALSE\nextendedKeyUsage=serverAuth")

# Clean up CSR
rm harmony.csr

# Keep your CA key (harmony-ca.key) and server key (key.pem) secure.
```

Then distribute `harmony-ca.crt` to users so they can add it to their browser/OS trust store:

- **Linux (system-wide):** `sudo cp harmony-ca.crt /usr/local/share/ca-certificates/harmony-ca.crt && sudo update-ca-certificates`
- **Firefox:** Settings → Privacy & Security → View Certificates → Authorities → Import `harmony-ca.crt` → check _Trust this CA to identify websites_
- **Chrome/Edge on Linux:** Uses the system trust store — the system-wide step above is sufficient.
</details>

### 4. Log in to Harmony Container Registry

With the credentials you received at purchase, log in to Harmony Docker Registry:

```bash
docker login registry.harmony.ac
```

### 5. Start services

```bash
# Start all services
docker compose up -d
```

### 6. Test the installation

Open your browser and navigate to the server's domain (e.g., `https://harmony.mycompany.intra`). You should see a login page.

### Database Persistence

Database data is persisted in a Docker volume. To backup:

```bash
# Backup database
docker compose exec db pg_dump -U harmony harmony > backup.sql

# Restore database
docker compose exec -T db psql -U harmony harmony < backup.sql
```

## Updating

```bash
# Fetch the latest version of this configuration
git pull
# See changes in .env.example since latest pull, and update your .env if needed
git diff HEAD@{1}..HEAD -- .env.example
nano .env
# Pull latest images and restart services if needed
docker compose up -d --pull=always
# Optional: Remove unused images to free up space
docker image prune
```

## Password resets

The on-premises version does not have email-based password resets. To reset a user's password, use the following command:

```bash
docker compose exec web password_reset joe@exmaple.com
```

## Operations

### Start Services

```bash
# Start all services
docker compose up -d
```

### Stop Services

```bash
# Stop all services
docker compose down

# Stop and remove volumes (WARNING: This will delete all data)
docker compose down -v
```

### View Logs

```bash
# View logs for all services
docker compose logs

# View logs for specific service
docker compose logs web
docker compose logs db
docker compose logs runner
docker compose logs proxy

# Follow logs in real-time
docker compose logs -f runner
```

### Health Checks

```bash
# Check service status
docker compose ps
```

## Troubleshooting

### Common Issues

#### Port Conflicts

```bash
# Check if ports are already in use
sudo netstat -tlnp | grep -E ':(80|443)'
```

#### Services Won't Start

```bash
# Check Docker daemon
sudo systemctl status docker

# Check logs for errors
docker compose logs

# Verify environment configuration
cat .env
```

#### SSL Certificate Errors

```bash
# Verify certificate files exist and are readable (see paths in .env)
ls -la cert.pem key.pem

# Test certificate validity
openssl x509 -in cert.pem -text -noout

# Check certificate and key match
openssl x509 -noout -modulus -in cert.pem | openssl md5
openssl rsa -noout -modulus -in key.pem | openssl md5
```

### Monitoring

```bash
# Monitor resource usage
docker stats

# Check disk usage
docker system df

# Monitor logs
tail -f /var/log/nginx/access.log
```

## Backup and Recovery

### Backup Strategy

```bash
#!/bin/bash
# backup-harmony.sh
BACKUP_DIR="/backups/harmony-$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

# Backup database
docker compose exec -T db pg_dump -U harmony harmony > "$BACKUP_DIR/database.sql"

# Backup environment configuration
cp .env "$BACKUP_DIR/"

# Backup certificates
cp cert.pem key.pem "$BACKUP_DIR/"

echo "Backup completed: $BACKUP_DIR"
```

### Recovery

```bash
# Restore database from backup
docker compose exec -T db psql -U harmony harmony < /path/to/backup/database.sql

# Restore configuration
cp /path/to/backup/.env ./
cp /path/to/backup/*.pem ./

# Restart services
docker compose down && docker compose up -d
```

## Support

### Logs Collection

When contacting support, please collect:

```bash
# System information
uname -a
docker --version
docker compose version

# Service status
docker compose ps
docker compose logs > harmony-logs.txt

# Configuration (remove sensitive data)
cp .env .env.redacted
sed -i 's/=.*/=***REDACTED***/' .env.redacted
```

---

For additional support or questions, please contact support@harmony.ac.
