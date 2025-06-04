# Docker DNS Stack

A containerized DNS infrastructure solution for home labs combining Unbound recursive DNS resolver with AdGuard Home for network-wide ad blocking and DNS filtering, plus optional synchronization capabilities.

## 🏗️ Architecture Overview

This stack implements a layered DNS architecture:

- **Unbound**: Recursive DNS resolver providing fast, secure DNS resolution
- **AdGuard Home**: DNS filtering and ad blocking with web interface
- **AdGuard-Sync**: Optional service for synchronizing AdGuard configurations across multiple instances

```
Client → AdGuard Home (Filter/Block) → Unbound (Recursive Resolution) → Internet
```

## 🌐 Network Topology

### Networks
- **dns_backend** (`172.23.0.0/24`): Internal communication between services
- **dns_frontend** (`172.16.0.0/24`): External DNS access using IPvlan

### IP Assignments
- **Unbound**: `172.23.0.10` (backend only)
- **AdGuard**: `172.23.0.11` (backend), `172.16.0.2` (frontend)
- **AdGuard-Sync**: Backend network only (when enabled)

## 🚀 Quick Start

### Prerequisites
- Docker Engine 20.10+
- Docker Compose 2.0+
- Network interface available for IPvlan mode - `eth0` in this case

### Basic Setup

1. **Clone and prepare environment**
   ```bash
   git clone <repository-url>
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
   - Web Interface: `http://172.16.0.2` or `http://<your-server-ip>`
   - Initial setup will prompt for admin credentials, default are admin/admin

### With Synchronization

```bash
# Start with AdGuard sync service
docker compose --profile sync up -d

# Access sync dashboard
# http://<your-server-ip>:8080
```

## 📋 Configuration

### Environment Variables

Update variables in `.env`:

```bash
# AdGuard Sync - Origin Instance
ORIGIN_USERNAME=admin
ORIGIN_PASSWORD=your_secure_password

# AdGuard Sync - Replica Instance  
REPLICA1_USERNAME=admin
REPLICA1_PASSWORD=your_secure_password

# AdGuard Sync - API Access
API_USERNAME=api_user
API_PASSWORD=your_api_password
```

### Network Configuration

The setup uses IPvlan networking for direct network access. Ensure your host system supports this and adjust the parent interface if needed:

```yaml
# In docker-compose.yml, modify if your interface isn't eth0
parent: eth0  # Change to your network interface
```

### DNS Upstream Configuration

- **Unbound** forwards to Google DNS (8.8.8.8, 8.8.4.4)
- **AdGuard** uses Unbound as upstream (`172.23.0.10`)
- Modify `unbound/unbound.conf` to change upstream resolvers

## 🔧 Service Management

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

## 📊 Monitoring & Health Checks

All services include health checks:

- **Unbound**: `unbound-control status`
- **AdGuard**: DNS lookup test for `healthcheck.local`
- **AdGuard-Sync**: HTTP endpoint check with authentication

### Metrics (when sync enabled)
- AdGuard dashboard: `http://<ag-frontend-ip>`
- Prometheus metrics: `http://<server-ip>:8080/metrics`
- Sync dashboard: `http://<server-ip>:8080`

## 🛡️ Security Features

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

## 🎯 DNS Filtering

AdGuard Home includes several pre-configured filter lists:
- AdGuard DNS filter
- Dan Pollock's List
- Peter Lowe's Blocklist
- Ukrainian Security Filter
- Steven Black's List
- Phishing protection lists

### Custom DNS Rewrites
Local domain resolution is configured for `homelab.blue`:
- `ns1.lan.homelab.blue` → `172.16.0.2`
- Various infrastructure hosts

## 📂 Directory Structure

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
    └── config/
        └── adguardhome-sync.yaml  # Sync configuration
```

## 🔄 Synchronization Setup

The AdGuard-Sync service can synchronize configurations between multiple AdGuard instances:

1. **Origin**: Primary AdGuard instance (source of truth)
2. **Replica(s)**: Secondary instances that receive updates

Synchronization runs every 5 minutes and includes:
- Filter lists and rules
- Client settings
- DNS rewrites
- General settings

## 🚨 Troubleshooting

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
dig @172.23.0.10 google.com

# Test AdGuard
dig @172.16.0.2 google.com

# Check service health
docker compose ps
```

**AdGuard Home password reset**
The [official wiki](https://github.com/AdguardTeam/AdGuardHome/wiki/Configuration#password-reset) process for resetting a password is a little complicated. 
Generate a new password with the below command, and them continnue from step 3
```sh
mkpasswd -m bcrypt -R 10 <super-strong-password>
```

**AdGuard-Sync connection issues**
```bash
# Verify credentials in .env
# Check origin AdGuard accessibility
curl -u "admin:password" http://172.16.0.10:30004/control/status
```

### Performance Tuning

- Adjust cache sizes in `unbound.conf`
- Modify `num-threads` based on CPU cores
- Tune AdGuard cache settings via web interface

## 📝 Contributing

1. Fork the repository
2. Create a feature branch
3. Test changes thoroughly
4. Submit a pull request

## 📄 License

[Add your license information here]

---

**Note**: This setup is designed for home lab environments. Ensure proper firewall rules and security measures are in place before exposing to external networks.
