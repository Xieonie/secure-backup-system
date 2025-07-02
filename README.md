# Secure Backup System with Borg Backup 🗄️🔒

This repository documents the implementation of a comprehensive, encrypted, and automated backup strategy using [Borg Backup](https://www.borgbackup.org/). The system follows the 3-2-1 backup rule and ensures data integrity, confidentiality, and rapid recovery capabilities for critical systems and personal data.

**Important Note:** Backup strategies are highly individual and depend on your specific data, infrastructure, and recovery requirements. The configurations provided here serve as templates and must be carefully adapted to your environment.

## 🎯 Goals

* Implement a robust 3-2-1 backup strategy (3 copies, 2 different media types, 1 offsite).
* Ensure data confidentiality through strong encryption.
* Automate backup processes to minimize human error.
* Provide fast and reliable data recovery capabilities.
* Monitor backup health and receive alerts on failures.
* Maintain data integrity through regular verification.

## 🛠️ Technologies Used

* [Borg Backup](https://www.borgbackup.org/) - Deduplicating backup program
* [Borgmatic](https://torsion.org/borgmatic/) - Configuration-driven backup automation
* SSH for secure remote backup repositories
* Systemd timers for scheduling (alternative to cron)
* Shell scripting for automation and monitoring
* Email notifications for alerts
* Optional: Cloud storage providers (AWS S3, Google Cloud, etc.) for offsite backups

## ✨ Key Features/Highlights

* **Deduplication:** Borg's advanced deduplication saves storage space and bandwidth.
* **Encryption:** AES-256 encryption with authenticated encryption modes.
* **Compression:** Multiple compression algorithms (LZ4, ZLIB, LZMA, ZSTD) for optimal storage efficiency.
* **Incremental Backups:** Only changed data is backed up after the initial full backup.
* **Cross-Platform:** Works on Linux, macOS, and Windows (via WSL).
* **Automated Scheduling:** Systemd timers ensure regular, unattended backups.
* **Health Monitoring:** Automated verification and alerting on backup failures.
* **Multiple Repositories:** Support for local, remote, and cloud backup destinations.
* **Retention Policies:** Automatic cleanup of old backups based on configurable policies.

## 🏛️ Repository Structure

```
secure-backup-system/
├── README.md
├── docs/
│   ├── installation-guide.md
│   ├── repository-setup.md
│   ├── recovery-procedures.md
│   └── troubleshooting.md
├── scripts/
│   ├── backup/
│   │   ├── borg-backup.sh
│   │   ├── pre-backup-hooks.sh
│   │   └── post-backup-hooks.sh
│   ├── monitoring/
│   │   ├── backup-health-check.sh
│   │   └── send-notification.sh
│   └── recovery/
│       ├── emergency-recovery.sh
│       └── selective-restore.sh
├── config-examples/
│   ├── borgmatic.yaml.example
│   ├── systemd-timer.example
│   └── ssh-config.example
└── systemd/
    ├── borg-backup.service
    └── borg-backup.timer
```

## 🚀 Getting Started / Configuration

### Prerequisites

1. **Install Borg Backup:**
   ```bash
   # Ubuntu/Debian
   sudo apt install borgbackup borgmatic
   
   # Arch Linux
   sudo pacman -S borg borgmatic
   
   # macOS
   brew install borgbackup borgmatic
   ```

2. **Prepare Backup Destinations:**
   - Local storage (external drives, NAS)
   - Remote SSH-accessible servers
   - Cloud storage (optional)

### Basic Setup

1. **Clone this repository:**
   ```bash
   git clone https://github.com/Xieonie/secure-backup-system.git
   cd secure-backup-system
   ```

2. **Initialize Borg repositories:**
   ```bash
   # Local repository
   borg init --encryption=repokey-blake2 /path/to/local/backup/repo
   
   # Remote repository
   borg init --encryption=repokey-blake2 user@backup-server:/path/to/remote/repo
   ```

3. **Configure Borgmatic:**
   ```bash
   cp config-examples/borgmatic.yaml.example ~/.config/borgmatic/config.yaml
   # Edit the configuration file to match your environment
   ```

4. **Set up automated backups:**
   ```bash
   sudo cp systemd/borg-backup.service /etc/systemd/system/
   sudo cp systemd/borg-backup.timer /etc/systemd/system/
   sudo systemctl enable borg-backup.timer
   sudo systemctl start borg-backup.timer
   ```

5. **Test your backup:**
   ```bash
   borgmatic --verbosity 1 --dry-run
   borgmatic --verbosity 1
   ```

## 📋 Backup Strategy

### 3-2-1 Rule Implementation

1. **3 Copies of Data:**
   - Original data on primary systems
   - Local backup repository (external drive/NAS)
   - Remote/cloud backup repository

2. **2 Different Media Types:**
   - Primary: SSDs/HDDs in production systems
   - Backup: External drives, remote servers, cloud storage

3. **1 Offsite Copy:**
   - Remote server via SSH
   - Cloud storage provider
   - Physical media stored offsite

### Retention Policy

```yaml
retention:
    keep_daily: 7      # Keep daily backups for 1 week
    keep_weekly: 4     # Keep weekly backups for 1 month
    keep_monthly: 12   # Keep monthly backups for 1 year
    keep_yearly: 5     # Keep yearly backups for 5 years
```

## 🔧 Configuration Examples

### Borgmatic Configuration

The `config-examples/borgmatic.yaml.example` includes:
- Repository locations (local and remote)
- Source directories to backup
- Exclusion patterns
- Encryption settings
- Retention policies
- Health check URLs
- Notification settings

### Systemd Timer

Automated scheduling with systemd timers provides better logging and dependency management compared to cron:

```ini
[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=1h

[Install]
WantedBy=timers.target
```

## 🔍 Monitoring and Alerting

### Health Checks

- **Repository integrity:** Regular `borg check` operations
- **Backup completion:** Monitoring backup exit codes
- **Storage space:** Monitoring available space on backup destinations
- **Network connectivity:** Testing remote repository accessibility

### Notification Methods

- Email alerts for backup failures
- Integration with monitoring systems (Prometheus, Grafana)
- Webhook notifications to messaging platforms
- Log aggregation for centralized monitoring

## 🚨 Recovery Procedures

### Emergency Recovery

1. **Boot from rescue media** if system is completely compromised
2. **Install Borg Backup** on rescue system
3. **Mount backup repository:**
   ```bash
   borg mount /path/to/repo::archive-name /mnt/backup
   ```
4. **Restore critical data** to new system

### Selective Restore

```bash
# List available archives
borg list /path/to/repo

# Extract specific files
borg extract /path/to/repo::archive-name path/to/specific/file

# Mount archive for browsing
borg mount /path/to/repo::archive-name /mnt/backup
```

## 🔒 Security Considerations

### Encryption

- **Repository encryption:** AES-256 with authenticated encryption
- **Key management:** Secure storage of repository keys
- **Passphrase security:** Strong, unique passphrases for each repository

### Access Control

- **SSH key authentication** for remote repositories
- **Restricted SSH commands** using `command=` in authorized_keys
- **Network segmentation** for backup infrastructure
- **Regular key rotation** for long-term security

### Data Protection

- **Immutable backups:** Using append-only mode where possible
- **Air-gapped backups:** Offline storage for critical data
- **Geographic distribution:** Multiple offsite locations
- **Regular testing:** Periodic restore tests to verify backup integrity

## 🔮 Potential Improvements/Future Plans

* Integration with cloud storage providers (AWS S3, Google Cloud Storage)
* Automated disaster recovery testing
* Integration with configuration management tools (Ansible, Terraform)
* Advanced monitoring with Prometheus metrics
* Backup encryption key escrow for enterprise environments
* Support for database-specific backup tools (pg_dump, mysqldump)

## ⚠️ Important Notes

* **Test your backups regularly** - A backup that can't be restored is useless
* **Store encryption keys securely** - Loss of keys means loss of data
* **Monitor backup health** - Failed backups should trigger immediate alerts
* **Document recovery procedures** - Ensure team members can perform recoveries
* **Consider legal requirements** - Some data may have specific retention requirements

## 📚 Additional Resources

* [Borg Backup Documentation](https://borgbackup.readthedocs.io/)
* [Borgmatic Documentation](https://torsion.org/borgmatic/)
* [3-2-1 Backup Strategy Guide](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/)
* [NIST Backup and Recovery Guidelines](https://csrc.nist.gov/publications/detail/sp/800-34/rev-1/final)