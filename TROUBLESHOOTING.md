# Troubleshooting Guide

Common issues and solutions for ACE IoT Sentinel Container.

## Table of Contents

- [Container Issues](#container-issues)
- [Network Issues](#network-issues)
- [BACnet Discovery Issues](#bacnet-discovery-issues)
- [Web UI Issues](#web-ui-issues)
- [ACE Manager Integration Issues](#ace-manager-integration-issues)
- [Performance Issues](#performance-issues)
- [Data and Storage Issues](#data-and-storage-issues)

---

## Container Issues

### Container Won't Start

**Symptom:** Container exits immediately after starting

**Check logs:**
```bash
docker logs aceiot-sentinel
```

**Common causes:**

1. **Missing BACNET_ADDRESS:**
   ```
   Error: BACNET_ADDRESS environment variable required
   ```
   **Solution:** Set `BACNET_ADDRESS` in `.env` file

2. **Invalid BACNET_ADDRESS format:**
   ```
   Error: Invalid BACNET_ADDRESS format
   ```
   **Solution:** Use format `IP/CIDR:PORT` (e.g., `192.168.1.100/24:47808`)

3. **Port already in use:**
   ```
   Error: Address already in use
   ```
   **Solution:**
   - Check if another container is using the same network/ports
   - Change `WEBAPP_PORT` or use different IP address
   - Check host has no service on port 5000

4. **Permissions error:**
   ```
   Error: Permission denied
   ```
   **Solution:**
   - Ensure Docker has necessary network permissions
   - On Linux: May need `--cap-add=NET_ADMIN`

### Container Keeps Restarting

**Check health status:**
```bash
docker inspect aceiot-sentinel | grep -A 10 Health
```

**Common causes:**

1. **VOLTTRON startup failure:**
   ```bash
   docker exec aceiot-sentinel cat /tmp/volttron.log
   ```
   Look for errors in VOLTTRON log

2. **Agent installation failure:**
   ```bash
   docker exec aceiot-sentinel vctl status
   ```
   Check if agents are installed

**Solution:**
```bash
# Remove container and volume
docker-compose down -v

# Recreate with fresh state
docker-compose up -d
```

### Cannot Access Container Shell

**Symptom:** `docker exec` fails

**Solutions:**

1. **Container not running:**
   ```bash
   docker ps -a | grep sentinel
   # If not running, check logs
   docker logs aceiot-sentinel
   ```

2. **Wrong container name:**
   ```bash
   docker ps
   # Use exact container name/ID
   ```

---

## Network Issues

### Cannot Reach Container IP

**Symptom:** Cannot ping or access container from network

**Diagnose:**
```bash
# Check container IP
docker exec aceiot-sentinel ip addr show

# Check container can reach gateway
docker exec aceiot-sentinel ping -c 3 192.168.1.1

# Check routes
docker exec aceiot-sentinel ip route show
```

**Solutions:**

1. **MACVLAN/IPVLAN host isolation:**
   - Host cannot reach MACVLAN containers by default
   - **Solution:** Create host bridge interface
   ```bash
   sudo ip link add macvlan-host link eth0 type macvlan mode bridge
   sudo ip addr add 192.168.1.254/32 dev macvlan-host
   sudo ip link set macvlan-host up
   sudo ip route add 192.168.1.200/32 dev macvlan-host
   ```

2. **Wrong subnet/gateway:**
   - Verify network settings match physical network
   - Check `docker network inspect bacnet-network`

3. **Firewall blocking:**
   ```bash
   # Check firewall rules
   sudo iptables -L -n

   # Allow container IP
   sudo iptables -A INPUT -s 192.168.1.200 -j ACCEPT
   ```

### Container Has Wrong IP

**Symptom:** Container gets DHCP IP instead of static IP

**Check configuration:**
```bash
docker inspect aceiot-sentinel | grep IPAddress
```

**Solution:**
- Ensure `ipv4_address` is set in docker-compose.yml
- Or use `--ip` flag with `docker run`
- Recreate container with correct settings:
```bash
docker-compose down
docker-compose up -d
```

### Network Not Found

**Symptom:** `Error: network bacnet-network not found`

**Solution:**
```bash
# Create network first
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.200/29 \
  -o parent=eth0 \
  bacnet-network

# Then start container
docker-compose up -d
```

---

## BACnet Discovery Issues

### No Devices Discovered

**Symptom:** Web UI shows no devices after scan

**Diagnose:**

1. **Check BACnet traffic:**
   ```bash
   docker exec aceiot-sentinel tcpdump -i eth0 -n udp port 47808
   ```
   Should see Who-Is broadcasts and I-Am responses

2. **Verify BACnet port listening:**
   ```bash
   docker exec aceiot-sentinel netstat -uln | grep 47808
   ```

3. **Check scan status in web UI:**
   - Go to http://CONTAINER_IP:5000
   - Check for scan progress/errors

**Solutions:**

1. **Wrong network/subnet:**
   - Ensure `BACNET_ADDRESS` is on same network as devices
   - Check device IP addresses are in expected range

2. **Broadcast not working:**
   - Host network: Should work automatically
   - Bridge network: Use MACVLAN or IPVLAN L2 instead
   - IPVLAN L3: **Not compatible with BACnet** (no broadcast support)

3. **Firewall blocking:**
   ```bash
   # Check if broadcast packets are blocked
   sudo iptables -L -n | grep 47808

   # Allow BACnet port
   sudo iptables -A INPUT -p udp --dport 47808 -j ACCEPT
   sudo iptables -A OUTPUT -p udp --sport 47808 -j ACCEPT
   ```

4. **Device range too narrow:**
   - Check `LOW_LIMIT` and `HIGH_LIMIT` in configuration
   - Expand range to cover all device instance numbers
   ```bash
   LOW_LIMIT=0
   HIGH_LIMIT=4194303
   ```

5. **Scan not triggered:**
   - Check scan interval: `SCAN_INTERVAL_SECS`
   - Manually trigger scan via web UI
   - Restart container to force immediate scan

### Partial Device Discovery

**Symptom:** Some devices found, others missing

**Causes:**

1. **Device instance numbers outside scan range:**
   - Expand `LOW_LIMIT` and `HIGH_LIMIT`

2. **Devices on different subnet:**
   - Use BBMD/Foreign Device Registration
   ```bash
   BACNET_FOREIGN="192.168.1.1:47808"
   BACNET_TTL=30
   ```

3. **Slow devices:**
   - Increase scan interval
   - Reduce step sizes for more thorough scanning
   ```bash
   DEVICE_BROADCAST_FULL_STEP=10
   DEVICE_BROADCAST_EMPTY_STEP=100
   ```

### Duplicate Device Instance Errors

**Symptom:** Errors about duplicate BACnet instance numbers

**Cause:** Sentinel instance number conflicts with existing device

**Solution:**
```bash
# Change Sentinel instance number
BACNET_INSTANCE=999  # Use unique number
```

---

## Web UI Issues

### Cannot Access Web UI

**Symptom:** Browser cannot load http://CONTAINER_IP:5000

**Diagnose:**
```bash
# Check if web server is running
docker exec aceiot-sentinel curl -f http://localhost:5000

# Check port is listening
docker exec aceiot-sentinel netstat -tln | grep 5000
```

**Solutions:**

1. **Container not ready:**
   - Wait 60 seconds for startup
   - Check logs: `docker logs aceiot-sentinel`

2. **Wrong IP/port:**
   - Verify container IP: `docker inspect aceiot-sentinel | grep IPAddress`
   - Verify `WEBAPP_PORT` setting

3. **Firewall blocking:**
   ```bash
   # Allow port 5000
   sudo iptables -A INPUT -p tcp --dport 5000 -j ACCEPT
   ```

4. **WEBAPP_ENABLED=false:**
   - Check configuration
   - Set `WEBAPP_ENABLED=true` and restart

### SSL/TLS Certificate Errors

**Symptom:** Browser shows certificate warning

**Solutions:**

1. **Self-signed certificate:**
   - Expected behavior
   - Click "Advanced" → "Proceed anyway"
   - Or add exception in browser

2. **Certificate expired:**
   - Generate new certificate
   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout key.pem -out cert.pem
   ```

3. **Wrong certificate path:**
   - Verify certificate files are mounted
   - Check paths in configuration:
   ```bash
   WEBAPP_CERTFILE=/certs/cert.pem
   WEBAPP_KEYFILE=/certs/key.pem
   ```

### Web UI Shows No Data

**Symptom:** Web UI loads but shows empty network

**Causes:**

1. **No scan completed yet:**
   - Wait for first scan (check `SCAN_INTERVAL_SECS`)
   - Manually trigger scan via UI

2. **No devices discovered:**
   - See [BACnet Discovery Issues](#bacnet-discovery-issues)

3. **Data persistence issue:**
   - Check volume is mounted correctly
   ```bash
   docker inspect aceiot-sentinel | grep -A 10 Mounts
   ```

---

## ACE Manager Integration Issues

### ACE Config Agent Not Starting

**Symptom:** Logs show ACE Config Agent errors

**Check logs:**
```bash
docker logs aceiot-sentinel | grep -i "ace.*config"
```

**Solutions:**

1. **Missing required configuration:**
   ```
   ERROR: ACE Config Agent enabled but missing required configuration
   ```
   **Solution:** Set all required variables:
   ```bash
   ACE_CONFIG_ENABLED=true
   ACE_CONFIG_URL=https://manage.aceiot.cloud
   ACE_CONFIG_JWT=your-jwt-token
   ACE_CONFIG_GATEWAY=your-gateway-name
   ```

2. **Invalid JWT:**
   ```
   ERROR: Authentication failed
   ```
   **Solution:**
   - Verify JWT is correct
   - Generate new JWT from ACE Manager
   - Check JWT hasn't expired

3. **Cannot reach ACE Manager:**
   ```
   ERROR: Connection refused
   ```
   **Solution:**
   - Check `ACE_CONFIG_URL` is correct
   - Verify network connectivity
   ```bash
   docker exec aceiot-sentinel curl -I https://manage.aceiot.cloud
   ```
   - Check firewall allows outbound HTTPS

### Data Not Uploading to ACE Manager

**Symptom:** Scans complete but data doesn't appear in ACE Manager

**Diagnose:**
```bash
# Check cloud upload configuration
docker exec aceiot-sentinel env | grep CLOUD_UPLOAD

# Check ACE Config Agent status
docker exec aceiot-sentinel vctl status | grep ace-config
```

**Solutions:**

1. **Cloud upload not enabled:**
   ```bash
   CLOUD_UPLOAD_ENABLED=true
   ```

2. **JWT not retrieved:**
   - ACE Config Agent should provide JWT automatically
   - Check ACE Config Agent logs
   - Verify `ACE_CONFIG_ENABLED=true`

3. **Upload interval not reached:**
   - Check `CLOUD_UPLOAD_INTERVAL` (default 24 hours)
   - Force upload by restarting container

---

## Performance Issues

### Slow Scanning

**Symptom:** Scans take very long to complete

**Solutions:**

1. **Narrow device range:**
   ```bash
   # Only scan known device range
   LOW_LIMIT=1000
   HIGH_LIMIT=2000
   ```

2. **Increase step sizes:**
   ```bash
   # Skip more device numbers when empty
   DEVICE_BROADCAST_EMPTY_STEP=5000
   ```

3. **Reduce scan frequency:**
   ```bash
   # Scan less often
   SCAN_INTERVAL_SECS=86400  # Once per day
   ```

### High CPU Usage

**Check container resources:**
```bash
docker stats aceiot-sentinel
```

**Solutions:**

1. **Too many devices:**
   - Split across multiple sentinels
   - See [docker-compose.multi.yml](./examples/docker-compose.multi.yml)

2. **Frequent scans:**
   - Increase `SCAN_INTERVAL_SECS`

3. **Limit container resources:**
   ```yaml
   deploy:
     resources:
       limits:
         cpus: '2'
         memory: 2G
   ```

### High Memory Usage

**Solutions:**

1. **Reduce device range:**
   ```bash
   HIGH_LIMIT=10000
   ```

2. **Restart container periodically:**
   - Add to cron for weekly restart
   ```bash
   0 2 * * 0 docker restart aceiot-sentinel
   ```

---

## Data and Storage Issues

### Volume Space Full

**Check volume usage:**
```bash
docker system df -v | grep sentinel
```

**Solutions:**

1. **Clean up old data:**
   ```bash
   # Access volume
   docker exec aceiot-sentinel du -sh /home/volttron/.aceiot-sentinel-volttron/*

   # Remove old TTL files (if needed)
   docker exec aceiot-sentinel find /home/volttron/.aceiot-sentinel-volttron -name "*.ttl" -mtime +30 -delete
   ```

2. **Increase volume size:**
   - Backup data
   - Recreate volume with more space
   - Restore data

### Data Loss After Restart

**Cause:** Volume not properly mounted

**Solution:**

1. **Verify volume in docker-compose.yml:**
   ```yaml
   volumes:
     - volttron-data:/home/volttron/.aceiot-sentinel-volttron
   ```

2. **Check volume exists:**
   ```bash
   docker volume ls | grep volttron
   ```

3. **Restore from backup:**
   ```bash
   docker run --rm \
     -v volttron-data:/data \
     -v $(pwd):/backup \
     alpine tar xzf /backup/sentinel-backup.tar.gz -C /data
   ```

---

## Getting Help

If issues persist:

1. **Collect diagnostic information:**
   ```bash
   # Container logs
   docker logs aceiot-sentinel > sentinel-logs.txt

   # Container configuration
   docker inspect aceiot-sentinel > sentinel-config.json

   # Network configuration
   docker network inspect bacnet-network > network-config.json

   # VOLTTRON logs
   docker exec aceiot-sentinel cat /tmp/volttron.log > volttron-logs.txt
   ```

2. **Contact support:**
   - Email: support@aceiotsolutions.com
   - Include: Logs, configuration files, network diagram
   - Describe: Expected behavior vs actual behavior

3. **Report issues:**
   - GitHub Issues: https://github.com/ACE-IoT-Solutions/aceiot-sentinel-container/issues
   - Include: Version, environment details, reproduction steps

---

## See Also

- [Configuration Reference](./CONFIGURATION.md)
- [Network Configuration](./NETWORKING.md)
- [Deployment Guide](./DEPLOYMENT.md)
