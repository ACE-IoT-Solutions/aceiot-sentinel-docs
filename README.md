# ACE IoT Sentinel Container

A containerized VOLTTRON edge platform with the Grasshopper BACnet network monitoring agent.

## Overview

The ACE IoT Sentinel Container packages:
- **VOLTTRON 9.0.4** - Open-source agent platform for smart buildings
- **Grasshopper Agent** - BACnet/IP network discovery and monitoring
- **ACE Config Agent** - Centralized configuration and JWT management
- **Web UI** - Interactive visualization of discovered BACnet devices

## Features

✅ **Full BACnet/IP Discovery** - Scans network and identifies all devices
✅ **Web-Based Visualization** - Interactive network topology viewer
✅ **ACE IoT Manager Integration** - Automatic JWT rotation and data upload
✅ **Environment-Based Config** - Easy deployment with env vars
✅ **Production Ready** - Security hardened, health checks, data persistence

## Quick Start

### Prerequisites

- **Docker** or **Podman**
- Access to a BACnet/IP network
- Network configuration allowing BACnet broadcasts (UDP port 47808)
- **Recommended**: Linux host for production deployments

### Basic Deployment

1. Create your configuration file:
```bash
cp .env.example .env
```

2. Edit `.env` and configure at minimum:
```bash
# Set your BACnet network address
BACNET_ADDRESS=192.168.1.100/24:47808

# Confirm web UI port
WEBAPP_PORT=5000
```

3. Start the container:
```bash
docker-compose up -d
```

4. Access the Grasshopper web UI:
```
http://localhost:5000
```

## Documentation

📚 **Configuration**
- [Configuration Reference](./CONFIGURATION.md) - Complete guide to all environment variables
- [Network Configuration](./NETWORKING.md) - Bridge, Host, MACVLAN, and IPVLAN networking modes

📚 **Deployment Scenarios**
- [Bridge Network](./examples/docker-compose.bridge.yml) - Container-to-container with auto-IP detection
- [Host Network Mode](./examples/docker-compose.host.yml) - Simple deployment (default)
- [MACVLAN Network](./examples/docker-compose.macvlan.yml) - Production deployment with isolated IP
- [IPVLAN L2 Network](./examples/docker-compose.ipvlan-l2.yml) - Alternative to MACVLAN
- [Multi-Container](./examples/docker-compose.multi.yml) - Deploy multiple sentinels

📚 **Integration**
- [ACE IoT Manager Integration](./ACE_INTEGRATION.md) - Connect to ACE Manager platform

📚 **Guides**
- [Deployment Guide](./DEPLOYMENT.md) - Step-by-step production deployment
- [Troubleshooting](./TROUBLESHOOTING.md) - Common issues and solutions

## Container Images

Official images are available on GitHub Container Registry:

```bash
# Pull latest version
docker pull ghcr.io/ace-iot-solutions/aceiot-sentinel:latest

# Run with host networking
docker run -d \
  --name aceiot-sentinel \
  --network host \
  -e BACNET_ADDRESS=192.168.1.100/24:47808 \
  -v sentinel-data:/home/volttron/.aceiot-sentinel-volttron \
  ghcr.io/ace-iot-solutions/aceiot-sentinel:latest
```

**Platform Support:**
- linux/amd64
- linux/arm64

## Required Configuration

The only **required** configuration is `BACNET_ADDRESS`. This can be set to `auto` for bridge networks or an explicit IP for other deployment modes.

**Format:** `auto`, `IP`, `IP/CIDR`, or `IP/CIDR:PORT`

**Examples:**
```bash
# Auto-detect (recommended for bridge networks)
BACNET_ADDRESS=auto

# Standard BACnet/IP on network 192.168.1.0/24
BACNET_ADDRESS=192.168.1.100/24:47808

# Different subnet
BACNET_ADDRESS=10.0.50.25/16:47808

# Network namespace deployment (Iotium, edge platforms)
BACNET_ADDRESS=192.168.1.206/24:47808
```

See [CONFIGURATION.md](./CONFIGURATION.md) for all available options.

## Support

For issues, questions, or contributions:
- GitHub: [ACE-IoT-Solutions/aceiot-sentinel-container](https://github.com/ACE-IoT-Solutions/aceiot-sentinel-container)
- Email: support@aceiotsolutions.com

## License

See LICENSE file for details.

## Credits

Built by ACE IoT Solutions
- Grasshopper Agent: Justice Lee, Jackson Giles, Andrew Rodgers
- Based on VOLTTRON platform (PNNL)
- BACpypes3 library
