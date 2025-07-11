# Downloading Files via FTP with curl

## Overview

This guide covers automated downloading of files from FTP servers using `curl` in HPC environments. The approach demonstrated here is particularly useful for downloading large genomic datasets that require robust error handling and parallel processing.

## Key Features

- **Parallel Downloads**: Uses SLURM job arrays to download multiple files simultaneously
- **Error Handling**: Comprehensive retry mechanisms and integrity checks
- **Progress Monitoring**: Detailed logging and progress tracking
- **File Validation**: Automatic verification of downloaded files
- **Resume Capability**: Skips already downloaded files

## Prerequisites

- Access to an HPC cluster with SLURM
- `curl` command-line tool
- Basic understanding of bash scripting

## Script Components

### SLURM Configuration

The script uses SLURM job arrays to parallelize downloads:

```bash
#SBATCH --array=1-15    # Download 15 files in parallel
#SBATCH --cpus-per-task=1
#SBATCH --mem=4G
#SBATCH --time=06:00:00
```

### File Management

Files are organized in arrays with proper indexing:

```bash
declare -a FILES=(
    ""  # Index 0 is empty to align with SLURM index (1-15)
    "ag1000g.phase2.ar1.pass.2L.vcf.gz"
    "ag1000g.phase2.ar1.pass.2R.vcf.gz"
    # ... more files
)
```

### Download Logic

The script implements several safety checks:

1. **Connection Test**: Verifies FTP server accessibility
2. **File Existence**: Checks if files are already downloaded
3. **Integrity Verification**: Validates VCF.gz files using `gzip -t`
4. **Progress Tracking**: Logs download progress and duration

## Complete Script

```bash
#!/bin/bash
################### SLURM configuration ##############################
#SBATCH -A invalbo
#SBATCH -p fast
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --job-name=ag1000g_download
#SBATCH --output=logs/ag1000g_download_%A_%a.out
#SBATCH --error=logs/ag1000g_download_%A_%a.err
#SBATCH --time=06:00:00
#SBATCH --cpus-per-task=1
#SBATCH --mem=4G
#SBATCH --array=1-15
#SBATCH --mail-user=loic.talignani@ird.fr
#SBATCH --mail-type=FAIL
################### SLURM configuration ##############################

# Configuration
FTP_SERVER="ngs.sanger.ac.uk"
REMOTE_DIR="/production/ag1000g/phase2/AR1/variation/main/vcf/pass"
LOCAL_DIR="./multiallelic"
LOG_DIR="./logs"
LOG_FILE="$LOG_DIR/download_log_${SLURM_ARRAY_JOB_ID}_${SLURM_ARRAY_TASK_ID}.log"

# Create necessary directories
mkdir -p "$LOCAL_DIR" "$LOG_DIR"

# Download file list (index starts from 1)
declare -a FILES=(
    ""  # Index 0 is empty to align with SLURM index (1-15)
    "ag1000g.phase2.ar1.pass.2L.vcf.gz"
    "ag1000g.phase2.ar1.pass.2R.vcf.gz"
    "ag1000g.phase2.ar1.pass.3L.vcf.gz"
    "ag1000g.phase2.ar1.pass.3R.vcf.gz"
    "ag1000g.phase2.ar1.pass.X.vcf.gz"
    "ag1000g.phase2.ar1.pass.2L.vcf.gz.csi"
    "ag1000g.phase2.ar1.pass.2L.vcf.gz.tbi"
    "ag1000g.phase2.ar1.pass.2R.vcf.gz.csi"
    "ag1000g.phase2.ar1.pass.2R.vcf.gz.tbi"
    "ag1000g.phase2.ar1.pass.3L.vcf.gz.csi"
    "ag1000g.phase2.ar1.pass.3L.vcf.gz.tbi"
    "ag1000g.phase2.ar1.pass.3R.vcf.gz.csi"
    "ag1000g.phase2.ar1.pass.3R.vcf.gz.tbi"
    "ag1000g.phase2.ar1.pass.X.vcf.gz.csi"
    "ag1000g.phase2.ar1.pass.X.vcf.gz.tbi"
)

# Logging function
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [Task ${SLURM_ARRAY_TASK_ID}] $1" | tee -a "$LOG_FILE"
}

# Get the file to download for this task
CURRENT_FILE="${FILES[$SLURM_ARRAY_TASK_ID]}"

if [[ -z "$CURRENT_FILE" ]]; then
    log_message "ERROR: No file assigned to task $SLURM_ARRAY_TASK_ID"
    exit 1
fi

log_message "========================================="
log_message "STARTING TASK: $SLURM_ARRAY_TASK_ID"
log_message "Job Array ID: $SLURM_ARRAY_JOB_ID"
log_message "File to Download: $CURRENT_FILE"
log_message "FTP Server: $FTP_SERVER"
log_message "Remote Directory: $REMOTE_DIR"
log_message "Local Directory: $LOCAL_DIR"

# Test FTP connection
log_message "Testing FTP connection..."
if ! curl -s --connect-timeout 30 "ftp://$FTP_SERVER/" > /dev/null 2>&1; then
    log_message "ERROR: Unable to connect to FTP server"
    exit 1
fi
log_message "FTP connection successful"

# Check if the file is already downloaded
if [[ -f "$LOCAL_DIR/$CURRENT_FILE" ]]; then
    log_message "File already exists, checking integrity..."
    
    # Check if file is non-empty
    if [[ -s "$LOCAL_DIR/$CURRENT_FILE" ]]; then
        log_message "File already downloaded and non-empty"
        
        # For VCF files, check validity
        if [[ "$CURRENT_FILE" == *.vcf.gz ]]; then
            if gzip -t "$LOCAL_DIR/$CURRENT_FILE" 2>/dev/null; then
                log_message "✓ VCF.gz is valid, no download needed"
                exit 0
            else
                log_message "⚠ VCF.gz is corrupted, re-downloading..."
                rm -f "$LOCAL_DIR/$CURRENT_FILE"
            fi
        else
            log_message "✓ Index file is valid, no download needed"
            exit 0
        fi
    else
        log_message "Empty file detected, removing it..."
        rm -f "$LOCAL_DIR/$CURRENT_FILE"
    fi
fi

# Begin download
log_message "Starting download: $CURRENT_FILE"

# Adjust timeout based on file type
if [[ "$CURRENT_FILE" == *.vcf.gz ]]; then
    TIMEOUT=7200  # 2 hours for VCF files
    log_message "VCF file detected, timeout set to 2 hours"
else
    TIMEOUT=1800  # 30 minutes for index files
    log_message "Index file detected, timeout set to 30 minutes"
fi

# Start download with curl
START_TIME=$(date +%s)
log_message "Command: curl -f -s -S --retry 5 --retry-delay 30 --connect-timeout 60 --max-time $TIMEOUT"

if curl -f -s -S --retry 5 --retry-delay 30 \
    --connect-timeout 60 --max-time $TIMEOUT \
    --progress-bar \
    -o "$LOCAL_DIR/$CURRENT_FILE" \
    "ftp://$FTP_SERVER$REMOTE_DIR/$CURRENT_FILE" 2>&1 | tee -a "$LOG_FILE"; then
    
    END_TIME=$(date +%s)
    DURATION=$((END_TIME - START_TIME))
    FILE_SIZE=$(du -h "$LOCAL_DIR/$CURRENT_FILE" | cut -f1)
    
    log_message "✓ Download successful: $CURRENT_FILE"
    log_message "  Size: $FILE_SIZE"
    log_message "  Duration: ${DURATION}s"
    
    # Post-download check for VCF files
    if [[ "$CURRENT_FILE" == *.vcf.gz ]]; then
        log_message "Verifying VCF file integrity..."
        if gzip -t "$LOCAL_DIR/$CURRENT_FILE" 2>/dev/null; then
            log_message "✓ VCF.gz file is valid"
        else
            log_message "✗ VCF.gz file is corrupted after download"
            rm -f "$LOCAL_DIR/$CURRENT_FILE"
            exit 1
        fi
    fi
    
    log_message "========================================="
    log_message "TASK $SLURM_ARRAY_TASK_ID COMPLETED SUCCESSFULLY"
    exit 0
    
else
    log_message "✗ Download failed: $CURRENT_FILE"
    log_message "curl error code: $?"
    
    # Delete partially downloaded file
    if [[ -f "$LOCAL_DIR/$CURRENT_FILE" ]]; then
        log_message "Deleting partially downloaded file"
        rm -f "$LOCAL_DIR/$CURRENT_FILE"
    fi
    
    log_message "========================================="
    log_message "TASK $SLURM_ARRAY_TASK_ID FAILED"
    exit 1
fi
```

## Usage Instructions

### 1. Prepare the Script

1. Save the script as `download_ag1000g.sh`
2. Make it executable: `chmod +x download_ag1000g.sh`
3. Create the logs directory: `mkdir -p logs`

### 2. Submit the Job

```bash
sbatch download_ag1000g.sh
```

### 3. Monitor Progress

Check job status:
```bash
squeue -u your_username
```

View logs:
```bash
tail -f logs/download_log_*.log
```

## Key curl Options Explained

| Option | Purpose |
|--------|---------|
| `-f` | Fail silently on server errors |
| `-s` | Silent mode (no progress bar in stdout) |
| `-S` | Show errors even in silent mode |
| `--retry 5` | Retry up to 5 times on failure |
| `--retry-delay 30` | Wait 30 seconds between retries |
| `--connect-timeout 60` | Connection timeout in seconds |
| `--max-time 7200` | Maximum time for the entire operation |
| `--progress-bar` | Show simple progress bar |

## Best Practices

### Error Handling
- Always test FTP connectivity before downloads
- Implement file integrity checks
- Clean up partially downloaded files on failure
- Use appropriate timeouts for different file types

### Logging
- Include timestamps in all log messages
- Log both successes and failures
- Track download duration and file sizes
- Use unique log files per job/task

### Resource Management
- Set appropriate memory and CPU limits
- Use job arrays for parallel downloads
- Implement resume capability to avoid re-downloading

## Troubleshooting

### Common Issues

**Connection Timeouts**
- Increase `--connect-timeout` value
- Check network connectivity
- Verify FTP server availability

**File Corruption**
- Implement post-download verification
- Use `--retry` options for transient errors
- Check available disk space

**Permission Errors**
- Verify write permissions in target directory
- Check SLURM account and partition access
- Ensure proper file ownership

### Log Analysis

Monitor download progress by examining log files:

```bash
# Check overall progress
grep "COMPLETED SUCCESSFULLY" logs/download_log_*.log | wc -l

# Check for failures
grep "FAILED" logs/download_log_*.log

# View download statistics
grep "Duration" logs/download_log_*.log
```

## Conclusion

This approach provides a robust framework for downloading large datasets from FTP servers in HPC environments. The combination of SLURM job arrays, comprehensive error handling, and file verification ensures reliable data acquisition for genomic analyses.

For additional customization, modify the file list, server configuration, and validation logic according to your specific requirements.