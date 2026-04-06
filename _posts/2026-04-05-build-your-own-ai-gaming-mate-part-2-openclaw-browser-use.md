---
layout: page
title: "Build Your Own AI Gaming Mate Part 2: OpenClaw Browser Use"
permalink: /build-your-own-ai-gaming-mate-part-2-openclaw-browser-use/
description: |
  Connect OpenClaw to a Chrome session so it can observe the local gameplay stream
  delivered by MediaMTX.
category: build-your-own-ai-gaming-mate
---

## Introduction

This post covers the observation layer:
how OpenClaw attaches to a Chrome session and watches the game stream there.
For an overview of the full system, see [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/).
For the video pipeline setup, see [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/).

## What This Part Does

At the end of this post, OpenClaw should be able to:

1. Attach to the right Chrome tab showing the MediaMTX stream
2. Take a screenshot of the stream on demand and describe what is on screen

This setup gives OpenClaw access to the game stream — it does not watch every frame continuously.
The agent only looks at the stream when you ask it to.
If you ask "what choices should I pick?", OpenClaw takes a screenshot of the tab and replies based on what it sees.
You can establish whatever convention works for you, such as saying "take a look" to trigger a screenshot.
Continuous stream watching will be explored in a later post.

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

## End-to-End Verification

Once OBS is publishing and the Gateway is running, you can hand the entire setup to OpenClaw with a single prompt:

> open `http://IP:8889/mystream` in my chrome and watch my game stream

OpenClaw will pick a profile, open the URL, and confirm when it can see the stream.

To verify at any point during a session, just ask:

> can you see my game stream in chrome?

OpenClaw should respond with a screenshot of the current tab and a short description of what is on screen — for example, the game title, the current scene, or the match state.
If it cannot see anything useful (spinner, blank frame, wrong tab), its reply will say so and you can reload the tab or restart OBS before trying again.

## Links

- [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/)
- [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/)
- [OpenClaw Browser Tool documentation](https://docs.openclaw.ai/tools/browser)
