# Ansible Role: pgbackrest

## Description

This role installs and configures [pgBackRest](https://pgbackrest.org/), a backup and restore solution designed specifically for PostgreSQL. pgBackRest provides enterprise-grade backup features including:

- **Full, differential, and incremental backups**
- **Point-in-Time Recovery (PITR)**
- **Parallel backup and restore** for improved performance
- **Backup archiving** to multiple destinations (local, S3, Azure, GCS)
- **Backup verification and testing**
- **Encryption and compression**
- **Retention management** with automatic expiration

pgBackRest is the recommended backup solution for production PostgreSQL clusters.

## Requirements

### System Requirements
- **Operating System**: Debian/Ubuntu, RedHat/CentOS, Rocky Linux, AlmaLinux
- **PostgreSQL**: 9.5 or higher (12+ recommended)
- **Perl**: 5.10.1 or higher
- **SSH**: For remote backup repository access

### Storage Requirements
- Sufficient space for backup repository (3x database size minimum recommended)
- Fast storage for backup repository (SSD recommended)

### Network Requirements
- SSH access between PostgreSQL nodes and backup repository (if using dedicated backup server)
- HTTP/HTTPS access to cloud storage (if using S3/Azure/GCS)

### Dependencies
- `postgresql` - PostgreSQL must be installed and configured
- `common` - For shared variables

## Role Variables

### Installation Configuration

```yaml
# Installation method: "repo" (from pgdg repository)
installation_method: "repo"

# Install from PostgreSQL Global Development Group repo
pgbackrest_install_from_pgdg_repo: true
```

### Basic Configuration

```yaml
# pgBackRest configuration file path
pgbackrest_conf_file: "/etc/pgbackrest/pgbackrest.conf"

# Log path
pgbackrest_log_path: "/var/log/pgbackrest"

# Repository path (local backups)
pgbackrest_repo_path: "/var/lib/pgbackrest"

# Repository type: "posix", "s3", "azure", "gcs"
pgbackrest_repo_type: "posix"

# Stanza name (backup set identifier)
pgbackrest_stanza: "{{ patroni_cluster_name }}"
```

### Cloud Storage Configuration

#### Amazon S3

```yaml
pgbackrest_repo_type: "s3"
pgbackrest_repo_s3_bucket: "my-backup-bucket"
pgbackrest_repo_s3_region: "us-east-1"
pgbackrest_repo_s3_endpoint: "s3.amazonaws.com"
pgbackrest_repo_s3_key: "{{ vault_aws_access_key }}"
pgbackrest_repo_s3_key_secret: "{{ vault_aws_secret_key }}"
```

#### Azure Blob Storage

```yaml
pgbackrest_repo_type: "azure"
pgbackrest_repo_azure_container: "my-backup-container"
pgbackrest_repo_azure_account: "{{ vault_azure_account }}"
pgbackrest_repo_azure_key: "{{ vault_azure_key }}"
```

#### Google Cloud Storage

```yaml
pgbackrest_repo_type: "gcs"
pgbackrest_repo_gcs_bucket: "my-backup-bucket"
pgbackrest_repo_gcs_key: "/path/to/service-account.json"
```

### Performance Configuration

```yaml
# Number of parallel processes for backup/restore
pgbackrest_process_max: 4

# Enable delta restore (only restore changed files)
pgbackrest_delta: true

# Compress backups (none, gz, bz2, lz4, zst)
pgbackrest_compress_type: "lz4"
pgbackrest_compress_level: 3
```

### Retention Configuration

```yaml
# Full backup retention
pgbackrest_retention_full: 4  # Keep 4 full backups

# Differential backup retention
pgbackrest_retention_diff: 4  # Keep 4 differential per full

# Archive retention (days)
pgbackrest_retention_archive: 7  # Keep 7 days of archives

# Retention type: full, diff, incr
pgbackrest_retention_archive_type: "full"
```

### Encryption Configuration

```yaml
# Enable encryption
pgbackrest_repo_cipher_type: "aes-256-cbc"
pgbackrest_repo_cipher_pass: "{{ vault_pgbackrest_encryption_pass }}"
```

### Dedicated Backup Server

```yaml
# Dedicated backup repository host
pgbackrest_repo_host: "backup.example.com"
pgbackrest_repo_user: "pgbackrest"

# SSH configuration for repo access
pgbackrest_repo_host_user: "pgbackrest"
```

### Scheduled Backups

```yaml
# Cron jobs for automated backups
pgbackrest_cron_jobs:
  - name: "pgbackrest full backup"
    file: "/etc/cron.d/pgbackrest-full"
    user: "postgres"
    minute: "0"
    hour: "2"
    day: "*"
    month: "*"
    weekday: "0"  # Sunday
    job: "pgbackrest --stanza={{ pgbackrest_stanza }} --type=full backup"

  - name: "pgbackrest diff backup"
    file: "/etc/cron.d/pgbackrest-diff"
    user: "postgres"
    minute: "0"
    hour: "2"
    day: "*"
    month: "*"
    weekday: "1-6"  # Monday-Saturday
    job: "pgbackrest --stanza={{ pgbackrest_stanza }} --type=diff backup"
```

## Dependencies

This role depends on:
- `postgresql` - PostgreSQL must be installed first

## Example Playbook

### Basic Local Backup

```yaml
- name: Configure pgBackRest
  hosts: postgres_cluster
  become: true
  roles:
    - role: pgbackrest
      vars:
        pgbackrest_stanza: "prod-cluster"
        pgbackrest_repo_path: "/backup/pgbackrest"
        pgbackrest_process_max: 4
```

### Cloud Backup to S3

```yaml
- name: Configure pgBackRest with S3
  hosts: postgres_cluster
  become: true
  roles:
    - role: pgbackrest
      vars:
        pgbackrest_stanza: "prod-cluster"
        pgbackrest_repo_type: "s3"
        pgbackrest_repo_s3_bucket: "prod-pg-backups"
        pgbackrest_repo_s3_region: "us-west-2"
        pgbackrest_repo_s3_key: "{{ vault_aws_access_key }}"
        pgbackrest_repo_s3_key_secret: "{{ vault_aws_secret_key }}"

        # Encryption
        pgbackrest_repo_cipher_type: "aes-256-cbc"
        pgbackrest_repo_cipher_pass: "{{ vault_encryption_pass }}"

        # Performance
        pgbackrest_process_max: 8
        pgbackrest_compress_type: "lz4"

        # Retention
        pgbackrest_retention_full: 7
        pgbackrest_retention_archive: 14
```

### Dedicated Backup Server

```yaml
- name: Configure dedicated backup repository
  hosts: pgbackrest
  become: true
  roles:
    - role: pgbackrest
      vars:
        pgbackrest_repo_path: "/var/lib/pgbackrest"
        pgbackrest_repo_user: "pgbackrest"

- name: Configure PostgreSQL nodes
  hosts: postgres_cluster
  become: true
  roles:
    - role: pgbackrest
      vars:
        pgbackrest_stanza: "prod-cluster"
        pgbackrest_repo_host: "backup.example.com"
        pgbackrest_repo_host_user: "pgbackrest"
```

## Handlers

This role defines the following handlers:

- **restart pgbackrest**: Restarts pgBackRest-related services (if any)

## Files and Directories Created

### Configuration Files
- `/etc/pgbackrest/pgbackrest.conf`: Main configuration file
- `/etc/pgbackrest/conf.d/`: Additional configuration directory

### Directories
- `/var/lib/pgbackrest/`: Default backup repository
- `/var/log/pgbackrest/`: Log directory
- `/var/spool/pgbackrest/`: Spool directory for parallel operations

### SSH Keys (if repo_host is set)
- `/var/lib/postgresql/.ssh/id_rsa`: SSH key for backup repository access

## Post-Installation

### Initialize Stanza

```bash
# Create the stanza
sudo -u postgres pgbackrest --stanza=prod-cluster stanza-create

# Verify stanza
sudo -u postgres pgbackrest --stanza=prod-cluster check
```

### Perform First Backup

```bash
# Full backup
sudo -u postgres pgbackrest --stanza=prod-cluster --type=full backup

# Check backup info
sudo -u postgres pgbackrest --stanza=prod-cluster info
```

### Common Operations

```bash
# List backups
pgbackrest --stanza=prod-cluster info

# Differential backup
pgbackrest --stanza=prod-cluster --type=diff backup

# Incremental backup
pgbackrest --stanza=prod-cluster --type=incr backup

# Verify latest backup
pgbackrest --stanza=prod-cluster --type=full verify

# Expire old backups
pgbackrest --stanza=prod-cluster expire
```

### Point-in-Time Recovery

```bash
# Restore to latest
pgbackrest --stanza=prod-cluster --delta restore

# Restore to specific time
pgbackrest --stanza=prod-cluster \
  --type=time \
  --target="2024-01-15 14:30:00" \
  --delta restore

# Restore to specific transaction ID
pgbackrest --stanza=prod-cluster \
  --type=xid \
  --target="12345678" \
  --delta restore
```

## Troubleshooting

### Backup Fails

**Check logs:**
```bash
tail -f /var/log/pgbackrest/prod-cluster-backup.log
```

**Common issues:**
- Insufficient disk space in repository
- PostgreSQL not in archive mode
- SSH key permissions incorrect
- Cloud credentials invalid

### Archive Command Fails

**Check PostgreSQL logs:**
```bash
tail -f /var/log/postgresql/postgresql-*.log | grep archive
```

**Verify archive_command:**
```sql
SHOW archive_command;
```

### Restore Fails

**Common issues:**
- Target directory not empty (use --delta or clean directory)
- Insufficient permissions
- Backup repository unreachable

### Performance Issues

**Increase parallelism:**
```yaml
pgbackrest_process_max: 8  # Increase based on CPU cores
```

**Use faster compression:**
```yaml
pgbackrest_compress_type: "lz4"  # Faster than gz or bz2
```

**Check repository performance:**
```bash
# Test I/O speed
dd if=/dev/zero of=/var/lib/pgbackrest/test bs=1M count=1024
```

## Security Considerations

### 1. Enable Encryption

```yaml
pgbackrest_repo_cipher_type: "aes-256-cbc"
pgbackrest_repo_cipher_pass: "{{ vault_pgbackrest_encryption_pass }}"
```

### 2. Secure Cloud Credentials

Use Ansible Vault for all cloud credentials:
```bash
ansible-vault create group_vars/all/vault.yml
```

### 3. SSH Key Security

```bash
# Ensure proper permissions
chmod 600 /var/lib/postgresql/.ssh/id_rsa
chmod 644 /var/lib/postgresql/.ssh/id_rsa.pub
chown postgres:postgres /var/lib/postgresql/.ssh/id_rsa*
```

### 4. Repository Permissions

```bash
# Restrict access to backup repository
chmod 750 /var/lib/pgbackrest
chown postgres:postgres /var/lib/pgbackrest
```

### 5. Test Backups Regularly

```bash
# Automated backup verification
pgbackrest --stanza=prod-cluster --type=full verify
```

## Performance Tuning

### For Large Databases (>1TB)

```yaml
pgbackrest_process_max: 8
pgbackrest_compress_type: "lz4"
pgbackrest_compress_level: 1
pgbackrest_delta: true
```

### For Fast Networks

```yaml
# Increase parallelism
pgbackrest_process_max: 16
pgbackrest_protocol_timeout: 1800
```

### For Slow Storage

```yaml
# Reduce parallelism to avoid I/O contention
pgbackrest_process_max: 2
pgbackrest_compress_type: "none"  # Reduce CPU usage
```

## Monitoring

### Key Metrics to Monitor

- Backup success/failure rate
- Backup duration
- Backup size growth
- Repository disk usage
- Archive command success rate
- Last successful backup timestamp

### Check Backup Status

```bash
# Get backup info with details
pgbackrest --stanza=prod-cluster info --output=json

# Check for stale backups
pgbackrest --stanza=prod-cluster info | grep -A5 "full backup"
```

## Best Practices

1. **Use incremental/differential backups** for daily operations
2. **Schedule full backups** weekly
3. **Enable encryption** for production
4. **Test restores** regularly (monthly minimum)
5. **Monitor backup completion**
6. **Use dedicated backup server** for large clusters
7. **Implement off-site backups** (cloud or remote datacenter)
8. **Document recovery procedures**
9. **Set appropriate retention** based on RTO/RPO requirements
10. **Verify backups** automatically with pgbackrest verify

## Retention Strategy Examples

### Conservative (Long Retention)

```yaml
pgbackrest_retention_full: 12  # 12 weeks of full backups
pgbackrest_retention_archive: 30  # 30 days of archives
```

### Standard Production

```yaml
pgbackrest_retention_full: 4  # 4 weeks of full backups
pgbackrest_retention_archive: 7  # 7 days of archives
```

### Development/Testing

```yaml
pgbackrest_retention_full: 2  # 2 weeks of full backups
pgbackrest_retention_archive: 3  # 3 days of archives
```

## Known Limitations

- Requires PostgreSQL to be in archive mode
- Full backup required before differential/incremental
- Cloud storage requires network connectivity
- Parallel processes limited by I/O capacity
- Encryption slightly impacts performance

## Additional Resources

- [Official pgBackRest Documentation](https://pgbackrest.org/documentation.html)
- [pgBackRest User Guide](https://pgbackrest.org/user-guide.html)
- [pgBackRest GitHub Repository](https://github.com/pgbackrest/pgbackrest)
- [pgBackRest Configuration Reference](https://pgbackrest.org/configuration.html)

## License

Same as the main project

## Author Information

Part of the PostgreSQL HA Cluster automation project
