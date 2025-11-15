# Ansible Role: consul

## Description

This role installs and configures [HashiCorp Consul](https://www.consul.io/), a service mesh solution providing service discovery, configuration, and segmentation functionality. In this PostgreSQL HA cluster setup, Consul serves as an alternative Distributed Configuration Store (DCS) for Patroni, providing:

- **Service discovery** and health checking
- **Distributed consensus** for leader election (using Raft algorithm)
- **Key/value store** for PostgreSQL cluster metadata
- **Multi-datacenter support** for geographically distributed clusters

Consul is a viable alternative to etcd, particularly useful for organizations already using HashiCorp ecosystem or requiring multi-datacenter deployments.

## Requirements

### System Requirements
- **Operating System**: Debian/Ubuntu, RedHat/CentOS, Rocky Linux, AlmaLinux
- **Architecture**: x86_64, arm, arm64
- **Memory**: Minimum 512MB RAM (2GB+ recommended for production)
- **Disk**: Moderate disk I/O requirements

### Network Requirements
- **Ports**:
  - `8300`: Server RPC (cluster communication)
  - `8301`: Serf LAN (gossip protocol)
  - `8302`: Serf WAN (multi-datacenter gossip)
  - `8500`: HTTP API
  - `8600`: DNS interface
- **Odd number of server nodes**: Deploy 3 or 5 server nodes for quorum
- **Client nodes**: Can deploy on PostgreSQL nodes for local agent

### Dependencies
- None (this is a foundational component)

## Role Variables

### Installation Configuration

```yaml
# Installation method: "repo" or "file"
installation_method: "repo"

# Consul version to install
consul_version: "1.17.1"

# Consul download URL pattern
consul_download_url: "https://releases.hashicorp.com/consul/{{ consul_version }}/consul_{{ consul_version }}_linux_{{ consul_architecture }}.zip"
```

### Cluster Configuration

```yaml
# Consul datacenter name
consul_datacenter: "dc1"

# Consul cluster name/domain
consul_domain: "consul"

# Node role: "server" or "client"
consul_node_role: "server"  # Set to "client" for PostgreSQL nodes

# Number of server nodes (for bootstrap expect)
consul_bootstrap_expect: 3  # Must match number of server nodes

# Retry join addresses (list of server IPs/hostnames)
consul_retry_join:
  - "10.0.1.10"
  - "10.0.1.11"
  - "10.0.1.12"
```

### Data and Logging

```yaml
# Consul data directory
consul_data_dir: "/var/lib/consul"

# Consul configuration directory
consul_config_dir: "/etc/consul.d"

# Log level: trace, debug, info, warn, err
consul_log_level: "info"

# Enable syslog
consul_enable_syslog: false
```

### Network Configuration

```yaml
# Bind address (internal cluster communication)
consul_bind_address: "{{ inventory_hostname }}"

# Client address (API/HTTP access)
consul_client_address: "0.0.0.0"

# Advertise address (how others should reach this node)
consul_advertise_address: "{{ inventory_hostname }}"
```

### Security Configuration

```yaml
# Enable ACLs (Access Control Lists)
consul_acl_enabled: false
consul_acl_default_policy: "deny"  # or "allow"
consul_acl_master_token: ""  # Set in production

# Enable encryption
consul_encrypt_enabled: false
consul_encrypt_key: ""  # Generate with: consul keygen

# Enable TLS
consul_tls_enabled: false
consul_tls_cert_file: ""
consul_tls_key_file: ""
consul_tls_ca_file: ""
```

### Performance Tuning

```yaml
# Raft multiplier (higher = more tolerance for slow networks)
consul_raft_multiplier: 1  # 1-10, higher for high-latency networks

# Leave on terminate
consul_leave_on_terminate: false

# Rejoin after leave
consul_rejoin_after_leave: true
```

## Dependencies

This role has no dependencies but is required by:
- `patroni` - Can use Consul as DCS instead of etcd

## Example Playbook

### Basic 3-Server Cluster

```yaml
- name: Deploy Consul servers
  hosts: consul_servers
  become: true
  roles:
    - role: consul
      vars:
        consul_node_role: "server"
        consul_bootstrap_expect: 3
        consul_retry_join:
          - "consul1"
          - "consul2"
          - "consul3"

- name: Deploy Consul clients on PostgreSQL nodes
  hosts: postgres_cluster
  become: true
  roles:
    - role: consul
      vars:
        consul_node_role: "client"
        consul_retry_join:
          - "consul1"
          - "consul2"
          - "consul3"
```

### Production Deployment with Security

```yaml
- name: Deploy Consul with ACLs and encryption
  hosts: consul_servers
  become: true
  roles:
    - role: consul
      vars:
        consul_version: "1.17.1"
        consul_node_role: "server"
        consul_datacenter: "production"
        consul_bootstrap_expect: 3

        # Security
        consul_acl_enabled: true
        consul_acl_default_policy: "deny"
        consul_acl_master_token: "{{ vault_consul_master_token }}"
        consul_encrypt_enabled: true
        consul_encrypt_key: "{{ vault_consul_encrypt_key }}"

        # Performance
        consul_raft_multiplier: 1
        consul_log_level: "warn"

        # Retry join
        consul_retry_join:
          - "10.0.1.10"
          - "10.0.1.11"
          - "10.0.1.12"
```

## Handlers

This role defines the following handlers:

- **restart consul**: Restarts the Consul service
- **reload consul**: Gracefully reloads Consul configuration
- **consul systemd daemon-reload**: Reloads systemd after service file changes

## Files and Directories Created

### Binary Files
- `/usr/local/bin/consul`: Consul binary

### Configuration Files
- `/etc/systemd/system/consul.service`: Systemd service unit
- `/etc/consul.d/consul.json`: Main configuration file
- `/etc/consul.d/server.json` or `/etc/consul.d/client.json`: Role-specific config

### Data Directories
- `/var/lib/consul/`: Data directory (Raft logs, snapshots, state)

### Log Files
- Logs via systemd journal: `journalctl -u consul`
- Or syslog if enabled

## Post-Installation

### Verify Consul is Running

```bash
# Check service status
systemctl status consul

# Verify cluster members
consul members

# Check cluster leader
consul operator raft list-peers

# Verify health
consul monitor
```

### Using Consul CLI

```bash
# List members
consul members

# Get catalog services
consul catalog services

# Get cluster leader
consul operator raft list-peers

# KV operations
consul kv put mykey myvalue
consul kv get mykey
consul kv delete mykey
```

### Check Cluster Health

```bash
# Via API
curl http://localhost:8500/v1/status/leader
curl http://localhost:8500/v1/agent/members

# Via DNS
dig @127.0.0.1 -p 8600 consul.service.consul
```

## Cluster Operations

### Bootstrap ACLs

```bash
# Initialize ACL system (run on one server)
consul acl bootstrap

# This outputs the initial management token
# Save it securely!
```

### Generate Encryption Key

```bash
# Generate gossip encryption key
consul keygen

# Output example: pUqJrVyVRj5jsiYEkM/tFQYfWyJIv4s3XkvDwy7Cu5s=
# Add to configuration: consul_encrypt_key
```

### Add/Remove Members

```bash
# Members auto-join via retry_join configuration
# To manually join:
consul join <server-ip>

# To gracefully remove:
consul leave

# To forcefully remove failed member:
consul force-leave <node-name>
```

### Backup and Restore

```bash
# Create snapshot
consul snapshot save backup.snap

# Inspect snapshot
consul snapshot inspect backup.snap

# Restore snapshot
consul snapshot restore backup.snap
```

## Troubleshooting

### Consul Fails to Start

**Check logs:**
```bash
journalctl -u consul -n 50 --no-pager
```

**Common issues:**
- Port conflicts (8300, 8301, 8500)
- Incorrect bind/advertise addresses
- Data directory permissions
- Bootstrap expect mismatch

### Cluster Not Forming

**Verify members:**
```bash
consul members
```

**Check connectivity:**
```bash
# Test TCP connectivity
nc -zv <peer-ip> 8300
nc -zv <peer-ip> 8301
```

**Check configuration:**
- Verify retry_join addresses are correct
- Ensure bootstrap_expect matches actual server count
- Check firewall rules allow Consul ports

### High CPU or Memory Usage

**Check Raft state:**
```bash
consul operator raft list-peers
consul monitor
```

**Common causes:**
- Too many KV writes
- Excessive service registrations
- Network instability causing frequent elections
- Insufficient resources

**Solutions:**
- Increase memory allocation
- Reduce KV write frequency
- Stabilize network
- Review and optimize service checks

### Split-Brain

**Identify:**
```bash
# Multiple leaders or no leader
consul operator raft list-peers
```

**Resolution:**
1. Identify which servers have quorum
2. Stop minority servers
3. Let majority elect new leader
4. Rejoin minority servers

## Security Considerations

### 1. Enable ACLs (Recommended)

```yaml
consul_acl_enabled: true
consul_acl_default_policy: "deny"
consul_acl_master_token: "{{ vault_consul_master_token }}"
```

```bash
# Create token for Patroni
consul acl policy create -name patroni -rules @patroni-policy.hcl
consul acl token create -description "Patroni token" -policy-name patroni
```

### 2. Enable Gossip Encryption

```yaml
consul_encrypt_enabled: true
consul_encrypt_key: "{{ vault_consul_encrypt_key }}"
```

### 3. Enable TLS

```yaml
consul_tls_enabled: true
consul_tls_cert_file: "/etc/consul.d/ssl/consul.crt"
consul_tls_key_file: "/etc/consul.d/ssl/consul.key"
consul_tls_ca_file: "/etc/consul.d/ssl/ca.crt"
```

### 4. Network Security

- Restrict ports via firewall
- Use private network for cluster communication
- Enable TLS for API access in production

### 5. Regular Backups

```bash
# Daily snapshots via cron
0 2 * * * /usr/local/bin/consul snapshot save /backup/consul-$(date +\%Y\%m\%d).snap
```

## Performance Tuning

### For Low-Latency Networks

```yaml
consul_raft_multiplier: 1
```

### For High-Latency Networks (>50ms)

```yaml
consul_raft_multiplier: 5
```

### For Large Clusters

```yaml
# Use clients instead of all servers
# Keep server count to 3 or 5
# Deploy client agents on application nodes
```

## Monitoring

### Key Metrics

- **consul.raft.leader**: Leader status
- **consul.raft.commitTime**: Raft commit latency
- **consul.raft.apply**: Raft apply rate
- **consul.serf.member.flap**: Member flapping
- **consul.runtime.alloc_bytes**: Memory usage

### Prometheus Integration

Consul exposes metrics at `/v1/agent/metrics?format=prometheus`

## Consul vs etcd

| Feature | Consul | etcd |
|---------|--------|------|
| **Service Discovery** | Built-in | Requires external |
| **Health Checks** | Built-in | Requires external |
| **Multi-DC** | Native support | Requires setup |
| **UI** | Built-in web UI | External tools |
| **Maturity** | Mature | Very mature |
| **Best for** | Service mesh, multi-DC | Kubernetes, simple KV |

## Best Practices

1. **Use 3 or 5 server nodes** (odd number for quorum)
2. **Deploy client agents** on application nodes
3. **Enable ACLs** in production
4. **Enable gossip encryption** for security
5. **Regular snapshots** for backup
6. **Monitor Raft metrics** for health
7. **Use consistent Consul version** across cluster
8. **Test failover** before production
9. **Document recovery procedures**
10. **Set up alerting** for leader changes

## Known Limitations

- Higher memory footprint than etcd
- More complex than etcd for simple KV use cases
- WAN federation requires careful planning
- ACL system requires initial setup

## Additional Resources

- [Official Consul Documentation](https://www.consul.io/docs)
- [Consul GitHub Repository](https://github.com/hashicorp/consul)
- [Consul Architecture](https://www.consul.io/docs/architecture)
- [Consul Security Model](https://www.consul.io/docs/security)
- [Raft Consensus in Consul](https://www.consul.io/docs/architecture/consensus)

## License

Same as the main project

## Author Information

Part of the PostgreSQL HA Cluster automation project
