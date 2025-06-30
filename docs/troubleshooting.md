# Borg Backup Troubleshooting Guide

This guide covers common issues, error messages, and solutions for Borg Backup and Borgmatic configurations.

## Table of Contents

1. [Common Error Messages](#common-error-messages)
2. [Installation Issues](#installation-issues)
3. [Configuration Problems](#configuration-problems)
4. [Repository Issues](#repository-issues)
5. [Network and SSH Problems](#network-and-ssh-problems)
6. [Performance Issues](#performance-issues)
7. [Debugging Tools](#debugging-tools)

## Common Error Messages

### "Repository does not exist"
```
Error: Repository /path/to/repo does not exist.
```

**Causes:**
- Repository path is incorrect
- Repository hasn't been initialized
- Permission issues

**Solutions:**
```bash
# Check if path exists
ls -la /path/to/repo

# Initialize repository if it doesn't exist
borg init --encryption=repokey-blake2 /path/to/repo

# Check permissions
sudo chown -R $USER:$USER /path/to/repo
```

### "Failed to create/acquire the lock"
```
Error: Failed to create/acquire the lock /path/to/repo/lock.roster
```

**Causes:**
- Another borg process is running
- Previous backup was interrupted
- Stale lock file

**Solutions:**
```bash
# Check for running borg processes
ps aux | grep borg

# Break lock if no borg processes are running
borg break-lock /path/to/repo

# For remote repositories
borg break-lock ssh://user@server/repo
```

### "Passphrase wrong"
```
Error: passphrase supplied in BORG_PASSPHRASE is wrong.
```

**Causes:**
- Incorrect passphrase
- Wrong repository
- Corrupted keyfile

**Solutions:**
```bash
# Try entering passphrase manually
unset BORG_PASSPHRASE
borg list /path/to/repo

# Check if you're using the correct repository
borg info /path/to/repo

# Export and re-import key if corrupted
borg key export /path/to/repo backup.key
borg key import /path/to/repo backup.key
```

### "Archive already exists"
```
Error: Archive name already exists: hostname-2023-01-01-120000
```

**Causes:**
- Duplicate archive names
- Clock synchronization issues
- Borgmatic configuration issue

**Solutions:**
```bash
# Use unique archive names in borgmatic config
archive_name_format: '{hostname}-{now:%Y-%m-%d-%H%M%S}-{user}'

# Check system time
timedatectl status

# Sync time if necessary
sudo ntpdate -s time.nist.gov
```

### "No space left on device"
```
Error: [Errno 28] No space left on device
```

**Causes:**
- Target storage is full
- Temporary directory is full
- Insufficient space for deduplication

**Solutions:**
```bash
# Check disk space
df -h

# Clean up old archives
borg prune /path/to/repo --keep-daily=7 --keep-weekly=4

# Change temporary directory
export TMPDIR=/path/to/larger/tmp

# Compact repository
borg compact /path/to/repo
```

## Installation Issues

### "borg: command not found"
**Solutions:**
```bash
# Install via package manager (Ubuntu/Debian)
sudo apt update && sudo apt install borgbackup

# Install via pip
pip3 install borgbackup

# Install from source
git clone https://github.com/borgbackup/borg.git
cd borg
pip install -e .
```

### "borgmatic: command not found"
**Solutions:**
```bash
# Install via package manager
sudo apt install borgmatic

# Install via pip
pip3 install borgmatic

# Add to PATH if installed locally
echo 'export PATH=$PATH:~/.local/bin' >> ~/.bashrc
source ~/.bashrc
```

### Python/Dependency Issues
```bash
# Update pip
pip3 install --upgrade pip

# Install missing dependencies
sudo apt install python3-dev libssl-dev libacl1-dev libxxhash-dev

# Use virtual environment
python3 -m venv borg-env
source borg-env/bin/activate
pip install borgbackup borgmatic
```

## Configuration Problems

### Borgmatic Configuration Validation
```bash
# Validate configuration
borgmatic config validate

# Generate new configuration
borgmatic config generate

# Show effective configuration
borgmatic config show
```

### Invalid YAML Syntax
```
Error: while parsing a block mapping
```

**Solutions:**
```bash
# Check YAML syntax
python3 -c "import yaml; yaml.safe_load(open('config.yaml'))"

# Use online YAML validator
# Fix indentation (use spaces, not tabs)
# Check for missing colons or quotes
```

### Path Issues in Configuration
```yaml
# Correct path specification
source_directories:
    - /home
    - /etc
    - /var/log

# Avoid these common mistakes
source_directories:
    - ~/home  # Don't use ~ in borgmatic
    - /home/  # Trailing slash not needed
```

### Exclude Patterns Not Working
```yaml
# Correct exclude patterns
exclude_patterns:
    - '*.tmp'
    - '*.cache'
    - '*/.cache'
    - '/tmp'
    - '/var/tmp'

# Test exclude patterns
borg create --dry-run --list /path/to/repo::test /path/to/backup
```

## Repository Issues

### Repository Corruption
```bash
# Check repository integrity
borg check /path/to/repo

# Attempt repair (use with caution)
borg check --repair /path/to/repo

# If repair fails, restore from backup repository
```

### Repository Migration
```bash
# Migrate to new location
borg init --encryption=repokey-blake2 /new/path/to/repo

# Transfer archives
borg list /old/repo --short | while read archive; do
    borg transfer /old/repo::$archive /new/repo
done
```

### Key Management Issues
```bash
# Export repository key
borg key export /path/to/repo key-backup.txt

# Change passphrase
borg key change-passphrase /path/to/repo

# Import key to new repository
borg key import /path/to/repo key-backup.txt
```

## Network and SSH Problems

### SSH Connection Issues
```bash
# Test SSH connection
ssh -v user@backup-server.com

# Check SSH key
ssh-add -l
ssh-add ~/.ssh/borg_backup

# Test borg over SSH
borg info ssh://user@backup-server.com/repo
```

### SSH Configuration
```bash
# Create SSH config
cat >> ~/.ssh/config << EOF
Host backup-server
    HostName backup-server.com
    User backup-user
    IdentityFile ~/.ssh/borg_backup
    Port 22
    Compression yes
    ServerAliveInterval 60
EOF
```

### Network Timeouts
```bash
# Increase SSH timeout
export BORG_RSH="ssh -o ConnectTimeout=60 -o ServerAliveInterval=20"

# Use compression for slow connections
export BORG_RSH="ssh -C"

# Limit bandwidth
export BORG_RSH="ssh -o 'ProxyCommand=pv -q -L 1m | nc %h %p'"
```

### Firewall Issues
```bash
# Check if SSH port is open
telnet backup-server.com 22

# Test from different network
# Check firewall rules on both ends
sudo ufw status
```

## Performance Issues

### Slow Backups
**Diagnosis:**
```bash
# Run with verbose output
borg create -v --stats /path/to/repo::test /path/to/backup

# Check system resources
htop
iotop
```

**Solutions:**
```bash
# Use faster compression
borg create --compression lz4 /path/to/repo::archive /path/to/backup

# Exclude unnecessary files
exclude_patterns:
    - '*.tmp'
    - '*/.cache'
    - '*/node_modules'

# Use SSD for cache
export BORG_CACHE_DIR=/path/to/ssd/cache
```

### High Memory Usage
```bash
# Limit memory usage
export BORG_WORKAROUND_BASESYNCFILE=yes

# Use chunker-params for large files
borg create --chunker-params=19,23,21,4095 /path/to/repo::archive /path/to/backup
```

### Slow Repository Checks
```bash
# Quick check only
borg check --repository-only /path/to/repo

# Skip data verification for speed
borg check --archives-only /path/to/repo

# Parallel checks for multiple repositories
for repo in repo1 repo2 repo3; do
    borg check $repo &
done
wait
```

## Debugging Tools

### Enable Debug Logging
```bash
# Set debug level
export BORG_LOGGING_CONF=/path/to/logging.conf

# Create logging configuration
cat > logging.conf << EOF
[loggers]
keys=root

[handlers]
keys=console,file

[formatters]
keys=detailed

[logger_root]
level=DEBUG
handlers=console,file

[handler_console]
class=StreamHandler
level=INFO
formatter=detailed
args=(sys.stderr,)

[handler_file]
class=FileHandler
level=DEBUG
formatter=detailed
args=('/var/log/borg.log',)

[formatter_detailed]
format=%(asctime)s %(levelname)s %(name)s %(message)s
EOF
```

### Borgmatic Debug Mode
```bash
# Run borgmatic with debug output
borgmatic create --verbosity 2

# Show what would be done
borgmatic create --dry-run --verbosity 1

# Test configuration
borgmatic config validate --verbosity 2
```

### System Monitoring
```bash
# Monitor disk I/O
sudo iotop -o

# Monitor network usage
sudo nethogs

# Monitor system calls
strace -f -e trace=file borg create /path/to/repo::test /path/to/backup
```

### Log Analysis
```bash
# Analyze borg logs
grep ERROR /var/log/borg.log
grep WARNING /var/log/borg.log

# Analyze system logs
journalctl -u borg-backup.service
tail -f /var/log/syslog | grep borg
```

## Diagnostic Scripts

### Repository Health Check
```bash
#!/bin/bash
# borg-health-check.sh

REPO="$1"
if [[ -z "$REPO" ]]; then
    echo "Usage: $0 <repository>"
    exit 1
fi

echo "=== Borg Repository Health Check ==="
echo "Repository: $REPO"
echo

# Basic connectivity
echo "Testing repository access..."
if borg info "$REPO" &>/dev/null; then
    echo "✓ Repository accessible"
else
    echo "✗ Repository not accessible"
    exit 1
fi

# Repository integrity
echo "Checking repository integrity..."
if borg check "$REPO" &>/dev/null; then
    echo "✓ Repository integrity OK"
else
    echo "✗ Repository integrity issues found"
fi

# Archive count
archive_count=$(borg list "$REPO" --short | wc -l)
echo "Archive count: $archive_count"

# Repository size
borg info "$REPO" | grep -E "(Original size|Compressed size|Deduplicated size)"

echo
echo "Health check completed!"
```

### Performance Benchmark
```bash
#!/bin/bash
# borg-benchmark.sh

REPO="$1"
TEST_DIR="/tmp/borg-test-data"
TEST_SIZE="100M"

# Create test data
mkdir -p "$TEST_DIR"
dd if=/dev/urandom of="$TEST_DIR/testfile" bs=1M count=100

echo "=== Borg Performance Benchmark ==="

# Test backup speed
echo "Testing backup speed..."
time borg create "$REPO::benchmark-$(date +%s)" "$TEST_DIR"

# Test extraction speed
echo "Testing extraction speed..."
time borg extract "$REPO::benchmark-$(date +%s)" --dry-run

# Cleanup
rm -rf "$TEST_DIR"
borg delete "$REPO::benchmark-$(date +%s)"

echo "Benchmark completed!"
```

## Emergency Recovery

### When Everything Fails
```bash
# 1. Check if repository exists
ls -la /path/to/repo

# 2. Try different borg version
pip install borgbackup==1.2.0

# 3. Use rescue mode
borg check --repair /path/to/repo

# 4. Extract what you can
mkdir /tmp/emergency-recovery
borg mount /path/to/repo /tmp/emergency-recovery
cp -r /tmp/emergency-recovery/important-files /safe/location/

# 5. Contact borg community
# - GitHub issues: https://github.com/borgbackup/borg/issues
# - Mailing list: borgbackup@python.org
```

### Data Recovery Services
If all else fails:
1. Stop using the storage device immediately
2. Contact professional data recovery services
3. Provide them with repository format information
4. Consider the cost vs. value of recovered data

## Prevention

### Best Practices
1. **Test backups regularly**
2. **Monitor backup health**
3. **Keep multiple repository copies**
4. **Document your configuration**
5. **Train team members**
6. **Update software regularly**
7. **Monitor system resources**

### Monitoring Setup
```bash
# Add to crontab for regular health checks
0 6 * * * /usr/local/bin/borg-health-check.sh /path/to/repo | mail -s "Backup Health" admin@example.com
```

This troubleshooting guide should help you resolve most common issues with Borg Backup. Remember to always test your backups and keep documentation updated!