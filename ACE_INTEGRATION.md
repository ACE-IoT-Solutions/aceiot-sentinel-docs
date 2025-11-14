# ACE IoT Manager Integration Guide

This guide explains how to integrate the Sentinel Container with ACE IoT Manager platform for centralized management and data collection.

## Overview

The **ACE Config Agent** provides:
- 🔐 **Automatic JWT rotation** - No manual credential management
- ⚙️ **Centralized configuration** - Manage settings from ACE Manager
- 📊 **Automated data upload** - Grasshopper scans automatically uploaded
- 🔄 **Platform management** - Remote agent deployment and configuration
- 📈 **Gateway monitoring** - Real-time status and health reporting

## Architecture

```
┌─────────────────────────────────────────┐
│       ACE IoT Manager Platform          │
│                                         │
│  - Central Configuration                │
│  - JWT Management                       │
│  - Data Collection                      │
│  - Gateway Management                   │
└────────────────┬────────────────────────┘
                 │
                 │ HTTPS + JWT
                 │
┌────────────────▼────────────────────────┐
│     Sentinel Container (Edge)           │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │   ACE Config Agent               │  │
│  │   - Polls for configuration      │  │
│  │   - Rotates JWT                  │  │
│  │   - Provides credentials         │  │
│  └─────────────┬────────────────────┘  │
│                │ RPC                    │
│  ┌─────────────▼────────────────────┐  │
│  │   Grasshopper Agent              │  │
│  │   - Scans BACnet network         │  │
│  │   - Gets JWT from ace-config     │  │
│  │   - Uploads scan results         │  │
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

## Quick Setup

### Prerequisites

1. Access to ACE IoT Manager instance
2. Admin credentials to create gateways
3. Network connectivity from edge to manager

### Step 1: Create Gateway in ACE Manager

1. Log into ACE IoT Manager web interface
2. Navigate to **Gateways** section
3. Click **Add Gateway**
4. Configure gateway:
   - **Name**: Unique identifier (e.g., `building-123-edge`)
   - **Site**: Associate with a site (optional)
   - **Client**: For multi-tenant setups (optional)
5. **Save** - You'll receive a JWT token

### Step 2: Configure Sentinel Container

Copy and edit your `.env` file:

```bash
cp .env.example .env
```

Set the ACE Config variables:

```bash
# ===========================================
# ACE Config Agent Integration
# ===========================================
ACE_CONFIG_ENABLED=true

# ACE Manager URL
ACE_CONFIG_URL=https://manage.aceiot.cloud

# JWT from gateway creation (will auto-rotate)
ACE_CONFIG_JWT=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Gateway name (must match what you created)
ACE_CONFIG_GATEWAY=building-123-edge

# Optional: Site and client for organization
ACE_CONFIG_SITE=main-campus
ACE_CONFIG_CLIENT=my-company

# Poll interval (default: 60 seconds)
ACE_CONFIG_POLL_INTERVAL=60
```

### Step 3: Enable Cloud Upload

Configure Grasshopper to upload scan results:

```bash
# Enable upload (uses JWT from ace-config automatically)
CLOUD_UPLOAD_ENABLED=true

# Upload interval (default: daily)
CLOUD_UPLOAD_INTERVAL=86400
```

**Important**: When `ACE_CONFIG_ENABLED=true`, you do NOT need to set `CLOUD_UPLOAD_URL` or `CLOUD_UPLOAD_JWT`. These are provided automatically by the ACE Config Agent.

### Step 4: Configure BACnet

Set your BACnet network address:

```bash
BACNET_ADDRESS=192.168.1.100/24:47808
```

### Step 5: Start Container

```bash
docker-compose up -d
```

### Step 6: Verify Integration

Check logs to confirm connection:

```bash
docker logs -f aceiot-sentinel
```

Look for:
```
========================================
Configuring ACE Config Agent...
========================================
ACE Config Agent installed and started successfully!
...
Container ready!
VOLTTRON Platform: Running (PID: ...)
ACE Config Agent: Running
Grasshopper Agent: Running
Cloud Upload: Enabled (URL: https://manage.aceiot.cloud)
========================================
```

Check agent status:
```bash
docker exec aceiot-sentinel vctl status
```

You should see:
```
AGENT                    IDENTITY       TAG        STATUS
ace-config               ace-config     ace-config running [12345]
grasshopper              grasshopper    grasshopper running [12346]
```

## Configuration Details

### Required Settings

When `ACE_CONFIG_ENABLED=true`, these are **required**:

| Variable | Description | Example |
|----------|-------------|---------|
| `ACE_CONFIG_URL` | ACE Manager API endpoint | `https://manage.aceiot.cloud` |
| `ACE_CONFIG_JWT` | Initial JWT token from gateway creation | `eyJhbG...` |
| `ACE_CONFIG_GATEWAY` | Unique gateway identifier | `building-123-edge` |

### Optional Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `ACE_CONFIG_CLIENT` | `""` | Client/tenant identifier |
| `ACE_CONFIG_SITE` | `""` | Site name for organization |
| `ACE_CONFIG_POLL_INTERVAL` | `60` | How often to check for config updates (seconds) |

### Cloud Upload Settings

| Variable | Default | Description |
|----------|---------|-------------|
| `CLOUD_UPLOAD_ENABLED` | `false` | Enable uploading scan results |
| `CLOUD_UPLOAD_INTERVAL` | `86400` | Upload frequency (seconds, default daily) |

**Note**: `CLOUD_UPLOAD_URL` and `CLOUD_UPLOAD_JWT` are NOT needed when using ACE Config Agent.

## How It Works

### JWT Management

1. Initial JWT provided during gateway setup
2. ACE Config Agent automatically rotates JWT before expiration
3. New JWT stored in agent's secure config
4. Grasshopper retrieves current JWT via RPC when uploading

### Configuration Updates

1. ACE Config Agent polls manager every `ACE_CONFIG_POLL_INTERVAL` seconds
2. Manager can push configuration updates (agent deployment, settings)
3. Agent applies changes automatically
4. Changes logged and reported back to manager

### Data Upload

1. Grasshopper performs BACnet scan (per `SCAN_INTERVAL_SECS`)
2. Network topology saved as RDF/TTL file
3. On upload interval, Grasshopper:
   - Contacts ace-config agent via RPC
   - Retrieves current valid JWT
   - Uploads TTL file to ACE Manager
   - Deletes local file on success

## Troubleshooting

### ACE Config Agent Won't Start

**Check logs:**
```bash
docker logs aceiot-sentinel | grep -A 20 "ACE Config"
```

**Common issues:**
- Missing required env vars → Error message will specify which
- Invalid JWT → Check JWT format and expiration
- Can't reach manager → Check `ACE_CONFIG_URL` and network connectivity

### JWT Not Rotating

**Check agent status:**
```bash
docker exec aceiot-sentinel vctl status
```

Ace-config should show `running`.

**Check ace-config logs:**
```bash
docker exec aceiot-sentinel grep "ace-config" /tmp/volttron.log
```

Look for JWT refresh messages.

### Uploads Not Working

**Verify cloud upload is enabled:**
```bash
docker exec aceiot-sentinel cat /home/volttron/.volttron/grasshopper-config/config | grep -A 5 ttl_post_to_cloud
```

Should show:
```json
"ttl_post_to_cloud": {
    "enabled": true,
    "url": "https://manage.aceiot.cloud",
    ...
}
```

**Check Grasshopper logs for upload attempts:**
```bash
docker logs aceiot-sentinel | grep -i upload
```

### Gateway Not Appearing in Manager

**Verify configuration:**
1. Gateway name matches exactly (case-sensitive)
2. JWT is valid
3. Manager URL is correct
4. Network allows HTTPS egress

**Test connectivity:**
```bash
docker exec aceiot-sentinel curl -I https://manage.aceiot.cloud
```

## Advanced Configuration

### Custom Poll Interval

Reduce polling frequency for bandwidth-constrained environments:

```bash
ACE_CONFIG_POLL_INTERVAL=300  # Check every 5 minutes
```

### Site Hierarchy

Organize gateways by site and client:

```bash
ACE_CONFIG_CLIENT=acme-corp
ACE_CONFIG_SITE=building-north
ACE_CONFIG_GATEWAY=north-hvac-edge-1
```

### Multiple Gateways

Run multiple containers with different gateway names:

```bash
# Gateway 1
ACE_CONFIG_GATEWAY=hvac-zone-1
WEBAPP_PORT=5001

# Gateway 2
ACE_CONFIG_GATEWAY=hvac-zone-2
WEBAPP_PORT=5002
```

## Security Considerations

### JWT Security
- JWT tokens are sensitive credentials
- Store in `.env` file (not committed to git)
- Container stores JWT in VOLTTRON secure config
- JWTs auto-rotate - old tokens become invalid

### Network Security
- Use HTTPS for ACE Manager connections
- Consider VPN or private network for edge-to-cloud
- Firewall rules to restrict egress if needed

### Container Security
- Runs as non-root user (`volttron`)
- Minimal attack surface
- Regular security updates recommended

## Monitoring

### Health Checks

```bash
# Check all agent status
docker exec aceiot-sentinel vctl status

# Check ACE config health
docker exec aceiot-sentinel vctl health ace-config
```

### Logs

```bash
# Follow all logs
docker logs -f aceiot-sentinel

# Just ace-config logs
docker exec aceiot-sentinel grep "ace-config" /tmp/volttron.log

# Just Grasshopper upload logs
docker exec aceiot-sentinel grep -i "upload" /tmp/volttron.log
```

### Platform Status

View in ACE Manager:
1. Navigate to **Gateways**
2. Find your gateway
3. View:
   - Connection status
   - Last check-in time
   - Installed agents
   - Recent uploads
   - System metrics

## Migration from Direct Upload

If you're currently using direct upload (without ACE Config):

1. Create gateway in ACE Manager
2. Get JWT token
3. Update `.env`:
   ```bash
   # Enable ace-config
   ACE_CONFIG_ENABLED=true
   ACE_CONFIG_URL=https://manage.aceiot.cloud
   ACE_CONFIG_JWT=<your-jwt>
   ACE_CONFIG_GATEWAY=<gateway-name>

   # Keep cloud upload enabled
   CLOUD_UPLOAD_ENABLED=true

   # Remove these (no longer needed):
   # CLOUD_UPLOAD_URL=...
   # CLOUD_UPLOAD_JWT=...
   ```
4. Restart container: `docker-compose restart`

## Support

For issues with:
- **ACE Config Agent**: Contact ACE IoT Solutions support
- **ACE Manager**: Check ACE Manager documentation
- **Container issues**: See main [README.md](README.md) troubleshooting

Email: support@aceiotsolutions.com
