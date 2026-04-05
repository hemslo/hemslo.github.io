---
layout: page
title: "Build Your Own AI Gaming Mate Part 2: OpenClaw Browser Use"
permalink: /build-your-own-ai-gaming-mate-part-2-openclaw-browser-use/
description: |
  Connect OpenClaw to a Chrome session so it can observe the local gameplay stream
  delivered by MediaMTX.
series: local-ai-gaming-mate
part: 2
---

## Introduction

This post covers the observation layer:
how OpenClaw attaches to a Chrome session and watches the game stream there.
For an overview of the full system, see [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/).
For the video pipeline setup, see [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/).

## What This Part Does

At the end of this post, OpenClaw should be able to:

1. Attach to the right Chrome tab showing the MediaMTX stream
2. See the stream consistently across a full play session
3. Reconnect cleanly when the browser restarts

## Why Chrome Is in the Loop

After MediaMTX receives the stream from OBS, something has to decode and render the video so the agent can observe it.
The cleanest option for OpenClaw is a browser tab: Chrome handles WebRTC playback natively, and OpenClaw has a first-class extension that can attach to existing tabs without spinning up a separate headless browser.

The alternative would be to point OpenClaw at the MediaMTX WebRTC endpoint directly and have the agent consume raw video frames.
That path is more complex to set up and harder to debug.
A Chrome tab gives me a visible surface I can inspect myself, and the OpenClaw extension gives the agent access to the same surface without any custom plumbing.

The other reason is session persistence.
Chrome tabs survive a MediaMTX restart better than most custom clients.
If I restart the stream, I can reload the tab and the agent picks up again without any extra configuration.

## Architecture for This Step

```text
MediaMTX -> Chrome tab -> OpenClaw Chrome extension -> OpenClaw Gateway -> agent
```

OpenClaw does not read from MediaMTX directly.
It observes the tab through the Chrome Debugger API, which the extension relays to the OpenClaw Gateway running locally on macOS.

## Prerequisites

- Chrome installed on macOS
- OpenClaw installed and the Gateway running locally
- `OPENCLAW_GATEWAY_TOKEN` set (or `gateway.auth.token` in the OpenClaw config)
- The MediaMTX stream from Part 1 is reachable from the macOS machine at `http://IP:8889/mystream`

## Step 1: Open the Stream in Chrome

On the macOS machine, open Chrome and navigate to the MediaMTX WebRTC player URL:

```
http://IP:8889/mystream
```

Replace `IP` with the Windows machine's local IP address or its Tailscale IP.
The page will start playing the stream automatically once OBS is publishing.

Keep this tab open and do not navigate away from it during a play session.
The tab is the observation surface that OpenClaw will attach to.
If you open a dedicated Chrome window just for the stream, it is easier to manage focus separately from your regular browsing.

## Step 2: Install the OpenClaw Chrome Extension

OpenClaw ships an extension that relays Chrome's debugger output to the local Gateway.
Install the extension files with:

```shell
openclaw browser extension install
openclaw browser extension path
```

The second command prints the directory you need to load in Chrome.

In Chrome, go to `chrome://extensions` and enable **Developer mode** in the top-right corner.
Click **Load unpacked** and select the directory printed by the previous command.
Pin the extension icon to the toolbar so it is easy to click during a session.

The first time you open the extension options, set two values:

- **Port:** `18792` (the default relay port)
- **Gateway token:** the same value as `OPENCLAW_GATEWAY_TOKEN` in your environment

Save the options.
The extension needs to match the local Gateway's token to relay debugger events.
If the token does not match, the extension badge will show `!` instead of connecting.

## Step 3: Create a Browser Profile

Create a named profile so you can reference this Chrome session consistently from the agent:

```shell
openclaw browser create-profile \
  --name my-chrome \
  --driver extension \
  --cdp-url http://127.0.0.1:18792 \
  --color "#00AA00"
```

The `--driver extension` flag tells OpenClaw to use the Chrome extension relay rather than launching a managed browser of its own.
The `--cdp-url` points at the relay port the extension listens on.
The `--color` flag is cosmetic but useful when you have multiple profiles.

You only need to run this once.
The profile is saved in your OpenClaw configuration.

## Step 4: Attach to the Stream Tab

Switch to the Chrome tab showing the MediaMTX stream.
Click the OpenClaw extension icon in the toolbar.
The badge should change to **ON** to confirm the tab is attached.

To detach, click the icon again.
Only attached tabs are visible to the agent.
OpenClaw does not have blanket access to every open tab.

Verify that the agent can see the tab:

```shell
openclaw browser --browser-profile my-chrome tabs
```

The output should list the attached tab with its title and URL.
If the tab does not appear, check that the extension badge shows **ON** and that the Gateway is running.

## Practical Issues

**Tab focus on macOS**

macOS routes keyboard events to the focused window.
If OpenClaw needs to interact with the tab (for example, to trigger a reload), it may steal focus from the game window.
For a pure observation setup where the agent only reads the stream, this is not a problem.
If you add interaction later, you will want to think about how focus changes affect the game input.

**Tab lifecycle**

The stream tab must stay open for the agent to observe it.
If you close or navigate away from the tab, the agent loses its observation surface.
I keep a separate Chrome window with only the stream tab open, minimized but not closed.
Minimized windows still render WebRTC video on macOS, unlike some other platforms.

**Reconnecting after a browser restart**

If Chrome restarts, the extension reloads automatically, but you need to reattach the tab manually.
Open the stream tab again, click the extension icon, and verify with `openclaw browser tabs`.
There is no automatic reconnect on tab close or Chrome restart.

**Extension reloads after OpenClaw upgrades**

When you upgrade OpenClaw, rerun `openclaw browser extension install` and then reload the extension at `chrome://extensions`.
The relay port and token settings are preserved, but the extension files themselves need to be updated.

**macOS security prompts**

On first use, macOS may prompt you to allow Chrome to access the local network.
Allow it.
The extension relay communicates over localhost (`127.0.0.1`), so the prompt is about loopback access, not external network exposure.

## Failure Modes

**Extension badge shows `!` or `…`**

This means the extension cannot reach the Gateway or the token does not match.
Check that the Gateway is running and that the token in the extension options matches `OPENCLAW_GATEWAY_TOKEN`.

**Attached to the wrong tab**

The `openclaw browser tabs` command shows exactly which tab is attached.
If the URL is not the MediaMTX stream page, click the extension icon to detach, switch to the correct tab, and attach again.
Avoid keeping the extension attached to other tabs at the same time.

**Stream visible in browser but not useful to the agent**

The agent observes the rendered frame, not the raw video stream.
If the stream is paused, buffering, or showing a spinner, the agent sees that frame instead of gameplay.
Keep the stream in an unpaused, healthy state before starting a session.
The MediaMTX player page will not autostart if there is no active OBS publish, so always start OBS before opening the agent session.

**Reconnects after browser restart**

There is no automatic reconnect.
After a Chrome restart, open the stream tab, click the extension icon to reattach, and confirm with `openclaw browser tabs`.

**Browser UI getting in the way**

The MediaMTX player page is a full-page video element with minimal UI.
Browser chrome (address bar, bookmarks, tab bar) does not overlap the video area in a maximized or fullscreen window.
If the agent starts navigating away from the tab for any reason, reload the stream URL and reattach.

## Links

- [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/)
- [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/)
- [OpenClaw Browser Tool documentation](https://docs.openclaw.ai/tools/browser)
