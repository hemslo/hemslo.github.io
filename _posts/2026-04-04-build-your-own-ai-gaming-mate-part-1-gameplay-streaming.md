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

- Output resolution: 1280x720
- Frame rate: 30 fps

Higher resolutions and frame rates are possible, but downstream steps in the pipeline will resize frames before sending them to the model anyway.
Starting at 720p keeps encoding overhead low and makes it easier to spot frame drop issues early.

In OBS Settings, go to Output and confirm the encoder is set to a hardware encoder if one is available (e.g., Apple VT H264 on macOS, NVENC on NVIDIA hardware).
Software encoding at 720p 30fps is fine for this step, but hardware encoding reduces the CPU load on the same machine.

## Step 2: MediaMTX Setup

Download the MediaMTX binary for your platform from the [MediaMTX releases page](https://github.com/bluenviron/mediamtx/releases).

Create a `mediamtx.yml` config file with the following minimum settings:

```yaml
paths:
  gameplay:
    source: publisher
```

This defines a single path called `gameplay` that accepts a published stream and holds it for readers.
The default ports MediaMTX uses are:

- `8554` for RTSP
- `8889` for WebRTC (HTTP-based signaling)
- `8888` for HLS

For this pipeline, only the WebRTC port matters.
You can disable the others if you want a smaller surface, but leaving defaults in place is fine for local use.

Start MediaMTX:

```bash
./mediamtx mediamtx.yml
```

You should see log output confirming the server started and the `gameplay` path is ready.

## Step 3: WebRTC Publish Flow

In OBS, go to Settings → Stream.

Set the following:

- Service: **WHIP**
- Server: `http://localhost:8889/gameplay/whip`
- Bearer Token: leave blank (not required for local MediaMTX without auth)

Click Apply and then start the stream in OBS.
OBS will negotiate a WebRTC session with MediaMTX and begin pushing frames.

If OBS does not show a WHIP option in your version, update to OBS 30 or later.
WHIP support was added in OBS 30.0.0.

## Verification

Once OBS is streaming and MediaMTX is running, open a browser and go to:

```
http://localhost:8889/gameplay
```

MediaMTX serves a built-in WebRTC player at that URL.
If you can see your gameplay in the browser, the full publish-to-read path is working.

For a non-browser check, you can use `ffprobe` to inspect the stream:

```bash
ffprobe -v quiet -print_format json -show_streams http://localhost:8888/gameplay/index.m3u8
```

This hits the HLS endpoint instead of WebRTC, but it confirms MediaMTX is receiving and serving the stream.
A successful response will include stream metadata with the video codec, resolution, and frame rate you configured in OBS.

## Failure Modes

**Stream not connecting**

Check that MediaMTX is running and the `gameplay` path is defined in your config.
OBS will silently fail to connect if the WHIP URL is wrong or MediaMTX is not listening.
Look at the MediaMTX log for incoming connection attempts.

**Wrong codec or browser compatibility issue**

MediaMTX and OBS default to H.264 for WebRTC, which has the widest browser support.
If you see a blank player in the browser, check whether OBS is encoding with H.265 or AV1.
Force H.264 in OBS encoder settings if you run into codec mismatches.

**High latency**

WebRTC should give you sub-second latency on a local connection.
If you are seeing delays of several seconds, check whether OBS is falling back to an HLS or RTMP path.
Confirm the stream is going through the WHIP URL, not a different output plugin.

**Resolution too high for the rest of the pipeline**

Downstream, the agent resizes frames before passing them to the model.
If you capture at 4K and the agent is slow, reduce the OBS output resolution to 1280x720.
The model does not need the extra pixels, and the lower resolution will reduce both encode and decode overhead.

**Session drops after long play sessions**

MediaMTX keeps the session alive as long as OBS is publishing.
If you see drops after 30 to 60 minutes, check system sleep settings and network interface power management.
On macOS, App Nap can throttle background processes.
Keep OBS in the foreground or disable App Nap for OBS using `defaults write -g NSAppSleepDisabled -bool YES`.

## Notes for the Final Draft

Keep this post practical.
Readers should be able to stop here and still have a working local gameplay stream.
