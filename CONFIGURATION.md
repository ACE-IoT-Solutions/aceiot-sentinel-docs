# Configuration Reference

Complete reference for all ACE IoT Sentinel Container environment variables.

## Quick Start

1. Copy the example configuration:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` with your settings
3. Start the container:
   ```bash
   docker-compose up -d
   ```

## Required Configuration

### BACNET_ADDRESS (Required)

The BACnet/IP address this container will use to communicate on the network.

- **Format:** `IP/CIDR:PORT`
- **Default:** None (must be set)
- **Example:** `192.168.1.100/24:47808`

```bash
# Standard BACnet/IP on network 192.168.1.0/24
BACNET_ADDRESS=192.168.1.100/24:47808

# Different subnet
BACNET_ADDRESS=10.0.50.25/16:47808

# Custom port
BACNET_ADDRESS=192.168.1.100/24:47809
```

**Important:** This IP must be on the same network as your BACnet devices for discovery to work.

---

## VOLTTRON Platform Configuration

### VOLTTRON_VIP_ADDRESS

VIP (VOLTTRON Interconnect Protocol) address for VOLTTRON communication.

- **Format:** `tcp://HOST:PORT`
- **Default:** `tcp://127.0.0.1:22917`
- **Example:** `tcp://0.0.0.0:22917` (allow external connections)

```bash
# Local only (recommended)
VOLTTRON_VIP_ADDRESS=tcp://127.0.0.1:22917

# Allow external VIP connections
VOLTTRON_VIP_ADDRESS=tcp://0.0.0.0:22917
```

**Note:** Port 22917 is used by default to avoid conflicts with other VOLTTRON instances on port 22916.

### VOLTTRON_WEB_ADDRESS

VOLTTRON web service address (optional, disabled by default).

- **Format:** `http://HOST:PORT`
- **Default:** Empty (disabled)
- **Example:** `http://0.0.0.0:8080`

```bash
# Disabled (recommended)
VOLTTRON_WEB_ADDRESS=

# Enable VOLTTRON web service
VOLTTRON_WEB_ADDRESS=http://0.0.0.0:8080
```

**Warning:** This is separate from the Grasshopper web UI. Usually not needed.

### VOLTTRON_INSTANCE_NAME

Instance name for this VOLTTRON platform.

- **Default:** `sentinel-edge`
- **Example:** `building-a-sentinel`

```bash
VOLTTRON_INSTANCE_NAME=sentinel-edge
```

### VOLTTRON_MESSAGE_BUS

Message bus type for VOLTTRON.

- **Options:** `zmq` or `rmq`
- **Default:** `zmq`

```bash
VOLTTRON_MESSAGE_BUS=zmq
```

**Note:** RabbitMQ (`rmq`) requires additional configuration not covered here.

---

## BACnet Configuration

### BACNET_NAME

BACnet device name.

- **Default:** `Sentinel`
- **Example:** `Building A Gateway`

```bash
BACNET_NAME=Sentinel
```

### BACNET_INSTANCE

BACnet device instance number (must be unique on network).

- **Range:** 0 to 4194303
- **Default:** `999`
- **Example:** `1001`

```bash
BACNET_INSTANCE=999
```

**Important:** Each BACnet device on the network must have a unique instance number.

### BACNET_NETWORK

BACnet network number.

- **Default:** `0`
- **Range:** 0 to 65535

```bash
BACNET_NETWORK=0
```

**Note:** Usually 0 for single-network deployments.

### BACNET_VENDOR_ID

BACnet vendor identifier.

- **Default:** `1318` (ACE IoT Solutions)
- **Range:** 0 to 65535

```bash
BACNET_VENDOR_ID=1318
```

### BACNET_FOREIGN

BACnet Foreign Device Registration (for connecting across subnets via BBMD).

- **Format:** `"IP:PORT"` or `null`
- **Default:** `null`
- **Example:** `"192.168.1.1:47808"`

```bash
# No foreign device registration
BACNET_FOREIGN=null

# Register with BBMD at 192.168.1.1
BACNET_FOREIGN="192.168.1.1:47808"
```

**Use case:** When Sentinel is on a different subnet than BACnet devices and needs to connect through a BBMD.

### BACNET_TTL

Time-to-live for foreign device registration (seconds).

- **Default:** `30`
- **Range:** 1 to 65535

```bash
BACNET_TTL=30
```

**Note:** Only applies when `BACNET_FOREIGN` is set.

### BACNET_BBMD

Configure this device as a BBMD (BACnet Broadcast Management Device).

- **Format:** `"IP:PORT"` or `null`
- **Default:** `null`

```bash
# Not a BBMD
BACNET_BBMD=null

# Configure as BBMD
BACNET_BBMD="192.168.1.100:47808"
```

**Use case:** Advanced networking scenarios where this Sentinel acts as a BBMD for other devices.

---

## Grasshopper Web UI Configuration

### WEBAPP_ENABLED

Enable the Grasshopper web interface.

- **Options:** `true` or `false`
- **Default:** `true`

```bash
WEBAPP_ENABLED=true
```

### WEBAPP_HOST

Host to bind web server to.

- **Default:** `0.0.0.0` (all interfaces)
- **Example:** `127.0.0.1` (localhost only)

```bash
# All interfaces (recommended for container)
WEBAPP_HOST=0.0.0.0

# Localhost only
WEBAPP_HOST=127.0.0.1
```

### WEBAPP_PORT

Port for web UI.

- **Default:** `5000`
- **Range:** 1024 to 65535

```bash
WEBAPP_PORT=5000
```

**Note:** This port will be exposed from the container.

### WEBAPP_CERTFILE

Path to SSL/TLS certificate file (for HTTPS).

- **Default:** `null` (HTTP only)
- **Example:** `/certs/cert.pem`

```bash
# HTTP (default)
WEBAPP_CERTFILE=null

# HTTPS
WEBAPP_CERTFILE=/certs/cert.pem
```

### WEBAPP_KEYFILE

Path to SSL/TLS private key file (for HTTPS).

- **Default:** `null` (HTTP only)
- **Example:** `/certs/key.pem`

```bash
# HTTP (default)
WEBAPP_KEYFILE=null

# HTTPS
WEBAPP_KEYFILE=/certs/key.pem
```

**Note:** Both `WEBAPP_CERTFILE` and `WEBAPP_KEYFILE` must be set for HTTPS.

---

## Network Scanning Configuration

### SCAN_INTERVAL_SECS

How often to scan the BACnet network (seconds).

- **Default:** `86400` (24 hours)
- **Example:** `3600` (1 hour)

```bash
# Daily scan
SCAN_INTERVAL_SECS=86400

# Hourly scan
SCAN_INTERVAL_SECS=3600

# Weekly scan
SCAN_INTERVAL_SECS=604800
```

### LOW_LIMIT

Lower bound of BACnet device instance range to scan.

- **Default:** `0`
- **Range:** 0 to 4194303

```bash
LOW_LIMIT=0
```

### HIGH_LIMIT

Upper bound of BACnet device instance range to scan.

- **Default:** `4194303` (maximum BACnet range)
- **Range:** 0 to 4194303

```bash
# Scan full range
HIGH_LIMIT=4194303

# Scan limited range
HIGH_LIMIT=1000
```

**Tip:** Limit the range if you know your devices use a specific instance number range to speed up scanning.

### DEVICE_BROADCAST_FULL_STEP

Step size when devices are found during scanning.

- **Default:** `100`
- **Range:** 1 to 4194303

```bash
DEVICE_BROADCAST_FULL_STEP=100
```

**Smaller values:** More thorough but slower
**Larger values:** Faster but may miss devices with sparse instance numbers

### DEVICE_BROADCAST_EMPTY_STEP

Step size when no devices are found during scanning.

- **Default:** `1000`
- **Range:** 1 to 4194303

```bash
DEVICE_BROADCAST_EMPTY_STEP=1000
```

**Optimization:** Use larger steps when no devices are found to scan faster through empty ranges.

---

## ACE Config Agent Integration

The ACE Config Agent manages connection to ACE IoT Manager platform and provides centralized configuration, JWT rotation, and platform management.

### ACE_CONFIG_ENABLED

Enable ACE Config Agent integration.

- **Options:** `true` or `false`
- **Default:** `false`

```bash
ACE_CONFIG_ENABLED=false
```

**Recommended:** Set to `true` for production deployments with ACE IoT Manager.

### ACE_CONFIG_URL

ACE Manager API URL (required if ACE_CONFIG_ENABLED=true).

- **Format:** `https://hostname` or `http://hostname:port`
- **Default:** Empty
- **Example:** `https://manage.aceiot.cloud`

```bash
# ACE IoT Cloud
ACE_CONFIG_URL=https://manage.aceiot.cloud

# Self-hosted
ACE_CONFIG_URL=http://172.26.0.3:8888
```

### ACE_CONFIG_JWT

JWT token for authentication with ACE Manager (required if ACE_CONFIG_ENABLED=true).

- **Default:** Empty
- **Example:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`

```bash
ACE_CONFIG_JWT=your-jwt-token-here
```

**Note:** This JWT will be auto-rotated by the ace-config agent.

### ACE_CONFIG_CLIENT

Client identifier (optional, for multi-tenant environments).

- **Default:** Empty
- **Example:** `acme-corp`

```bash
ACE_CONFIG_CLIENT=
```

### ACE_CONFIG_SITE

Site name (optional, for organizing gateways).

- **Default:** Empty
- **Example:** `headquarters`

```bash
ACE_CONFIG_SITE=
```

### ACE_CONFIG_GATEWAY

Gateway name (required if ACE_CONFIG_ENABLED=true).

- **Default:** Empty
- **Example:** `building-a-sentinel`

```bash
ACE_CONFIG_GATEWAY=building-a-sentinel
```

**Important:** Must be unique across all gateways in your organization.

### ACE_CONFIG_POLL_INTERVAL

How often to poll ACE Manager for configuration updates (seconds).

- **Default:** `60`
- **Range:** 1 to 86400

```bash
ACE_CONFIG_POLL_INTERVAL=60
```

---

## Cloud Upload Configuration (Alternative)

Direct cloud upload WITHOUT ACE Config Agent. Use this ONLY if `ACE_CONFIG_ENABLED=false`.

**Note:** If ACE Config is enabled, credentials come from there automatically.

### CLOUD_UPLOAD_ENABLED

Enable uploading scan results to cloud API.

- **Options:** `true` or `false`
- **Default:** `false`

```bash
CLOUD_UPLOAD_ENABLED=false
```

### CLOUD_UPLOAD_URL

API endpoint URL (not needed if using ACE Config).

- **Format:** `https://hostname/path`
- **Default:** Empty

```bash
CLOUD_UPLOAD_URL=https://api.example.com/v1/scan-results
```

### CLOUD_UPLOAD_JWT

JWT token for authentication (not needed if using ACE Config).

- **Default:** Empty
- **Example:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`

```bash
CLOUD_UPLOAD_JWT=your-jwt-token-here
```

**Warning:** This JWT is static and won't auto-rotate. Use ACE Config Agent for automatic rotation.

### CLOUD_UPLOAD_INTERVAL

Upload interval (seconds).

- **Default:** `86400` (24 hours)
- **Example:** `3600` (1 hour)

```bash
CLOUD_UPLOAD_INTERVAL=86400
```

---

## Configuration Examples

### Basic Development Setup

```bash
# Minimal configuration for testing
BACNET_ADDRESS=192.168.1.100/24:47808
WEBAPP_PORT=5000
SCAN_INTERVAL_SECS=3600
```

### Production with ACE Manager

```bash
# BACnet Configuration
BACNET_ADDRESS=192.168.1.150/24:47808
BACNET_NAME="Building A Sentinel"
BACNET_INSTANCE=1001

# Web UI
WEBAPP_ENABLED=true
WEBAPP_PORT=5000
WEBAPP_CERTFILE=/certs/cert.pem
WEBAPP_KEYFILE=/certs/key.pem

# Scanning (daily)
SCAN_INTERVAL_SECS=86400

# ACE Manager Integration
ACE_CONFIG_ENABLED=true
ACE_CONFIG_URL=https://manage.aceiot.cloud
ACE_CONFIG_JWT=your-jwt-token
ACE_CONFIG_GATEWAY=building-a-sentinel
```

### Standalone with Cloud Upload

```bash
# BACnet Configuration
BACNET_ADDRESS=192.168.1.100/24:47808
BACNET_NAME=Sentinel

# Scanning (hourly)
SCAN_INTERVAL_SECS=3600

# Cloud Upload (without ACE Config)
CLOUD_UPLOAD_ENABLED=true
CLOUD_UPLOAD_URL=https://api.example.com/v1/scans
CLOUD_UPLOAD_JWT=your-jwt-token
CLOUD_UPLOAD_INTERVAL=3600
```

### Multi-Subnet with BBMD

```bash
# BACnet Configuration
BACNET_ADDRESS=10.0.50.100/16:47808
BACNET_NAME=Sentinel
BACNET_INSTANCE=1001

# Foreign Device Registration
BACNET_FOREIGN="10.0.1.1:47808"
BACNET_TTL=30

# Web UI
WEBAPP_PORT=5000
```

---

## See Also

- [Network Configuration](./NETWORKING.md) - Host, MACVLAN, and IPVLAN networking
- [ACE Integration](./ACE_INTEGRATION.md) - ACE IoT Manager integration guide
- [Deployment Guide](./DEPLOYMENT.md) - Production deployment best practices
