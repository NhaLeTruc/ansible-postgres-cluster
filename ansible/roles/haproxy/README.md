# Ansible Role: haproxy

## Description

This role installs and configures [HAProxy](http://www.haproxy.org/), a reliable, high-performance TCP/HTTP load balancer. In this PostgreSQL HA cluster setup, HAProxy provides:

- **Load balancing** for PostgreSQL connections
- **Health checking** to route traffic only to healthy nodes
- **Automatic failover** by detecting primary/replica status
- **Connection pooling** to reduce overhead
- **Statistics dashboard** for monitoring
- **SSL termination** (optional)

HAProxy sits between applications and PostgreSQL, routing read/write traffic to the primary and read-only traffic to replicas.

## Requirements

### System Requirements
- **Operating System**: Debian/Ubuntu, RedHat/CentOS, Rocky Linux, AlmaLinux
- **Memory**: Minimum 512MB RAM (1GB+ recommended)
- **CPU**: Minimal (HAProxy is very efficient)

### Network Requirements
- **Ports**:
  - `5000`: PostgreSQL primary (default)
  - `5001`: PostgreSQL replicas (default)
  - `5002`: PostgreSQL replicas (sync priority, optional)
  - `5003`: PostgreSQL replicas (async, optional)
  - `7000`: HAProxy statistics page (default)

### Dependencies
- `patroni` - HAProxy checks Patroni REST API for node health

## Role Variables

### Installation Configuration

```yaml
# Install from OS repository
installation_method: "repo"
```

### Basic Configuration

```yaml
# HAProxy configuration file
haproxy_config_file: "/etc/haproxy/haproxy.cfg"

# Listen address
haproxy_listen_address: "0.0.0.0"

# Primary port (read-write)
haproxy_listen_port_primary: 5000

# Replica port (read-only)
haproxy_listen_port_replica: 5001

# Replica sync port (synchronous replicas only)
haproxy_listen_port_replica_sync: 5002

# Replica async port
haproxy_listen_port_replica_async: 5003
```

### PostgreSQL Backend Configuration

```yaml
# Backend servers (auto-populated from inventory)
haproxy_backend_servers: "{{ groups['postgres_cluster'] }}"

# PostgreSQL port
postgresql_port: 5432

# Patroni REST API port
patroni_rest_api_port: 8008
```

### Health Check Configuration

```yaml
# Health check interval
haproxy_check_interval: "3s"

# Health check timeout
haproxy_check_timeout: "1s"

# Number of failed checks before marking down
haproxy_check_fall: 3

# Number of successful checks before marking up
haproxy_check_rise: 2
```

### Performance Tuning

```yaml
# Maximum connections
haproxy_maxconn_global: 10000
haproxy_maxconn_frontend: 4000

# Timeout settings
haproxy_timeout_connect: "10s"
haproxy_timeout_client: "1m"
haproxy_timeout_server: "1m"
haproxy_timeout_check: "10s"
```

### Statistics Configuration

```yaml
# Enable statistics page
haproxy_stats_enable: true

# Statistics port
haproxy_stats_port: 7000

# Statistics URI
haproxy_stats_uri: "/stats"

# Statistics username
haproxy_stats_user: "admin"

# Statistics password
haproxy_stats_password: "{{ vault_haproxy_stats_password }}"

# Allow statistics refresh
haproxy_stats_refresh: "10s"
```

### Logging Configuration

```yaml
# Syslog server
haproxy_syslog_server: "127.0.0.1:514"

# Log level: emerg, alert, crit, err, warning, notice, info, debug
haproxy_log_level: "info"
```

## Dependencies

This role depends on:
- `patroni` - For health check endpoint

## Example Playbook

### Basic Load Balancer

```yaml
- name: Deploy HAProxy
  hosts: balancers
  become: true
  roles:
    - role: haproxy
      vars:
        haproxy_listen_port_primary: 5000
        haproxy_listen_port_replica: 5001
        haproxy_stats_password: "{{ vault_haproxy_password }}"
```

### Production Setup with Custom Timeouts

```yaml
- name: Deploy HAProxy for production
  hosts: balancers
  become: true
  roles:
    - role: haproxy
      vars:
        # Connection settings
        haproxy_maxconn_global: 20000
        haproxy_maxconn_frontend: 8000

        # Timeouts
        haproxy_timeout_connect: "5s"
        haproxy_timeout_client: "5m"
        haproxy_timeout_server: "5m"

        # Health checks
        haproxy_check_interval: "2s"
        haproxy_check_fall: 3
        haproxy_check_rise: 2

        # Statistics
        haproxy_stats_enable: true
        haproxy_stats_port: 7000
        haproxy_stats_password: "{{ vault_haproxy_password }}"
```

### Multiple Replica Pools

```yaml
- name: Deploy HAProxy with separated replica pools
  hosts: balancers
  become: true
  roles:
    - role: haproxy
      vars:
        haproxy_listen_port_primary: 5000
        haproxy_listen_port_replica_sync: 5001  # Sync replicas only
        haproxy_listen_port_replica_async: 5002  # Async replicas
        haproxy_listen_port_replica: 5003  # All replicas
```

## Handlers

This role defines the following handlers:

- **restart haproxy**: Restarts the HAProxy service
- **reload haproxy**: Gracefully reloads HAProxy configuration
- **haproxy systemd daemon-reload**: Reloads systemd after service file changes

## Files and Directories Created

### Configuration Files
- `/etc/haproxy/haproxy.cfg`: Main configuration file

### Service Files
- `/etc/systemd/system/haproxy.service`: Systemd service unit (if customized)

### Log Files
- Logs via syslog: `/var/log/haproxy.log` (if configured)
- Or via systemd journal: `journalctl -u haproxy`

## Post-Installation

### Verify HAProxy is Running

```bash
# Check service status
systemctl status haproxy

# Verify listening ports
ss -tlnp | grep haproxy

# Check configuration syntax
haproxy -c -f /etc/haproxy/haproxy.cfg
```

### Access Statistics Page

```
http://<haproxy-ip>:7000/stats
```

Login with username: `admin` and configured password.

### Test PostgreSQL Connections

```bash
# Connect to primary (read-write)
psql -h <haproxy-ip> -p 5000 -U postgres

# Connect to replica (read-only)
psql -h <haproxy-ip> -p 5001 -U postgres
```

### Monitor Backend Status

```bash
# Check backend status via stats socket
echo "show stat" | socat stdio /var/run/haproxy.sock
```

## Architecture

```
Application
    |
    v
HAProxy (balancer nodes)
    |
    +--- Port 5000 -> Primary PostgreSQL (read-write)
    |
    +--- Port 5001 -> Replica PostgreSQL (read-only)
    |
    +--- Port 5002 -> Sync Replicas (read-only, synchronous only)
    |
    +--- Port 5003 -> Async Replicas (read-only, asynchronous)
```

## Health Check Mechanism

HAProxy uses Patroni's REST API to determine node status:

1. **Primary Detection**: Checks `/primary` endpoint
2. **Replica Detection**: Checks `/replica` endpoint
3. **Sync Replica Detection**: Checks `/sync` or `/replica?lag=0` endpoint
4. **Health Status**: Returns HTTP 200 if healthy, 503 if not

## Troubleshooting

### HAProxy Fails to Start

**Check logs:**
```bash
journalctl -u haproxy -n 50 --no-pager
```

**Common issues:**
- Port already in use
- Configuration syntax error
- Insufficient permissions

### Cannot Connect to PostgreSQL via HAProxy

**Check backend status:**
```bash
# Via stats page
curl http://admin:password@localhost:7000/stats

# Or check logs
tail -f /var/log/haproxy.log
```

**Common issues:**
- All backends down (check Patroni health)
- Firewall blocking connections
- Wrong PostgreSQL port in configuration
- HAProxy not running

### All Backends Showing as DOWN

**Verify Patroni health:**
```bash
# Check Patroni REST API directly
curl http://<postgres-node>:8008/health
curl http://<postgres-node>:8008/primary
curl http://<postgres-node>:8008/replica
```

**Common issues:**
- Patroni not running
- Wrong Patroni API port
- Network connectivity issues
- Firewall blocking health checks

### High Latency

**Check HAProxy stats:**
```bash
echo "show stat" | socat stdio /var/run/haproxy.sock
```

**Look for:**
- High queue times
- Connection limits reached
- Slow backend responses

**Solutions:**
- Increase `maxconn` values
- Adjust timeout settings
- Add more backend servers
- Optimize PostgreSQL queries

## Security Considerations

### 1. Protect Statistics Page

```yaml
haproxy_stats_user: "admin"
haproxy_stats_password: "{{ vault_haproxy_stats_password }}"
```

Restrict access via firewall:
```bash
# Allow only from admin network
iptables -A INPUT -p tcp --dport 7000 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 7000 -j DROP
```

### 2. Use Strong Passwords

Store in Ansible Vault:
```bash
ansible-vault edit group_vars/all/vault.yml
```

### 3. Restrict PostgreSQL Access

Configure `pg_hba.conf` to only allow connections from HAProxy:
```
# Only allow HAProxy nodes
host  all  all  <haproxy-ip>/32  md5
```

### 4. Enable SSL (if needed)

```yaml
# In haproxy configuration
haproxy_ssl_cert: "/etc/haproxy/ssl/cert.pem"
haproxy_ssl_enable: true
```

### 5. Network Segmentation

- Place HAProxy in DMZ or application network
- Restrict PostgreSQL backend network
- Use VPC/firewall rules

## Performance Tuning

### For High-Traffic Applications

```yaml
haproxy_maxconn_global: 50000
haproxy_maxconn_frontend: 20000
haproxy_timeout_connect: "3s"
haproxy_timeout_client: "2m"
haproxy_timeout_server: "2m"
```

### For Long-Running Queries

```yaml
haproxy_timeout_client: "10m"
haproxy_timeout_server: "10m"
```

### For Low-Latency Requirements

```yaml
haproxy_check_interval: "1s"
haproxy_check_fall: 2
haproxy_check_rise: 1
haproxy_timeout_connect: "1s"
```

## Monitoring

### Key Metrics to Monitor

- **Backend status**: UP/DOWN status of PostgreSQL nodes
- **Queue size**: Number of queued connections
- **Session rate**: Connections per second
- **Response time**: Backend response latency
- **Error rate**: Failed health checks

### Prometheus Integration

HAProxy can expose metrics for Prometheus:

1. Install haproxy_exporter
2. Configure to scrape HAProxy stats
3. Set up Grafana dashboards

### Log Analysis

```bash
# Watch connections
tail -f /var/log/haproxy.log | grep "Connect from"

# Watch backend changes
tail -f /var/log/haproxy.log | grep "Server.*is UP"
```

## Connection Flow Examples

### Application Connection to Primary

```
App -> HAProxy:5000 -> Check /primary -> Route to Primary Node:5432
```

### Application Connection to Replica

```
App -> HAProxy:5001 -> Check /replica -> Round-robin to Replica:5432
```

## Best Practices

1. **Deploy multiple HAProxy instances** for HA (use keepalived or VIP)
2. **Monitor backend health** continuously
3. **Set appropriate timeouts** based on query patterns
4. **Use connection pooling** in applications
5. **Separate read and write traffic** (different ports)
6. **Enable statistics page** for troubleshooting
7. **Log connection events** for auditing
8. **Test failover scenarios** regularly
9. **Document port assignments**
10. **Monitor connection limits**

## High Availability Setup

For HAProxy HA, combine with keepalived:

```yaml
# Deploy HAProxy on multiple nodes
- hosts: balancer1, balancer2
  roles:
    - haproxy
    - keepalived  # Manages VIP
```

Virtual IP (VIP) floats between HAProxy instances.

## Known Limitations

- Does not perform connection pooling (use pgBouncer for that)
- Health checks create constant low-level traffic
- Statistics page is basic (consider using Prometheus + Grafana)
- No built-in query routing (all reads go to replicas equally)

## Additional Resources

- [Official HAProxy Documentation](http://www.haproxy.org/#docs)
- [HAProxy Configuration Manual](http://cbonte.github.io/haproxy-dconv/)
- [HAProxy Management Guide](http://www.haproxy.org/download/2.0/doc/management.txt)
- [Patroni REST API Reference](https://patroni.readthedocs.io/en/latest/rest_api.html)

## License

Same as the main project

## Author Information

Part of the PostgreSQL HA Cluster automation project
