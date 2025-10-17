# ATLAS Docker Containerization & Deployment Guide

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Development Deployment](#development-deployment)
5. [Production Deployment](#production-deployment)
6. [Export & Import Strategy](#export--import-strategy)
7. [Data Migration](#data-migration)
8. [Configuration](#configuration)
9. [Monitoring & Maintenance](#monitoring--maintenance)
10. [Troubleshooting](#troubleshooting)
11. [Security Considerations](#security-considerations)

---

## Overview

This guide provides comprehensive instructions for containerizing and deploying the ATLAS system using Docker. The deployment strategy is designed for **portability** using `docker save/load` for offline deployment without requiring a Docker registry.

### Key Features
- **Multi-stage builds** for optimized image sizes
- **Non-root containers** for security
- **Health checks** for reliability
- **Volume management** for data persistence
- **Export/import capability** for offline deployment
- **Horizontal scaling** for RQ workers

---

## Architecture

### Container Components

| Component | Image | Port | Description |
|-----------|--------|------|-------------|
| **Frontend** | atlas-frontend | 3000 | Next.js application |
| **Flask Backend** | atlas-flask-backend | 5002 | Quart/Flask API |
| **AI-ML Backend** | atlas-ai-backend | 8000 | FastAPI ML service |
| **RQ Worker** | atlas-rq-worker | - | PCAP processing workers (scalable) |
| **Redis** | redis:7-alpine | 6379 | Queue, cache, pub/sub |
| **PostgreSQL** | postgres:14-alpine | 5432 | PCAP cache database |
| **Nginx** | nginx:alpine | 80/443 | Reverse proxy (optional) |

### Data Flow Architecture

```
    [User Browser]
           ↓
    [Nginx :80/443]  ←── Optional SSL termination
           ↓
    [Frontend :3000]
           ↓
    [Flask Backend :5002]
           ↓
    [AI-ML Backend :8000]
           ↓
    [Redis Queue :6379]
           ↓
    [RQ Workers] ←── Scale based on load
           ↓
    [File System / PostgreSQL]
```

### Volume Strategy

| Volume | Path | Purpose |
|--------|------|---------|
| `atlas_data` | `/srv/atlas-Final/data` | Analysis results, SQLite DBs |
| `atlas_uploads` | `/srv/atlas-Final/uploads` | PCAP file uploads |
| `atlas_logs` | `/srv/atlas-Final/logs` | Application logs |
| `redis_data` | `/data` | Redis persistence |
| `postgres_data` | `/var/lib/postgresql/data` | PostgreSQL data |

---

## Prerequisites

### System Requirements
- **Docker Engine**: 20.10+
- **Docker Compose**: 2.0+
- **RAM**: Minimum 8GB (16GB recommended)
- **Storage**: 20GB free space minimum
- **CPU**: 4+ cores recommended

### Software Installation

#### Ubuntu/Debian
```bash
# Install Docker
curl -fsSL https://get.docker.com | sudo bash

# Install Docker Compose
sudo apt update
sudo apt install docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

#### CentOS/RHEL
```bash
# Install Docker
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Start Docker
sudo systemctl start docker
sudo systemctl enable docker
```

---

## Development Deployment

### Quick Start (Development)

1. **Clone the repository**:
   ```bash
   cd /srv/atlas-Final
   ```

2. **Create environment file**:
   ```bash
   cp .env.example .env
   # Edit .env with your configuration
   nano .env
   ```

3. **Build and start services**:
   ```bash
   # Using existing docker-compose.yml for development
   docker-compose up -d

   # Or use production compose file
   docker-compose -f docker-compose.production.yml up -d
   ```

4. **Verify services**:
   ```bash
   docker-compose ps
   docker-compose logs -f
   ```

5. **Access application**:
   - Frontend: http://localhost:3000
   - Flask API: http://localhost:5002
   - AI-ML API: http://localhost:8000

---

## Production Deployment

### Step 1: Prepare Production Environment

1. **Create directory structure**:
   ```bash
   sudo mkdir -p /opt/atlas/volumes/{data,uploads,logs,redis,postgres}
   sudo mkdir -p /opt/atlas/volumes/data/analysis_results
   sudo mkdir -p /opt/atlas/volumes/logs/{audit,ai-backend,flask-backend,frontend,rq-worker,nginx}
   ```

2. **Set permissions**:
   ```bash
   sudo chown -R 1000:1000 /opt/atlas/volumes
   ```

### Step 2: Build Production Images

1. **Navigate to project root**:
   ```bash
   cd /srv/atlas-Final
   ```

2. **Build all images**:
   ```bash
   # Build frontend
   docker build -f docker/frontend/Dockerfile -t atlas-frontend:latest \
     --build-arg NEXT_PUBLIC_API_BASE_URL=https://your-domain.com/api \
     ./frontend

   # Build Flask backend
   docker build -f docker/flask-backend/Dockerfile -t atlas-flask-backend:latest ./main_backend

   # Build AI-ML backend
   docker build -f docker/ai-ml-backend/Dockerfile -t atlas-ai-backend:latest ./ai-ml_backend

   # Build RQ worker
   docker build -f docker/rq-worker/Dockerfile -t atlas-rq-worker:latest ./ai-ml_backend
   ```

### Step 3: Configure Production Environment

1. **Create production .env file**:
   ```bash
   cp .env.example .env.production
   nano .env.production
   ```

2. **Essential production settings**:
   ```bash
   # Environment
   ENVIRONMENT=production
   DEBUG=false

   # Security Keys (generate unique keys!)
   SECRET_KEY=$(openssl rand -hex 32)
   MASTER_KEY=$(openssl rand -hex 32)
   DB_ENCRYPTION_KEY=$(openssl rand -hex 32)
   REDIS_PASSWORD=$(openssl rand -hex 16)

   # API Keys
   OPENAI_API_KEY=your-production-api-key

   # Domain Configuration
   DOMAIN=your-domain.com
   NEXT_PUBLIC_API_BASE_URL=https://your-domain.com/api
   CORS_ORIGINS=https://your-domain.com

   # Authentication
   AUTH_MODE=saml  # or 'local' for development
   ENABLE_LOCAL_AUTH=false
   SAML_ENABLED=true
   ```

### Step 4: Deploy Services

1. **Start services**:
   ```bash
   docker-compose -f docker-compose.production.yml up -d
   ```

2. **Initialize databases**:
   ```bash
   # Wait for services to be ready
   sleep 30

   # Create initial admin user
   docker-compose -f docker-compose.production.yml exec flask-backend \
     python3 create_users.py
   ```

3. **Configure SSL/TLS** (if using Nginx):
   ```bash
   # Copy SSL certificates
   sudo cp /path/to/cert.pem /opt/atlas/ssl/
   sudo cp /path/to/key.pem /opt/atlas/ssl/

   # Update Nginx configuration
   nano docker/nginx/nginx.conf
   ```

---

## Export & Import Strategy

### Exporting for Offline Deployment

1. **Run the export script**:
   ```bash
   cd /srv/atlas-Final
   chmod +x docker/scripts/export-images.sh
   sudo ./docker/scripts/export-images.sh /tmp/atlas-export
   ```

2. **This creates**:
   - Compressed Docker images (.tar.gz)
   - Configuration files
   - Import scripts
   - Deployment instructions

3. **Transfer the archive**:
   ```bash
   # The script creates: atlas-docker-TIMESTAMP.tar.gz
   scp /tmp/atlas-export/atlas-docker-*.tar.gz user@target-server:/tmp/
   ```

### Importing on Target Server

1. **Extract archive**:
   ```bash
   cd /opt
   tar -xzf /tmp/atlas-docker-*.tar.gz
   cd atlas-docker-*
   ```

2. **Import images**:
   ```bash
   chmod +x import-images.sh
   ./import-images.sh
   ```

3. **Configure environment**:
   ```bash
   cp configs/.env.template .env
   nano .env  # Configure for production
   ```

4. **Create volumes and start**:
   ```bash
   # Create volume directories
   sudo mkdir -p /opt/atlas/volumes/{data,uploads,logs,redis,postgres}
   sudo chown -R 1000:1000 /opt/atlas/volumes

   # Start services
   docker-compose -f configs/docker-compose.production.yml up -d
   ```

---

## Data Migration

### Migrating from Existing Installation

1. **Run migration script**:
   ```bash
   cd /srv/atlas-Final
   chmod +x docker/scripts/migrate-data.sh
   sudo ./docker/scripts/migrate-data.sh
   ```

2. **Manual migration** (if needed):
   ```bash
   # Stop existing services
   ./restart-servers.sh stop

   # Copy data to volumes
   sudo cp -r /srv/atlas-Final/data/* /opt/atlas/volumes/data/
   sudo cp -r /srv/atlas-Final/uploads/* /opt/atlas/volumes/uploads/

   # Copy databases
   sudo cp /srv/atlas-Final/ai-ml_backend/agents/*.db /opt/atlas/volumes/data/

   # Set permissions
   sudo chown -R 1000:1000 /opt/atlas/volumes
   ```

### Backup and Restore

#### Create Backup
```bash
cd /srv/atlas-Final
./docker/scripts/backup-volumes.sh
```

#### Restore from Backup
```bash
./docker/scripts/restore-volumes.sh backups/backup-TIMESTAMP
```

---

## Configuration

### Environment Variables

Key environment variables for production:

| Variable | Description | Example |
|----------|-------------|---------|
| `ENVIRONMENT` | Deployment environment | `production` |
| `SECRET_KEY` | JWT signing key | Generate with `openssl rand -hex 32` |
| `MASTER_KEY` | API key encryption | Generate with `openssl rand -hex 32` |
| `DB_ENCRYPTION_KEY` | Database encryption | Generate with `openssl rand -hex 32` |
| `REDIS_PASSWORD` | Redis authentication | Generate with `openssl rand -hex 16` |
| `OPENAI_API_KEY` | OpenAI API access | Your API key |
| `DOMAIN` | Production domain | `atlas.example.com` |
| `NEXT_PUBLIC_API_BASE_URL` | Backend API URL | `https://atlas.example.com/api` |

### Resource Limits

Adjust in `docker-compose.production.yml`:

```yaml
deploy:
  resources:
    limits:
      cpus: '2.0'
      memory: 2G
    reservations:
      cpus: '1.0'
      memory: 1G
```

### Scaling Workers

Scale RQ workers based on load:

```bash
# Scale to 5 workers
docker-compose -f docker-compose.production.yml up -d --scale rq-worker=5
```

---

## Monitoring & Maintenance

### Health Checks

Monitor service health:

```bash
# Check all services
docker-compose -f docker-compose.production.yml ps

# Check specific service health
docker inspect atlas-ai-backend --format='{{.State.Health.Status}}'
```

### Logs

View logs:

```bash
# All services
docker-compose -f docker-compose.production.yml logs -f

# Specific service
docker-compose -f docker-compose.production.yml logs -f ai-backend

# Save logs
docker-compose -f docker-compose.production.yml logs > atlas-logs-$(date +%Y%m%d).txt
```

### Performance Monitoring

```bash
# Container stats
docker stats

# Redis monitoring
docker exec atlas-redis redis-cli INFO

# Check queue status
docker exec atlas-redis redis-cli -n 0 LLEN rq:queue:default
```

### Updates

1. **Pull latest code**:
   ```bash
   git pull origin main
   ```

2. **Rebuild images**:
   ```bash
   docker-compose -f docker-compose.production.yml build
   ```

3. **Rolling update**:
   ```bash
   docker-compose -f docker-compose.production.yml up -d --no-deps ai-backend
   docker-compose -f docker-compose.production.yml up -d --no-deps flask-backend
   docker-compose -f docker-compose.production.yml up -d --no-deps frontend
   ```

---

## Troubleshooting

### Common Issues

#### Port Conflicts
```bash
# Check what's using a port
sudo lsof -i :3000

# Change port in docker-compose.yml
ports:
  - "3001:3000"  # Changed external port
```

#### Permission Errors
```bash
# Fix volume permissions
sudo chown -R 1000:1000 /opt/atlas/volumes

# If using different UID/GID
DOCKER_UID=1001 DOCKER_GID=1001 docker-compose up -d
```

#### Memory Issues
```bash
# Check memory usage
docker system df

# Clean up
docker system prune -a --volumes

# Increase memory limits in docker-compose.yml
```

#### Service Won't Start
```bash
# Check logs
docker-compose -f docker-compose.production.yml logs service-name

# Rebuild image
docker-compose -f docker-compose.production.yml build --no-cache service-name

# Check dependencies
docker-compose -f docker-compose.production.yml up -d redis postgres
```

### Debug Mode

Enable debug mode for troubleshooting:

```bash
# In .env file
DEBUG=true
LOG_LEVEL=DEBUG

# Restart services
docker-compose -f docker-compose.production.yml restart
```

---

## Security Considerations

### Container Security

1. **Non-root users**: All containers run as non-root (UID 1001)
2. **Read-only filesystems**: Where possible
3. **Security options**: `no-new-privileges:true`
4. **Network isolation**: Internal networks for backend services

### Secret Management

1. **Environment files**: Never commit `.env` files
2. **Secret rotation**: Regularly rotate API keys and passwords
3. **Encryption keys**: Use different keys for different purposes
4. **Volume encryption**: Consider encrypted volumes for sensitive data

### Network Security

1. **Firewall rules**:
   ```bash
   # Only expose necessary ports
   sudo ufw allow 80/tcp
   sudo ufw allow 443/tcp
   ```

2. **SSL/TLS**: Always use HTTPS in production
3. **CORS**: Configure allowed origins explicitly

### Monitoring & Auditing

1. **Log aggregation**: Centralize logs for analysis
2. **Audit trails**: Enable audit logging in production
3. **Vulnerability scanning**:
   ```bash
   # Scan images for vulnerabilities
   docker scout cves atlas-frontend:latest
   ```

---

## Appendix

### Quick Reference Commands

```bash
# Start all services
docker-compose -f docker-compose.production.yml up -d

# Stop all services
docker-compose -f docker-compose.production.yml down

# Restart a service
docker-compose -f docker-compose.production.yml restart ai-backend

# View logs
docker-compose -f docker-compose.production.yml logs -f

# Execute command in container
docker-compose -f docker-compose.production.yml exec flask-backend bash

# Scale workers
docker-compose -f docker-compose.production.yml up -d --scale rq-worker=3

# Backup volumes
./docker/scripts/backup-volumes.sh

# Export for offline deployment
./docker/scripts/export-images.sh
```

### File Structure

```
/srv/atlas-Final/
├── docker/
│   ├── frontend/
│   │   └── Dockerfile
│   ├── flask-backend/
│   │   ├── Dockerfile
│   │   └── docker-entrypoint.sh
│   ├── ai-ml-backend/
│   │   ├── Dockerfile
│   │   └── docker-entrypoint.sh
│   ├── rq-worker/
│   │   ├── Dockerfile
│   │   └── docker-entrypoint.sh
│   ├── nginx/
│   │   └── nginx.conf
│   └── scripts/
│       ├── export-images.sh
│       ├── migrate-data.sh
│       ├── backup-volumes.sh
│       └── restore-volumes.sh
├── docker-compose.yml              # Development
├── docker-compose.production.yml   # Production
├── .env.example                    # Environment template
└── DOCKER_DEPLOYMENT_GUIDE.md     # This file
```

### Support

For issues or questions:
1. Check the logs first: `docker-compose logs`
2. Review this guide's troubleshooting section
3. Verify all environment variables are set correctly
4. Ensure sufficient system resources

---

**Last Updated**: October 2024
**Version**: 1.0.0
**Status**: Production Ready