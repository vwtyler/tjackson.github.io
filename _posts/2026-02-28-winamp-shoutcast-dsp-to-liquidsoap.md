---
title: "Transitioning from Winamp + Shoutcast DSP to Liquidsoap"
date: 2026-02-28 15:00:00 -0800
categories: [kaad-one]
tags: [winamp, shoutcast, liquidsoap, migration, streaming]
---

I am working with KAAD-LP 103.5 FM in Sonora ([kaad-lp.org](https://kaad-lp.org)) on a reliability upgrade for the station's streaming path.

KAAD-LP is a community station with live operators and volunteers, so the requirement is practical: the stream should stay up even during normal, busy studio use. The legacy workflow (Winamp + Shoutcast DSP) had become too fragile for that.

This post covers the first phase: server bring-up, Linux audio ingest, and initial Liquidsoap output testing.

## Context and problem statement

The previous stream chain was tied to a desktop workflow. It worked, but it was sensitive to local machine state and end-user interactions. In real station conditions, DJs could accidentally knock the stream off-air while doing routine show tasks.

The issue was not operator error. The issue was architecture.

## Infrastructure baseline: `kaad-one`

We bought a lightweight dedicated machine so streaming would no longer depend on a DJ workstation. That host is `kaad-one`.

Current baseline:

- Model: Dell OptiPlex 7080
- OS: Ubuntu 24.04.4 LTS (kernel 6.8.0-100-generic)
- CPU: Intel Core i5-10500T (6 cores / 12 threads)
- RAM: 8 GB
- Storage: 238.5 GB SSD (LVM root volume)

In the first wiring pass, board output was split into two paths:

1. Existing FM chain (unchanged)
2. `kaad-one` ingest path for streaming via a MOTU M2 USB interface

This let us modernize the streaming side without disrupting on-air FM operations.

## First implementation pass: from MOTU input to test stream

The initial sequence was:

1. Confirm ALSA sees the MOTU M2
2. Create a stable ALSA alias (`motu_in`)
3. Build a minimal Liquidsoap config
4. Validate on MyRadioStream before changing station-facing output

### 1) Confirm the ALSA device

I started by verifying device names and capture paths on Ubuntu.

```bash
sudo apt update
sudo apt install -y alsa-utils liquidsoap

arecord -l
arecord -L
```

Example output looked like this (abridged):

```text
card 2: M2 [MOTU M2], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```

Then a quick capture sanity check:

```bash
arecord -D hw:M2,0 -f S16_LE -r 48000 -c 2 -d 5 /tmp/motu-test.wav
aplay /tmp/motu-test.wav
```

### 2) Create a persistent ALSA alias: `motu_in`

Rather than hardcoding card indexes (which can shift on reboot), I added a named ALSA PCM in `/etc/asound.conf`:

```conf
pcm.motu_in {
  type dsnoop
  ipc_key 1024
  slave {
    pcm "hw:M2,0"
    channels 2
    rate 48000
    format S16_LE
  }
}

ctl.motu_in {
  type hw
  card M2
}
```

That gave us a stable device name Liquidsoap can target directly.

Quick validation:

```bash
arecord -D motu_in -f S16_LE -r 48000 -c 2 -d 5 /tmp/motu-alias-test.wav
aplay /tmp/motu-alias-test.wav
```

### 3) Build the first Liquidsoap config

I started with a minimal config focused on capture stability and predictable output. Since free MyRadioStream accounts are limited to Shoutcast, the first external test used `output.shoutcast`.

`/etc/liquidsoap/kaad-test.liq`

```liquidsoap
set("log.level", 3)
set("server.telnet", true)
set("server.telnet.port", 1234)

# MOTU capture via ALSA alias
motu = input.alsa(id="motu_in", bufferize=true, channels=2, "motu_in")

# Keep a little headroom in early testing
radio = amplify(0.9, motu)

# Send to test Shoutcast endpoint (MyRadioStream free tier)
output.shoutcast(
  %mp3(bitrate=128, samplerate=44100, stereo=true),
  host="YOUR_MYRADIOSTREAM_HOST",
  port=8000,
  password="YOUR_SOURCE_PASSWORD",
  mount="/stream",
  name="KAAD-LP Test",
  description="KAAD test chain from kaad-one",
  genre="Community Radio",
  url="https://kaad-lp.org",
  public=false,
  radio
)
```

Then run it interactively first:

```bash
liquidsoap /etc/liquidsoap/kaad-test.liq
```

### 4) Push to MyRadioStream as a safe external test

Before any production cutover, I pointed output to a free MyRadioStream test endpoint. That gave me a safe way to verify:

- end-to-end ingest from the MOTU M2,
- encoder stability over time,
- reconnect behavior when network or endpoint dropped.

Useful runtime checks:

```bash
ss -tulpn | grep liquidsoap
journalctl -u liquidsoap -f
```

Once this was stable, the migration had a real baseline instead of a lab-only config.

After that verification, I connected the chain to the station's MyRadioStream service. Early in that cutover, it became clear Shoutcast would limit the next phase. I replaced the server type there with Icecast, which better fits forward-looking needs like multiple mounts and live DJ routing.

At that point, the output side switched from `output.shoutcast` to `output.icecast`:

```liquidsoap
output.icecast(
  %mp3(bitrate=192, samplerate=44100, stereo=true),
  host="YOUR_ICECAST_HOST",
  port=8000,
  password="YOUR_SOURCE_PASSWORD",
  mount="/stream",
  name="KAAD-LP 103.5 FM",
  description="KAAD-LP live stream",
  genre="Community Radio",
  url="https://kaad-lp.org",
  public=true,
  radio
)
```

I also raised bitrate from the initial test profile to improve stream quality after stability was confirmed.

You can listen to the station stream here: [kaad-lp.org](https://kaad-lp.org).

## Result and next stage

By the end of this phase, KAAD had a dedicated Linux ingest host, validated MOTU-to-stream signal flow, and a tested path from Shoutcast-limited proof-of-concept to Icecast-based output.

Next post: the first production-oriented Liquidsoap script, including fallback behavior, startup ordering, and guardrails to prevent accidental stream drops during normal DJ operations.
