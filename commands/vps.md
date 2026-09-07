VPS operations — SSH, deploy, logs, status, and file transfers.

**Template — replace `$VPS_HOST` with your own `user@host`, and the example service
table with your own.**

## VPS basics

- **Host**: `$VPS_HOST` (e.g. `root@your.server.ip`)
- **Reverse proxy**: Caddy at `/etc/caddy/Caddyfile` (auto-HTTPS)
- **Service manager**: systemd — services under `/opt/`

```bash
ssh $VPS_HOST "command"
scp localfile $VPS_HOST:/remote/path
```

## Running services

Keep a table like this current — one row per systemd service, with subdomain/port/path.
Re-verify periodically against live `systemctl`/Caddyfile state rather than trusting it
forever; it goes stale fast.

| Service | Subdomain | Port | Path |
|---------|-----------|------|------|
| example-web | example.yourdomain.com | 8000 | /opt/example |

Flag any service that's real-money / production-critical so a restart isn't done
without checking timing first.

## Common operations

### Check status of a service
```bash
ssh $VPS_HOST "systemctl status <service>.service --no-pager"
```

### Pull logs (last 50 lines)
```bash
ssh $VPS_HOST "journalctl -u <service>.service -n 50 --no-pager"
```

### Deploy a Python service (standard pattern)
```bash
ssh $VPS_HOST "cd /opt/<service> && git pull origin main && venv/bin/pip install -e . && systemctl restart <service>"
```

### Deploy a static site (copy files)
```bash
scp -r ./dist/* $VPS_HOST:/opt/<sitename>/
```

### Add a new subdomain (Caddy)
```bash
ssh $VPS_HOST "cat >> /etc/caddy/Caddyfile" << 'EOF'
newsite.yourdomain.com {
    reverse_proxy 127.0.0.1:PORT
}
EOF
ssh $VPS_HOST "systemctl reload caddy"
```

### Quick health check
```bash
ssh $VPS_HOST "systemctl list-units --type=service --state=failed --no-pager; free -h; df -h /"
```

## Deploying a brand new service

1. Write the code locally, test it.
2. Create `/opt/<service>` on the VPS, set up venv, copy files:
   ```bash
   ssh $VPS_HOST "mkdir -p /opt/<service>"
   scp -r ./* $VPS_HOST:/opt/<service>/
   ssh $VPS_HOST "cd /opt/<service> && python3 -m venv venv && venv/bin/pip install -r requirements.txt"
   ```
3. Write a `.env` file with secrets:
   ```bash
   ssh $VPS_HOST "echo 'KEY=value' > /opt/<service>/.env && chmod 600 /opt/<service>/.env"
   ```
4. Install and start the systemd service:
   ```bash
   scp <service>.service $VPS_HOST:/etc/systemd/system/
   ssh $VPS_HOST "systemctl daemon-reload && systemctl enable --now <service>"
   ```
5. Add a Caddy block and reload.

## Notes
- Deploy by pulling from git on the VPS, not by scp'ing loose files, for anything
  git-backed — keeps prod in sync with a commit you can point to.
- Prefer key-only SSH (`PermitRootLogin prohibit-password`, `PasswordAuthentication no`)
  and `fail2ban` on any internet-facing box.
- Only open firewall ports that have an active listener (`ss -ltn`) — prune stale
  `ufw allow` rules.
