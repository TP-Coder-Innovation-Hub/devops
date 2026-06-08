# Linux Fundamentals

## Why Linux

Most production servers run Linux. CI runners, containers, cloud VMs — all Linux. You need to navigate, debug, and automate on Linux systems daily.

## File System Navigation

```bash
# Where am I
pwd                          # /home/user/projects

# List files
ls -la                       # all files, details, hidden files

# Move around
cd /var/log                  # absolute path
cd ../config                 # relative path

# Find things
find /etc -name "*.conf"     # find config files
which nginx                  # locate a binary
```

## File Permissions

Every file has three permission sets: owner, group, others.

```
-rwxr-xr-- 1 user group 4096 Jun  8 10:00 deploy.sh
│├──┤├──┤├──┤
│ │   │   └── others: read
│ │   └────── group: read + execute
│ └────────── owner: read + write + execute
└──────────── file (-) not directory (d)
```

```bash
chmod 755 deploy.sh          # rwxr-xr-x
chmod +x script.sh           # add execute permission
chown appuser:appgroup app.log
```

## Processes

```bash
# List running processes
ps aux                       # all processes, full output
ps aux | grep nginx          # find nginx processes

# Real-time view
top                          # CPU, memory usage
htop                         # better top (install separately)

# Kill a process
kill -9 12345                # force kill PID 12345
killall nginx                # kill all nginx processes

# Background processes
./server &                   # run in background
nohup ./server &             # survive terminal close
```

## Networking

```bash
# Check open ports
ss -tlnp                     # listening TCP ports with process info

# Test connectivity
curl -I https://example.com  # HTTP headers
wget --spider http://localhost:8080/health

# DNS lookup
dig example.com
nslookup example.com

# Trace route
traceroute 10.0.0.5
```

## SSH

```bash
# Generate key pair
ssh-keygen -t ed25519 -C "deploy-key"

# Connect to server
ssh user@10.0.0.5 -i ~/.ssh/deploy_key

# Copy key to server (password auth)
ssh-copy-id user@10.0.0.5

# Config file for aliases: ~/.ssh/config
cat <<EOF > ~/.ssh/config
Host prod
    HostName 10.0.0.5
    User deploy
    IdentityFile ~/.ssh/deploy_key
EOF

ssh prod                     # connects using config
```

## Package Management

```bash
# Debian/Ubuntu
apt update && apt install -y nginx

# RHEL/CentOS
yum install -y nginx

# Alpine (common in containers)
apk add --no-cache nginx
```

## Text Processing

```bash
# Search in files
grep -r "ERROR" /var/log/    # recursive search

# Stream editing
cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
# Top 10 IPs by request count

# Real-time tail
tail -f /var/log/app.log | grep --color "ERROR\|WARN"
```

## What to Remember

You will spend most of your time using: `cd`, `ls`, `grep`, `cat`, `tail`, `ps`, `curl`, `ssh`, `chmod`. Master these first.
