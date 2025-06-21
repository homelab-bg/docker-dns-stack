# Docker DNS Stack

A containerized DNS infrastructure solution for home labs combining Unbound recursive DNS resolver with AdGuard Home for network-wide ad blocking and DNS filtering, plus optional synchronisation capabilities.

## Architecture Overview

This stack implements a layered DNS architecture:

- **Unbound**: Recursive DNS resolver providing fast, secure DNS resolution
- **AdGuard Home**: DNS filtering and ad blocking with web interface
- **AdGuard-Sync**: Optional service for synchronizing AdGuard configurations across multiple instances

```
Client → AdGuard Home (Filter/Block) → Unbound (Recursive Resolution) → Internet
```

## Network Topology

### Networks
- **dns_backend** (`172.23.0.0/24`): Internal communication between services
- **dns_frontend** (`192.168.0.0/24`): External DNS access using IPvlan

### IP Assignments
- **Unbound**: `172.23.0.10` (backend only)
- **AdGuard**: `172.23.0.11` (backend), `192.168.0.2` (frontend)
- **AdGuard-Sync**: Backend network only (when enabled)

## Quick Start

### Prerequisites
- Docker Engine 20.10+
- Docker Compose 2.0+
- Network interface available for IPvlan mode - `eth0` in this case

### Basic Setup

1. **Clone and prepare environment**
   ```bash
   git clone https://github.com/homelab-bg/docker-dns-stack.git
   cd docker-dns-stack
   cp .env.example .env
   ```

2. **Configure environment variables**
   ```bash
   # Edit .env with your credentials
   nano .env
   ```

3. **Start core DNS services**
   ```bash
   # Basic DNS stack (Unbound + AdGuard)
   docker compose up -d
   ```

4. **Access AdGuard Home**
   - Web interface for initial setup: `http://192.168.0.2:3000` or `http://<your-server-ip>:3000`
   - AdGuard Home will step through an initial setup, including configuration of admin credentials, then restart for normal operations.
   - Web interface for normal operations: `http://192.168.0.2` or `http://<your-server-ip>`

### With Synchronization

```bash
# Start with AdGuard sync service
docker compose --profile sync up -d

# Access sync dashboard
# http://<your-server-ip>:8080
```

## Configuration

### Environment Variables

Update variables in `.env`:

```bash
# =============================================================================
# Docker DNS Stack Configuration
# =============================================================================

# Deployment identification
DEPLOYMENT_NAME=dns-primary
ENVIRONMENT=production

# Network Configuration
# Backend network (internal communication)
DNS_BACKEND_SUBNET=172.23.0.0/24
DNS_BACKEND_GATEWAY=172.23.0.1
UNBOUND_IP=172.23.0.10
ADGUARD_BACKEND_IP=172.23.0.11

# Frontend network (external access via IPvlan)
DNS_FRONTEND_SUBNET=192.168.0.0/24
DNS_FRONTEND_GATEWAY=192.168.0.1
DNS_FRONTEND_IP_RANGE=192.168.0.0/24
ADGUARD_FRONTEND_IP=192.168.0.5
PARENT_INTERFACE=eth0

# Port Configuration
ADGUARD_SYNC_PORT=8080

# Domain Configuration
LOCAL_DOMAIN=homelab.blue
TIMEZONE=Europe/London

# Container Resource Limits
UNBOUND_MEMORY_LIMIT=512m
ADGUARD_MEMORY_LIMIT=1g
SYNC_MEMORY_LIMIT=256m

# AdGuard Sync Configuration
ORIGIN_USERNAME=admin
ORIGIN_PASSWORD=your_secure_password_here
REPLICA1_USERNAME=admin
REPLICA1_PASSWORD=your_secure_password_here
API_USERNAME=sync_api_user
API_PASSWORD=your_api_password_here

# External Origin AdGuard (for sync)
ORIGIN_ADGUARD_URL=http://192.168.0.2

# Health Check Configuration
HEALTH_CHECK_INTERVAL=30s
HEALTH_CHECK_TIMEOUT=3s
HEALTH_CHECK_RETRIES=3

# Logging Configuration
LOG_LEVEL=warn
UNBOUND_VERBOSITY=3

# User/Group IDs
PUID=1000
PGID=1000
```

### Network Configuration

The setup uses IPvlan networking for direct network access. Ensure your host system supports this and adjust the parent interface if needed:

```bash
# In .env, modify if your interface is not eth0
PARENT_INTERFACE=eth0  # Change to your network interface
```

### DNS Upstream Configuration

- **Unbound** performs recursive DNS resolution (no forwarding)
- **AdGuard** uses Unbound as upstream (`172.23.0.10`)
- Modify `unbound/unbound.conf` to enable forwarding if desired

## Service Management

### Profiles Available
- **Default**: Core DNS services (unbound + adguard)
- **sync**: Includes AdGuard synchronization

### Common Commands

```bash
# Start core services
docker compose up -d

# Start with sync
docker compose --profile sync up -d

# View logs
docker compose logs -f adguard
docker compose logs -f unbound

# Restart a service
docker compose restart adguard

# Stop all services
docker compose down

# Update and restart
docker compose pull
docker compose up -d
```

## Monitoring & Health Checks

All services include health checks:

- **Unbound**: `unbound-control status`
- **AdGuard**: DNS lookup test for `healthcheck.local`
- **AdGuard-Sync**: HTTP endpoint check with authentication

### Metrics (when sync enabled)
- AdGuard dashboard: `http://<ag-frontend-ip>`
- Prometheus metrics: `http://<server-ip>:8080/metrics`
- Sync dashboard: `http://<server-ip>:8080`

## Security Features

- **Network isolation**: Backend services not directly accessible
- **Rate limiting**: 20 queries/second per client
- **Access control**: Restricted to specified IP ranges
- **No new privileges**: Security constraints on containers
- **DNS rebinding protection**: Private address filtering

### Access Control
The following networks are allowed DNS access:
- `10.0.0.0/8` - Class A private IP range 
- `172.16.0.0/12` - Class B private IP range
- `192.168.0.0/16` - Class C private IP range

Modify `unbound/access_lists.conf` to adjust access permissions.

## DNS Filtering

AdGuard Home includes options for several pre-configured filter lists:
- AdGuard DNS filter
- Dan Pollock's List
- Peter Lowe's Blocklist
- Ukrainian Security Filter
- Steven Black's List
- Phishing protection lists

### Custom DNS Rewrites
Local domain resolution can be configured using DNS rewrites inside AdGuard:
```
AdGuard Home → Filters → DNS rewrites → Add DNS rewrite
```

## Directory Structure

```
├── docker-compose.yml          # Main compose file
├── .env.example               # Environment template
├── unbound/
│   ├── unbound.conf          # Unbound configuration
│   ├── access_lists.conf     # Access control rules
│   └── host_entries.conf     # Local DNS entries
├── adguard/
│   ├── config/               # AdGuard config (auto-generated)
│   └── work/                 # AdGuard working files
└── adguard-sync/
    └── adguardhome-sync.yaml  # Sync configuration
```

## Synchronization Setup

The AdGuard-Sync service can synchronize configurations between multiple AdGuard instances:

1. **Origin**: Primary AdGuard instance (source of truth)
2. **Replica(s)**: Secondary instances that receive updates

Synchronization runs every 5 minutes and includes:
- Filter lists and rules
- Client settings
- DNS rewrites
- General settings

## Troubleshooting

### Common Issues

**Services won't start**
```bash
# Check logs
docker compose logs

# Verify network interfaces
ip link show eth0
```

**DNS resolution not working**
```bash
# Test Unbound directly
#docker compose exec unbound nslookup [HOST] [SERVER]
docker compose exec unbound nslookup google.com unbound

# Test AdGuard
dig @172.16.0.2 google.com

# Check service health
docker compose ps
```

**AdGuard Home password reset**
The [official wiki](https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration#password-reset) process for resetting a password is a little complicated, however you can generate a new password with the below command, and continue from step 3
```sh
mkpasswd -m bcrypt -R 10 <super-strong-password>
```

**AdGuard-Sync connection issues**
```bash
# Verify credentials in .env
# Check origin AdGuard accessibility
curl -u "admin:password" http://192.168.0.2/control/status
```

### Performance Tuning

- Adjust cache sizes in `unbound.conf`
- Modify `num-threads` based on CPU cores
- Tune AdGuard cache settings via web interface