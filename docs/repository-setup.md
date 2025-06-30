# Borg Backup Repository Setup Guide

This guide explains how to set up and configure Borg backup repositories for different storage types including local, remote, and cloud storage.

## Repository Types

### 1. Local Repository Setup

#### External Drive Repository
```bash
# Mount external drive
sudo mkdir -p /mnt/backup
sudo mount /dev/sdb1 /mnt/backup

# Create repository directory
sudo mkdir -p /mnt/backup/borg-repo
sudo chown $USER:$USER /mnt/backup/borg-repo

# Initialize repository
borg init --encryption=repokey-blake2 /mnt/backup/borg-repo

# Add to /etc/fstab for automatic mounting
echo "/dev/sdb1 /mnt/backup ext4 defaults,noatime 0 2" | sudo tee -a /etc/fstab
```

#### NAS Repository
```bash
# Mount NAS share
sudo mkdir -p /mnt/nas-backup
sudo mount -t cifs //nas.local/backup /mnt/nas-backup -o username=backup,password=password

# Initialize repository
borg init --encryption=repokey-blake2 /mnt/nas-backup/borg-repo

# Add to /etc/fstab
echo "//nas.local/backup /mnt/nas-backup cifs username=backup,password=password,uid=1000,gid=1000 0 0" | sudo tee -a /etc/fstab
```

### 2. Remote Repository Setup

#### SSH-based Remote Repository
```bash
# Generate SSH key for backup
ssh-keygen -t rsa -b 4096 -f ~/.ssh/borg_backup -N ""

# Copy public key to remote server
ssh-copy-id -i ~/.ssh/borg_backup.pub user@backup-server.com

# Test SSH connection
ssh -i ~/.ssh/borg_backup user@backup-server.com

# Create repository directory on remote server
ssh -i ~/.ssh/borg_backup user@backup-server.com "mkdir -p ~/borg-repo"

# Initialize remote repository
borg init --encryption=repokey-blake2 ssh://user@backup-server.com:22/~/borg-repo
```

#### SSH Configuration
```bash
# Create SSH config for backup connections
cat >> ~/.ssh/config << EOF
Host backup-server
    HostName backup-server.com
    User backup-user
    IdentityFile ~/.ssh/borg_backup
    Port 22
    Compression yes
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF
```

### 3. Cloud Repository Setup

#### rsync.net
```bash
# Initialize cloud repository
borg init --encryption=repokey-blake2 ssh://username@usw-s001.rsync.net/borg-repo
```

#### BorgBase
```bash
# Initialize BorgBase repository
borg init --encryption=repokey-blake2 ssh://username@username.repo.borgbase.com/./repo
```

#### Custom Cloud Provider
```bash
# Using rclone for cloud storage
rclone config  # Configure your cloud provider
rclone mount remote:backup /mnt/cloud-backup --daemon

# Initialize repository
borg init --encryption=repokey-blake2 /mnt/cloud-backup/borg-repo
```

## Repository Configuration

### Encryption Options

#### Repokey (Recommended)
```bash
# Repository key stored in repository
borg init --encryption=repokey-blake2 /path/to/repo
```

#### Keyfile
```bash
# Repository key stored in separate file
borg init --encryption=keyfile-blake2 /path/to/repo
```

#### Authenticated
```bash
# No encryption, only authentication
borg init --encryption=authenticated-blake2 /path/to/repo
```

### Repository Maintenance

#### Check Repository Integrity
```bash
# Quick check
borg check /path/to/repo

# Full check (slower)
borg check --verify-data /path/to/repo

# Repair repository (use with caution)
borg check --repair /path/to/repo
```

#### Compact Repository
```bash
# Remove unused space
borg compact /path/to/repo
```

#### Repository Information
```bash
# Show repository info
borg info /path/to/repo

# Show archive list
borg list /path/to/repo

# Show archive contents
borg list /path/to/repo::archive-name
```

## Multi-Repository Setup

### Configuration for Multiple Repositories
```yaml
# borgmatic.yaml
repositories:
    - path: /mnt/backup/borg-repo
      label: local-backup
    - path: ssh://user@backup-server.com/~/borg-repo
      label: remote-backup
    - path: ssh://username@usw-s001.rsync.net/borg-repo
      label: cloud-backup

# Different retention policies per repository
retention:
    keep_daily: 7
    keep_weekly: 4
    keep_monthly: 12
    keep_yearly: 5

# Repository-specific settings
storage:
    archive_name_format: '{hostname}-{now:%Y-%m-%d-%H%M%S}'
    relocated_repo_access_is_ok: true
    unknown_unencrypted_repo_access_is_ok: false
```

### Selective Repository Backup
```bash
# Backup to specific repository only
borgmatic create --repository /mnt/backup/borg-repo

# Backup to multiple repositories
borgmatic create --repository /mnt/backup/borg-repo --repository ssh://user@backup-server.com/~/borg-repo
```

## Security Best Practices

### Key Management
```bash
# Export repository key
borg key export /path/to/repo backup-key.txt

# Import repository key
borg key import /path/to/repo backup-key.txt

# Change repository passphrase
borg key change-passphrase /path/to/repo
```

### Access Control
```bash
# Restrict SSH access for backup user
# Add to ~/.ssh/authorized_keys on backup server:
command="borg serve --restrict-to-path ~/borg-repo",restrict ssh-rsa AAAAB3...
```

### Network Security
```bash
# Use SSH tunneling for additional security
ssh -L 2222:backup-server:22 jump-host
borg init --encryption=repokey-blake2 ssh://user@localhost:2222/~/borg-repo
```

## Monitoring and Alerting

### Repository Health Checks
```bash
#!/bin/bash
# check-repo-health.sh

REPOS=(
    "/mnt/backup/borg-repo"
    "ssh://user@backup-server.com/~/borg-repo"
)

for repo in "${REPOS[@]}"; do
    echo "Checking repository: $repo"
    
    if borg check "$repo" 2>/dev/null; then
        echo "✓ Repository $repo is healthy"
    else
        echo "✗ Repository $repo has issues"
        # Send alert
        echo "Repository check failed: $repo" | mail -s "Backup Alert" admin@example.com
    fi
done
```

### Storage Space Monitoring
```bash
#!/bin/bash
# monitor-storage.sh

THRESHOLD=90  # Alert when 90% full

for repo in "${REPOS[@]}"; do
    if [[ "$repo" =~ ^/ ]]; then  # Local repository
        usage=$(df "$repo" | awk 'NR==2 {print $5}' | sed 's/%//')
        if [[ $usage -gt $THRESHOLD ]]; then
            echo "Storage warning: $repo is ${usage}% full" | mail -s "Storage Alert" admin@example.com
        fi
    fi
done
```

## Troubleshooting

### Common Issues

#### Repository Lock
```bash
# Break lock if backup was interrupted
borg break-lock /path/to/repo
```

#### Corrupted Repository
```bash
# Check and repair
borg check --repair /path/to/repo

# If repair fails, restore from another repository
```

#### SSH Connection Issues
```bash
# Test SSH connection
ssh -i ~/.ssh/borg_backup user@backup-server.com

# Debug SSH issues
ssh -v -i ~/.ssh/borg_backup user@backup-server.com
```

#### Permission Issues
```bash
# Fix repository permissions
sudo chown -R $USER:$USER /path/to/repo
chmod -R 750 /path/to/repo
```

### Recovery Procedures

#### Emergency Repository Access
```bash
# If passphrase is lost but key file exists
borg key export /path/to/repo --paper

# Restore from paper key
borg key import /path/to/repo
```

#### Repository Migration
```bash
# Create new repository
borg init --encryption=repokey-blake2 /new/path/to/repo

# Transfer archives
borg list /old/repo --short | while read archive; do
    borg transfer /old/repo::$archive /new/repo
done
```

## Performance Optimization

### Network Optimization
```bash
# Use compression for remote repositories
export BORG_RSH="ssh -C"

# Limit bandwidth
export BORG_RSH="ssh -o 'ProxyCommand=pv -q -L 1m | nc %h %p'"
```

### Storage Optimization
```bash
# Use different compression algorithms
borg create --compression lz4 /path/to/repo::archive /path/to/backup
borg create --compression zlib,6 /path/to/repo::archive /path/to/backup
borg create --compression zstd,3 /path/to/repo::archive /path/to/backup
```

### Parallel Operations
```bash
# Run multiple borg operations in parallel (different repositories)
borgmatic create --repository /mnt/backup/borg-repo &
borgmatic create --repository ssh://user@backup-server.com/~/borg-repo &
wait
```

## Automation

### Systemd Timer Setup
```bash
# Create timer for repository maintenance
sudo tee /etc/systemd/system/borg-maintenance.timer << EOF
[Unit]
Description=Borg Repository Maintenance
Requires=borg-maintenance.service

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo tee /etc/systemd/system/borg-maintenance.service << EOF
[Unit]
Description=Borg Repository Maintenance

[Service]
Type=oneshot
User=backup
ExecStart=/usr/local/bin/borg-maintenance.sh
EOF

sudo systemctl enable borg-maintenance.timer
sudo systemctl start borg-maintenance.timer
```

### Maintenance Script
```bash
#!/bin/bash
# borg-maintenance.sh

REPOS=(
    "/mnt/backup/borg-repo"
    "ssh://user@backup-server.com/~/borg-repo"
)

for repo in "${REPOS[@]}"; do
    echo "Maintaining repository: $repo"
    
    # Check repository
    borg check "$repo"
    
    # Compact repository
    borg compact "$repo"
    
    # Prune old archives
    borg prune "$repo" --keep-daily=7 --keep-weekly=4 --keep-monthly=12
done
```

This guide provides comprehensive instructions for setting up and maintaining Borg backup repositories across different storage types and environments.