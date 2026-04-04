---
layout: page
title: "Build Your Own AI Gaming Mate Part 1: Gameplay Streaming"
permalink: /build-your-own-ai-gaming-mate-part-1-gameplay-streaming/
description: |
  Set up the gameplay video pipeline for a local AI gaming mate using OBS,
  WebRTC, and MediaMTX.
series: local-ai-gaming-mate
part: 1
published: false
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

## Why This Layer Exists

The most obvious alternative to this stack is direct screen capture inside the agent process itself.
That approach works for one-off demos but breaks down in practice.
Screen capture libraries vary by platform, introduce their own latency, and tie the capture lifecycle to the agent process.
If the agent crashes or restarts, you lose the capture context.

OBS separates the capture concern completely.
It handles the platform-specific work, lets me tune the output independently, and keeps running even when I restart or redeploy the agent.
WebRTC over MediaMTX gives me a local HTTP endpoint that any process can attach to without caring about how the capture is done.
That separation is the whole point.
If something feels wrong with the image quality or the frame rate, I can fix it in OBS without touching the agent code.

## Architecture for This Step

```text
Game -> OBS -> WebRTC -> MediaMTX
```

## Prerequisites

- OBS Studio 30 or later (WebRTC output support is built in)
- MediaMTX 1.9 or later
- macOS, Linux, or Windows host with a GPU capable of capturing at your target resolution
- The game, OBS, and MediaMTX all running on the same local machine or local network
- Basic familiarity with OBS scenes and sources

No cloud account or external port forwarding is required.
Everything in this post stays on your local machine.

## Step 1: OBS Setup

Create a new scene in OBS and add a Game Capture or Display Capture source depending on how your game runs.
Game Capture works best for games running in a window or full-screen exclusive mode on Windows.
Display Capture is more portable and works well on macOS.

Resolution and frame rate settings that work well for this pipeline:

- Output resolution: 1920x1080
- Frame rate: 30 fps

In OBS Settings, go to Output and confirm the encoder is set to a hardware encoder if one is available (e.g., Apple VT H264 on macOS, NVENC on NVIDIA hardware).
Software encoding at 1080p 30fps is fine for this step, but hardware encoding reduces the CPU load on the same machine.

## Step 2: MediaMTX Setup

Download the MediaMTX binary for your platform from the [MediaMTX releases page](https://github.com/bluenviron/mediamtx/releases).

The default configuration accepts streams on any path, so no custom config file is required to get started.
Start MediaMTX with the default config:

```bash
./mediamtx
```

You should see log output confirming the server started.
The WebRTC endpoint listens on port `8889` by default.

## Step 3: WebRTC Publish Flow

In OBS, go to Settings → Stream.

Set the following:

- Service: `WHIP`
- Server: `http://localhost:8889/gameplay/whip`

Save the configuration and click `Start streaming`.

OBS will negotiate a WebRTC session with MediaMTX and the stream will be available on path `/gameplay`.

For more details, see the [MediaMTX WebRTC clients documentation](https://mediamtx.org/docs/publish/webrtc-clients) and the [OBS Studio guide](https://mediamtx.org/docs/publish/obs-studio).

If OBS does not show a WHIP option in your version, update to OBS 30 or later.
WHIP support was added in OBS 30.0.0.

## Verification

Once OBS is streaming and MediaMTX is running, open a browser and go to:

```
http://localhost:8889/gameplay
```

MediaMTX serves a built-in WebRTC player at that URL.
If you can see your gameplay in the browser, the full publish-to-read path is working.

## Notes for the Final Draft

Keep this post practical.
Readers should be able to stop here and still have a working local gameplay stream.
