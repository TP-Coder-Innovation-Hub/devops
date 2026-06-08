# Networking for DevOps `[Entry]`

## Why Networking Matters

When production breaks at 2 AM, the problem is often network-related: DNS fails, a port is blocked, a TLS certificate expired, or a firewall drops traffic. You need to diagnose these fast.

## The OSI Model (Simplified)

You rarely need all seven layers. Focus on these four:

| Layer | Protocol | What Breaks |
|-------|----------|-------------|
| Application | HTTP, HTTPS, DNS | Wrong URLs, cert issues |
| Transport | TCP, UDP | Port blocked, connection refused |
| Network | IP, routing | Subnet misconfiguration |
| Link | Ethernet, ARP | Network cable, switch failure |

## DNS

DNS maps names to IP addresses. Most outages start here.

```bash
# Resolve a domain
dig +short example.com          # 93.184.216.34

# Check full DNS response
dig example.com A
dig example.com CNAME
dig example.com MX

# Reverse lookup (IP to name)
dig -x 93.184.216.34

# Trace DNS resolution path
dig +trace example.com

# Flush local DNS cache
sudo systemd-resolve --flush-caches   # systemd
sudo dscacheutil -flushcache           # macOS
```

Common DNS issues: TTL too high (stale records), missing records, CNAME pointing to deleted resource.

## HTTP

The protocol your services speak. Know the status codes:

| Code Range | Meaning | Action |
|------------|---------|--------|
| 2xx | Success | None |
| 3xx | Redirect | Check redirect target |
| 4xx | Client error | Fix the request |
| 5xx | Server error | Check application logs |

```bash
# Health check
curl -sf http://localhost:8080/health || echo "UNHEALTHY"

# Inspect response headers
curl -I https://example.com

# Debug TLS certificate
curl -vI https://example.com 2>&1 | grep -E "SSL|subject|expire"

# POST with JSON payload
curl -X POST http://localhost:8080/api/items \
    -H "Content-Type: application/json" \
    -d '{"name": "test", "value": 42}'

# Follow redirects, show timing
curl -L -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
    -o /dev/null -s https://example.com
```

## SSH

Secure remote access and file transfer.

```bash
# Remote command execution
ssh prod-server "systemctl status nginx"

# Port forwarding (tunnel)
ssh -L 5432:db.internal:5432 bastion-host
# Now localhost:5432 tunnels to db.internal:5432 via bastion

# Copy files
scp deploy.sh prod-server:/opt/app/
rsync -avz ./dist/ prod-server:/var/www/html/

# SSH config (~/.ssh/config)
Host bastion
    HostName 52.10.10.10
    User admin
    IdentityFile ~/.ssh/bastion_key

Host db-*
    ProxyJump bastion
    User postgres
```

## Firewalls

Control which traffic reaches your servers.

```bash
# iptables (traditional Linux firewall)
sudo iptables -L -n                # list rules
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT   # allow HTTPS
sudo iptables -A INPUT -j DROP     # deny all other

# ufw (Ubuntu, simpler)
sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# Check what's listening
sudo ss -tlnp
```

## Debugging Network Issues

When something cannot connect, check in order:

```bash
# 1. Is DNS resolving?
dig +short myservice.internal

# 2. Is the port open?
nc -zv 10.0.0.5 5432

# 3. Is there a route?
traceroute 10.0.0.5

# 4. Is HTTP responding?
curl -v http://10.0.0.5:8080/health

# 5. Is TLS valid?
openssl s_client -connect example.com:443 -servername example.com </dev/null
```

## What to Remember

DNS, TCP ports, and TLS certificates cause most production network issues. Master `dig`, `curl`, `nc`, and `ssh`. They are your debugging tools.
