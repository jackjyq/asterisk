# Hello World with Docker Asterisk

This guide walks you through the classic Asterisk "Hello World" example using your Docker-based Asterisk image with chan_audiosocket support.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Step 1: Project Setup](#step-1-project-setup)
- [Step 2: Start Asterisk with Docker Compose](#step-2-start-asterisk-with-docker-compose)
- [Step 3: Configure Your SIP Phone](#step-3-configure-your-sip-phone)
- [Step 4: Make the Call](#step-4-make-the-call)
- [Troubleshooting](#troubleshooting)
  - [WSL2 + Docker + Windows Softphone](#wsl2--docker--windows-softphone-zoipermicrosip)
  - [Registration Failed](#registration-failed)
  - [Call Fails or No Audio](#call-fails-or-no-audio)
  - [Audio File Not Found](#audio-file-not-found)
  - [Container Won't Start](#container-wont-start)
- [Next Steps](#next-steps)
- [Project Structure](#project-structure)
- [Reference](#reference)

## Overview

This tutorial will:
1. Set up a basic Asterisk dialplan that answers calls and plays "hello-world"
2. Configure a SIP peer (extension 6001) using chan_pjsip
3. Connect a softphone to place a test call

## Prerequisites

Before starting, ensure you have:

1. **Docker** and **Docker Compose** installed and running
2. Your **Asterisk Docker image** built with chan_audiosocket support (see [build-image.md](./build-image.md)):
   - Image: `ghcr.io/jackjyq/asterisk:22.8.2_debian-trixie`
3. A **SIP softphone** installed (we recommend [Zoiper](http://www.zoiper.com/) or [MicroSIP](https://www.microsip.org/))
4. Both your computer and softphone on the **same network**

## Quick Start

If you already have Docker Compose and just want to get started:

```bash
# 1. Navigate to the hello-world example directory
cd /path/to/asterisk/examples/hello-world

# 2. Start the services
docker compose up -d

# 3. Verify Asterisk is running
docker compose exec asterisk-hello asterisk -rx "core show uptime"

# 4. Configure your SIP phone (see Step 3)

# 5. Dial 100 from your softphone to hear "Hello, world!"
```

## Step 1: Project Setup

The hello-world example is located in `examples/hello-world/` and contains:

```
examples/hello-world/
├── docker-compose.yml          # Docker Compose configuration
└── config/
    ├── extensions.conf         # Dialplan configuration
    ├── pjsip.conf             # SIP channel driver configuration
    └── modules.conf           # Module loading configuration
```

All configuration files are ready to use. You can customize them if needed, but they work out of the box for the Hello World example.

## Step 2: Start Asterisk with Docker Compose

Navigate to the hello-world directory and start the services:

```bash
cd examples/hello-world
docker compose up -d
```

This will:
1. Pull the Docker image `ghcr.io/jackjyq/asterisk:22.8.2_debian-trixie` (if not already present)
2. Create a Docker network for the Asterisk service
3. Start the Asterisk container with all necessary port mappings
4. Mount the configuration files from the `config/` directory

### Verify Container is Running

```bash
# Check container status
docker compose ps

# Check Asterisk logs
docker compose logs -f

# Connect to Asterisk CLI
docker compose exec asterisk asterisk -rvvvvv
```

### Verify Configuration Loaded

In the Asterisk CLI, check:

```
; Verify pjsip is loaded
asterisk*CLI> pjsip show endpoints

; You should see endpoint 6001 listed

; Verify dialplan
asterisk*CLI> dialplan show from-internal

; You should see extension 100
```

Exit the CLI with `Ctrl+C` or `exit`.

### Comprehensive Verification Checklist

Run these commands to verify your Asterisk installation is fully operational:

```bash
# 1. Verify Container Status
docker ps --filter "name=asterisk" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
# Expected: asterisk-hello | Up X minutes (healthy)

# 2. Verify Asterisk Core
docker exec asterisk-hello asterisk -rx "core show version"
# Expected: Asterisk 22.8.2 built by root @ buildkitsandbox...

docker exec asterisk-hello asterisk -rx "core show uptime"
# Expected: System uptime: X minutes, Y seconds

# 3. Verify AudioSocket Modules (Key Feature)
docker exec asterisk-hello asterisk -rx "module show like audiosocket"
# Expected: 3 modules - app_audiosocket.so, chan_audiosocket.so, res_audiosocket.so

# 4. Verify PJSIP Endpoint
docker exec asterisk-hello asterisk -rx "pjsip show endpoints"
# Expected: Endpoint 6001 listed with state "Unavailable" (waiting for registration)

# 5. Verify Dialplan
docker exec asterisk-hello asterisk -rx "dialplan show from-internal"
# Expected: Extension 100 with Answer, Wait(1), Playback(hello-world), Hangup

# 6. Check Port Bindings
docker port asterisk-hello
# Expected: Port mappings for 5060/udp/tcp and 10000-10100/udp (if configured)
```

**Expected Results Summary:**

| Component | Status | Details |
|-----------|--------|---------|
| Container | ✅ HEALTHY | Up X minutes |
| Asterisk Core | ✅ RUNNING | Version 22.8.2 |
| AudioSocket | ✅ LOADED | 3/3 modules active |
| PJSIP Endpoint | ✅ READY | Extension 6001 configured |
| Dialplan | ✅ READY | Extension 100 (hello-world) |

If all checks pass, your Asterisk container is ready for testing with a SIP softphone!

## Step 3: Configure Your SIP Phone

### Option A: Zoiper (Recommended for beginners)

1. **Download and install Zoiper** from [zoiper.com](http://www.zoiper.com/)

2. **Open Zoiper** and click the **wrench icon** (Settings)

3. **Add a new account**:
   - Click "Add new SIP account"
   - Account name: `6001`
   - Click OK

4. **Configure SIP settings**:
   - **Domain**: Your Asterisk server IP address (e.g., `127.0.0.1`)
   - **Username**: `6001`
   - **Password**: `unsecurepassword`
   - **Caller ID Name**: (optional) `Test User`
   - Click OK

5. **Register the account**:
   - From the main Zoiper window, select account `6001`
   - Click **Register**
   - Status should show "Registered"

### Option B: MicroSIP (Lightweight Windows option)

1. **Download MicroSIP** from [microsip.org](https://www.microsip.org/)

2. **Configure account**:
   - Account: `6001`
   - Domain: Your Asterisk IP address
   - Password: `unsecurepassword`
   - Click Save

### Finding Your Asterisk IP Address

```bash
# On the Asterisk host
ip addr show

# Or from the container
docker compose exec asterisk ip addr show
```

## Step 4: Make the Call

1. **Open your SIP phone** (Zoiper/MicroSIP)

2. **Ensure the account is registered** (status shows "Registered")

3. **Dial extension 100**

4. **You should hear**: "Hello, world!" followed by silence, then the call hangs up

### On the Asterisk CLI

If you have the CLI open, you'll see logs like:

```
  == Using SIP RTP CoS mark 5
    -- Executing [100@from-internal:1] Answer("PJSIP/6001-00000001", "") in new stack
    -- Executing [100@from-internal:2] Wait("PJSIP/6001-00000001", "1") in new stack
    -- Executing [100@from-internal:3] Playback("PJSIP/6001-00000001", "hello-world") in new stack
    -- <PJSIP/6001-00000001> Playing 'hello-world.slin' (language 'en')
    -- Executing [100@from-internal:4] Hangup("PJSIP/6001-00000001", "") in new stack
  == Spawn extension (from-internal, 100, 4) exited non-zero on 'PJSIP/6001-00000001'
```

## Troubleshooting

### WSL2 + Docker + Windows Softphone (Zoiper/MicroSIP)

**Problem**: Zoiper or other softphone on Windows host cannot connect to Asterisk running in Docker inside WSL2.

**Root Cause**: WSL2 uses a virtual network adapter (e.g., 192.168.10.10) that requires explicit port proxy configuration for UDP ports to be accessible from Windows host. The automatic WSL2 port forwarding does not work for all protocols by default.

**Current Status**:
- ✅ Asterisk is running in Docker (host network mode)
- ✅ Asterisk is listening on 0.0.0.0:5060 (UDP) inside WSL2
- ✅ WSL2 can reach 192.168.10.10:5060 from inside
- ❌ Windows CANNOT reach 192.168.10.10:5060 from outside

**Solutions**:

#### Solution 1: Configure WSL2 Port Proxy (Required for WSL2)

WSL2 requires explicit port proxy configuration to forward UDP traffic from Windows to WSL2.

**Step 1: Configure .wslconfig (Optional but Recommended)**

Create or edit `C:\Users\<YourUsername>\.wslconfig`:

```ini
[wsl2]
hostAddressLoopback=true
```

Then restart WSL2:
```cmd
wsl --shutdown
# Then reopen WSL2
```

**Step 2: Configure Port Proxy (REQUIRED)**

Run these commands in **PowerShell as Administrator**:

```powershell
# Add UDP 5060 proxy (SIP signaling)
netsh interface portproxy add v4tov4 listenport=5060 listenaddress=0.0.0.0 connectport=5060 connectaddress=192.168.10.10 protocol=udp

# Add UDP 10000-10009 proxy (RTP audio range)
netsh interface portproxy add v4tov4 listenport=10000 listenaddress=0.0.0.0 connectport=10000 connectaddress=192.168.10.10 protocol=udp
netsh interface portproxy add v4tov4 listenport=10001 listenaddress=0.0.0.0 connectport=10001 connectaddress=192.168.10.10 protocol=udp
netsh interface portproxy add v4tov4 listenport=10002 listenaddress=0.0.0.0 connectport=10002 connectaddress=192.168.10.10 protocol=udp
netsh interface portproxy add v4tov4 listenport=10003 listenaddress=0.0.0.0 connectport=10003 connectaddress=192.168.10.10 protocol=udp
netsh interface portproxy add v4tov4 listenport=10004 listenaddress=0.0.0.0 connectport=10004 connectaddress=192.168.10.10 protocol=udp
netsh interface portproxy add v4tov4 listenport=10005 listenaddress=0.0.0.0 connectport=10005 connectaddress=192.168.10.10 protocol=udp
netsh interface portproxy add v4tov4 listenport=10006 listenaddress=0.0.0.0 connectport=10006 connectaddress=192.168.10.10 protocol=udp
netsh interface portproxy add v4tov4 listenport=10007 listenaddress=0.0.0.0 connectport=10007 connectaddress=192.168.10.10 protocol=udp
netsh interface portproxy add v4tov4 listenport=10008 listenaddress=0.0.0.0 connectport=10008 connectaddress=192.168.10.10 protocol=udp
netsh interface portproxy add v4tov4 listenport=10009 listenaddress=0.0.0.0 connectport=10009 connectaddress=192.168.10.10 protocol=udp
```

Verify port proxy rules:
```powershell
netsh interface portproxy show all
```

Expected output:
```
Listen on addr:port        Connect to addr:port
---------------------------  ------------------------
0.0.0.0:5060              192.168.10.10:5060
0.0.0.0:10000             192.168.10.10:10000
...
```

**Step 3: Add Windows Firewall Rules**

Run in **PowerShell as Administrator**:

```powershell
# Allow SIP signaling (UDP)
netsh advfirewall firewall add rule name="Asterisk SIP UDP" dir=in action=allow protocol=udp localport=5060

# Allow RTP audio
netsh advfirewall firewall add rule name="Asterisk RTP" dir=in action=allow protocol=udp localport=10000-10009
```

**Step 4: Test Connectivity from Windows**

Before trying Zoiper, verify Windows can reach Asterisk server.

From **Windows Command Prompt** (not WSL2):

```cmd
:: Test TCP connectivity
telnet 192.168.10.10 5060

:: Or using PowerShell
powershell -Command "Test-NetConnection -ComputerName 192.168.10.10 -Port 5060"
```

Note: Test-NetConnection tests TCP only. For UDP testing, use Zoiper directly after configuring port proxy.

**Step 5: Configure Zoiper**
   ```

3. In Zoiper, use your Windows IP instead of 192.168.10.10

#### WSL2 Zoiper Configuration Summary

Once you have connectivity working, use these settings in Zoiper:

| Setting | Value |
|---------|-------|
| Domain | 192.168.10.10 (your WSL2 IP) |
| Username | 6001 |
| Password | unsecurepassword |
| Transport | UDP |
| Port | 5060 |

**Important Notes**:

1. **Port proxy is one-way (Windows → WSL2)**: You need proxy rules only in this direction
2. **Port proxy resets after Windows reboot**: You may need to recreate rules after reboot
3. **Protocol matters**: Specify `protocol=udp` for UDP ports in port proxy commands
4. **Address changes**: If WSL2 IP changes, update `connectaddress` in all port proxy rules

**Troubleshooting WSL2 Port Proxy**:

If port proxy doesn't work:
1. Verify rules exist: `netsh interface portproxy show all`
2. Check if port is already in use: `netstat -an | findstr 5060`
3. Restart Windows Networking Service: `netsh winsock reset` (requires reboot)

If connection still fails:
1. Check Windows Firewall: `netsh advfirewall show currentprofile`
2. Try localhost instead of IP: Use `127.0.0.1` in Zoiper
3. Check WSL2 is running: `wsl --status`

### Registration Failed

**Problem**: Softphone shows "Registration Failed"

**Solutions**:

1. **Check network connectivity**:
   ```bash
   # From your computer, test if Asterisk is reachable
   ping <asterisk-ip>
   
   # Test SIP port
   nc -vu <asterisk-ip> 5060
   ```

2. **Verify Asterisk is listening**:
   ```bash
   docker compose exec asterisk netstat -tulpn | grep 5060
   ```

3. **Check credentials**:
   - Username: `6001`
   - Password: `unsecurepassword`
   - Domain: Asterisk IP address

4. **Check PJSIP configuration**:
```bash
# Check Asterisk is running
docker compose exec asterisk asterisk -rx "core show uptime"
```

**Expected output:**

```
Unable to open specified master config file '/etc/asterisk/asterisk.conf', using built-in defaults
No ethernet interface found for seeding global EID. You will have to set it manually.
System uptime: 4 minutes, 48 seconds
Last reload: 4 minutes, 48 seconds
```

**Note:** The warnings about missing configuration files and ethernet interface are expected for a minimal test container and don't affect Asterisk functionality.

### Call Fails or No Audio

**Problem**: Phone registers but call fails or no audio

**Solutions**:

1. **Check dialplan**:
   ```bash
   docker compose exec asterisk asterisk -rx "dialplan show from-internal"
   ```

2. **Verify audio file exists**:
   ```bash
   docker compose exec asterisk ls -la /usr/share/asterisk/sounds/en/hello-world.*
   ```

3. **Check RTP port range** (for audio):
   - Ensure RTP ports 10000-10100 are mapped:
   ```bash
   docker compose ps
   ```

4. **Check firewall** on host:
   ```bash
   sudo iptables -L | grep 5060
   sudo iptables -L | grep 10000
   ```

### Audio File Not Found

**Problem**: Error "File hello-world does not exist"

**Solution**:

```bash
# Check what sound files are available
docker compose exec asterisk ls /usr/share/asterisk/sounds/en/

# If missing, install core sounds
docker compose exec asterisk bash
# Inside container:
# apt-get update && apt-get install -y asterisk-core-sounds-en
```

### Container Won't Start

**Problem**: Container exits immediately

**Solutions**:

```bash
# Check logs
docker compose logs

# Check if ports are already in use
sudo netstat -tulpn | grep 5060

# Stop and remove existing containers, then start fresh
docker compose down
docker compose up -d
```

## Next Steps

Now that you have a working "Hello World" setup, you can:

1. **Explore the dialplan** - Try adding more extensions with different applications
2. **Add more SIP peers** - Configure additional extensions in pjsip.conf
3. **Experiment with AudioSocket** - Since your image includes chan_audiosocket, try integrating external applications
4. **Read the documentation** - Visit [Asterisk Documentation](https://docs.asterisk.org/) for more advanced topics

## Project Structure

The hello-world example is located at `examples/hello-world/`:

```
examples/hello-world/
├── docker-compose.yml          # Docker Compose configuration
└── config/
    ├── extensions.conf         # Dialplan configuration
    ├── pjsip.conf             # SIP channel driver configuration
    └── modules.conf           # Module loading configuration
```

## Reference

### Key Files

| File | Purpose |
|------|---------|
| `extensions.conf` | Dialplan - defines call flow |
| `pjsip.conf` | SIP channel driver configuration |
| `modules.conf` | Module loading configuration |

### Default Credentials

| Setting | Value |
|---------|-------|
| SIP Peer | 6001 |
| Password | unsecurepassword |
| Dialplan Context | from-internal |
| Test Extension | 100 |

### Docker Ports

| Port | Protocol | Purpose |
|------|----------|---------|
| 5060 | UDP/TCP | SIP signaling |
| 10000-10100 | UDP | RTP (audio) |

---

**Related Documentation**: [build-image.md](./build-image.md) - How to build the Asterisk Docker image used in this guide.