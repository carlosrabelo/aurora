# Aurora - Debian Desktop via RDP

Docker container with Debian 13 (Trixie) + Xfce4 + XRDP for remote access via Remote Desktop Protocol.

## Features

- **Base**: Debian 13 (Trixie) - Testing release
- **Desktop Environment**: Xfce4 (with auto-configuration on first login)
- **Remote Access**:
  - XRDP (port 3389) - Remote Desktop Protocol
  - SSH (port 2222) - Secure Shell access
- **Backend**: xorgxrdp (native Xorg)
- **Multiple Users**: Full support (simultaneous sessions)
- **Default User**: aurora (UID 1000)
- **Hostname**: aurora
- **Timezone**: America/Cuiaba
- **Persistence**: All home directories in `/mnt/aurora/home`
- **Healthcheck**: Automatic monitoring of XRDP service
- **Logging**: Structured logs for startup and XRDP events
- **Pre-installed Tools**: git, vim, htop, tmux, tree, and more
- **Custom Profile**: Useful aliases and colorful terminal prompt

## Quick Start

### 1. Configure Environment (Required)

Copy the example configuration and customize:

```bash
cp .env.example .env
```

Then edit `.env` to set your values:

```bash
# Docker Hub Configuration
DOCKER_USER=carlosrabelo
IMAGE_NAME=aurora
VERSION=0.0.1

# User Configuration
USER_PASSWORD=your_secure_password_here

# System Configuration
TZ=America/Cuiaba
USER_NAME=aurora
```

**IMPORTANT**: Never commit the `.env` file to Git!

### 2. Initialize Directories (First time)

```bash
make init CTX=hostname
```

### 3. Build Image

```bash
make build CTX=hostname
```

### 4. Start Container

```bash
make start CTX=hostname
```

### 5. Connect to Aurora

**Option 1: RDP (Graphical Desktop)**

Use an RDP client (such as Remmina, Microsoft Remote Desktop, or rdesktop):

```bash
# Linux
rdesktop -u aurora -p your_password hostname:3389

# Or with Remmina
# Host: hostname:3389
# User: aurora
# Password: the one you defined in .env
```

**Option 2: SSH (Terminal Access)**

Connect via SSH for command-line access:

```bash
# SSH access
ssh -p 2222 aurora@hostname

# Or with SCP to transfer files
scp -P 2222 file.txt aurora@hostname:/home/aurora/
```

## Available Make Commands

| Command | Description |
|---------|-----------|
| `make init` | Initialize directory structure on remote host |
| `make build` | Build Docker image (creates VERSION and latest tags) |
| `make push` | Push images to Docker Hub (VERSION and latest) |
| `make start` | Start container |
| `make stop` | Stop container |
| `make restart` | Restart container |
| `make ps` | List running containers |
| `make logs` | Display Docker Compose logs in real-time |
| `make view-logs` | View Aurora startup and XRDP logs |
| `make sessions` | Show active user sessions |
| `make backup` | Create backup of /mnt/aurora/home |
| `make exec SVC=aurora` | Open shell in container |
| `make config` | Display Docker Compose configuration |
| `make clean` | Remove local Docker images (current version) |
| `make clean-all` | Stop containers and remove all project images |

All commands accept `CTX=<context>` to specify the remote Docker host and can override variables from `.env`.

## Multiple Users

The container works like a normal Debian system - you can add as many users as you need!

### Default User

- **User**: aurora
- **UID**: 1000
- **Password**: Defined via `.env` (`USER_PASSWORD` variable)

### Adding New Users

Like a normal Debian system, use native commands:

**Method 1: Using `adduser` (Recommended - Interactive)**

```bash
# Enter the container
make exec SVC=aurora CTX=hostname

# Add the user (Debian interactive command)
adduser john

# This will:
# 1. Create the user
# 2. Request password
# 3. Create home directory
# 4. Request optional information (full name, etc)

# Add to sudo group (optional)
usermod -aG sudo john

# Configure Xfce4 session
echo "startxfce4" > /home/john/.xsession
```

**Method 2: Using `useradd` (Non-interactive)**

```bash
# Enter the container
make exec SVC=aurora CTX=hostname

# Create the user
useradd -m -s /bin/bash mary

# Set the password
passwd mary

# Add to sudo group (optional)
usermod -aG sudo mary

# Configure Xfce4 session
echo "startxfce4" > /home/mary/.xsession
```

### Connecting with Different Users

Each user can connect simultaneously via RDP:

```bash
# User 1: aurora
rdesktop -u aurora -p aurora_password hostname:3389

# User 2: john (in another session)
rdesktop -u john -p john_password hostname:3389

# User 3: mary (in another session)
rdesktop -u mary -p mary_password hostname:3389
```

### Multiple Users Features

**Each user has:**
- Separate and persistent home directory
- Independent Xfce4 session
- Own configurations
- Sudo permissions (by default)

**Support for:**
- Multiple simultaneous sessions
- Data persisted in `/mnt/aurora/home` volume
- Standard structure (Desktop, Documents, etc)

### Managing Users

Enter the container and use standard Debian commands:

```bash
make exec SVC=aurora CTX=hostname
```

**List users:**
```bash
cat /etc/passwd | grep /home
# or
cut -d: -f1 /etc/passwd
```

**Remove user:**
```bash
deluser --remove-home john  # Debian command
# or
userdel -r john             # Traditional Linux command
```

**Change password:**
```bash
passwd john
```

**View user information:**
```bash
id john
groups john
```

## Security Considerations

### Important Warnings

1. **NOPASSWD sudo**: The `aurora` user can execute sudo commands WITHOUT password
   - **Risk**: If someone gains access to the user, they have full control of the container
   - **Recommendation**: Use only in controlled/development environments
   - **Production**: Remove `NOPASSWD` from line 181 in Dockerfile

2. **Password via Environment Variable**
   - Passwords are defined at runtime via `USER_PASSWORD`
   - **Best practice**: Use Docker secrets in production
   - Never expose `USER_PASSWORD` in logs or repositories

3. **Port 3389 Exposed**
   - RDP is not encrypted by default
   - **Recommendation**: Use VPN or SSH tunnel on untrusted networks
   - Consider using `ports: - "127.0.0.1:3389:3389"` and SSH tunnel

4. **Chromium Browser Sandbox**
   - Chromium runs with `--no-sandbox` flag (required for containers)
   - **Safe**: Docker provides container-level isolation (namespaces, cgroups, seccomp)
   - The browser is still isolated from the host system
   - This is the standard approach for running browsers in containers

### Recommended Security Improvements

For production environments:

1. **Remove NOPASSWD**:
   ```dockerfile
   # Line 181 in Dockerfile
   echo "$USER_NAME ALL=(ALL) ALL" >> /etc/sudoers
   ```

2. **Use Docker Secrets**:
   ```yaml
   secrets:
     - user_password
   ```

3. **SSH Tunnel for RDP**:
   ```bash
   ssh -L 3389:localhost:3389 user@hostname
   # Then connect RDP to localhost:3389
   ```

4. **Firewall/Network Policies**:
   - Restrict access to port 3389 only from trusted IPs

## Volume Structure

```
/mnt/aurora/
    ├── home/                    → /home (in container)
    │   ├── aurora/              # Default user (UID 1000)
    │   │   ├── Desktop/
    │   │   ├── Documents/
    │   │   ├── Downloads/
    │   │   ├── Pictures/
    │   │   └── Videos/
    │   ├── john/                # Additional users
    │   ├── mary/
    │   └── ...
    │
    └── logs/                    → /var/log/aurora (in container)
        ├── startup.log          # Aurora startup logs
        └── xrdp.log             # XRDP server logs
```

**Persisted data on the host:**
- **Home directories**: `/mnt/aurora/home/*` - All user data, configurations, and files
- **Logs**: `/mnt/aurora/logs/*` - Startup and XRDP logs for troubleshooting

## Pre-installed Tools

Aurora comes with essential development and system tools pre-installed:

**Development Tools:**
- git - Version control
- vim - Text editor
- tmux - Terminal multiplexer
- tree - Directory visualization

**System Monitoring:**
- htop - Interactive process viewer
- net-tools - Network utilities
- iputils-ping - Network diagnostics

**Desktop Applications:**
- Chromium - Web browser (optimized for containers)
- Thunar - File manager
- Mousepad - Text editor
- Xfce4 Terminal - Terminal emulator
- Task Manager - System monitor
- Screenshot Tool - Screen capture

**Custom Aliases (available in terminal):**
```bash
ll          # Detailed file listing (ls -lah)
gs          # Git status
gp          # Git pull
gc          # Git commit
gd          # Git diff
update      # System update (apt update && upgrade)
ports       # Show network ports (netstat)
```

## Logging and Monitoring

Aurora implements structured logging for better troubleshooting:

**View startup logs:**
```bash
make view-logs CTX=hostname
```

**Monitor active sessions:**
```bash
make sessions CTX=hostname
```

**View real-time Docker logs:**
```bash
make logs CTX=hostname
```

**Log files (persisted on host at `/mnt/aurora/logs/`):**
- `startup.log` - Startup process logs
- `xrdp.log` - XRDP server logs

**Note**: Logs are persisted on the host, so they survive container restarts and rebuilds.

## Backup and Restore

**Create backup:**
```bash
make backup CTX=hostname
```

This creates a timestamped backup file in `/tmp/aurora-backup-YYYYMMDD-HHMMSS.tar.gz` on the remote host.

The backup includes:
- All user home directories (`/mnt/aurora/home/*`)
- All logs (`/mnt/aurora/logs/*`)

**Restore from backup:**
```bash
# On the remote host
ssh root@hostname
cd /tmp
tar -xzf aurora-backup-20250113-120000.tar.gz -C /
```

**Manual backup of specific directories:**
```bash
# Backup only home directories
ssh root@hostname "tar -czf /tmp/aurora-home-backup.tar.gz /mnt/aurora/home"

# Backup only logs
ssh root@hostname "tar -czf /tmp/aurora-logs-backup.tar.gz /mnt/aurora/logs"
```

## Troubleshooting

### Cannot connect via RDP

1. Check if the container is running:
   ```bash
   make ps CTX=hostname
   ```

2. Check the container health:
   ```bash
   docker ps  # Look for "healthy" status
   ```

3. Check the logs:
   ```bash
   make logs CTX=hostname
   # Or view Aurora-specific logs
   make view-logs CTX=hostname
   ```

4. Test connectivity:
   ```bash
   telnet hostname 3389
   ```

### Cannot connect via SSH

1. Verify SSH port is exposed:
   ```bash
   docker ps | grep aurora
   # Should show 0.0.0.0:2222->22/tcp
   ```

2. Test SSH connectivity:
   ```bash
   telnet hostname 2222
   ```

3. Try connecting with verbose mode:
   ```bash
   ssh -v -p 2222 aurora@hostname
   ```

### Black screen after login

1. Check Aurora logs for errors:
   ```bash
   make view-logs CTX=hostname
   ```

2. Check home directory permissions:
   ```bash
   make exec SVC=aurora CTX=hostname
   ls -la /home/aurora
   ```

3. Verify XFCE4 configuration:
   ```bash
   make exec SVC=aurora CTX=hostname
   cat /home/aurora/.xsession
   # Should contain: startxfce4
   ```

4. Restart the container:
   ```bash
   make restart CTX=hostname
   ```

### "USER_PASSWORD not defined"

Verify that the `.env` file exists and contains:
```bash
USER_PASSWORD=your_password
```

## Environment Variables

All environment variables should be configured in the `.env` file (copy from `.env.example`):

| Variable | Default | Description |
|----------|--------|-----------|
| `DOCKER_USER` | carlosrabelo | Docker Hub username |
| `IMAGE_NAME` | aurora | Docker image name |
| `VERSION` | 0.0.1 | Image version tag |
| `TZ` | America/Cuiaba | System timezone |
| `USER_NAME` | aurora | Default username |
| `USER_PASSWORD` | aurora | RDP password (change this!) |

## Architecture

```
┌─────────────────────────────────────┐
│  RDP Client (Remmina/mstsc)         │
│  SSH Client (Terminal)              │
└──────────┬──────────────┬───────────┘
           │ Port 3389    │ Port 2222
           │ (RDP)        │ (SSH)
┌──────────▼──────────────▼───────────┐
│         Docker Container            │
│  ┌───────────────────────────────┐  │
│  │         XRDP Server           │  │
│  └──────────────┬────────────────┘  │
│  ┌──────────────▼────────────────┐  │
│  │      XRDP Session Manager     │  │
│  └──────────────┬────────────────┘  │
│  ┌──────────────▼────────────────┐  │
│  │      Xorg (xorgxrdp)          │  │
│  └──────────────┬────────────────┘  │
│  ┌──────────────▼────────────────┐  │
│  │     Xfce4 Desktop             │  │
│  └───────────────────────────────┘  │
│  ┌───────────────────────────────┐  │
│  │         SSH Server            │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

## License

MIT License - Use at your own risk.

## Contributing

PRs are welcome! For major changes, please open an issue first.

## Support

For issues or questions, please open an issue in the repository.
