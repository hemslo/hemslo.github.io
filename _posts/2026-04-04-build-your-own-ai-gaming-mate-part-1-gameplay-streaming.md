---
layout: page
title: "Build Your Own AI Gaming Mate Part 1: Gameplay Streaming"
permalink: /build-your-own-ai-gaming-mate-part-1-gameplay-streaming/
description: |
  Set up the gameplay video pipeline for a local AI gaming mate using OBS,
  WebRTC, and MediaMTX.
series: local-ai-gaming-mate
part: 1
---

## Introduction

This post covers the video pipeline for the gaming mate setup.
The goal is to get gameplay from OBS into a local endpoint that the rest of the system can consume reliably.

## What This Part Does

At the end of this post, you should have:

1. OBS capturing your gameplay
2. WebRTC publishing enabled
3. MediaMTX receiving the stream locally
4. A repeatable way to verify that the stream is healthy

## What I Tried First

Before settling on OBS + MediaMTX, I went through a couple of simpler options.

**Google Meet screen share**

The first attempt was just sharing the game screen in a Google Meet call.
It requires zero setup and works from any browser.
The problem is that latency and quality are both at the mercy of Google's servers.
Even on a local network the stream routes through the cloud, which adds unpredictable delay and makes it unsuitable for anything that needs to react to what is happening on screen.

**Screego**

[Screego](https://screego.net) is a self-hosted WebRTC screen-sharing server.
Both the game machine and the macOS machine connect through a browser, no extra software required.
This was promising until I noticed the resource usage.
Screego publishes at display resolution, and my display is 4K.
Encoding and decoding a 4K WebRTC stream on the same machine running the game was too expensive and caused frame drops in both the game and the agent.

**OBS + MediaMTX**

OBS lets me set an explicit output resolution independent of the display.
Publishing at 1080p 30fps uses a fraction of the resources that a raw 4K screen capture would.
MediaMTX acts as a local relay so any process on the same machine can read the stream without going through a browser at all.
This combination is generic enough to work with any game and stable enough to leave running in the background.

## Architecture for This Step

```text
Game -> OBS -> WebRTC (WHIP) -> MediaMTX   [Windows machine]
                                    |
                               browser / agent  [macOS machine]
```

## Prerequisites

- OBS Studio 30 or later (WebRTC/WHIP output support is built in) — Windows
- MediaMTX 1.9 or later — Windows
- The agent and browser — macOS
- Both machines on the same local network

## Step 1: OBS Setup

[OBS Studio](https://obsproject.com) is a free, open-source screen recording and live streaming application.
It is widely used by streamers but works just as well as a local capture tool.
OBS gives you control over output resolution and encoder independently of your display, which is exactly what this pipeline needs.

Create a new scene in OBS and add a Game Capture source for the game.
Game Capture works best for games running in a window or full-screen exclusive mode on Windows.

Resolution and frame rate settings that work well for this pipeline:

- Output resolution: 1920x1080
- Frame rate: 30 fps

In OBS Settings, go to Output and confirm the encoder is set to a hardware encoder if one is available (e.g., NVENC on NVIDIA hardware).
Software encoding at 1080p 30fps is fine for this step, but hardware encoding reduces the CPU load on the same machine.

## Step 2: MediaMTX Setup

[MediaMTX](https://mediamtx.org) is a lightweight, zero-dependency media server that supports WebRTC, RTSP, HLS, and more.
It acts as a local relay: OBS pushes the stream to it, and any client on the network can pull it back out.

Install MediaMTX using winget:

```bat
winget install -e --id bluenviron.mediamtx
```

Or download the binary manually from the [MediaMTX releases page](https://github.com/bluenviron/mediamtx/releases).

Start MediaMTX:

```bat
mediamtx.exe
```

You should see log output confirming the WebRTC listener started on port `8889`.

Start MediaMTX before OBS.
The stream path is created dynamically when OBS begins publishing, so there is nothing to pre-configure.

## Step 3: WebRTC Publish Flow

In OBS, go to Settings → Stream.

Set the following:

- Service: `WHIP`
- Server: `http://localhost:8889/mystream/whip`

Save the configuration and click `Start streaming`.

OBS will negotiate a WebRTC session with MediaMTX and the stream will be available on path `/mystream`.

For more details, see the [MediaMTX WebRTC clients documentation](https://mediamtx.org/docs/publish/webrtc-clients) and the [OBS Studio and WebRTC guide](https://mediamtx.org/docs/publish/obs-studio#obs-studio-and-webrtc).

If OBS does not show a WHIP option in your version, update to OBS 30 or later.
WHIP support was added in OBS 30.0.0.

## Verification

Start MediaMTX first, then start OBS streaming.
Once OBS is publishing, open a browser on the macOS machine and go to:

```
http://IP:8889/mystream
```

Replace `IP` with the Windows machine's local IP address.
MediaMTX serves a built-in WebRTC player at that URL.
The page will not show a stream until OBS has started publishing — opening it before OBS starts streaming will show nothing.
If you can see your gameplay in the browser, the full publish-to-read path is working.

**Network access note:** if the macOS machine cannot reach the stream, the Windows firewall may be blocking port `8889`.
Add an inbound rule to allow TCP/UDP on that port, or skip firewall changes entirely by using [Tailscale](https://tailscale.com) — replace `IP` with the Windows machine's Tailscale IP and the traffic will tunnel through the VPN without touching firewall rules.
