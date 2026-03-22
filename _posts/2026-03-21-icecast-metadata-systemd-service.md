---
title: "Automating Icecast Song Metadata with a systemd Service"
date: 2026-03-21 22:35:00 -0700
categories: [kaad-one]
tags: [icecast, systemd, acrcloud, metadata, linux, automation, security]
---

When we listen in the car, it is a lot nicer when the stream shows the song title and artist instead of a blank field. It is a small thing, but it makes the station feel more complete and easier to use in players like VLC and CarPlay.

Our audio setup sends the music fine, but it does not send track names on its own. To fix that, I use ACRCloud, which is a music recognition service (basically Shazam-style matching for streams). It listens to the audio feed, identifies what song is playing, and gives that result back through an API.

From there, I run a small tool called `slogger` that takes that song info and updates Icecast automatically.

This post shows how I run `slogger` on `kaad-one` so now-playing metadata stays current without anyone needing to type it in by hand.

## What we're talking about

- **slogger**: The local metadata updater tooling and scripts.
- **ACRCloud**: Provides recent track recognition results from the monitored stream.
- **Icecast admin metadata**: Endpoint used to update the current song text.
- **Bash updater**: Polls ACRCloud, parses JSON, and updates Icecast on change.
- **systemd service**: Keeps the updater running across reboots and crashes.
- **Env file**: Stores API tokens and credentials outside the script.

## Why this setup

- It avoids manual metadata entry during live operation.
- The script is easy to audit and troubleshoot.
- `systemd` is reliable and already present on modern Linux hosts.
- Secrets stay in a local env file instead of hardcoded script text.
- Metadata updates are deduplicated, so Icecast is not spammed with repeats.

## Files

- Script (`slogger`): `/home/user/slogger/icecast-meta.sh`
- Env file (`slogger`): `/home/user/slogger/icecast-meta.env`
- Service on `kaad-one`: `/etc/systemd/system/icecast-meta.service`

## 1) Keep runtime values in an env file

Create an env file with API and Icecast settings:

`/home/user/slogger/icecast-meta.env`

<details>
<summary>Show <code>icecast-meta.env</code> example</summary>

```bash
# ACRCloud API endpoint + token
ACR_URL="https://api-v2.acrcloud.com/api/bm-cs-projects/PROJECT_ID/streams/STREAM_ID/results?type=last"
ACR_TOKEN="REDACTED"

# Icecast server/admin settings
ICECAST_SCHEME="https"
ICECAST_HOST="your-icecast-host"
ICECAST_PORT="11555"
ICECAST_USER="admin"
ICECAST_PASS="REDACTED"
ICECAST_MOUNT="stream"

# Polling + TLS behavior
POLL_SECONDS="20"
CURL_INSECURE="0"
```

</details>

Minimum permission baseline:

```bash
sudo chown user:user /home/user/slogger/icecast-meta.env
sudo chmod 600 /home/user/slogger/icecast-meta.env
```

## 2) Metadata update script

The updater script:

- loads env values,
- polls ACRCloud JSON with bearer auth,
- extracts `artist` and `title` with `jq` fallbacks,
- builds a normalized `Artist - Title` value,
- deduplicates by track ID (or normalized song fallback),
- updates Icecast metadata only on change.

`/home/user/slogger/icecast-meta.sh`

<details>
<summary>Show <code>icecast-meta.sh</code></summary>

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
ENV_FILE="${ENV_FILE:-${SCRIPT_DIR}/icecast-meta.env}"
if [[ -f "$ENV_FILE" ]]; then
  # shellcheck disable=SC1090
  source "$ENV_FILE"
fi

ACR_URL="${ACR_URL:-https://api-v2.acrcloud.com/api/bm-cs-projects/PROJECT_ID/streams/STREAM_ID/results?type=last}"
ACR_TOKEN="${ACR_TOKEN:-}"

ICECAST_HOST="${ICECAST_HOST:-your-icecast-host}"
ICECAST_PORT="${ICECAST_PORT:-11555}"
ICECAST_SCHEME="${ICECAST_SCHEME:-https}"
ICECAST_USER="${ICECAST_USER:-admin}"
ICECAST_PASS="${ICECAST_PASS:-}"
ICECAST_MOUNT="${ICECAST_MOUNT:-stream}"

POLL_SECONDS="${POLL_SECONDS:-20}"
CURL_INSECURE="${CURL_INSECURE:-0}"

: "${ACR_TOKEN:?ACR_TOKEN is required}"
: "${ICECAST_PASS:?ICECAST_PASS is required}"

auth_header_file="$(mktemp)"
chmod 600 "$auth_header_file"
printf 'Authorization: Bearer %s\n' "$ACR_TOKEN" > "$auth_header_file"
trap 'rm -f "$auth_header_file"' EXIT

last_song_key=""
curl_tls_opts=()
if [[ "$CURL_INSECURE" == "1" ]]; then
  curl_tls_opts+=(-k)
fi

while true; do
  json="$({
    curl -fsS "${curl_tls_opts[@]}" \
      -H "Accept: application/json" \
      -H "@${auth_header_file}" \
      "$ACR_URL"
  } || true)"

  if [[ -n "$json" ]]; then
    title="$(jq -r '
      [
        (.. | objects | .metadata?.music?[0]?.title?),
        (.. | objects | .music?[0]?.title?),
        (.. | objects | .title?)
      ]
      | map(select(type == "string" and length > 0))
      | .[0] // empty
    ' <<<"$json")"

    artist="$(jq -r '
      [
        (.. | objects | .metadata?.music?[0]?.artists?[0]?.name?),
        (.. | objects | .music?[0]?.artists?[0]?.name?),
        (.. | objects | .artist?.name?),
        (.. | objects | .artist?)
      ]
      | map(select(type == "string" and length > 0))
      | .[0] // empty
    ' <<<"$json")"

    if [[ -n "$artist" && -n "$title" ]]; then
      song="${artist} - ${title}"
    elif [[ -n "$title" ]]; then
      song="$title"
    else
      song=""
    fi

    song="$(jq -rn --arg s "$song" '$s | gsub("\\s+";" ") | gsub("^\\s+|\\s+$";"")')"

    track_id="$(jq -r '
      [
        (.. | objects | .metadata?.music?[0]?.acrid?),
        (.. | objects | .music?[0]?.acrid?),
        (.. | objects | .result_id?),
        (.. | objects | .id?)
      ]
      | map(select(type == "string" and length > 0))
      | .[0] // empty
    ' <<<"$json")"

    if [[ -n "$track_id" ]]; then
      song_key="$track_id"
    else
      song_key="$(jq -rn --arg s "$song" '$s | ascii_downcase')"
    fi

    if [[ -n "$song" && "$song_key" != "$last_song_key" ]]; then
      curl -fsS "${curl_tls_opts[@]}" \
        -u "${ICECAST_USER}:${ICECAST_PASS}" \
        --get "${ICECAST_SCHEME}://${ICECAST_HOST}:${ICECAST_PORT}/admin/metadata" \
        --data-urlencode "mount=/${ICECAST_MOUNT#/}" \
        --data-urlencode "mode=updinfo" \
        --data-urlencode "song=${song}" \
        >/dev/null

      echo "Updated metadata: ${song}"
      last_song_key="$song_key"
    fi
  fi

  sleep "$POLL_SECONDS"
done
```

</details>

Make it executable:

```bash
chmod 750 /home/user/slogger/icecast-meta.sh
```

## 3) systemd service

Create a dedicated service so metadata updates survive reboots and process exits.

`/etc/systemd/system/icecast-meta.service`

<details>
<summary>Show <code>icecast-meta.service</code></summary>

```ini
[Unit]
Description=Icecast metadata updater from ACRCloud
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=user
Group=user
WorkingDirectory=/home/user/slogger
Environment=ENV_FILE=/home/user/slogger/icecast-meta.env
ExecStart=/home/user/slogger/icecast-meta.sh
Restart=always
RestartSec=2
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/user/slogger

[Install]
WantedBy=multi-user.target
```

</details>

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now icecast-meta.service
```

## 4) Useful checks

For day-to-day operation:

<details>
<summary>Show useful systemd and journalctl commands</summary>

```bash
sudo systemctl status icecast-meta.service
sudo journalctl -u icecast-meta.service -f
sudo journalctl -u icecast-meta.service -n 50 --no-pager
sudo systemctl restart icecast-meta.service
```

</details>

Expected healthy log lines include:

<details>
<summary>Show expected log output example</summary>

```text
Updated metadata: Artist - Track Title
```

</details>

If you see repeated HTTP `403` errors, validate token scope/expiry and confirm the `ACR_URL` still points to the correct project stream.

## Security notes

- Never commit real `.env` files or raw tokens/passwords.
- Keep credentials file permissions tight (`chmod 600`).
- Use least-privilege API credentials where possible.
- Rotate secrets immediately if there is any chance they were exposed.

This setup gives a straightforward, auditable metadata path with native Linux components and very little operational overhead.
