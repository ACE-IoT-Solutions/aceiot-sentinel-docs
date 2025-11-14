# Network Configuration Guide

This guide covers different networking modes for the Sentinel Container, with a focus on production deployments requiring BACnet/IP functionality.

## Overview

BACnet/IP requires broadcast UDP packets, which means the container needs direct access to the physical network. This guide covers three networking approaches:

1. **Host Networking** - Simplest, shares host network stack
2. **MACVLAN** - Gives container its own MAC address (recommended)
3. **IPVLAN** - Gives container its own IP without separate MAC

## Quick Comparison

| Feature | Host Mode | MACVLAN | IPVLAN L2 | IPVLAN L3 |
|---------|-----------|---------|-----------|-----------|
| Setup Complexity | Easy | Medium | Medium | Hard |
| Container Isolation | None | Full | Full | Full |
| Host-Container Communication | ✅ | ❌* | ❌* | ✅ |
| BACnet Broadcast Support | ✅ | ✅ | ✅ | ❌ |
| MAC Address Limits | N/A | May hit switch limits | No limits | No limits |
| Port Conflicts | Possible | None | None | None |
| Production Ready | ⚠️ | ✅ | ✅ | ❌ |

*Requires additional bridge configuration

## Host Networking (Default)

### When to Use
- Development and testing
- Quick deployments
- Single container on host

### Limitations
- Shares network namespace with host
- Port conflicts possible (22917, 5000)
- Less isolation

### Usage

```bash
# Docker
docker run -e BACNET_ADDRESS=192.168.1.100/24:47808 \
  --network host \
  ghcr.io/acedrew/aceiot-sentinel:latest

# Podman
podman run -e BACNET_ADDRESS=192.168.1.100/24:47808 \
  --network host \
  ghcr.io/acedrew/aceiot-sentinel:latest

# Docker Compose (default in docker-compose.yml)
docker-compose up -d
```

## MACVLAN Networking (Recommended for Production)

### Overview
MACVLAN creates a virtual network interface with its own MAC address, making the container appear as a physical device on the network. This is the **recommended approach for production**.

### Advantages
- ✅ Full network isolation
- ✅ Container gets its own IP address
- ✅ No port conflicts
- ✅ BACnet broadcasts work perfectly
- ✅ Multiple containers can coexist

### Limitations
- ⚠️ Cannot communicate with host by default (requires additional bridge)
- ⚠️ Some switches limit MAC addresses per port
- ⚠️ Some cloud/VM providers don't support MACVLAN

### Prerequisites

1. **Available IP address** in your BACnet subnet
2. **Parent interface name** (e.g., `eth0`, `enp0s3`)
3. **Subnet and gateway** information

### Setup Script

Use the provided script:

```bash
./scripts/setup-macvlan.sh
```

Or manually:

```bash
# Find your network interface
ip addr show

# Create MACVLAN network
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.128/25 \
  -o parent=eth0 \
  sentinel-macvlan

# Run container with static IP
docker run -d \
  --name aceiot-sentinel \
  --network sentinel-macvlan \
  --ip 192.168.1.150 \
  -e BACNET_ADDRESS=192.168.1.150/24:47808 \
  ghcr.io/acedrew/aceiot-sentinel:latest
```

### Docker Compose with MACVLAN

```yaml
# docker-compose.macvlan.yml
version: '3.8'

networks:
  macvlan:
    driver: macvlan
    driver_opts:
      parent: eth0
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.1
          ip_range: 192.168.1.128/25

services:
  sentinel:
    image: ghcr.io/acedrew/aceiot-sentinel:latest
    container_name: aceiot-sentinel
    networks:
      macvlan:
        ipv4_address: 192.168.1.150
    environment:
      BACNET_ADDRESS: 192.168.1.150/24:47808
    volumes:
      - volttron-data:/home/volttron/.aceiot-sentinel-volttron
    restart: unless-stopped

volumes:
  volttron-data:
```

Usage:
```bash
docker-compose -f docker-compose.macvlan.yml up -d
```

### Accessing Container from Host

By default, the host cannot communicate with MACVLAN containers. To enable:

```bash
# Create a macvlan interface on the host
sudo ip link add macvlan-host link eth0 type macvlan mode bridge
sudo ip addr add 192.168.1.254/32 dev macvlan-host
sudo ip link set macvlan-host up
sudo ip route add 192.168.1.150/32 dev macvlan-host

# Now you can access the container
curl http://192.168.1.150:5000
```

## IPVLAN Networking

### Overview
IPVLAN is similar to MACVLAN but doesn't create separate MAC addresses. Better for environments with MAC address restrictions.

### IPVLAN L2 Mode (Recommended)

Similar to MACVLAN but uses the same MAC address as parent interface.

#### Setup

```bash
# Create IPVLAN L2 network
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.128/25 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  sentinel-ipvlan-l2

# Run container
docker run -d \
  --name aceiot-sentinel \
  --network sentinel-ipvlan-l2 \
  --ip 192.168.1.150 \
  -e BACNET_ADDRESS=192.168.1.150/24:47808 \
  ghcr.io/acedrew/aceiot-sentinel:latest
```

#### Docker Compose

```yaml
# docker-compose.ipvlan-l2.yml
version: '3.8'

networks:
  ipvlan:
    driver: ipvlan
    driver_opts:
      parent: eth0
      ipvlan_mode: l2
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.1
          ip_range: 192.168.1.128/25

services:
  sentinel:
    image: ghcr.io/acedrew/aceiot-sentinel:latest
    container_name: aceiot-sentinel
    networks:
      ipvlan:
        ipv4_address: 192.168.1.150
    environment:
      BACNET_ADDRESS: 192.168.1.150/24:47808
    volumes:
      - volttron-data:/home/volttron/.aceiot-sentinel-volttron
    restart: unless-stopped

volumes:
  volttron-data:
```

### IPVLAN L3 Mode

⚠️ **Not recommended for BACnet** - L3 mode doesn't support broadcast/multicast.

L3 mode is layer 3 routing only, which breaks BACnet broadcast discovery. Only use if you don't need BACnet functionality.

## Troubleshooting

### BACnet Devices Not Discovered

**Symptom**: Container starts but no BACnet devices are found.

**Solutions**:
1. Verify container has correct IP on BACnet subnet
2. Check firewall allows UDP port 47808
3. Verify broadcast is working:
   ```bash
   # Inside container
   docker exec aceiot-sentinel tcpdump -i any -n udp port 47808
   ```
4. Ensure switch/network allows broadcasts

### Cannot Access Container from Host

**MACVLAN/IPVLAN Issue**: By default, host cannot reach container.

**Solution**: Create host interface (see MACVLAN section above)

### MAC Address Limit Exceeded

**Symptom**: MACVLAN containers fail to get network connectivity.

**Solution**: Switch to IPVLAN L2 mode (uses same MAC as parent)

### Container Gets Wrong IP

**Symptom**: Container gets DHCP IP instead of static IP.

**Solution**: Always specify `--ip` flag or `ipv4_address` in compose file

## Network Verification

Once your container is running, verify networking:

```bash
# Check container IP
docker exec aceiot-sentinel ip addr show

# Verify BACnet port is listening
docker exec aceiot-sentinel netstat -uln | grep 47808

# Check if container can reach network
docker exec aceiot-sentinel ping -c 3 192.168.1.1

# Test broadcast capability
docker exec aceiot-sentinel tcpdump -i eth0 -c 10 broadcast

# Access web UI
curl http://CONTAINER_IP:5000
```

## Production Recommendations

For production deployments:

1. **Use MACVLAN or IPVLAN L2** - Not host networking
2. **Reserve IP addresses** - Use DHCP reservations or static IPs outside DHCP range
3. **Document network config** - Keep track of which IPs are assigned to which containers
4. **Use Docker Compose** - Easier to manage and reproduce
5. **Enable restart policies** - `restart: unless-stopped` in compose
6. **Monitor connectivity** - Set up health checks

## Example Production Deployment

```bash
# 1. Create network (one time)
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.200/29 \
  -o parent=eth0 \
  bacnet-network

# 2. Deploy container
docker run -d \
  --name sentinel-building-a \
  --network bacnet-network \
  --ip 192.168.1.200 \
  -e BACNET_ADDRESS=192.168.1.200/24:47808 \
  -e BACNET_NAME="Building A Sentinel" \
  -e BACNET_INSTANCE=1001 \
  -v sentinel-a-data:/home/volttron/.aceiot-sentinel-volttron \
  --restart unless-stopped \
  ghcr.io/acedrew/aceiot-sentinel:latest

# 3. Verify
docker logs sentinel-building-a
curl http://192.168.1.200:5000

# 4. Deploy additional containers on same network
docker run -d \
  --name sentinel-building-b \
  --network bacnet-network \
  --ip 192.168.1.201 \
  -e BACNET_ADDRESS=192.168.1.201/24:47808 \
  -e BACNET_NAME="Building B Sentinel" \
  -e BACNET_INSTANCE=1002 \
  -v sentinel-b-data:/home/volttron/.aceiot-sentinel-volttron \
  --restart unless-stopped \
  ghcr.io/acedrew/aceiot-sentinel:latest
```

## References

- [Docker MACVLAN Documentation](https://docs.docker.com/network/macvlan/)
- [Docker IPVLAN Documentation](https://docs.docker.com/network/ipvlan/)
- [BACnet/IP Protocol Specification](http://www.bacnet.org/)
