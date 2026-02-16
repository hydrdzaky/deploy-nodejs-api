# Troubleshooting Report
# Issue 1 : Docker Daemon Configuration Fix

Docker service was not running, blocking all container operations for example-nodejs-api service.

```
Service Status: ❌ INACTIVE/FAILED
Impact: Cannot start, manage, or access any containers
```
## 2. Investigation Steps

### 2.1 Check Docker Service Status
```bash
sudo systemctl status docker
```

**Output**: Service showed as `failed` or `inactive`

### 2.2 Review Docker Logs
```bash
journalctl -u docker
```

**Finding**: Log showed critical error:
```
unable to configure the Docker daemon with file /etc/docker/daemon.json:
the following directives don't match any configuration options: userns-remap-to
```

---

## 3. Root Cause Identified

### Issue Details
| Component | Finding |
|-----------|---------|
| **Config File** | `/etc/docker/daemon.json` |
| **Invalid Directive** | `userns-remap-to` |
| **Root Cause** | Configuration typo - should be `userns-remap` (not `userns-remap-to`) |
| **Impact** | Docker daemon fails to parse config and won't start |

### Why This Happened
- Previous team attempted to configure user namespace remapping
- Made typo: `userns-remap-to` instead of correct `userns-remap`
- Configuration not validated before deployment
- No backup or version control for daemon.json

---

## 4. Troubleshooting & Resolution

### 4.1 Backup Current Configuration
```bash
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.backup.20260216
```

**Result**: Backup created for future reference

### 4.2 Inspect Current daemon.json
```bash
sudo cat /etc/docker/daemon.json
```

**Output**: Showed `userns-remap-to` directive

### 4.3 Fix Configuration

**Changed from**:
```json
{
  "userns-remap-to": "..."
}
```

**Changed to**:
```json
{
  "userns-remap": "default"
}
```

**Method**: 
```bash
sudo nano /etc/docker/daemon.json
# Edited: userns-remap-to → userns-remap
```

### 4.4 Validate JSON Syntax
```bash
sudo python3 -m json.tool /etc/docker/daemon.json
```

**Result**: ✅ Valid JSON output (no syntax errors)

### 4.5 Reset Systemd Rate Limiter
```bash
sudo systemctl reset-failed docker.service
```

**Reason**: Previous failed restart attempts triggered systemd rate limiting

### 4.6 Start Docker Service
```bash
sudo systemctl start docker
```

**Result**: Service started successfully ✅

### 4.7 Verify Docker is Running
```bash
sudo systemctl status docker
```

**Output**: `● docker.service - Docker Application Container Engine`  
**Status**: `Active: active (running)` ✅

### 4.8 Verify Docker Functionality
```bash
docker ps
docker version
docker info
```

**Result**: All commands executed successfully ✅