# Ansible Role: patroni

## Description

This role installs and configures [Patroni](https://github.com/zalando/patroni), a template for PostgreSQL High Availability (HA) with automatic failover using distributed consensus stores (etcd or Consul).

Patroni manages PostgreSQL clusters by:
- Handling automatic failover and leader election
- Performing health checks and monitoring
- Managing replication and synchronous commits
- Integrating with DCS (Distributed Configuration Store) for cluster state
- Providing REST API for cluster management

## Requirements

### System Requirements
- **Operating System**: Debian/Ubuntu, RedHat/CentOS, Rocky Linux, AlmaLinux
- **Python**: 3.6 or higher
- **PostgreSQL**: 12 or higher (must be installed before running this role)

### Dependencies
- **DCS (Distributed Configuration Store)**: One of the following must be configured:
  - etcd (recommended) - configured via `etcd` role
  - Consul - configured via `consul` role
- **PostgreSQL**: Installed and configured (via `postgres` role)

### Required Ansible Collections
- `ansible.builtin`
- `community.general`

## Role Variables

### Installation Method

```yaml
# Installation method: "repo", "file"
installation_method: "repo"

# Patroni installation method: "pip", "rpm", "deb"
patroni_installation_method: "pip"  # default

# Patroni version to install (use "latest" for most recent)
patroni_install_version: "latest"
```

### Cluster Configuration

```yaml
# Patroni cluster name (must be unique)
patroni_cluster_name: "postgres-cluster"

# Patroni scope (namespace for the cluster)
patroni_scope: "{{ patroni_cluster_name }}"

# DCS type: "etcd" or "consul"
dcs_type: "etcd"

# Patroni log level: DEBUG, INFO, WARNING, ERROR
patroni_log_level: "INFO"
```

### Authentication

```yaml
# PostgreSQL superuser credentials
patroni_superuser_username: "postgres"
patroni_superuser_password: ""  # Auto-generated if empty

# PostgreSQL replication user credentials
patroni_replication_username: "replicator"
patroni_replication_password: ""  # Auto-generated if empty
```

### Network Configuration

```yaml
# Patroni REST API settings
patroni_restapi_listen: "0.0.0.0:8008"
patroni_restapi_connect_address: "{{ inventory_hostname }}:8008"

# PostgreSQL connection settings
patroni_postgresql_listen: "0.0.0.0:{{ postgresql_port }}"
patroni_postgresql_connect_address: "{{ inventory_hostname }}:{{ postgresql_port }}"
```

### Advanced Configuration

```yaml
# Synchronous mode settings
patroni_synchronous_mode: false
patroni_synchronous_mode_strict: false

# Use slots for replication
patroni_use_slots: true

# TTL for leader key in DCS (seconds)
patroni_ttl: 30

# Loop wait time (seconds)
patroni_loop_wait: 10

# Retry timeout (seconds)
patroni_retry_timeout: 30

# Maximum lag for synchronous replica (bytes)
patroni_maximum_lag_on_syncnode: 1048576

# Enable pg_rewind for diverged replicas
patroni_use_pg_rewind: true
```

### Installation from Files

```yaml
# When installation_method: "file"
patroni_pip_requirements_file: []  # List of requirement files
patroni_pip_package_file: []        # List of patroni package files
patroni_deb_package_file: ""        # Debian package file
patroni_rpm_package_file: ""        # RPM package file
```

### Installation from Repositories

```yaml
# When installation_method: "repo"
patroni_pip_requirements_repo: []  # List of requirement URLs
patroni_pip_package_repo: []       # List of patroni package URLs
patroni_deb_package_repo: []       # List of Debian package URLs
patroni_rpm_package_repo: []       # List of RPM package URLs
```

## Dependencies

This role depends on:
- `common` - For shared variables and configurations
- `etcd` or `consul` - For distributed consensus
- `postgresql` - For database installation

## Example Playbook

### Basic Usage

```yaml
- name: Deploy PostgreSQL HA with Patroni
  hosts: postgres_cluster
  become: true
  roles:
    - role: patroni
      vars:
        patroni_cluster_name: "production-pg"
        patroni_log_level: "INFO"
        dcs_type: "etcd"
```

### Advanced Usage with Custom Configuration

```yaml
- name: Deploy PostgreSQL HA with Patroni
  hosts: postgres_cluster
  become: true
  roles:
    - role: patroni
      vars:
        patroni_cluster_name: "production-pg"
        patroni_installation_method: "pip"
        patroni_install_version: "3.2.0"
        patroni_log_level: "DEBUG"

        # Enable synchronous replication
        patroni_synchronous_mode: true
        patroni_synchronous_mode_strict: false

        # DCS configuration
        dcs_type: "etcd"

        # Authentication
        patroni_superuser_password: "{{ vault_postgres_superuser_password }}"
        patroni_replication_password: "{{ vault_postgres_replication_password }}"

        # Performance tuning
        patroni_loop_wait: 5
        patroni_ttl: 30
        patroni_retry_timeout: 30
```

## Handlers

This role defines the following handlers:

- **restart patroni**: Restarts the Patroni service
- **reload patroni**: Reloads Patroni configuration without restart
- **patroni config file updated**: Triggered when patroni.yml is modified

## Files and Directories Created

### Configuration Files
- `/etc/patroni/patroni.yml`: Main Patroni configuration file
- `/etc/systemd/system/patroni.service`: Systemd service file (if using systemd)

### Log Files
- `/var/log/patroni/patroni.log`: Patroni application logs

### State Files
- `/var/lib/postgresql/{{ postgresql_version }}/main/`: PostgreSQL data directory
- DCS stores cluster state in etcd or Consul

## Installation Methods

### 1. From PyPI (pip) - Recommended
```yaml
installation_method: "repo"
patroni_installation_method: "pip"
patroni_install_version: "latest"
```

### 2. From OS Repository
```yaml
installation_method: "repo"
patroni_installation_method: "deb"  # or "rpm"
```

### 3. From Local Files
```yaml
installation_method: "file"
patroni_installation_method: "pip"
patroni_pip_package_file:
  - "/path/to/patroni-3.2.0.tar.gz"
```

### 4. From Remote URLs
```yaml
installation_method: "repo"
patroni_installation_method: "pip"
patroni_pip_package_repo:
  - "https://example.com/patroni-3.2.0.tar.gz"
```

## Post-Installation

After running this role:

1. **Verify Patroni is running**:
   ```bash
   systemctl status patroni
   ```

2. **Check cluster status**:
   ```bash
   patronictl -c /etc/patroni/patroni.yml list
   ```

3. **View Patroni logs**:
   ```bash
   journalctl -u patroni -f
   ```

4. **Access REST API**:
   ```bash
   curl http://localhost:8008/
   ```

## Troubleshooting

### Patroni fails to start
- Check DCS (etcd/Consul) is running and accessible
- Verify PostgreSQL is installed
- Check logs: `journalctl -u patroni -n 50`
- Verify configuration: `/etc/patroni/patroni.yml`

### Cannot connect to DCS
- Verify etcd/Consul is running
- Check network connectivity
- Verify DCS endpoints in patroni.yml

### Replication not working
- Check replication user credentials
- Verify pg_hba.conf allows replication connections
- Check network connectivity between nodes
- Review PostgreSQL logs

### Split-brain scenario
- Check DCS connectivity
- Verify network between all nodes
- Review patroni logs on all nodes
- Check `patroni_ttl` value (may need adjustment)

## Security Considerations

1. **Passwords**: Always use Ansible Vault for sensitive passwords
   ```yaml
   patroni_superuser_password: "{{ vault_postgres_superuser_password }}"
   ```

2. **REST API**: Consider enabling authentication on the REST API:
   ```yaml
   patroni_restapi_username: "admin"
   patroni_restapi_password: "{{ vault_patroni_api_password }}"
   ```

3. **Firewall**: Ensure ports are properly restricted:
   - 8008: Patroni REST API (between cluster nodes only)
   - 5432: PostgreSQL (from application servers)
   - 2379: etcd (between DCS nodes only) OR
   - 8500: Consul (between DCS nodes only)

4. **SSL/TLS**: Enable SSL for PostgreSQL connections in production

## Performance Tuning

### For low-latency requirements:
```yaml
patroni_loop_wait: 5
patroni_ttl: 20
patroni_retry_timeout: 20
```

### For high-availability over network partitions:
```yaml
patroni_synchronous_mode: true
patroni_synchronous_mode_strict: true
patroni_maximum_lag_on_syncnode: 1048576
```

### For large clusters (>10 nodes):
```yaml
patroni_loop_wait: 10
patroni_ttl: 30
patroni_maximum_lag_on_failover: 10485760
```

## Known Limitations

- Requires functioning DCS (etcd or Consul) before installation
- Password changes require manual intervention
- Switchover/failover operations should be tested in staging first
- Not compatible with PostgreSQL versions < 12

## Additional Resources

- [Official Patroni Documentation](https://patroni.readthedocs.io/)
- [Patroni GitHub Repository](https://github.com/zalando/patroni)
- [PostgreSQL High Availability Best Practices](https://www.postgresql.org/docs/current/high-availability.html)

## License

Same as the main project

## Author Information

Part of the PostgreSQL HA Cluster automation project
