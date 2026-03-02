---
title: "Cloudflare Dynamic DNS with systemd Timer and .env Secrets"
date: 2026-03-01 12:30:00 -0800
categories: [homelab]
tags: [cloudflare, dns, ddns, systemd, linux, automation, security, kaad-one]
---

I wanted a simple dynamic DNS setup to serve various needs at home. At this stage I think I primarily use it for wireguard, but that's another post. For now let's focus on the simple DDNS setup.

All my domains are on managed in Cloudflare, so I used Cloudflare's DNS API, a small shell script, and a `systemd` timer.

This is now running on more than one server of mine serving in different capacities. It is versatile. There may be better ways to do this out there, so take this as you will. I'm going to use "mydomain.com" as the placeholder.

## Why this setup

- Cloudflare already hosts my DNS.
- A shell script is easy to audit and debug.
- `systemd` timer is native, reliable, and easy to inspect.
- Secrets stay in a local `.env` file, not in published code.

## Files 

- Script: `/usr/local/bin/cf-ddns-kaad.sh`
- Service: `/etc/systemd/system/cf-ddns-kaad.service`
- Timer: `/etc/systemd/system/cf-ddns-kaad.timer`
- Env file: `/etc/cloudflare-ddns/kaad.env`
- Log file: `/var/log/cf-ddns-kaad.log`

## 1) Keep Cloudflare credentials in `.env`

I use an env file that is readable only by root:

```bash
CF_API_TOKEN="REDACTED"
CF_ZONE_NAME="tjackson.me"
CF_RECORD_NAME="kaad.tjackson.me"
CF_RECORD_TYPE="A"
```

Permissions:

```bash
sudo mkdir -p /etc/cloudflare-ddns
sudo chown root:root /etc/cloudflare-ddns
sudo chmod 700 /etc/cloudflare-ddns
sudo chmod 600 /etc/cloudflare-ddns/kaad.env
```

## 2) DDNS update script

The script loads the env file, detects public IP, checks the current DNS record, and only updates Cloudflare when the IP changed.

`/usr/local/bin/cf-ddns-kaad.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

ENV_FILE="/etc/cloudflare-ddns/kaad.env"
LOG_FILE="/var/log/cf-ddns-kaad.log"

log() {
  printf '%s %s\n' "$(date -Is)" "$*" | tee -a "$LOG_FILE" >/dev/null
}

if [[ ! -f "$ENV_FILE" ]]; then
  echo "Missing $ENV_FILE" >&2
  exit 1
fi

set -a
source "$ENV_FILE"
set +a

: "${CF_API_TOKEN:?Missing CF_API_TOKEN}"
: "${CF_ZONE_NAME:?Missing CF_ZONE_NAME (e.g. tjackson.me)}"
: "${CF_RECORD_NAME:?Missing CF_RECORD_NAME (e.g. kaad.tjackson.me)}"
: "${CF_RECORD_TYPE:=A}"

AUTH_HEADER="Authorization: Bearer ${CF_API_TOKEN}"
JSON_HEADER="Content-Type: application/json"

PUBLIC_IP="$(curl -fsS4 --max-time 5 https://api.ipify.org 2>/dev/null || true)"
if [[ -z "$PUBLIC_IP" ]]; then
  PUBLIC_IP="$(curl -fsS4 --max-time 5 https://checkip.amazonaws.com 2>/dev/null | tr -d ' \n\r\t' || true)"
fi
if [[ -z "$PUBLIC_IP" ]]; then
  PUBLIC_IP="$(curl -fsS4 --max-time 5 https://icanhazip.com 2>/dev/null | tr -d ' \n\r\t' || true)"
fi
if [[ -z "$PUBLIC_IP" ]]; then
  log "ERROR: Could not determine public IP"
  exit 1
fi

ZONE_ID="$({
  curl -fsS -H "$AUTH_HEADER" \
    "https://api.cloudflare.com/client/v4/zones?name=${CF_ZONE_NAME}&status=active"
} | jq -r '.result[0].id // empty')"
if [[ -z "$ZONE_ID" ]]; then
  log "ERROR: Zone not found or not accessible: ${CF_ZONE_NAME}"
  exit 1
fi

REC_JSON="$({
  curl -fsS -H "$AUTH_HEADER" \
    "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records?type=${CF_RECORD_TYPE}&name=${CF_RECORD_NAME}"
})"

RECORD_ID="$(printf '%s' "$REC_JSON" | jq -r '.result[0].id // empty')"
CURRENT_IP="$(printf '%s' "$REC_JSON" | jq -r '.result[0].content // empty')"

if [[ -z "$RECORD_ID" ]]; then
  log "ERROR: Record not found: ${CF_RECORD_NAME} (${CF_RECORD_TYPE}) in zone ${CF_ZONE_NAME}"
  exit 1
fi

if [[ "$CURRENT_IP" == "$PUBLIC_IP" ]]; then
  log "OK: ${CF_RECORD_NAME} already ${PUBLIC_IP} (no change)"
  exit 0
fi

UPDATE_JSON="$({
  curl -fsS -X PUT \
    -H "$AUTH_HEADER" \
    -H "$JSON_HEADER" \
    --data "{\"type\":\"${CF_RECORD_TYPE}\",\"name\":\"${CF_RECORD_NAME}\",\"content\":\"${PUBLIC_IP}\",\"ttl\":120,\"proxied\":false}" \
    "https://api.cloudflare.com/client/v4/zones/${ZONE_ID}/dns_records/${RECORD_ID}"
})"

if printf '%s' "$UPDATE_JSON" | jq -e '.success == true' >/dev/null; then
  log "UPDATED: ${CF_RECORD_NAME} ${CURRENT_IP:-unknown} -> ${PUBLIC_IP}"
else
  ERR="$(printf '%s' "$UPDATE_JSON" | jq -c '.errors // empty' 2>/dev/null || true)"
  log "ERROR: Update failed: ${ERR:-unknown}"
  exit 1
fi
```

## 3) systemd service

`/etc/systemd/system/cf-ddns-kaad.service`:

```ini
[Unit]
Description=Cloudflare DDNS update for kaad.tjackson.me

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cf-ddns-kaad.sh
```

## 4) systemd timer

`/etc/systemd/system/cf-ddns-kaad.timer`:

```ini
[Unit]
Description=Run Cloudflare DDNS update for kaad.tjackson.me every 24 hours

[Timer]
OnBootSec=2min
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cf-ddns-kaad.timer
```

## 5) Useful checks

```bash
systemctl status cf-ddns-kaad.timer
systemctl list-timers --all | grep cf-ddns-kaad
sudo systemctl start cf-ddns-kaad.service
journalctl -u cf-ddns-kaad.service -f
tail -f /var/log/cf-ddns-kaad.log
```

## Security notes

- Use a scoped Cloudflare API token (DNS edit only for the target zone).
- Never commit real API tokens or raw `.env` files.
- Keep env files root-only (`chmod 600`).
- Rotate credentials if there is any chance they were exposed.

This setup has been very low maintenance so far and gives me a clear, auditable DDNS path with native Linux tooling.
