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
The cleanest option for OpenClaw is a browser tab: Chrome handles WebRTC playback natively, and OpenClaw can control a Chrome session directly without any custom plumbing.

The alternative would be to point OpenClaw at the MediaMTX WebRTC endpoint directly and have the agent consume raw video frames.
That path is more complex to set up and harder to debug.
A Chrome tab gives me a visible surface I can inspect myself, and OpenClaw's browser tool gives the agent access to the same surface.

The other reason is session persistence.
Chrome tabs survive a MediaMTX restart better than most custom clients.
If I restart the stream, I can reload the tab and the agent picks up again without any extra configuration.

## Architecture for This Step

OpenClaw supports two ways to connect to Chrome, described in the sections below.

**Option A — `openclaw` profile (fresh instance):**

```text
MediaMTX -> OpenClaw-managed Chrome tab -> OpenClaw -> agent
```

OpenClaw launches and fully manages its own Chrome instance.

**Option B — `user` profile (attach to existing session):**

```text
MediaMTX -> your Chrome tab -> Chrome DevTools MCP -> OpenClaw -> agent
```

OpenClaw attaches to the Chrome session you already have open via Chrome DevTools MCP.

In both cases OpenClaw does not read from MediaMTX directly.
It observes the browser tab that is playing the stream.

## Prerequisites

- Chrome installed on macOS
- OpenClaw installed and the Gateway running locally
- The MediaMTX stream from Part 1 is reachable from the macOS machine at `http://IP:8889/mystream`

For the `user` profile only:

- Chrome 144 or later (Chrome DevTools MCP support)

## Option A: `openclaw` Profile (Fresh Chrome Instance)

This is the simpler path.
OpenClaw launches its own Chrome instance, isolated from your personal profile.
No cookies, extensions, or login state from your daily browser carry over.
That isolation is also the limitation: if you need an authenticated session you would use Option B instead.

### Step 1: Open the Stream in the OpenClaw Browser

Start the OpenClaw-managed browser and navigate to the stream URL:

```shell
openclaw browser --browser-profile openclaw open http://IP:8889/mystream
```

Replace `IP` with the Windows machine's local IP address or its Tailscale IP.
OpenClaw launches a dedicated Chrome window, opens the page, and keeps the connection live.

Verify that the tab is visible to the agent:

```shell
openclaw browser --browser-profile openclaw tabs
```

The output should list the stream tab with its title and URL.
The stream will not autoplay until OBS is publishing, so start OBS first.

### Step 2: Keep the Session Running

The OpenClaw-managed browser stays open as long as the Gateway is running.
If you need to restart, run the `open` command again.
There is no persistent tab state between sessions.

---

## Option B: `user` Profile (Attach to Existing Chrome Session)

This path lets OpenClaw observe the Chrome session you already have open,
using [Chrome DevTools MCP](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session).
You keep your existing tabs, login state, and extensions.
The tradeoff is that Chrome must be running and you need to approve the connection the first time.

### Step 1: Open the Stream Tab in Chrome

On the macOS machine, open Chrome and navigate to the MediaMTX WebRTC player URL:

```
http://IP:8889/mystream
```

Replace `IP` with the Windows machine's local IP address or its Tailscale IP.
Keep this tab open and do not navigate away from it during a play session.
If you open a dedicated Chrome window just for the stream, it is easier to manage focus separately from your regular browsing.

### Step 2: Enable Chrome DevTools MCP

Chrome DevTools MCP allows external tools to attach to your running Chrome session.
In Chrome, go to `chrome://inspect/#remote-debugging` and follow the prompts to enable remote debugging.

Chrome will show a confirmation dialog the first time OpenClaw tries to connect, and a banner will appear while the session is active.
You need to approve the connection each time.

### Step 3: Connect OpenClaw to Your Session

Use the `user` profile to list tabs in your current Chrome session:

```shell
openclaw browser --browser-profile user tabs
```

The output should include the stream tab.
If the tab does not appear, confirm that Chrome is running and that you accepted the DevTools MCP connection dialog.

---

## Practical Issues

**Tab lifecycle**

The stream tab must stay open for the agent to observe it.
If you close or navigate away from the tab, the agent loses its observation surface.
I keep a separate Chrome window with only the stream tab open, minimized but not closed.
Minimized windows still render WebRTC video on macOS, unlike some other platforms.

**Tab focus on macOS**

macOS routes keyboard events to the focused window.
If OpenClaw needs to interact with the tab (for example, to trigger a reload), it may steal focus from the game window.
For a pure observation setup where the agent only reads the stream, this is not a problem.
If you add interaction later, you will want to think about how focus changes affect the game input.

**Reconnecting after a browser restart**

For the `openclaw` profile, run `openclaw browser --browser-profile openclaw open http://IP:8889/mystream` again after a restart.

For the `user` profile, open the stream tab in Chrome again and accept the DevTools MCP connection prompt.
Then confirm with `openclaw browser --browser-profile user tabs`.
There is no automatic reconnect in either case.

## Failure Modes

**Tab does not appear in `openclaw browser tabs`**

For the `openclaw` profile: confirm the Gateway is running and that you ran the `open` command to start the managed browser.

For the `user` profile: confirm Chrome is running, that you navigated to the stream URL, and that you accepted the Chrome DevTools MCP connection dialog.

**Attached to the wrong tab**

The `openclaw browser tabs` command shows exactly which tabs are visible.
If the URL is not the MediaMTX stream page, close the wrong tab or navigate the correct tab to the stream URL and retry.

**Stream visible in browser but not useful to the agent**

The agent observes the rendered frame, not the raw video stream.
If the stream is paused, buffering, or showing a spinner, the agent sees that frame instead of gameplay.
Keep the stream in an unpaused, healthy state before starting a session.
The MediaMTX player page will not autoplay if there is no active OBS publish, so always start OBS before opening the agent session.

**Reconnects after browser restart**

There is no automatic reconnect.
For the `openclaw` profile, run the `open` command again.
For the `user` profile, open the stream tab, accept the DevTools MCP prompt, and confirm with `openclaw browser --browser-profile user tabs`.

**Browser UI getting in the way**

The MediaMTX player page is a full-page video element with minimal UI.
Browser chrome (address bar, bookmarks, tab bar) does not overlap the video area in a maximized or fullscreen window.
If the agent starts navigating away from the tab for any reason, reload the stream URL and re-verify the tab list.

## Links

- [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/)
- [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/)
- [OpenClaw Browser Tool documentation](https://docs.openclaw.ai/tools/browser)
