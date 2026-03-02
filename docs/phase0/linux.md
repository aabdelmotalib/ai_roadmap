# Linux & Networking for AI Engineers

Linux is where your code runs. Understanding it means the difference between "it works on my machine" and "it works in production."

---

## 1. Why This Matters in Production

An AI startup deployed their model server on EC2. One month later, the server filled with core dumps from crashed training processes. No one was monitoring disk space.

Result: Service went down. They lost data. Recovery took 8 hours.

Cost: $50k per hour in lost business = $400k total loss.

**Production reality:**
- Logs grow unbounded. Disks fill. Services crash.
- A single hung process can freeze your entire server.
- A misconfigured firewall can make your service invisible.
- SSH key management is how you actually log in to servers.
- systemd is how production services are managed (not `python server.py` in a terminal).

---

## 2. Conceptual Explanation: The Unix Philosophy

Unix (Linux, macOS) is built on one principle: **Do one thing, and do it well.**

Each tool has a single job:
- `ls` — list files
- `grep` — find patterns
- `curl` — transfer data
- `sed` — text transformation

You **compose** them together:

```bash
# Find all Python files modified in the last hour, count them
find . -name "*.py" -mmin -60 | wc -l

# Find hung processes, kill them
ps aux | grep python | grep -v grep | awk '{print $2}' | xargs kill -9
```

This composability is your superpower. Master it, and you can write one command to solve problems that would take pages of code.

---

## 3. Code Examples: Essential Commands

### 3.1 File System Navigation

```bash
# Where am I?
pwd
# Output: /home/user/ai_project

# List files with details
ls -lah
# -l: long format (permissions, owner, size, date)
# -a: all (including hidden)
# -h: human-readable (KB/MB instead of bytes)

# Find files by name (searches everything under current dir)
find . -name "*.py" -type f
# Find Python files modified in the last day
find . -name "*.py" -type f -mtime -1

# Find large files (over 100MB)
find . -type f -size +100M
# Get disk usage per directory
du -sh *  # Human-readable, summarize

# Total disk usage
df -h  # Remaining space

# Count lines of code
find src/ -name "*.py" | xargs wc -l
```

### 3.2 Process Management

```bash
# List all processes
ps aux
# USER, PID, CPU%, MEM%, VSZ, RSS, TTY, STAT, START, TIME, COMMAND

# Find a specific process (don't grep grep)
ps aux | grep python | grep -v grep

# Interactive process viewer (like Task Manager)
top  # Shows real-time CPU, memory, process list
# Press 'q' to quit

# Kill a process (graceful)
kill 12345  # Process exits cleanly

# Force-kill a process (immediately)
kill -9 12345  # Process dies instantly (dangerous)

# Run a command that survives terminal close (background)
nohup python train.py > training.log 2>&1 &
# nohup: survives terminal close
# &: run in background
# >training.log: stdout to file
# 2>&1: stderr to same file

# List background jobs
jobs

# Bring job to foreground
fg %1  # Job number 1

# Process priority (nice = politeness to other processes)
nice -n 10 python train.py  # Lower priority (10)
nice -n -10 python train.py  # Higher priority (-10)

# Change running process priority
renice +10 -p 12345  # Make PID 12345 lower priority

# Monitor processes in real-time
watch -n 1 'ps aux | head -20'  # Update every 1 second
```

### 3.3 Systemd: Running Services

```bash
# Service that starts at boot and restarts on crash
# File: /etc/systemd/system/ai-api.service
[Unit]
Description=AI Backend API
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=notify
User=ai_user
WorkingDirectory=/home/ai_user/ai_backend
Environment="PATH=/home/ai_user/.local/bin:/usr/local/bin:/usr/bin"
EnvironmentFile=/home/ai_user/.env
ExecStart=/home/ai_user/.local/bin/poetry run uvicorn src.main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target

# Commands
sudo systemctl daemon-reload  # Load new service files
sudo systemctl start ai-api  # Start service
sudo systemctl enable ai-api  # Start at boot
sudo systemctl status ai-api  # Check status
sudo systemctl logs -f ai-api  # View logs (tail)
sudo systemctl restart ai-api  # Restart service
sudo systemctl stop ai-api  # Stop service
```

### 3.4 SSH: Remote Access

```bash
# Generate SSH key (one-time)
ssh-keygen -t ed25519 -C "your.email@example.com"
# Creates: ~/.ssh/id_ed25519 (private key) + ~/.ssh/id_ed25519.pub (public key)

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server.example.com
# Or manually append to ~/.ssh/authorized_keys on server

# Connect to server
ssh user@server.example.com

# Config file for shortcuts (~/.ssh/config)
Host ai-prod
    HostName server.example.com
    User deploy_user
    IdentityFile ~/.ssh/id_ed25519
    Port 22

# Now just: ssh ai-prod

# Copy file from server to local
scp user@server.example.com:/path/to/file ~/local/path

# Copy directory recursively
scp -r user@server.example.com:/path/to/dir ~/local/path

# Sync entire directories (rsync is better)
rsync -azP user@server.example.com:/remote/models/ ./models/
# -a: archive (preserve permissions)
# -z: compress during transfer
# -P: progress

# SSH tunneling (forward port from server)
ssh -L 5432:localhost:5432 user@server.example.com
# Now: localhost:5432 → server's PostgreSQL

# Keep connection alive (for long-running jobs)
ssh -o ServerAliveInterval=300 user@server.example.com  # Send keepalive every 5 min
```

### 3.5 Environment Variables & Configuration

```bash
# Set variable (session only)
export API_KEY="sk-1234567890"
echo $API_KEY

# Read from .env file
set -a
source .env.example
set +a
# Now all variables in .env are available

# Check if variable is set
[ -z "$API_KEY" ] && echo "Variable not set"

# Use in scripts
if [ -z "$OPENAI_API_KEY" ]; then
    echo "ERROR: OPENAI_API_KEY is not set"
    exit 1
fi

# Default fallback
ENVIRONMENT=${ENV:-development}  # Use ENV, default to development
echo "Running in: $ENVIRONMENT"
```

### 3.6 curl/httpie: Testing APIs

```bash
# GET request (curl)
curl https://api.example.com/data

# GET with header (API key)
curl -H "X-API-Key: secret123" https://api.example.com/data

# POST with JSON (curl)
curl -X POST https://api.example.com/documents \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","content":"..."}'

# httpie (much cleaner)
http GET https://api.example.com/data
http POST https://api.example.com/documents title="Test" content="..."
http -b POST https://api.example.com/documents < data.json

# Save response to file
curl https://api.example.com/data > response.json

# Follow redirects
curl -L https://api.example.com/data

# Show response headers
curl -i https://api.example.com/data  # Includes headers + body
curl -I https://api.example.com/data  # Headers only

# Bearer token
curl -H "Authorization: Bearer YOUR_TOKEN" https://api.example.com/protected
```

### 3.7 Networking: Ports & Firewall

```bash
# Check if a port is open
nc -zv localhost 8000  # Connection successful if open

# List all listening ports
netstat -tlnp | grep LISTEN
# or (newer)
ss -tlnp | grep LISTEN

# Check specific service
lsof -iTCP:8000 -sTCP:LISTEN

# Open firewall port (Ubuntu)
sudo ufw allow 8000/tcp
sudo ufw enable

# Check firewall status
sudo ufw status

# Test connectivity to server
ping server.example.com
# Shows if server is reachable and latency

# Trace route to server
traceroute server.example.com
# Shows all hops between you and server

# Test DNS resolution
nslookup server.example.com
# or
dig server.example.com
```

### 3.8 Logs: Finding & Monitoring

```bash
# View application logs (systemd)
sudo journalctl -u ai-api -f
# -f: follow (like tail)

# Last 100 lines
sudo journalctl -u ai-api -n 100

# Logs from last hour
sudo journalctl -u ai-api --since "1 hour ago"

# Errors only
sudo journalctl -u ai-api -p err

# Search for pattern
grep "error" /var/log/application.log

# Real-time log monitoring
tail -f /var/log/application.log

# Monitor multiple logs
tail -f /var/log/syslog /var/log/application.log

# Count occurrences of error
grep -c "ERROR" application.log

# Find errors in last 24 hours
find /var/log -name "*.log" -mtime -1 -exec grep "ERROR" {} +
```

### 3.9 nginx: Reverse Proxy & Load Balancer

```nginx
# /etc/nginx/sites-available/ai-api
upstream ai_backend {
    # Direct requests to FastAPI backend server
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;  # Load balance across 2 instances
}

server {
    listen 80;  # Listen on port 80 (HTTP)
    server_name api.example.com;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # SSL certificates (see Let's Encrypt section)
    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;

    # Proxy requests to FastAPI
    location / {
        proxy_pass http://ai_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket support (for streaming)
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Health check endpoint
    location /health {
        access_log off;  # Don't log health checks
        proxy_pass http://ai_backend;
    }
}

# Enable the config
sudo ln -s /etc/nginx/sites-available/ai-api /etc/nginx/sites-enabled/ai-api
sudo nginx -t  # Test config
sudo systemctl reload nginx
```

### 3.10 Let's Encrypt: Free SSL Certificates

```bash
# Install certbot
sudo apt-get install certbot python3-certbot-nginx

# Obtain certificate (automatic nginx config)
sudo certbot --nginx -d api.example.com

# Obtain certificate (manual)
sudo certbot certonly --standalone -d api.example.com

# Auto-renewal (runs daily)
sudo systemctl enable certbot.timer

# Renew manually
sudo certbot renew

# Check certificate expiry
openssl x509 -in /etc/letsencrypt/live/api.example.com/fullchain.pem -noout -dates
```

### 3.11 Monitoring: Resource Usage

```bash
# Real-time CPU/Memory (interactive)
top
# Press 'M' to sort by memory
# Press 'P' to sort by CPU

# Disk usage
df -h  # Filesystem usage

du -sh /var/log  # Specific directory

# Network I/O
iftop  # Real-time network usage (install: apt-get install iftop)

# CPU cores
nproc  # Number of CPU cores
lscpu  # Detailed CPU info

# Memory info
free -h  # Used, available memory

cat /proc/cpuinfo  # Detailed CPU info

# GPU usage (if available)
nvidia-smi  # NVIDIA GPU stats
```

### 3.12 tmux: Terminal Multiplexing

```bash
# Start new session
tmux new-session -s training

# Now you're in tmux. Start your long-running job
python train.py

# Detach (leave job running): Ctrl+B then D
# Session keeps running even if you disconnect

# List sessions
tmux list-sessions

# Reattach to session
tmux attach-session -s training

# Kill session
tmux kill-session -s training

# Create window in session
tmux new-window -t training -n worker2

# Kill window
tmux kill-window -t training:1

# Split pane (horizontal)
# Ctrl+B then %

# Split pane (vertical)
# Ctrl+B then "

# Move between panes
# Ctrl+B then Arrow keys

# Sample tmux.conf (persistent config)
# ~/.tmux.conf
set -g mouse on  # Enable mouse support
bind r source-file ~/.tmux.conf  # Reload config with Ctrl+B R

# Basic commands
Ctrl+B C  # Create new window
Ctrl+B N  # Next window
Ctrl+B P  # Previous window
Ctrl+B D  # Detach
Ctrl+B [  # Enter copy mode (scroll)
```

---

## 4. Architecture Diagram: Linux System Stack

```mermaid
graph TB
    A["Application<br/>Python/FastAPI"] --> B["systemd<br/>Service Management"]
    B --> C["Linux Kernel<br/>Process, Memory, I/O"]
    C --> D["Hardware<br/>CPU, RAM, Disk"]
    
    E["nginx<br/>Reverse Proxy"] --> F["Networking Stack<br/>TCP/IP"]
    F --> G["Firewall<br/>UFW/iptables"]
    
    H["Monitoring<br/>logs, metrics"] --> I["Logging<br/>systemd journal"]
    
    A --> E
    B --> I
    C --> D
    G --> D
```

---

## 5. Tools Section: Essential Linux Tools

| Tool | Purpose | Example |
|------|---------|---------|
| **systemd** | Service management | `systemctl start service` |
| **journalctl** | Log viewing | `journalctl -u service -f` |
| **nginx** | Reverse proxy | Proxy FastAPI to port 80 |
| **certbot** | SSL certificates | Automatic HTTPS |
| **tmux** | Terminal sessions | Keep training jobs alive |
| **curl/httpie** | API testing | Test endpoints |
| **ssh** | Remote access | Connect to servers |
| **rsync** | File sync | Copy large model files |
| **top/htop** | Process monitoring | Watch CPU/memory |
| **lsof** | File descriptor listing | Debug port conflicts |

---

## 6. Decision Framework: When to Use What

| Scenario | Tool/Command | Why |
|----------|-------------|-----|
| Run app at startup | systemd service | Automatic restart, logging |
| Test API endpoint | curl or httpie | Quick, no dependencies |
| View recent errors | `journalctl -f` | Real-time, filtered |
| SSH to server | SSH key + config | Secure, no passwords |
| Monitor GPU | nvidia-smi | Real-time streaming data |
| Copy model files | rsync (not scp) | Much faster for large dirs |
| Proxy traffic | nginx | Load balancing, SSL |
| Get HTTPS cert | Let's Encrypt/certbot | Free, automatic renewal |
| Long-running job | tmux session | Survives disconnection |
| Find process crash | systemd logs | Full backtrace + context |

---

## 7. Step-by-Step: Deploying FastAPI on Linux

### Step 1: Create systemd Service

```bash
sudo nano /etc/systemd/system/ai-api.service
```

Paste:
```ini
[Unit]
Description=AI FastAPI Server
After=network.target
Wants=postgresql.service redis-server.service

[Service]
Type=notify
User=ai_user
WorkingDirectory=/home/ai_user/myapp
Environment="PATH=/home/ai_user/.local/bin"
EnvironmentFile=/home/ai_user/myapp/.env
ExecStart=/home/ai_user/.local/bin/poetry run uvicorn src.main:app --host 0.0.0.0 --port 8000 --workers 4
Restart=always
RestartSec=5

StandardOutput=append:/var/log/ai-api.log
StandardError=append:/var/log/ai-api.log

[Install]
WantedBy=multi-user.target
```

### Step 2: Enable & Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable ai-api
sudo systemctl start ai-api
sudo systemctl status ai-api

# View logs
sudo journalctl -u ai-api -f
```

### Step 3: Configure nginx

```bash
sudo nano /etc/nginx/sites-available/api
```

Paste:
```nginx
upstream backend { server 127.0.0.1:8000; }
server {
    listen 80; server_name api.example.com;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl http2; server_name api.example.com;
    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    location / { proxy_pass http://backend; }
}
```

### Step 4: Enable & Test

```bash
sudo ln -s /etc/nginx/sites-available/api /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Step 5: Verify

```bash
curl https://api.example.com/health
# Should return healthy
```

---

## 8. Practical Project: Monitoring Dashboard

**Goal:** Create scripts to monitor your deployed service.

```bash
#!/bin/bash
# monitor.sh - Health check script

# Check if service is running
systemctl is-active --quiet ai-api
if [ $? -ne 0 ]; then
    echo "ALERT: ai-api is not running"
    systemctl restart ai-api
fi

# Check disk space
DISK_USAGE=$(df / | awk 'NR==2 {print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 80 ]; then
    echo "ALERT: Disk usage is $DISK_USAGE%"
fi

# Check HTTP endpoint
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health)
if [ "$HTTP_CODE" != "200" ]; then
    echo "ALERT: API returned $HTTP_CODE"
fi

# Check memory
MEM_USAGE=$(free | awk 'NR==2 {print int($3/$2 * 100)}')
if [ $MEM_USAGE -gt 85 ]; then
    echo "ALERT: Memory usage is $MEM_USAGE%"
fi

echo "All checks passed at $(date)"
```

Run daily:
```bash
# Add to crontab
crontab -e
# Add line:
0 * * * * /home/ai_user/monitor.sh >> /var/log/monitor.log 2>&1
```

---

## 9. Debugging Playbook: 10 Linux Errors

### Error 1: "Permission denied" when running command

**Symptom:** `permission denied: /usr/local/bin/myapp`

**Explanation:** File doesn't have execute permission.

**Fix:**
```bash
chmod +x /usr/local/bin/myapp
# or
chmod 755 /usr/local/bin/myapp
```

**Prevention:** Set permissions correctly on install.

---

### Error 2: "Address already in use"

**Symptom:** `OSError: [Errno 48] Address already in use: ('::', 8000)`

**Explanation:** Another process is using port 8000.

**Fix:**
```bash
lsof -iTCP:8000
# Kill the process
kill -9 <PID>

# Or use different port
uvicorn main:app --port 8001
```

**Prevention:** Use unique ports. Configure properly in systemd.

---

### Error 3: "systemd service failed to start"

**Symptom:** `systemctl status myapp` shows failed.

**Explanation:** Check the logs.

**Fix:**
```bash
journalctl -u myapp -n 50  # Last 50 lines
journalctl -u myapp -p err  # Errors only

# Common causes:
# 1. Path doesn't exist (check WorkingDirectory)
# 2. User doesn't have permissions (check User=)
# 3. Environment variable missing (check EnvironmentFile)
```

**Prevention:** Test systemd locally before deploying.

---

### Error 4: "disk quota exceeded"

**Symptom:** Can't write files. `No space left on device`

**Explanation:** Disk is full.

**Fix:**
```bash
df -h  # Check usage
du -sh /var/log  # Find largest directory
sudo rm /var/log/old.log  # Delete old logs
# Or set up log rotation
```

**Prevention:** Monitor disk usage. Set up logrotate.

---

### Error 5: "SSH: connection refused"

**Symptom:** `ssh: connect to host server.example.com port 22: Connection refused`

**Explanation:** SSH server not running or firewall blocking.

**Fix:**
```bash
# On server
sudo systemctl start ssh
sudo systemctl enable ssh

# Check firewall
sudo ufw allow 22/tcp
sudo ufw enable
```

**Prevention:** SSH should auto-start. Enable explicitly in systemd.

---

### Error 6: "command not found"

**Symptom:** `python: command not found`

**Explanation:** Binary not in PATH.

**Fix:**
```bash
which python  # Show actual path
python3 main.py  # May need python3 instead of python

# Or add to PATH
export PATH="/usr/local/bin:$PATH"
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.bashrc
```

**Prevention:** Use absolute paths. Check PATH on deployment.

---

### Error 7: "too many open files"

**Symptom:** `OSError: [Errno 24] Too many open files`

**Explanation:** Process hit open file limit.

**Fix:**
```bash
# Increase limit
ulimit -n 65536  # Set limit to 65536

# Make permanent
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
```

**Prevention:** Monitor open file count. Increase limit on deployment.

---

### Error 8: "SSL certificate problem"

**Symptom:** `curl: (60) SSL certificate problem: unable to get local issuer certificate`

**Explanation:** SSL cert expired or invalid.

**Fix:**
```bash
# Check expiry
openssl x509 -in /etc/ssl/certs/cert.pem -noout -dates

# Renew with Let's Encrypt
sudo certbot renew

# Or use curl with insecure (dev only!)
curl --insecure https://localhost:8443
```

**Prevention:** Set up automatic renewal with certbot.

---

### Error 9: "Cannot connect to database"

**Symptom:** Application fails to start. Logs: "Could not connect to PostgreSQL"

**Explanation:** Database URL wrong or database not running.

**Fix:**
```bash
# Check database is running
docker ps | grep postgres

# Test connection
psql "postgresql://user:pass@localhost:5432/dbname"

# Check environment variable
echo $DATABASE_URL
```

**Prevention:** Health check DB before starting app.

---

### Error 10: "nginx: [emerg] bind() to failed"

**Symptom:** nginx won't start. `bind() [::]:443 failed`

**Explanation:** Port 443 already in use or permission denied.

**Fix:**
```bash
# Kill process using port
lsof -iTCP:443
kill -9 <PID>

# Or run as root (for ports < 1024)
sudo systemctl start nginx
```

**Prevention:** Run nginx as root. Use ports > 1024 for unprivileged services.

---

## 10. Common Mistakes: Linux Administration

### Mistake 1: Running App Directly (Not systemd)

```bash
# Wrong: direct terminal (dies when you disconnect)
ssh server.example.com
python main.py  # CTRL+C or disconnect → app stops

# Right: systemd service (restarts automatically)
systemctl start myapp
# App runs forever, restarts on crash
```

---

### Mistake 2: Ignoring Logs

```bash
# Wrong: no logging
app.run()  # Silent failures

# Right: structured logging to systemd
logger.info("Event happened")
# Appears in: journalctl -u myapp -f
```

---

### Mistake 3: SSH Without Keys

```bash
# Wrong: password-based SSH (vulnerable)
ssh user@server.example.com  # Prompts for password

# Right: SSH key (secure, automatic)
ssh -i ~/.ssh/mykey user@server.example.com
# Or configure ~/.ssh/config
```

---

### Mistake 4: One Endpoint Behind firewall

```bash
# Wrong: forgot to open port
sudo ufw allow 22  # SSH open
# But 8000 (app port) still blocked

# Right: open all necessary ports
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

---

### Mistake 5: Not Monitoring Resources

```bash
# Wrong: fires up training job, never checks
python train.py &

# Right: monitor before, during, after
top  # Check max memory needed
nvidia-smi  # Watch GPU
# Then schedule with resource limits:
# cpulimit -l 50% python train.py
```

---

## 11. Production Realism: Local vs Production

### Local Deployment

```bash
cd myapp
poetry install
uvicorn src.main:app --reload --port 8000
# Run in foreground, auto-reload on changes
```

### Production Deployment

```bash
# 1. Application files copied to /home/deploy/myapp
# 2. Systemd service installed (/etc/systemd/system/myapp.service)
# 3. Environment loaded from /home/deploy/.env (secrets separate)
# 4. nginx proxy in front (handles SSL, load balancing)
# 5. All logs in journalctl (persistent, searchable)
# 6. Health checks running (monitoring)
# 7. Auto-restarts on crash
# 8. Multiple workers (4-8 processes)

# Result: bulletproof service that runs 24/7
```

---

## 12. Cost & Performance Considerations

### Server Sizing

```
CPU Cores → Concurrent Users:
1 core: 1-5 users
2 cores: 5-10 users
4 cores: 20-40 users
8 cores: 50-100 users

Memory → Model Size + Concurrency:
4GB: small model
16GB: medium model
32GB: large model or high concurrency
64GB: production-grade
```

### Monitoring When It Matters

```
Check every:
- CPU: If training job → every minute
- Memory: If model loading → continuously
- Disk: If log generation → hourly
- Network: If streaming model → seconds
- DB connections: Always at startup
```

---

## 13. Security Considerations

### SSH Security

```bash
# Regenerate keys periodically
ssh-keygen -C "$(whoami)@$(hostname)-$(date -I)"

# Disable password authentication
sudo nano /etc/ssh/sshd_config
# Add: PasswordAuthentication no
sudo systemctl reload ssh

# Fail2ban (auto-block brute force)
sudo apt-get install fail2ban
sudo systemctl enable fail2ban
```

### Firewall Rules

```bash
# Start with everything blocked
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Only allow what's needed
sudo ufw allow 22/tcp  # SSH
sudo ufw allow 80/tcp  # HTTP
sudo ufw allow 443/tcp  # HTTPS
sudo ufw allow from 10.0.0.0/8 to any port 5432  # PostgreSQL from internal
```

### Secret Management

```bash
# Wrong: hardcoded in config
API_KEY = "sk-1234567890"

# Right: environment variable
API_KEY = os.getenv("OPENAI_API_KEY")
# Set via:
# 1. systemd EnvironmentFile=/path/to/.env
# 2. AWS Secrets Manager
# 3. HashiCorp Vault
```

---

## 14. Case Study: Netflix Deployment

**Why Netflix matters:** Serves 200M+ users, complex deployment.

**Their practices:**
- Multiple instances behind load balancer
- Canary deployments (5% of users on new version)
- Automatic health checks + rollback
- Centralized logs (not scattered across servers)
- Monitoring triggers alerts

**Lesson:** Automation is survival. Deploy with confidence, monitor everything.

---

## 15. Interview Cheat Sheet: Linux/Ops

### Concept 1: systemd Service Lifecycle

**Q:** Explain systemd service startup to shutdown.

**A:** ExecStartPre (setup) → ExecStart (run) → ExecStop (cleanup). If crashes, Restart=always restarts after RestartSec delay.

---

### Concept 2: Port Forwarding vs Reverse Proxy

**Q:** What's the difference?

**A:** Port forwarding (SSH -L) is temporary, local. Reverse proxy (nginx) is permanent, production, handles SSL.

---

### Concept 3: Log Rotation

**Q:** Why rotate logs?

**A:** Prevent disk fill. Old logs are compressed/deleted. logrotate handles this automatically.

---

### Concept 4: Process Signals

**Q:** Difference between SIGTERM and SIGKILL?

**A:** SIGTERM (15) = graceful shutdown. SIGKILL (9) = forceful, can't catch. Always SIGTERM first.

---

### Concept 5: Open File Descriptors

**Q:** What's "too many open files"?

**A:** Each connection (socket, file, pipe) is an FD. Limit is ~1024 by default. Hit it with many concurrent connections. Increase with ulimit.

---

## 16. Review Questions

1. How do you keep a Python process running after you disconnect from SSH?
2. Write a systemd service file for your FastAPI app.
3. Explain how nginx proxies requests to your backend.
4. How do you renew an SSL certificate with Let's Encrypt?
5. What does `kill -9 <PID>` do vs `kill <PID>`?
6. How do you find which process is using port 8000?
7. Explain how tmux keeps a session alive.
8. How do you monitor real-time resource usage?
9. What's the firewall command to allow port 443?
10. How do you tail logs from a systemd service?

---

## 17. Flashcards

### Card 1
**Front:** How do you make a FastAPI app start automatically on reboot?

**Back:** Create a systemd service in `/etc/systemd/system/`, use `ExecStart=`, enable with `systemctl enable`.

---

### Card 2
**Front:** Why use nginx instead of running FastAPI directly on port 80?

**Back:** Nginx handles SSL termination, load balancing, reverse proxy. FastAPI is for app logic, not infrastructure.

---

### Card 3
**Front:** What's the difference between tail and journalctl?

**Back:** tail shows files. journalctl shows systemd journal (structured, searchable, includes metadata).

---

### Card 4
**Front:** How do you test if a port is open?

**Back:** `nc -zv localhost 8000` or `curl localhost:8000`. Or use lsof.

---

### Card 5
**Front:** Why use SSH keys instead of passwords?

**Back:** Keys are automatic, no typos. Passwords are vulnerable to brute force. Keys allow failban/rate limiting.

---

### Card 6
**Front:** What does systemctl daemon-reload do?

**Back:** Tells systemd to re-read service files. Needed after editing .service files.

---

### Card 7
**Front:** How do you increase the open file limit?

**Back:** `ulimit -n 65536` (temporary) or add to `/etc/security/limits.conf` (permanent).

---

### Card 8
**Front:** What's the difference between HTTP and HTTPS?

**Back:** HTTP = unencrypted. HTTPS = encrypted with SSL/TLS. Always use HTTPS in production.

---

### Card 9
**Front:** How do you find a process by name?

**Back:** `ps aux | grep name | grep -v grep` (to avoid matching the grep itself).

---

### Card 10
**Front:** Why is `kill -9` dangerous?

**Back:** Process can't clean up (close files, release locks). Use `kill 15` (SIGTERM) first, only use -9 if needed.

---

## 18. Teach-It-Back Prompt

Explain why systemd is better than running `python app.py` in a tmux session.

**Model Answer:**

"systemd is the OS-level service manager. It starts your app on boot, restarts on crash, captures logs centrally, manages dependencies, and handles graceful shutdown. tmux is a terminal. Kill the terminal, session disappears (unless you detached). systemd persists across reboots and is production-standard. It's the difference between 'works while I'm watching' and 'works forever.'"

---

## 19. 1-Week Review Checklist

- [ ] Deploy a FastAPI app with systemd (not terminal)
- [ ] Configure nginx reverse proxy in front
- [ ] Set up HTTPS with Let's Encrypt
- [ ] Monitor logs with journalctl
- [ ] Create a health check script

---

## 20. Resources

- [systemd documentation](https://systemd.io/)
- [nginx documentation](https://nginx.org/en/docs/)
- [Let's Encrypt guide](https://letsencrypt.org/getting-started/)
- [SSH best practices](https://man.openbsd.org/ssh_config)
- [Linux command reference](https://linux.die.net/)
- [OWASP: Linux hardening](https://cheatsheetseries.owasp.org/)

---

## 21. Next: Capstone Project

You now have all the pieces. Time to build something real.

→ **[Phase 0 Milestone: Complete Backend →](milestone.md)**
