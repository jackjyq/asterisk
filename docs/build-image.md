# Building Asterisk Docker Images with chan_audiosocket Support

This guide explains how to build Asterisk Docker images with the `chan_audiosocket` module enabled.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Method: Template Modification](#method-template-modification)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Overview

The `chan_audiosocket` module provides external media streaming capabilities, allowing audio to be streamed to/from external applications over TCP sockets.

### AudioSocket Module Details

- **Module Name**: `chan_audiosocket`
- **Category**: Channel driver (external media)
- **Asterisk Versions**: Available in Asterisk 12+
- **Dependencies**: None (built into Asterisk source)
- **Use Case**: External media streaming, audio proxying, AI/ML integration

## Prerequisites

Before building, ensure you have:

1. **Docker** installed and running
2. **Git** to clone the repository
3. **Python 3.8+** for build scripts
4. At least **10GB free disk space** for build process

Clone the repository:

```bash
git clone https://github.com/andrius/asterisk.git
cd asterisk
```

## Method: Template Modification

This method integrates `chan_audiosocket` into your build template, ensuring it's always included in future builds.

### Step 1: Edit the Modern Template

Open the modern variant template:

```bash
vim templates/variants/modern.yml.template
```

### Step 2: Add chan_audiosocket to Channels

Locate the `menuselect.channels` section (around line 14-19) and add `chan_audiosocket`:

```yaml
  # Full feature menuselect for modern Asterisk
  menuselect:
    channels:
      - chan_pjsip
      - chan_iax2
      - chan_local
      - chan_bridge_media
      - chan_audiosocket      # <-- ADD THIS LINE
    apps:
      - app_voicemail
      # ... rest of config
```

### Step 3: Add app_audiosocket (Optional)

If you also need the AudioSocket application module, add it to the `apps` section:

```yaml
    apps:
      - app_voicemail
      - app_queue
      # ... other apps
      - app_audiosocket       # <-- ADD THIS LINE (optional)
```

### Step 4: Save and Build

Save the file and build your Asterisk version:

```bash
# Build specific version with --force-config to regenerate from templates
./scripts/build-asterisk.sh 22.8.2 debian amd64 --force-config --push
```

### Step 5: Verify the Build

Check that `chan_audiosocket` is included:

```bash
# Run container and check module
docker run --rm 22.8.2_debian-trixie asterisk -rx "module show like audiosocket"

# Or check if module file exists
docker run --rm 22.8.2_debian-trixie ls -la /usr/lib/asterisk/modules/chan_audiosocket.so
```

## Verification

After building, verify `chan_audiosocket` is properly included:

### Check Module File

```bash
# Verify module file exists in image
docker run --rm asterisk:22.8.2-audiosocket \
  ls -la /usr/lib/asterisk/modules/chan_audiosocket.so
```

### Check Module Loading

```bash
# Start container and check module
docker run -d --name asterisk-test asterisk:22.8.2-audiosocket
docker exec asterisk-test asterisk -rx "module show like audiosocket"
```

Expected output:
```
Module                         Description                              Use Count  Status      Support Level
chan_audiosocket.so            AudioSocket channel driver             0          Running     core
```

## Troubleshooting

### Module Not Found

**Problem**: `chan_audiosocket.so` doesn't exist in the image.

**Solutions**:

1. **Verify template changes**:
   ```bash
   grep -n "chan_audiosocket" templates/variants/modern.yml.template
   ```
   Should show your added line.

2. **Force config regeneration**:
   ```bash
   ./scripts/build-asterisk.sh 22.8.2 --force-config
   ```

3. **Check generated build script**:
   ```bash
   grep -n "chan_audiosocket" asterisk/22.8.2-trixie/build.sh
   ```

### Module Fails to Load

**Problem**: Module exists but won't load.

**Check**:

```bash
# Check for missing dependencies
docker exec asterisk-test ldd /usr/lib/asterisk/modules/chan_audiosocket.so

# Check Asterisk logs
docker exec asterisk-test cat /var/log/asterisk/full | grep audiosocket
```

**Common Fixes**:

1. **Missing dependencies**: Ensure all required libraries are installed in the Docker image.

2. **Version mismatch**: `chan_audiosocket` requires Asterisk 12+. For older versions, it's not available.

3. **Configuration issues**: Check that required config files exist:
   ```bash
   ls -la /etc/asterisk/audiosocket.conf
   ```

### Build Fails

**Problem**: Docker build fails when adding `chan_audiosocket`.

**Solutions**:

1. **Check Asterisk version compatibility**:
   - `chan_audiosocket` is available in Asterisk 12+
   - Not available in legacy versions (1.2-1.8)

2. **Verify menuselect syntax**:
   ```bash
   # In generated build.sh, check the exact command:
   menuselect/menuselect --enable chan_audiosocket menuselect.makeopts
   ```

3. **Check for conflicts**:
   Some modules may conflict. Check if `chan_audiosocket` is in both enable and disable lists:
   ```bash
   grep "chan_audiosocket" asterisk/22.8.2-trixie/build.sh
   ```

---

**Last Updated**: 2025-02-28  
**Asterisk Version Support**: 12+  
**Docker Image**: andrius/asterisk
