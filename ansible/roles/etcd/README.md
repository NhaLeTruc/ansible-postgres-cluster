# Ansible Role: etcd

## Description

This role installs and configures [etcd](https://etcd.io/), a distributed, reliable key-value store for the most critical data of a distributed system. In this PostgreSQL HA cluster setup, etcd serves as the Distributed Configuration Store (DCS) for Patroni, providing:

- **Distributed consensus** for leader election
- **Cluster state storage** for PostgreSQL cluster metadata
- **Configuration management** for dynamic cluster reconfiguration
- **High availability** through multi-node deployment

etcd is the recommended DCS for production PostgreSQL HA clusters due to its maturity, performance, and robust consensus algorithm (Raft).

## Requirements

### System Requirements
- **Operating System**: Debian/Ubuntu, RedHat/CentOS, Rocky Linux, AlmaLinux
- **Architecture**: x86_64, arm64
- **Memory**: Minimum 1GB RAM (2GB+ recommended for production)
- **Disk**: Fast storage recommended (SSD preferred for production)

### Network Requirements
- **Ports**:
  - `2379`: Client requests (etcd API)
  - `2380`: Peer communication (etcd cluster)
- **Network latency**: <10ms between etcd nodes recommended
- **Odd number of nodes**: Deploy 3, 5, or 7 nodes for quorum (never even numbers)

### Dependencies
- None (this is a foundational component)

## Role Variables

### Installation Configuration

```yaml
# Installation method: "repo" or "file"
installation_method: "repo"

# etcd version to install
etcd_version: "3.5.11"  # Specify exact version for production

# etcd download URL (when installation_method: "repo")
etcd_download_url: "https://github.com/etcd-io/etcd/releases/download/v{{ etcd_version }}/etcd-v{{ etcd_version }}-linux-{{ etcd_architecture }}.tar.gz"
```

### Cluster Configuration

```yaml
# etcd cluster name (must be unique in your network)
etcd_cluster_name: "etcd-cluster"

# etcd data directory
etcd_data_dir: "/var/lib/etcd"

# Initial cluster state: "new" or "existing"
etcd_initial_cluster_state: "new"  # Use "existing" when adding nodes to running cluster

# Initial cluster token (prevents cross-cluster communication)
etcd_initial_cluster_token: "etcd-cluster-token"
```

### Network Configuration

```yaml
# Client listen URLs (where etcd listens for client requests)
etcd_listen_client_urls: "http://{{ inventory_hostname }}:2379,http://127.0.0.1:2379"

# Advertise client URLs (how clients should connect to this etcd member)
etcd_advertise_client_urls: "http://{{ inventory_hostname }}:2379"

# Peer listen URLs (where etcd listens for peer communication)
etcd_listen_peer_urls: "http://{{ inventory_hostname }}:2380"

# Advertise peer URLs (how peers should connect to this etcd member)
etcd_initial_advertise_peer_urls: "http://{{ inventory_hostname }}:2380"
```

### Performance Tuning

```yaml
# Snapshot count (number of transactions before snapshot)
etcd_snapshot_count: 10000

# Heartbeat interval (milliseconds)
etcd_heartbeat_interval: 100

# Election timeout (milliseconds)
etcd_election_timeout: 1000

# Max snapshots to retain
etcd_max_snapshots: 5

# Max WAL files to retain
etcd_max_wals: 5
```

### Security Configuration

```yaml
# Enable TLS for client connections
etcd_client_cert_auth: false

# Enable TLS for peer connections
etcd_peer_cert_auth: false

# Certificate paths (when TLS is enabled)
etcd_cert_file: ""
etcd_key_file: ""
etcd_trusted_ca_file: ""
etcd_peer_cert_file: ""
etcd_peer_key_file: ""
etcd_peer_trusted_ca_file: ""
```

### Logging Configuration

```yaml
# Log level: debug, info, warn, error, panic, fatal
etcd_log_level: "info"

# Log output: default, stderr, stdout, or file path
etcd_log_outputs: "default"
```

## Dependencies

This role has no dependencies but is required by:
- `patroni` - Uses etcd for cluster coordination

## Example Playbook

### Basic 3-Node Cluster

```yaml
- name: Deploy etcd cluster
  hosts: etcd
  become: true
  roles:
    - role: etcd
      vars:
        etcd_cluster_name: "pg-etcd-cluster"
        etcd_version: "3.5.11"
```

### Production Deployment with Custom Configuration

```yaml
- name: Deploy production etcd cluster
  hosts: etcd
  become: true
  roles:
    - role: etcd
      vars:
        etcd_cluster_name: "production-etcd"
        etcd_version: "3.5.11"

        # Data directory on separate disk
        etcd_data_dir: "/data/etcd"

        # Performance tuning for low-latency network
        etcd_heartbeat_interval: 50
        etcd_election_timeout: 500

        # Increase snapshot retention
        etcd_max_snapshots: 10
        etcd_max_wals: 10

        # Logging
        etcd_log_level: "warn"
```

### Inventory Example

```ini
[etcd]
etcd1 ansible_host=10.0.1.10
etcd2 ansible_host=10.0.1.11
etcd3 ansible_host=10.0.1.12

[etcd:vars]
etcd_cluster_name=pg-etcd-cluster
```

## Handlers

This role defines the following handlers:

- **restart etcd**: Restarts the etcd service
- **reload etcd**: Gracefully reloads etcd configuration
- **etcd systemd daemon-reload**: Reloads systemd after service file changes

## Files and Directories Created

### Binary Files
- `/usr/local/bin/etcd`: etcd server binary
- `/usr/local/bin/etcdctl`: etcd command-line client
- `/usr/local/bin/etcdutl`: etcd utility for offline operations

### Configuration Files
- `/etc/systemd/system/etcd.service`: Systemd service unit
- `/etc/default/etcd`: Environment variables and configuration

### Data Directories
- `/var/lib/etcd/`: Data directory (contains member info, WAL, and snapshots)
  - `/var/lib/etcd/member/`: Member data
  - `/var/lib/etcd/member/wal/`: Write-Ahead Log
  - `/var/lib/etcd/member/snap/`: Snapshots

### Log Files
- Logs via systemd journal: `journalctl -u etcd`
- Or custom log file if `etcd_log_outputs` is configured

## Post-Installation

### Verify etcd is Running

```bash
# Check service status
systemctl status etcd

# Verify etcd is healthy
etcdctl endpoint health

# List cluster members
etcdctl member list

# Check cluster health
etcdctl endpoint status --cluster -w table
```

### Using etcdctl

```bash
# Set endpoint (if not using default)
export ETCDCTL_ENDPOINTS=http://10.0.1.10:2379

# Put a key-value pair
etcdctl put mykey myvalue

# Get a value
etcdctl get mykey

# Delete a key
etcdctl del mykey

# List all keys
etcdctl get "" --prefix --keys-only
```

### Monitor Cluster

```bash
# Check member list
etcdctl member list -w table

# Check endpoint health
etcdctl endpoint health --cluster

# Check endpoint status with metrics
etcdctl endpoint status --cluster -w table

# Check alarms (e.g., NOSPACE)
etcdctl alarm list
```

## Cluster Operations

### Adding a New Member

```bash
# On existing cluster, add new member
etcdctl member add etcd4 --peer-urls=http://10.0.1.13:2380

# On new node, set initial_cluster_state to "existing"
# and use the full cluster configuration including the new member
```

### Removing a Member

```bash
# Get member ID
etcdctl member list

# Remove member
etcdctl member remove <member-id>
```

### Disaster Recovery

```bash
# Create snapshot
etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db

# Restore from snapshot (on all nodes)
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --name etcd1 \
  --initial-cluster etcd1=http://10.0.1.10:2380,etcd2=http://10.0.1.11:2380,etcd3=http://10.0.1.12:2380 \
  --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --data-dir /var/lib/etcd-restore
```

## Troubleshooting

### etcd Fails to Start

**Check logs:**
```bash
journalctl -u etcd -n 50 --no-pager
```

**Common issues:**
- Port already in use (2379, 2380)
- Data directory permissions incorrect
- Network connectivity between nodes
- Initial cluster configuration mismatch

### Cluster Not Forming

**Verify connectivity:**
```bash
# From each node, test connectivity to peers
curl http://<peer-ip>:2380/version
```

**Check configuration:**
- Ensure `initial_cluster` includes all members
- Verify `initial_cluster_state` is "new" for new clusters
- Confirm hostnames/IPs are correct and reachable

### High Latency or Slow Performance

**Check disk performance:**
```bash
# etcd is very sensitive to disk I/O
# Check backend commit duration (should be <25ms)
etcdctl endpoint status --cluster -w json | jq '.[] | {endpoint: .Endpoint, commit_duration: .Status.raftAppliedIndex}'
```

**Solutions:**
- Move data directory to SSD
- Reduce `etcd_snapshot_count`
- Increase `etcd_heartbeat_interval` and `etcd_election_timeout`
- Add more memory

### NOSPACE Alarm

**Check space:**
```bash
etcdctl alarm list
etcdctl endpoint status --cluster -w table
```

**Solutions:**
```bash
# Compact old revisions
etcdctl compact <revision>

# Defragment (on each member)
etcdctl defrag --cluster

# Disarm alarm
etcdctl alarm disarm
```

### Split-Brain or Leader Election Issues

**Check cluster status:**
```bash
etcdctl endpoint status --cluster -w table
```

**Look for:**
- Multiple leaders (should only be one)
- Members unreachable
- High network latency between nodes

**Solutions:**
- Check network connectivity
- Verify firewall rules
- Increase election timeout for high-latency networks
- Ensure odd number of nodes

## Security Considerations

### 1. Network Security

```yaml
# Restrict etcd access to cluster nodes only via firewall
# Allow only PostgreSQL cluster nodes to access client port 2379
# Allow only etcd cluster nodes to access peer port 2380
```

### 2. Enable TLS (Recommended for Production)

```yaml
etcd_client_cert_auth: true
etcd_peer_cert_auth: true
etcd_cert_file: "/etc/etcd/ssl/server.crt"
etcd_key_file: "/etc/etcd/ssl/server.key"
etcd_trusted_ca_file: "/etc/etcd/ssl/ca.crt"
```

### 3. Enable Authentication

```bash
# Enable authentication
etcdctl user add root
etcdctl auth enable

# Create role for Patroni
etcdctl role add patroni
etcdctl role grant-permission patroni readwrite /service/
```

### 4. Regular Backups

```bash
# Automated snapshot backup (add to cron)
0 2 * * * /usr/local/bin/etcdctl snapshot save /backup/etcd-$(date +\%Y\%m\%d).db
```

## Performance Tuning

### For Low-Latency Networks (<5ms)

```yaml
etcd_heartbeat_interval: 50
etcd_election_timeout: 500
```

### For High-Latency Networks (>10ms)

```yaml
etcd_heartbeat_interval: 200
etcd_election_timeout: 2000
```

### For Large Data Sets

```yaml
etcd_snapshot_count: 5000  # More frequent snapshots
etcd_max_snapshots: 10     # Retain more snapshots
etcd_quota_backend_bytes: 8589934592  # 8GB quota
```

### For Write-Heavy Workloads

```yaml
# Use SSD storage
# Increase memory
# Consider larger cluster (5 or 7 nodes)
etcd_heartbeat_interval: 100
etcd_election_timeout: 1000
etcd_snapshot_count: 10000
```

## Monitoring

### Key Metrics to Monitor

- **Leader changes**: Should be rare (indicates instability)
- **Proposal commit duration**: Should be <25ms
- **Backend commit duration**: Should be <10ms
- **Disk sync duration**: Should be <10ms
- **Applied index lag**: Should be 0 or minimal
- **DB size**: Monitor for growth

### Prometheus Metrics

etcd exposes metrics at `http://localhost:2379/metrics`

## Best Practices

1. **Always use odd number of nodes** (3, 5, or 7)
2. **Use fast disks** (SSD recommended)
3. **Separate etcd from PostgreSQL** (dedicated nodes in production)
4. **Enable TLS** in production environments
5. **Regular backups** of etcd data
6. **Monitor disk space** and set up alerts
7. **Keep etcd version consistent** across cluster
8. **Test failover scenarios** before production
9. **Plan for disaster recovery** (document restore procedure)
10. **Limit cluster size** (3 nodes sufficient for most deployments)

## Known Limitations

- Maximum recommended cluster size: 7 nodes (more nodes = slower writes)
- Not designed for large data sets (use for configuration only)
- Requires low-latency network between nodes
- Database size limit (default 2GB, configurable up to 8GB)
- Not suitable for cross-datacenter deployments with high latency

## Additional Resources

- [Official etcd Documentation](https://etcd.io/docs/)
- [etcd GitHub Repository](https://github.com/etcd-io/etcd)
- [etcd Operations Guide](https://etcd.io/docs/v3.5/op-guide/)
- [etcd FAQ](https://etcd.io/docs/v3.5/faq/)
- [Raft Consensus Algorithm](https://raft.github.io/)

## License

Same as the main project

## Author Information

Part of the PostgreSQL HA Cluster automation project
