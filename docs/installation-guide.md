# Borg Backup Installation Guide

This guide provides step-by-step instructions for installing and configuring Borg Backup with Borgmatic for automated, secure backups.

## Prerequisites

### System Requirements
- Linux-based operating system (Ubuntu 20.04+ recommended)
- Python 3.6 or higher
- At least 1GB free disk space for installation
- Network connectivity for remote repositories
- Root or sudo access for system-wide installation

### Hardware Requirements
- **Local backups**: External drive or NAS with sufficient storage
- **Remote backups**: SSH access to remote server
- **Cloud backups**: Account with cloud storage provider

## Installation Methods

### Method 1: Package Manager Installation (Recommended)

#### Ubuntu/Debian
```bash
# Update package list
sudo apt update

# Install Borg Backup
sudo apt install borgbackup

# Install Borgmatic
sudo apt install borgmatic

# Verify installation
borg --version
borgmatic --version
```

#### CentOS/RHEL/Fedora
```bash
# Enable EPEL repository (CentOS/RHEL)
sudo yum install epel-release

# Install Borg Backup
sudo yum install borgbackup

# Install pip for Borgmatic
sudo yum install python3-pip

# Install Borgmatic via pip
sudo pip3 install borgmatic
```

#### Arch Linux
```bash
# Install from official repositories
sudo pacman -S borg borgmatic
```

### Method 2: Binary Installation

#### Download Latest Borg Binary
```bash
# Create installation directory
sudo mkdir -p /usr/local/bin

# Download latest Borg binary
wget https://github.com/borgbackup/borg/releases/download/1.2.4/borg-linux64
sudo mv borg-linux64 /usr/local/bin/borg
sudo chmod +x /usr/local/bin/borg

# Verify installation
borg --version
```

#### Install Borgmatic via pip
```bash
# Install pip if not available
sudo apt install python3-pip

# Install Borgmatic
sudo pip3 install borgmatic

# Verify installation
borgmatic --version
```

### Method 3: Docker Installation

```bash
# Pull Borg Docker image
docker pull borgbackup/borg-backup

# Create alias for easier usage
echo 'alias borg="docker run --rm -v /home:/home -v /etc:/etc:ro borgbackup/borg-backup"' >> ~/.bashrc
source ~/.bashrc
```

## Initial Configuration

### 1. Create Configuration Directory
```bash
# Create Borgmatic config directory
mkdir -p ~/.config/borgmatic

# Create log directory
sudo mkdir -p /var/log/borgmatic
sudo chown $USER:$USER /var/log/borgmatic
```

### 2. Generate Initial Configuration
```bash
# Generate sample configuration
borgmatic config generate

# This creates ~/.config/borgmatic/config.yaml
```

### 3. Copy Example Configuration
```bash
# Copy the example configuration from this repository
cp config-examples/borgmatic.yaml.example ~/.config/borgmatic/config.yaml

# Edit the configuration
nano ~/.config/borgmatic/config.yaml
```

## Repository Setup

### Local Repository

#### 1. Prepare Storage Location
```bash
# Mount external drive (example)
sudo mkdir -p /mnt/backup
sudo mount /dev/sdb1 /mnt/backup

# Create repository directory
sudo mkdir -p /mnt/backup/borg-repo
sudo chown $USER:$USER /mnt/backup/borg-repo
```

#### 2. Initialize Repository
```bash
# Initialize with encryption
borg init --encryption=repokey-blake2 /mnt/backup/borg-repo

# You'll be prompted to create a passphrase - SAVE THIS SECURELY!
```

### Remote Repository

#### 1. Set Up SSH Key Authentication
```bash
# Generate SSH key pair (if not exists)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/borg_backup

# Copy public key to remote server
ssh-copy-id -i ~/.ssh/borg_backup.pub user@backup-server.example.com
```

#### 2. Test SSH Connection
```bash
# Test connection
ssh -i ~/.ssh/borg_backup user@backup-server.example.com

# Create repository directory on remote server
mkdir -p ~/borg-repo
```

#### 3. Initialize Remote Repository
```bash
# Initialize remote repository
borg init --encryption=repokey-blake2 ssh://user@backup-server.example.com:22/~/borg-repo
```

### Cloud Repository (Optional)

#### Using rsync.net
```bash
# Initialize cloud repository
borg init --encryption=repokey-blake2 ssh://username@usw-s001.rsync.net/borg-repo
```

## Configuration Customization

### 1. Edit Borgmatic Configuration
```bash
nano ~/.config/borgmatic/config.yaml
```

### 2. Key Configuration Sections

#### Repositories
```yaml
repositories:
    - path: /mnt/backup/borg-repo
      label: local-backup
    - path: ssh://user@backup-server.example.com:22/~/borg-repo
      label: remote-backup
```

#### Source Directories
```yaml
source_directories:
    - /home
    - /etc
    - /var/log
    - /opt
```

#### Exclude Patterns
```yaml
exclude_patterns:
    - '*.tmp'
    - '*.cache'
    - '*/.cache'
    - '/tmp'
    - '/var/tmp'
```

#### Retention Policy
```yaml
retention:
    keep_daily: 7
    keep_weekly: 4
    keep_monthly: 12
    keep_yearly: 5
```

### 3. Validate Configuration
```bash
# Validate configuration syntax
borgmatic config validate

# Test configuration with dry run
borgmatic create --dry-run --verbosity 1
```

## Automation Setup

### 1. Install Systemd Timer
```bash
# Copy systemd files from this repository
sudo cp systemd/borg-backup.service /etc/systemd/system/
sudo cp systemd/borg-backup.timer /etc/systemd/system/

# Reload systemd
sudo systemctl daemon-reload

# Enable and start timer
sudo systemctl enable borg-backup.timer
sudo systemctl start borg-backup.timer
```

### 2. Verify Timer Status
```bash
# Check timer status
sudo systemctl status borg-backup.timer

# List all timers
sudo systemctl list-timers
```

### 3. Alternative: Cron Setup
```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /usr/local/bin/borgmatic create --verbosity 1 2>&1 | logger -t borgmatic
```

## Security Configuration

### 1. Secure Passphrase Storage
```bash
# Create secure passphrase file
sudo mkdir -p /etc/borgmatic
echo "your_secure_passphrase" | sudo tee /etc/borgmatic/passphrase
sudo chmod 600 /etc/borgmatic/passphrase
sudo chown root:root /etc/borgmatic/passphrase
```

### 2. Update Borgmatic Configuration
```yaml
# Add to config.yaml
encryption_passcommand: cat /etc/borgmatic/passphrase
```

### 3. SSH Configuration
```bash
# Create SSH config for backup connections
cat >> ~/.ssh/config << EOF
Host backup-server
    HostName backup-server.example.com
    User backup-user
    IdentityFile ~/.ssh/borg_backup
    Port 22
    Compression yes
    ServerAliveInterval 60
EOF
```

## Testing and Verification

### 1. First Backup Test
```bash
# Perform first backup
borgmatic create --verbosity 1 --stats

# List archives
borgmatic list

# Check repository info
borgmatic info
```

### 2. Restore Test
```bash
# Mount archive for browsing
mkdir /tmp/borg-mount
borgmatic mount --archive latest --mount-point /tmp/borg-mount

# Browse files
ls -la /tmp/borg-mount

# Unmount when done
borgmatic umount --mount-point /tmp/borg-mount
```

### 3. Consistency Check
```bash
# Run repository check
borgmatic check --verbosity 1
```

## Monitoring Setup

### 1. Enable Health Checks
```bash
# Sign up for a free account at healthchecks.io
# Add the ping URL to your borgmatic config:
```

```yaml
healthchecks:
    ping_url: https://hc-ping.com/your-uuid-here
```

### 2. Email Notifications
```bash
# Install mail command
sudo apt install mailutils

# Configure in borgmatic config:
```

```yaml
on_error:
    - echo "Backup failed at $(date)" | mail -s "Backup Failure" admin@example.com
```

## Troubleshooting

### Common Issues

#### 1. Permission Denied Errors
```bash
# Fix ownership of backup directories
sudo chown -R $USER:$USER ~/.config/borgmatic
sudo chown -R $USER:$USER /var/log/borgmatic
```

#### 2. SSH Connection Issues
```bash
# Test SSH connection manually
ssh -i ~/.ssh/borg_backup user@backup-server.example.com

# Check SSH agent
ssh-add ~/.ssh/borg_backup
```

#### 3. Repository Lock Issues
```bash
# Break lock if backup was interrupted
borg break-lock /path/to/repo

# Or for remote repository
borg break-lock ssh://user@server/repo
```

#### 4. Disk Space Issues
```bash
# Check available space
df -h

# Prune old archives manually
borgmatic prune --verbosity 1
```

### Log Analysis
```bash
# View recent logs
tail -f /var/log/borgmatic/backup.log

# Check systemd logs
sudo journalctl -u borg-backup.service -f
```

## Maintenance

### Regular Tasks

#### 1. Update Software
```bash
# Update Borg and Borgmatic
sudo apt update && sudo apt upgrade borgbackup borgmatic
```

#### 2. Monitor Repository Health
```bash
# Weekly repository check
borgmatic check --verbosity 1

# Monthly deep check
borgmatic check --verbosity 1 --repository-only
```

#### 3. Test Restores
```bash
# Monthly restore test
borgmatic extract --archive latest --destination /tmp/restore-test
```

#### 4. Review Retention Policy
```bash
# List all archives with sizes
borgmatic list --short --stats
```

## Advanced Configuration

### Multiple Backup Sets
```bash
# Create separate configs for different backup sets
mkdir -p ~/.config/borgmatic/configs
cp ~/.config/borgmatic/config.yaml ~/.config/borgmatic/configs/system.yaml
cp ~/.config/borgmatic/config.yaml ~/.config/borgmatic/configs/home.yaml

# Run specific config
borgmatic --config ~/.config/borgmatic/configs/system.yaml create
```

### Database Backups
```yaml
# Add to borgmatic config for database dumps
before_backup:
    - pg_dump mydb > /tmp/mydb.sql
    - mysqldump --all-databases > /tmp/mysql-backup.sql

after_backup:
    - rm -f /tmp/mydb.sql /tmp/mysql-backup.sql
```

### Bandwidth Limiting
```bash
# Limit bandwidth for remote backups
export BORG_RSH="ssh -o 'ProxyCommand=pv -q -L 1m | nc %h %p'"
borgmatic create
```

## Security Best Practices

1. **Use strong passphrases** for repository encryption
2. **Store passphrases securely** in encrypted files
3. **Use SSH key authentication** for remote repositories
4. **Regularly test restores** to ensure backup integrity
5. **Monitor backup health** with automated checks
6. **Keep multiple copies** following the 3-2-1 rule
7. **Encrypt sensitive data** before backing up
8. **Limit network access** to backup repositories
9. **Regularly update** Borg and Borgmatic
10. **Document recovery procedures** for emergency situations

## Next Steps

After successful installation:

1. **Configure monitoring** and alerting
2. **Set up automated testing** of backup integrity
3. **Document recovery procedures** for your team
4. **Plan disaster recovery scenarios**
5. **Regular review and optimization** of backup strategy

For more advanced configurations and troubleshooting, refer to:
- [Borg Documentation](https://borgbackup.readthedocs.io/)
- [Borgmatic Documentation](https://torsion.org/borgmatic/)
- [Repository troubleshooting guide](troubleshooting.md)