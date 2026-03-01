# Building Asterisk Docker Images with chan_audiosocket Support

This guide explains how to build Asterisk Docker images with the `chan_audiosocket` module enabled. AudioSocket is an Asterisk module that provides external media streaming capabilities, allowing audio to be streamed to/from external applications over TCP sockets.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Method 1: Template Modification (Recommended)](#method-1-template-modification-recommended)
- [Method 2: Build with Custom Menuselect Commands](#method-2-build-with-custom-menuselect-commands)
- [Method 3: Post-Build Module Enablement](#method-3-post-build-module-enablement)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Overview

The `chan_audiosocket` module is categorized as an **optional** channel module in this build system (`lib/menuselect.py`, line 55). It's not enabled by default in standard builds but can be easily added through several methods.

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

## Method 1: Template Modification (Recommended)

This is the cleanest method that integrates `chan_audiosocket` into your build template, ensuring it's always included in future builds.

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

### Step 3:  Add app_audiosocket

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
./scripts/build-asterisk.sh 22.8.2 --force-config

# Or with custom distribution
./scripts/build-asterisk.sh 22.8.2 debian amd64 --force-config --load

# Build AND push to registry in one step (using --push flag)
./scripts/build-asterisk.sh 22.8.2 --push

# Build with additional tags and push
./scripts/build-asterisk.sh 22.8.2 --push --tag latest --tag 22

# Build for multiple architectures and push
./scripts/build-asterisk.sh 22.8.2 --push --platform linux/amd64,linux/arm64
```

**Prerequisites for pushing with `--push`:**
1. Login to your container registry first:
   ```bash
   docker login ghcr.io -u jackjyq
   ```
   (Use your GitHub Personal Access Token as the password)

2. The image will be automatically tagged and pushed to `ghcr.io/jackjyq/asterisk`

**How the `--push` flag works:**
- Automatically tags the image with the correct registry prefix (`ghcr.io/jackjyq/asterisk`)
- Creates multi-architecture manifests (when using multiple platforms)
- Pushes all tags to the registry in one step
- No manual `docker tag` and `docker push` commands needed

### Step 5: Verify the Build

Check that `chan_audiosocket` is included:

```bash
# Run container and check module
docker run --rm 22.8.2_debian-trixie asterisk -rx "module show like audiosocket"

# Or check if module file exists
docker run --rm 22.8.2_debian-trixie ls -la /usr/lib/asterisk/modules/chan_audiosocket.so
```

## Method 2: Build with Custom Menuselect Commands

This method allows you to enable `chan_audiosocket` without modifying templates, useful for one-off builds.

### Step 1: Generate Base Configuration

First, generate the configuration for your version:

```bash
./scripts/build-asterisk.sh 22.8.2 --force-config --dry-run
```

This creates build files in `asterisk/22.8.2-trixie/` without actually building.

### Step 2: Edit the Build Script

Locate and edit the generated build script:

```bash
vim asterisk/22.8.2-trixie/build.sh
```

### Step 3: Add Menuselect Command for chan_audiosocket

Find the menuselect section (usually around lines 70-90) and add a line to enable `chan_audiosocket`:

```bash
# Look for existing menuselect commands like:
menuselect/menuselect --enable chan_pjsip menuselect.makeopts

# Add after the existing channel enables:
menuselect/menuselect --enable chan_audiosocket menuselect.makeopts
```

The section should look something like this after editing:

```bash
# Enable required channel drivers
menuselect/menuselect --enable chan_pjsip menuselect.makeopts
menuselect/menuselect --enable chan_iax2 menuselect.makeopts
menuselect/menuselect --enable chan_local menuselect.makeopts
menuselect/menuselect --enable chan_audiosocket menuselect.makeopts  # <-- ADD THIS
```

### Step 4: Build the Image

Now run the build using the modified script:

```bash
# Navigate to the version directory
cd asterisk/22.8.2-trixie

# Build the Docker image
docker build -t asterisk:22.8.2-audiosocket .

# Or use the build script directly
./build.sh
```

### Step 5: Verify

Test that the module is available:

```bash
docker run --rm asterisk:22.8.2-audiosocket asterisk -rx "module show like audiosocket"
```

## Method 3: Post-Build Module Enablement

If you already have a running Asterisk container and want to enable `chan_audiosocket` without rebuilding, you can compile and load the module separately.

**Note**: This method requires the Asterisk development headers and build tools inside the container.

### Step 1: Start Container with Build Tools

Run the container with additional packages:

```bash
docker run -it --name asterisk-build \
  -v $(pwd)/asterisk-src:/usr/src/asterisk \
  andrius/asterisk:22.8.2 /bin/bash

# Inside container, install build dependencies
apt-get update
apt-get install -y build-essential libssl-dev
```

### Step 2: Download Asterisk Source

Inside the container:

```bash
cd /usr/src
wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-22.8.2.tar.gz
tar xzf asterisk-22.8.2.tar.gz
cd asterisk-22.8.2
```

### Step 3: Build Only chan_audiosocket

```bash
# Configure Asterisk (minimal)
./configure --disable-all --enable chan_audiosocket

# Build only the module
make channels/chan_audiosocket.so
```

### Step 4: Install and Load Module

```bash
# Copy to Asterisk modules directory
cp channels/chan_audiosocket.so /usr/lib/asterisk/modules/

# Load the module
asterisk -rx "module load chan_audiosocket"

# Verify
asterisk -rx "module show like audiosocket"
```

**Note**: This method is not persistent. The module will be lost when the container is removed. Use Method 1 or 2 for permanent solutions.

## Verification

After building with any method, verify `chan_audiosocket` is properly included:

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

### Test AudioSocket Dialplan

Create a test dialplan to verify functionality:

```bash
docker exec -it asterisk-test /bin/bash
```

Add to `/etc/asterisk/extensions.conf`:

```ini
[audiosocket-test]
exten => 100,1,NoOp(AudioSocket Test)
 same => n,AudioSocket(127.0.0.1:8080)
 same => n,Hangup()
```

Reload and test:

```bash
asterisk -rx "dialplan reload"
asterisk -rx "core show application audiosocket"
```

## Pushing to Docker Registry

After building and verifying your image, you may want to push it to a Docker registry for distribution.

### Option 1: Build and Push in One Step (Recommended)

The build script supports building and pushing in a single command using the `--push` flag. This is the most efficient method.

```bash
# Build and push to your registry (ghcr.io/jackjyq/asterisk)
./scripts/build-asterisk.sh 22.8.2 --push

# Build with additional tags and push
./scripts/build-asterisk.sh 22.8.2 --push --tag latest --tag 22

# Build for multiple architectures and push
./scripts/build-asterisk.sh 22.8.2 --push --platform linux/amd64,linux/arm64
```

**Prerequisites for pushing:**
1. Login to your container registry first:
   ```bash
   docker login ghcr.io -u jackjyq
   ```
   (Use your GitHub Personal Access Token as the password)

2. The image will be tagged and pushed automatically to `ghcr.io/jackjyq/asterisk`

**How it works:**
The `--push` flag automatically:
1. Tags the image with the correct registry prefix (`ghcr.io/jackjyq/asterisk`)
2. Creates multi-architecture manifests (when using multiple platforms)
3. Pushes all tags to the registry

**Example output:**
```
# Building 22.8.2 with push
./scripts/build-asterisk.sh 22.8.2 --push

# The image will be tagged as:
# - ghcr.io/jackjyq/asterisk:22.8.2_debian-trixie
# And pushed to your registry
```

### Option 2: Manual Tag and Push

If you prefer to build first and push later, follow these steps:

#### Tag the Image

Before pushing, ensure your image is properly tagged with the registry URL:

```bash
# Tag for Docker Hub (default)
docker tag asterisk:22.8.2-audiosocket yourusername/asterisk:22.8.2-audiosocket

# Tag for GitHub Container Registry
docker tag asterisk:22.8.2-audiosocket ghcr.io/yourusername/asterisk:22.8.2-audiosocket

# Tag for other registries
docker tag asterisk:22.8.2-audiosocket registry.example.com/asterisk:22.8.2-audiosocket
```

### Login to Registry

Authenticate with your Docker registry:

```bash
# Docker Hub
docker login

# GitHub Container Registry (using Personal Access Token)
docker login ghcr.io -u USERNAME

# Other registries
docker login registry.example.com
```

### Push the Image

Upload your image to the registry:

```bash
# Push to Docker Hub
docker push yourusername/asterisk:22.8.2-audiosocket

# Push to GitHub Container Registry
docker push ghcr.io/yourusername/asterisk:22.8.2-audiosocket

# Push to other registries
docker push registry.example.com/asterisk:22.8.2-audiosocket
```

### Multi-Architecture Push (Optional)

For multi-architecture support, use buildx:

```bash
# Create a buildx builder (if not exists)
docker buildx create --use

# Build and push multi-arch image
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t yourusername/asterisk:22.8.2-audiosocket \
  --push .
```

### Verify the Push

Confirm your image is available in the registry:

```bash
# Pull the image back
docker pull yourusername/asterisk:22.8.2-audiosocket

# Verify audiosocket module
docker run --rm yourusername/asterisk:22.8.2-audiosocket \
  asterisk -rx "module show like audiosocket"
```

---

## Troubleshooting

### Module Not Found

**Problem**: `chan_audiosocket.so` doesn't exist in the image.

**Solutions**:

1. **Verify template changes** (Method 1):
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

## Summary

Three methods to enable `chan_audiosocket`:

| Method | Best For | Persistence | Complexity |
|--------|----------|-------------|------------|
| **1. Template Modification** | Production builds, multiple versions | Permanent | Low |
| **2. Custom Menuselect** | One-off builds, testing | Permanent | Medium |
| **3. Post-Build Enablement** | Development, debugging | Temporary | High |

**Recommendation**: Use **Method 1 (Template Modification)** for production deployments. It ensures `chan_audiosocket` is always included in future builds and maintains consistency across your infrastructure.

For testing or single builds, **Method 2** offers a good balance between simplicity and persistence.

## Additional Resources

- [Asterisk AudioSocket Documentation](https://docs.asterisk.org/Configuration/Channel-Drivers/AudioSocket/)
- [AudioSocket Application Documentation](https://docs.asterisk.org/Configuration/Applications/App_AudioSocket/)
- [GitHub Repository](https://github.com/andrius/asterisk)

---

**Last Updated**: 2025-02-28  
**Asterisk Version Support**: 12+  
**Docker Image**: andrius/asterisk
