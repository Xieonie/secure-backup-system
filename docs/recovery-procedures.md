# Backup Recovery Procedures

This document provides comprehensive procedures for recovering data from Borg backups in various scenarios, from simple file restoration to complete disaster recovery.

## Table of Contents

1. [Quick Recovery](#quick-recovery)
2. [File and Directory Recovery](#file-and-directory-recovery)
3. [System Recovery](#system-recovery)
4. [Disaster Recovery](#disaster-recovery)
5. [Emergency Procedures](#emergency-procedures)
6. [Recovery Testing](#recovery-testing)

## Quick Recovery

### List Available Archives
```bash
# List all archives in repository
borg list /path/to/repo

# List archives with timestamps
borg list /path/to/repo --format '{archive}{TAB}{time}'

# List recent archives only
borg list /path/to/repo --last 10
```

### Browse Archive Contents
```bash
# List files in specific archive
borg list /path/to/repo::archive-name

# List files with details
borg list /path/to/repo::archive-name --format '{mode} {user:6} {group:6} {size:8} {mtime} {path}'

# Search for specific files
borg list /path/to/repo::archive-name | grep filename
```

### Mount Archive for Browsing
```bash
# Create mount point
mkdir /tmp/borg-mount

# Mount archive
borg mount /path/to/repo::archive-name /tmp/borg-mount

# Browse files
ls -la /tmp/borg-mount
cd /tmp/borg-mount && find . -name "*.conf"

# Unmount when done
borg umount /tmp/borg-mount
```

## File and Directory Recovery

### Extract Specific Files
```bash
# Extract single file
borg extract /path/to/repo::archive-name path/to/file

# Extract multiple files
borg extract /path/to/repo::archive-name path/to/file1 path/to/file2

# Extract directory
borg extract /path/to/repo::archive-name path/to/directory

# Extract with original paths
borg extract /path/to/repo::archive-name --strip-components 0
```

### Extract to Specific Location
```bash
# Extract to different location
mkdir /tmp/restore
cd /tmp/restore
borg extract /path/to/repo::archive-name

# Extract specific files to current directory
borg extract /path/to/repo::archive-name home/user/documents --strip-components 3
```

### Selective Recovery
```bash
# Extract files matching pattern
borg extract /path/to/repo::archive-name --pattern '*.conf'
borg extract /path/to/repo::archive-name --pattern 'home/user/documents/**'

# Exclude certain files during extraction
borg extract /path/to/repo::archive-name --exclude '*.tmp' --exclude '*.cache'
```

### Recovery with Verification
```bash
# Extract and verify checksums
borg extract /path/to/repo::archive-name --verify-data

# Compare extracted files with archive
borg diff /path/to/repo::archive-name path/to/extracted/files
```

## System Recovery

### Configuration Recovery
```bash
# Recover system configuration files
sudo borg extract /path/to/repo::latest etc/

# Recover specific service configurations
sudo borg extract /path/to/repo::latest etc/nginx/ etc/apache2/ etc/mysql/

# Recover user configurations
borg extract /path/to/repo::latest home/user/.config/
```

### Database Recovery
```bash
# Extract database dumps
borg extract /path/to/repo::latest var/backups/mysql/

# Restore MySQL database
mysql -u root -p database_name < /var/backups/mysql/database_name.sql

# Restore PostgreSQL database
sudo -u postgres psql -d database_name -f /var/backups/postgresql/database_name.sql
```

### Application Data Recovery
```bash
# Recover web application data
sudo borg extract /path/to/repo::latest var/www/

# Recover application configurations
sudo borg extract /path/to/repo::latest opt/application/

# Set proper permissions after recovery
sudo chown -R www-data:www-data /var/www/
sudo chmod -R 755 /var/www/
```

## Disaster Recovery

### Complete System Recovery

#### 1. Boot from Recovery Media
```bash
# Boot from Ubuntu Live USB/CD
# Connect to network
# Install necessary tools
sudo apt update
sudo apt install borgbackup openssh-client
```

#### 2. Prepare Target System
```bash
# Partition and format disks
sudo fdisk /dev/sda
sudo mkfs.ext4 /dev/sda1
sudo mkfs.ext4 /dev/sda2

# Mount target filesystem
sudo mkdir /mnt/target
sudo mount /dev/sda1 /mnt/target
sudo mkdir /mnt/target/home
sudo mount /dev/sda2 /mnt/target/home
```

#### 3. Restore System Files
```bash
# Mount repository
mkdir /tmp/borg-mount
borg mount ssh://user@backup-server/repo::latest /tmp/borg-mount

# Copy system files
sudo cp -a /tmp/borg-mount/* /mnt/target/

# Or extract directly
cd /mnt/target
sudo borg extract ssh://user@backup-server/repo::latest
```

#### 4. Restore Boot Configuration
```bash
# Mount special filesystems
sudo mount --bind /dev /mnt/target/dev
sudo mount --bind /proc /mnt/target/proc
sudo mount --bind /sys /mnt/target/sys

# Chroot into restored system
sudo chroot /mnt/target

# Reinstall bootloader
grub-install /dev/sda
update-grub

# Exit chroot
exit
```

#### 5. Final Steps
```bash
# Update fstab if necessary
sudo nano /mnt/target/etc/fstab

# Unmount filesystems
sudo umount /mnt/target/dev
sudo umount /mnt/target/proc
sudo umount /mnt/target/sys
sudo umount /mnt/target/home
sudo umount /mnt/target

# Reboot
sudo reboot
```

### Partial System Recovery

#### Operating System Recovery
```bash
# Recover only OS files, preserve user data
borg extract /path/to/repo::latest \
    --exclude 'home/**' \
    --exclude 'var/log/**' \
    --exclude 'tmp/**'
```

#### User Data Recovery
```bash
# Recover user home directories
borg extract /path/to/repo::latest home/

# Recover specific user
borg extract /path/to/repo::latest home/username/
```

## Emergency Procedures

### Repository Access Issues

#### Lost Passphrase Recovery
```bash
# If you have the key file
borg key export /path/to/repo --paper > key-backup.txt

# Print key as QR code for secure storage
qrencode -t UTF8 < key-backup.txt
```

#### Corrupted Repository Recovery
```bash
# Check repository integrity
borg check /path/to/repo

# Attempt repair
borg check --repair /path/to/repo

# If repair fails, use backup repository
borg list /backup/repo
```

#### Network Issues Recovery
```bash
# Use alternative connection method
export BORG_RSH="ssh -o ConnectTimeout=30 -o ServerAliveInterval=10"

# Use local copy if available
rsync -av user@backup-server:/repo/ /tmp/local-repo/
borg list /tmp/local-repo
```

### Emergency Recovery Scripts

#### Quick System Recovery Script
```bash
#!/bin/bash
# emergency-recovery.sh

set -euo pipefail

REPO="${1:-}"
ARCHIVE="${2:-latest}"
TARGET="${3:-/mnt/recovery}"

if [[ -z "$REPO" ]]; then
    echo "Usage: $0 <repository> [archive] [target]"
    exit 1
fi

echo "Starting emergency recovery..."
echo "Repository: $REPO"
echo "Archive: $ARCHIVE"
echo "Target: $TARGET"

# Create target directory
sudo mkdir -p "$TARGET"

# Mount repository
echo "Mounting repository..."
mkdir -p /tmp/borg-mount
borg mount "$REPO::$ARCHIVE" /tmp/borg-mount

# Copy critical system files
echo "Recovering critical system files..."
sudo cp -a /tmp/borg-mount/etc "$TARGET/"
sudo cp -a /tmp/borg-mount/var "$TARGET/"
sudo cp -a /tmp/borg-mount/usr/local "$TARGET/"

# Copy user data
echo "Recovering user data..."
sudo cp -a /tmp/borg-mount/home "$TARGET/"

# Unmount
borg umount /tmp/borg-mount

echo "Emergency recovery completed!"
echo "Files recovered to: $TARGET"
```

#### Database Emergency Recovery
```bash
#!/bin/bash
# db-emergency-recovery.sh

REPO="$1"
ARCHIVE="${2:-latest}"
DB_NAME="$3"

if [[ -z "$REPO" || -z "$DB_NAME" ]]; then
    echo "Usage: $0 <repository> [archive] <database_name>"
    exit 1
fi

echo "Recovering database: $DB_NAME"

# Extract database dump
borg extract "$REPO::$ARCHIVE" var/backups/mysql/${DB_NAME}.sql

# Stop database service
sudo systemctl stop mysql

# Restore database
mysql -u root -p -e "DROP DATABASE IF EXISTS $DB_NAME; CREATE DATABASE $DB_NAME;"
mysql -u root -p "$DB_NAME" < "var/backups/mysql/${DB_NAME}.sql"

# Start database service
sudo systemctl start mysql

echo "Database recovery completed!"
```

## Recovery Testing

### Automated Recovery Testing
```bash
#!/bin/bash
# test-recovery.sh

REPO="/path/to/repo"
TEST_DIR="/tmp/recovery-test"
ARCHIVE="latest"

echo "Starting recovery test..."

# Clean test directory
rm -rf "$TEST_DIR"
mkdir -p "$TEST_DIR"

# Test file extraction
echo "Testing file extraction..."
cd "$TEST_DIR"
borg extract "$REPO::$ARCHIVE" home/user/test-file.txt

if [[ -f "home/user/test-file.txt" ]]; then
    echo "✓ File extraction successful"
else
    echo "✗ File extraction failed"
    exit 1
fi

# Test directory extraction
echo "Testing directory extraction..."
borg extract "$REPO::$ARCHIVE" etc/nginx/

if [[ -d "etc/nginx" ]]; then
    echo "✓ Directory extraction successful"
else
    echo "✗ Directory extraction failed"
    exit 1
fi

# Test mount operation
echo "Testing mount operation..."
mkdir -p /tmp/borg-test-mount
borg mount "$REPO::$ARCHIVE" /tmp/borg-test-mount

if mountpoint -q /tmp/borg-test-mount; then
    echo "✓ Mount operation successful"
    borg umount /tmp/borg-test-mount
else
    echo "✗ Mount operation failed"
    exit 1
fi

# Cleanup
rm -rf "$TEST_DIR"
rmdir /tmp/borg-test-mount

echo "✓ All recovery tests passed!"
```

### Recovery Performance Testing
```bash
#!/bin/bash
# test-recovery-performance.sh

REPO="/path/to/repo"
ARCHIVE="latest"
TEST_SIZE="1G"

echo "Testing recovery performance..."

# Test extraction speed
echo "Testing extraction speed..."
time borg extract "$REPO::$ARCHIVE" --dry-run

# Test mount performance
echo "Testing mount performance..."
mkdir -p /tmp/perf-test
time borg mount "$REPO::$ARCHIVE" /tmp/perf-test
time ls -la /tmp/perf-test > /dev/null
borg umount /tmp/perf-test

echo "Performance test completed!"
```

### Recovery Verification
```bash
#!/bin/bash
# verify-recovery.sh

ORIGINAL_PATH="$1"
RECOVERED_PATH="$2"

if [[ -z "$ORIGINAL_PATH" || -z "$RECOVERED_PATH" ]]; then
    echo "Usage: $0 <original_path> <recovered_path>"
    exit 1
fi

echo "Verifying recovery..."

# Compare file counts
original_count=$(find "$ORIGINAL_PATH" -type f | wc -l)
recovered_count=$(find "$RECOVERED_PATH" -type f | wc -l)

echo "Original files: $original_count"
echo "Recovered files: $recovered_count"

if [[ $original_count -eq $recovered_count ]]; then
    echo "✓ File count matches"
else
    echo "✗ File count mismatch"
fi

# Compare checksums
echo "Comparing checksums..."
diff <(find "$ORIGINAL_PATH" -type f -exec md5sum {} \; | sort) \
     <(find "$RECOVERED_PATH" -type f -exec md5sum {} \; | sort)

if [[ $? -eq 0 ]]; then
    echo "✓ All checksums match"
else
    echo "✗ Checksum mismatches found"
fi

echo "Verification completed!"
```

## Best Practices

### Recovery Planning
1. **Document Recovery Procedures**: Keep this guide accessible offline
2. **Test Recovery Regularly**: Schedule monthly recovery tests
3. **Maintain Recovery Media**: Keep updated rescue USB/CD
4. **Store Keys Securely**: Multiple secure locations for encryption keys
5. **Practice Procedures**: Regular drills for disaster scenarios

### Recovery Security
1. **Verify Integrity**: Always verify recovered data
2. **Scan for Malware**: Scan recovered files before use
3. **Update Systems**: Apply security updates after recovery
4. **Change Passwords**: Update passwords after system recovery
5. **Review Logs**: Check for suspicious activity

### Recovery Documentation
1. **Log All Actions**: Document what was recovered and when
2. **Note Issues**: Record any problems encountered
3. **Update Procedures**: Improve procedures based on experience
4. **Share Knowledge**: Train team members on procedures

This comprehensive recovery guide ensures you can restore your data and systems effectively in any situation.