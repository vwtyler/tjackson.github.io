---
title: "Replacing Shoutcast DSP with a Dedicated Liquidsoap Broadcast Server"
date: 2026-02-27 15:00:00 -0800
categories: [kaad-one]
tags: [winamp, shoutcast, liquidsoap, migration, streaming]
---

I am working with KAAD-LP 103.5 FM in Sonora ([kaad-lp.org](https://kaad-lp.org)) on a reliability upgrade for the station's streaming path.

KAAD-LP is a community station with a few live operators who are all volunteers, with a small but dedicated listenership. The requirement is practical: the stream should stay up even during studio use by a live DJ. The legacy workflow (Winamp + Shoutcast DSP) had become too fragile for that. The station is also going through a growth stage with a large technical upgrade, new board, broadcast desk, broadcast workstation, etc. In this forward motion, the intention is to make the station more reliable and user friendly.

I'm going to talk about this transition over time, primarily with respect to kaad-one, the server I set up. Today we start from the beginning.

## Context

The previous stream chain was tied to a desktop workflow. Computer A runs Winamp with the DSP plugin broadcasting to Myradiostream. It worked, but it was sensitive to local machine state and end-user interactions. DJs could accidentally knock the stream off-air while doing routine show tasks. This introduced massive unreliability with little ability to know something had gone wrong. So the stream would go off, then a few hours would go by before someone let us know.

After several attempts to communicate best practices with large notes taped to the computer monitor, I realized the issue was not operator error. The issue was architecture.

## kaad-one

We bought a lightweight dedicated machine so streaming would no longer depend on a DJ workstation. That host is **kaad-one**. This move would let me start to automate a lot of things, but first and foremost, allow me to take away the DJ variable in stream reliability.

Modest Specs:

- Model: Dell OptiPlex 7080
- OS: Ubuntu 24.04.4 LTS (kernel 6.8.0-100-generic)
- CPU: Intel Core i5-10500T (6 cores / 12 threads)
- RAM: 8 GB
- Storage: 238.5 GB SSD (LVM root volume)

Before I could switch everything over (including the FM stack) to kaad-one, I needed to show that it worked. So, in the first wiring pass, board output was split into two paths:

1. Existing FM chain (unchanged)
2. **kaad-one** ingest path for streaming via a MOTU M2 USB interface

This let me initiate the modernization of the streaming side without disrupting on-air FM operations.

## Setting up the stream

After doing a clean install of Ubuntu 24.04.4 LTS, I tackled configuring the MOTU input channels and set up a test Myradiostream server, then configured Liquidsoap to do its thing. I actually did this in a roundabout way, since I was initially setting this up at home. My first tests were relaying the main stream to my test stream using Liquidsoap without the M2, but I will show the config here with the M2.

To get started I went through these main steps:

1. Confirm ALSA sees the MOTU M2
2. Create a stable ALSA alias (`motu_in`)
3. Build a minimal Liquidsoap config
4. Validate on MyRadioStream before changing station-facing output

### Confirm the ALSA device

After plugging in the M2, I started by verifying device names and capture paths on Ubuntu.

```bash
sudo apt update
sudo apt install -y alsa-utils liquidsoap

arecord -l
```

For the M2, I was looking for this:

```text
card 2: M2 [MOTU M2], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```
But if you are using a different usb audio interface, it will be different.

Then a quick capture sanity check:

```bash
arecord -D hw:M2,0 -f S16_LE -r 48000 -c 2 -d 5 /tmp/motu-test.wav
aplay /tmp/motu-test.wav
```
If you are running headless, you could also test the audio via software tools:

```bash
# Check file metadata after capture
ffprobe -hide_banner /tmp/motu-test.wav

# Check if there is real signal (not just silence)
ffmpeg -i /tmp/motu-test.wav -af volumedetect -f null -
```

For me, this was enough to validate the ingest path even without local speakers or a desktop session.

### Create a persistent ALSA alias: `motu_in`

Rather than hardcoding card indexes (which can shift on reboot), I added a named ALSA PCM in `/etc/asound.conf`:

```conf
pcm.motu_in {
  type plug
  slave {
    pcm "hw:0,0"
    format S32_LE
    rate 44100
    channels 2
  }
}

ctl.motu_in {
  type hw
  card 0
}
```

That gave me a stable device name Liquidsoap can target directly.

Quick validation:

```bash
arecord -D motu_in -f S16_LE -r 44100 -c 2 -d 5 /tmp/motu-alias-test.wav
aplay /tmp/motu-alias-test.wav
```

### Build the first Liquidsoap config

I started with a minimal config focused on capture stability and predictable output. Since free MyRadioStream accounts are limited to Shoutcast, the first external test used `output.shoutcast`.

Create `/etc/liquidsoap/kaad-test.liq`:

```text
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
Then I listened to it on my test stream, voila, it worked.

### Push to MyRadioStream as a safe external test

At this point, before any production cutover, I streamed to a free MyRadioStream test endpoint. That gave me a safe way to verify:

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

```text
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

## Systemd service

After I got it all working, I needed to set up a service to keep it running all the time. I used systemd to accomplish this.

Before creating the service, I made sure the runtime user existed and had access to audio devices:

```bash
sudo useradd -r -m -d /opt/kaad -s /usr/sbin/nologin kaad
sudo usermod -aG audio kaad
sudo mkdir -p /opt/kaad/bin
sudo chown -R kaad:audio /opt/kaad
```

If the `kaad` user already exists, only the `usermod` and ownership commands are needed.

I created `/etc/systemd/system/kaad-stream.service`:

```ini
[Unit]
Description=KAAD Liquidsoap stream service
After=network-online.target sound.target
Wants=network-online.target

[Service]
Type=simple
User=kaad
Group=audio
WorkingDirectory=/opt/kaad
Environment=HOME=/opt/kaad
ExecStart=/usr/bin/liquidsoap /opt/kaad/bin/kaad_stream.liq
Restart=always
RestartSec=3
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Then enabled and started it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now kaad-stream.service
sudo systemctl status kaad-stream.service
```

And for ongoing checks:

```bash
journalctl -u kaad-stream.service -f
```

This made the stream resilient to reboots and process crashes, and removed another manual step from station operations.

You can listen to the station stream here: [kaad-lp.org](https://kaad-lp.org).

## Result and next stage

KAAD now has a dedicated Linux ingest host, validated MOTU-to-stream signal flow, and a tested path from Shoutcast-limited proof-of-concept to Icecast-based output. The stream is now more reliable, auto reconnects, and provides a starting point for more.
