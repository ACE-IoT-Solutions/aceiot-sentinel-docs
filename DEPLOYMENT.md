# Deployment Guide

Step-by-step guide for deploying ACE IoT Sentinel Container in production environments.

## Table of Contents

- [Planning Your Deployment](#planning-your-deployment)
- [Network Setup](#network-setup)
- [Container Deployment](#container-deployment)
- [Verification](#verification)
- [Monitoring](#monitoring)
- [Backup and Recovery](#backup-and-recovery)
- [Updates and Maintenance](#updates-and-maintenance)

---

## Planning Your Deployment

### 1. Network Assessment

Before deployment, gather the following information:

**BACnet Network:**
- Network subnet (e.g., `192.168.1.0/24`)
- Gateway IP
- Available IP addresses for container(s)
- BACnet port (usually `47808`)
- VLAN configuration (if applicable)

**Existing Devices:**
- BACnet device instance number ranges in use
- Number of devices to monitor
- Device locations (if multiple buildings)

**Infrastructure:**
- Host server specifications
- Operating system (Linux recommended)
- Docker/Podman version
- Network interface names

### 2. Capacity Planning

**Single Sentinel:**
- Suitable for: Up to 500 BACnet devices
- Resources: 2 CPU cores, 2GB RAM minimum
- Storage: 10GB for OS + data

**Multiple Sentinels:**
- Use when: More than 500 devices, multiple buildings, or geographic distribution
- Split device ranges across sentinels
- Each sentinel: Same resource requirements as single

### 3. IP Address Planning

Reserve static IP addresses:
- One IP per sentinel container
- Document in network inventory
- Configure DHCP reservations or static assignments

**Example allocation:**
```
192.168.1.200 - Building A Sentinel
192.168.1.201 - Building B Sentinel
192.168.1.202 - Campus Core Sentinel
```

---

## Network Setup

### Option 1: Host Network (Development/Testing)

**Pros:**
- Simplest setup
- Good for proof-of-concept

**Cons:**
- Less isolation
- Possible port conflicts
- Not recommended for production

**Setup:**
```bash
# No network setup needed
docker-compose -f examples/docker-compose.host.yml up -d
```

### Option 2: MACVLAN (Recommended for Production)

**Pros:**
- Full network isolation
- Container gets own MAC and IP
- Multiple containers supported
- Production-ready

**Setup:**

1. Identify your network interface:
```bash
ip addr show
# Look for interface with your network IP (e.g., eth0, enp0s3)
```

2. Create MACVLAN network:
```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.200/29 \
  -o parent=eth0 \
  bacnet-network
```

**Parameters explained:**
- `--subnet`: Your network subnet
- `--gateway`: Your network gateway
- `--ip-range`: Reserved range for containers (200-207)
- `parent`: Your network interface name

3. Verify network:
```bash
docker network ls
docker network inspect bacnet-network
```

### Option 3: IPVLAN L2 (Alternative)

Use when switches have MAC address limits.

```bash
docker network create -d ipvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.200/29 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  bacnet-network
```

See [NETWORKING.md](./NETWORKING.md) for detailed networking guide.

---

## Container Deployment

### Step 1: Prepare Configuration

1. Download configuration template:
```bash
curl -O https://raw.githubusercontent.com/ACE-IoT-Solutions/aceiot-sentinel-docs/main/examples/.env.example
```

2. Create configuration file:
```bash
cp .env.example .env
```

3. Edit configuration:
```bash
nano .env
```

**Minimum required settings:**
```bash
BACNET_ADDRESS=192.168.1.200/24:47808
BACNET_NAME="Building A Sentinel"
BACNET_INSTANCE=1001
```

**Recommended production settings:**
```bash
# BACnet
BACNET_ADDRESS=192.168.1.200/24:47808
BACNET_NAME="Building A Sentinel"
BACNET_INSTANCE=1001
BACNET_VENDOR_ID=1318

# Web UI
WEBAPP_ENABLED=true
WEBAPP_PORT=5000
WEBAPP_CERTFILE=/certs/cert.pem
WEBAPP_KEYFILE=/certs/key.pem

# Scanning
SCAN_INTERVAL_SECS=86400  # Daily
LOW_LIMIT=0
HIGH_LIMIT=1000

# ACE Manager Integration
ACE_CONFIG_ENABLED=true
ACE_CONFIG_URL=https://manage.aceiot.cloud
ACE_CONFIG_JWT=your-jwt-token
ACE_CONFIG_GATEWAY=building-a-sentinel
```

### Step 2: Deploy Container

**Using Docker Compose (Recommended):**

1. Download compose file:
```bash
curl -O https://raw.githubusercontent.com/ACE-IoT-Solutions/aceiot-sentinel-docs/main/examples/docker-compose.macvlan.yml
```

2. Edit IP address in compose file:
```bash
nano docker-compose.macvlan.yml
# Change ipv4_address to your chosen IP
```

3. Start container:
```bash
docker-compose -f docker-compose.macvlan.yml up -d
```

**Using Docker Run:**

```bash
docker run -d \
  --name sentinel-building-a \
  --network bacnet-network \
  --ip 192.168.1.200 \
  --env-file .env \
  -v sentinel-data:/home/volttron/.aceiot-sentinel-volttron \
  --restart unless-stopped \
  ghcr.io/ace-iot-solutions/aceiot-sentinel:latest
```

### Step 3: Enable SSL/TLS (Recommended)

1. Generate SSL certificate:
```bash
# Self-signed (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout key.pem -out cert.pem

# Or use Let's Encrypt for production
certbot certonly --standalone -d sentinel.example.com
```

2. Mount certificates:
```yaml
volumes:
  - ./certs/cert.pem:/certs/cert.pem:ro
  - ./certs/key.pem:/certs/key.pem:ro
```

3. Update configuration:
```bash
WEBAPP_CERTFILE=/certs/cert.pem
WEBAPP_KEYFILE=/certs/key.pem
```

---

## Verification

### 1. Check Container Status

```bash
# Container is running
docker ps | grep sentinel

# View logs
docker logs aceiot-sentinel

# Follow logs
docker logs -f aceiot-sentinel
```

**Expected output:**
```
=========================================
Container ready!
VOLTTRON Platform: Running (PID: 123)
Grasshopper Agent: Running
Web UI: http://0.0.0.0:5000
=========================================
```

### 2. Verify Network Connectivity

```bash
# Container has correct IP
docker exec aceiot-sentinel ip addr show

# Can reach gateway
docker exec aceiot-sentinel ping -c 3 192.168.1.1

# BACnet port is listening
docker exec aceiot-sentinel netstat -uln | grep 47808
```

### 3. Test Web UI

```bash
# HTTP
curl http://192.168.1.200:5000

# HTTPS
curl -k https://192.168.1.200:5000

# Or open in browser
```

### 4. Verify BACnet Communication

```bash
# Monitor BACnet traffic
docker exec aceiot-sentinel tcpdump -i eth0 -n udp port 47808

# Check for Who-Is broadcasts and I-Am responses
```

### 5. Check Agent Status

```bash
# VOLTTRON agent status
docker exec aceiot-sentinel vctl status

# Should show grasshopper agent running
```

---

## Monitoring

### Health Checks

Docker health checks run automatically:
```bash
docker inspect aceiot-sentinel | grep -A 10 Health
```

### Log Monitoring

**View VOLTTRON logs:**
```bash
docker exec aceiot-sentinel tail -f /tmp/volttron.log
```

**View container logs:**
```bash
docker logs -f --tail 100 aceiot-sentinel
```

### Metrics to Monitor

1. **Container health:** Check `docker ps` status
2. **Network connectivity:** Ping tests to gateway
3. **BACnet traffic:** Monitor UDP port 47808
4. **Web UI availability:** HTTP/HTTPS checks
5. **Disk usage:** Monitor volume size
6. **Scan completion:** Check web UI for recent scans

### Alerting

Set up monitoring with tools like:
- **Prometheus + Grafana:** Container metrics
- **ELK Stack:** Log aggregation
- **Nagios/Zabbix:** Infrastructure monitoring
- **ACE IoT Manager:** Built-in gateway monitoring

---

## Backup and Recovery

### Backup Data

**Create backup:**
```bash
docker run --rm \
  -v sentinel-data:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/sentinel-backup-$(date +%Y%m%d).tar.gz -C /data .
```

**Automated daily backup:**
```bash
# Create backup script
cat > backup-sentinel.sh << 'EOF'
#!/bin/bash
BACKUP_DIR=/backups/sentinel
DATE=$(date +%Y%m%d)
mkdir -p $BACKUP_DIR

docker run --rm \
  -v sentinel-data:/data \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/sentinel-$DATE.tar.gz -C /data .

# Keep only last 7 days
find $BACKUP_DIR -name "sentinel-*.tar.gz" -mtime +7 -delete
EOF

chmod +x backup-sentinel.sh

# Add to crontab
crontab -e
# Add: 0 2 * * * /path/to/backup-sentinel.sh
```

### Restore from Backup

```bash
# Stop container
docker-compose -f docker-compose.macvlan.yml down

# Restore data
docker run --rm \
  -v sentinel-data:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/sentinel-backup-20241114.tar.gz -C /data

# Start container
docker-compose -f docker-compose.macvlan.yml up -d
```

### Disaster Recovery

1. **Save configuration:**
   - `.env` file
   - `docker-compose.yml`
   - SSL certificates
   - Network configuration

2. **Regular backups:**
   - Automated daily backups
   - Store off-site or cloud storage
   - Test restore procedures

3. **Recovery steps:**
   ```bash
   # 1. Install Docker on new host
   # 2. Restore network configuration
   # 3. Restore .env and compose files
   # 4. Restore data volume
   # 5. Start container
   docker-compose up -d
   ```

---

## Updates and Maintenance

### Update Container Image

1. **Pull latest image:**
```bash
docker pull ghcr.io/ace-iot-solutions/aceiot-sentinel:latest
```

2. **Backup before updating:**
```bash
./backup-sentinel.sh
```

3. **Update container:**
```bash
docker-compose -f docker-compose.macvlan.yml pull
docker-compose -f docker-compose.macvlan.yml up -d
```

4. **Verify update:**
```bash
docker logs aceiot-sentinel
curl http://192.168.1.200:5000
```

### Maintenance Windows

**Plan maintenance for:**
- Container updates
- Host OS updates
- Network changes
- Certificate renewals

**Best practices:**
- Schedule during low-usage periods
- Notify stakeholders
- Have rollback plan
- Monitor after changes

### Rollback Procedure

If issues occur after update:

```bash
# 1. Stop current container
docker-compose -f docker-compose.macvlan.yml down

# 2. Restore from backup
docker run --rm \
  -v sentinel-data:/data \
  -v /backups/sentinel:/backup \
  alpine tar xzf /backup/sentinel-backup-previous.tar.gz -C /data

# 3. Use previous image version
docker-compose -f docker-compose.macvlan.yml up -d

# Or specify version
docker run -d \
  --name aceiot-sentinel \
  ... \
  ghcr.io/ace-iot-solutions/aceiot-sentinel:v1.0.0
```

---

## Production Checklist

Before going live, verify:

- [ ] Network configured (MACVLAN/IPVLAN)
- [ ] Static IP assigned and documented
- [ ] BACnet instance number unique
- [ ] SSL/TLS certificates configured
- [ ] Configuration backed up
- [ ] Automated backups configured
- [ ] Monitoring/alerting set up
- [ ] ACE Manager integration tested (if applicable)
- [ ] Web UI accessible
- [ ] BACnet device discovery working
- [ ] Documentation updated
- [ ] Stakeholders notified
- [ ] Rollback plan documented

---

## Support

For deployment assistance:
- Email: support@aceiotsolutions.com
- Documentation: [GitHub](https://github.com/ACE-IoT-Solutions/aceiot-sentinel-docs)
- Issues: [Report here](https://github.com/ACE-IoT-Solutions/aceiot-sentinel-container/issues)
