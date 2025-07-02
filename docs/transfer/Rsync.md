# Transfering Files using Rsync

Rsync is a powerful and efficient tool for transferring and synchronizing files between your local computer and a remote cluster. It offers several advantages over basic `scp` or `sftp`, including incremental transfers, compression, and the ability to resume interrupted transfers.

## Overview

Rsync (Remote Sync) is particularly useful for:

- **Incremental backups**: Only transfers changed files
- **Large file transfers**: Can resume interrupted transfers
- **Directory synchronization**: Keeps local and remote directories in sync
- **Bandwidth optimization**: Built-in compression reduces transfer time

## Prerequisites

Before using rsync, ensure you have:

- SSH access to the cluster
- Rsync installed on your local machine
- Your cluster username and hostname

## Installation

### Windows

**Option 1: WSL (Windows Subsystem for Linux)**
```bash
# Install WSL first, then in WSL terminal:
sudo apt update
sudo apt install rsync
```

**Option 2: Git Bash**
```bash
# Rsync comes pre-installed with Git for Windows
# Download from: https://git-scm.com/download/win
```

**Option 3: Cygwin**
```bash
# Install Cygwin and select rsync package during installation
# Download from: https://www.cygwin.com/
```

### Linux

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install rsync

# CentOS/RHEL/Fedora
sudo yum install rsync
# or for newer versions:
sudo dnf install rsync

# Arch Linux
sudo pacman -S rsync
```

### macOS

```bash
# Rsync is pre-installed on macOS
# To get the latest version via Homebrew:
brew install rsync
```

## Basic Syntax

The general syntax for rsync is:

```bash
rsync [OPTIONS] SOURCE DESTINATION
```

For cluster transfers:
```bash
# Upload to cluster
rsync [OPTIONS] /local/path/ username@cluster.example.com:/remote/path/

# Download from cluster
rsync [OPTIONS] username@cluster.example.com:/remote/path/ /local/path/
```

## Essential Options

| Option      | Description                                            |
| ----------- | ------------------------------------------------------ |
| `-a`        | Archive mode (preserves permissions, timestamps, etc.) |
| `-v`        | Verbose output                                         |
| `-z`        | Compress data during transfer                          |
| `-P`        | Show progress and keep partial files                   |
| `-r`        | Recursive (for directories)                            |
| `-u`        | Update only (skip files that are newer on destination) |
| `-n`        | Dry run (preview what would be transferred)            |
| `--delete`  | Delete files on destination that don't exist on source |
| `--exclude` | Exclude files/directories matching pattern             |
| `--progress`| Print information showing the progress of the transfer |


## Common Transfer Scenarios

### 1. Upload a Single File

```bash
# Basic file upload
rsync -avz myfile.txt username@cluster.example.com:~/

# Upload to specific directory
rsync -avz myfile.txt username@cluster.example.com:~/data/
```

### 2. Upload a Directory

```bash
# Upload entire directory (creates 'myproject' on cluster)
rsync -avz myproject/ username@cluster.example.com:~/myproject/

# Upload directory contents only
rsync -avz myproject/ username@cluster.example.com:~/existing_dir/
```

### 3. Download from Cluster

```bash
# Download a file
rsync -avz username@cluster.example.com:~/results/output.txt ./

# Download a directory
rsync -avz username@cluster.example.com:~/results/ ./local_results/
```

### 4. Synchronize Directories

```bash
# Keep local and remote directories in sync
rsync -avz --delete myproject/ username@cluster.example.com:~/myproject/
```

## Advanced Examples

### Large Dataset Transfer with Progress

```bash
# Transfer large files with detailed progress
rsync -avzP --human-readable \
    /path/to/large/dataset/ \
    username@cluster.example.com:~/data/
```

### Selective File Transfer

```bash
# Only transfer specific file types
rsync -avz --include="*.R" --include="*.py" --exclude="*" \
    myproject/ \
    username@cluster.example.com:~/scripts/

# Exclude certain directories
rsync -avz --exclude="*.git" --exclude="__pycache__" \
    myproject/ \
    username@cluster.example.com:~/myproject/
```

### Resume Interrupted Transfer

```bash
# Use -P to resume partial transfers
rsync -avzP myproject/ username@cluster.example.com:~/myproject/
```

### Dry Run Before Transfer

```bash
# Preview what will be transferred (recommended for large transfers)
rsync -avzn myproject/ username@cluster.example.com:~/myproject/

# If satisfied, remove -n to execute
rsync -avz myproject/ username@cluster.example.com:~/myproject/
```

## Platform-Specific Examples

### Windows (WSL/Git Bash)

```bash
# From Windows file system to cluster
rsync -avz /c/Users/YourName/Documents/myproject/ \
    username@cluster.example.com:~/myproject/

# Using WSL paths
rsync -avz /mnt/c/Users/YourName/Documents/myproject/ \
    username@cluster.example.com:~/myproject/
```

### Linux

```bash
# Standard Linux paths
rsync -avz ~/Documents/myproject/ \
    username@cluster.example.com:~/myproject/

# From external drive
rsync -avz /media/usb/myproject/ \
    username@cluster.example.com:~/myproject/
```

### macOS

```bash
# From Documents folder
rsync -avz ~/Documents/myproject/ \
    username@cluster.example.com:~/myproject/

# From external drive
rsync -avz /Volumes/ExternalDrive/myproject/ \
    username@cluster.example.com:~/myproject/
```

## Best Practices

### 1. Use SSH Keys

Set up SSH key authentication to avoid password prompts:

```bash
# Generate SSH key (if not already done)
ssh-keygen -t rsa -b 4096

# Copy public key to cluster
ssh-copy-id username@cluster.example.com
```

### 2. Create a Transfer Script

Create a shell script for frequent transfers:

```bash
#!/bin/bash
# upload_to_cluster.sh

PROJECT_DIR="~/Documents/myproject"
CLUSTER_USER="username"
CLUSTER_HOST="cluster.example.com"
CLUSTER_PATH="~/myproject"

echo "Starting transfer to cluster..."
rsync -avzP \
    --exclude=".git" \
    --exclude="__pycache__" \
    --exclude="*.tmp" \
    "$PROJECT_DIR/" \
    "$CLUSTER_USER@$CLUSTER_HOST:$CLUSTER_PATH/"

echo "Transfer completed!"
```

### 3. Use Configuration File

Create an rsync configuration file:

```bash
# ~/.rsync_cluster
--archive
--verbose
--compress
--progress
--human-readable
--exclude=.git
--exclude=__pycache__
--exclude=*.tmp
```

Use it with:
```bash
rsync --config=~/.rsync_cluster myproject/ username@cluster.example.com:~/myproject/
```

## Troubleshooting

### Permission Denied

```bash
# Check SSH connection first
ssh username@cluster.example.com

# Verify destination directory exists and is writable
ssh username@cluster.example.com "ls -la ~/target_directory"
```

### Slow Transfer Speed

```bash
# Adjust compression level (0-9, where 9 is maximum compression)
rsync -avz --compress-level=6 myproject/ username@cluster.example.com:~/myproject/

# Use different SSH cipher for faster transfers
rsync -avz -e "ssh -c aes128-ctr" myproject/ username@cluster.example.com:~/myproject/
```

### Large File Handling

```bash
# For very large files, use minimal compression
rsync -av --compress-level=1 -P largefile.dat username@cluster.example.com:~/data/

# Or disable compression entirely
rsync -avP largefile.dat username@cluster.example.com:~/data/
```

## Monitoring Transfer Progress

### Real-time Progress

```bash
# Show detailed progress with file names
rsync -avz --progress myproject/ username@cluster.example.com:~/myproject/

# Show overall progress
rsync -avzP myproject/ username@cluster.example.com:~/myproject/
```

### Log Transfer Details

```bash
# Create detailed log
rsync -avz --log-file=rsync.log myproject/ username@cluster.example.com:~/myproject/

# View log
tail -f rsync.log
```

## Security Considerations

- Always use SSH for remote transfers (rsync over SSH is the default)
- Verify fingerprints when connecting to new clusters
- Use SSH keys instead of passwords for automated transfers
- Be cautious with `--delete` option to avoid accidental data loss
- Consider using `--dry-run` first for critical transfers

## Conclusion

Rsync is an essential tool for efficient file transfers to computing clusters. Its incremental transfer capabilities, combined with compression and resumption features, make it ideal for scientific computing workflows where large datasets and frequent synchronization are common.

Remember to always test with `--dry-run` first, especially when using `--delete`, and consider creating transfer scripts for frequently used commands.

