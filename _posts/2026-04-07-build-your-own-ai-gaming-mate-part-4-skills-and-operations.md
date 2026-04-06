---
layout: page
title: "Build Your Own AI Gaming Mate Part 4: Skills and Operations"
permalink: /build-your-own-ai-gaming-mate-part-4-skills-and-operations/
description: |
  Tune agent behavior, startup flow, and operational details so your AI gaming mate
  is stable and enjoyable to use across real play sessions.
category: build-your-own-ai-gaming-mate
---

## Introduction

This post covers the layer that makes the system feel good to use over time:
skills, orchestration, and operational polish.
The previous posts set up the video pipeline, observation layer, and audio path.
This one is about shaping the agent behavior that sits on top of all of that.

For an overview of the full system, see [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/).

## What This Part Does

At the end of this post, you should have:

1. A clear interactive skill structure that keeps responses fast and focused
2. Game-specific skills that make the companion feel aware of the game you are actually playing
3. A reply length policy that works for voice output during gameplay
4. A clear sense of when the companion should speak and when it should stay quiet
5. A lightweight approach to game state and memory across a play session
6. A repeatable startup sequence for the whole stack

## Why This Layer Matters

Without this layer, the system may work technically but still feel awkward.
A companion that takes too long to respond, speaks over critical gameplay moments,
or forgets what you discussed five minutes ago is more distraction than help.

The skill and operations layer is where you shape timing, tone, interruption behavior,
and recovery. It is the most opinionated part of the project and the hardest to copy
from someone else's setup directly, because it depends on the games you play and the
kind of interaction you want.

Think of this post as a set of principles with concrete examples, not a fixed recipe.

## Interactive Skill Design

OpenClaw skills split into two categories: interactive skills and agent skills.
Interactive skills are the backbone of the conversational loop.
They run synchronously — the agent processes your message, picks the right skill,
and replies.

The most important thing to get right is scope.
An interactive skill should do one thing and do it quickly.
A skill that tries to answer the question, look up lore, check the stream, and summarize
the match history in one pass will feel slow and unpredictable.

Design interactive skills around the questions you actually ask during play:

1. **React** — respond to a remark or short question without consulting external state
2. **Observe** — take a screenshot of the stream and describe what is on screen
3. **Recall** — answer a question using context already in the conversation

The react skill is the default.
It covers most of what you say during gameplay: short comments, quick questions,
and back-and-forth remarks.
Because it does not call any tools, it is fast.
Keep the model small and the temperature low for this skill.
You want consistency and speed, not creativity.

The observe skill is the one that uses the browser tool to take a screenshot.
Design it to be explicit: the companion only looks at the screen when you ask it to.
Ask "take a look" or "what do you see?" to trigger it.
Passive, unsolicited observation runs on every reply only if you want the latency and token cost.

The recall skill helps when you reference something from earlier in the session:
"what did you say about my gold income?" or "how many turns did you say I had?".
A good recall implementation just searches the conversation history.
It does not need to call any external tools unless your session transcripts live outside the context window.

## Game-Specific Skill Design

Generic companion behavior gets you halfway there.
The other half comes from skills that know something about the game you are playing.

A game-specific skill is just an interactive skill with a specialized prompt or a narrow
tool set. For example:

1. **Civ VI advisor** — knows the victory conditions, evaluates your position if you give it numbers, and avoids guessing exact tech counts from a screenshot alone
2. **Combat commentator** — knows the game's faction names, unit types, and common tactical patterns; uses the right vocabulary
3. **Quest tracker** — accepts updates you dictate ("I finished the main story quest") and keeps a short structured list in the conversation

You do not need to build all of these at once.
Start with a single game and a single skill that covers the questions you ask most.
Add vocabulary by listing proper nouns in the system prompt: hero names, faction names,
ability names, and any term that macOS dictation gets wrong.
Once the model knows the vocabulary, transcription errors become easier to spot and correct.

For games with publicly available data, you can include a short reference block in the
system prompt.
For example, a tech tree summary, a table of unit costs, or a list of faction bonuses.
Keep it short: a few dozen lines covers the facts the companion actually needs.
Anything longer starts to dilute the prompt and slow things down.

## Short Versus Long Replies

Reply length is one of the highest-leverage settings in the whole system.

The companion is speaking over a game.
Two to four sentences is the right range for almost every interactive reply.
Anything longer takes more time to generate, more time to speak, and is harder to follow
while you are still playing.

Add an explicit instruction to the system prompt:

```
Keep replies to three sentences or fewer.
If you need to say more, break it into a follow-up.
Shorter is better.
```

Short replies also reduce the awkward gap between asking something and being back in the game.
The companion speaks, stops, and you are still in the same moment.
A long reply means the game has moved on before the companion finishes speaking.

There is one exception: analysis on demand.
If you pause the game and ask for a detailed evaluation, longer is fine.
The companion can tell from context that you are waiting for a real answer.
You can give it explicit permission in the prompt:

```
For analysis questions where the user has paused or asked for a full breakdown,
longer replies are appropriate.
For all other replies, keep it to three sentences.
```

## When the Mate Should Speak and When It Should Stay Quiet

The companion should not volunteer observations constantly.
If it comments on every frame, every trade route, and every unit move, it becomes noise.

The simplest rule: speak when spoken to.
The companion replies when you send a message.
It does not interrupt on its own.

That rule alone handles most of the awkwardness.
The companion is not a co-pilot making live calls.
It is a friend on the couch — present, watching, ready to react when you say something,
but not narrating everything you do.

Beyond that baseline, there are two moments where proactive output makes sense:

1. **You ask it to watch something specific.** "Tell me if my score drops below 500." The companion can set a lightweight reminder in the session and flag it when you next interact.
2. **You ask for a hot-take after a major event.** "I just won the battle — what do you think?" Triggering commentary on demand is better than building a continuous event-detection loop.

Avoid building event detection that triggers the companion unprompted.
It is tempting — detecting "you just took damage" or "enemy hero spotted" sounds useful.
In practice, unsolicited interruptions during tense gameplay are more annoying than helpful.
Reserve proactive output for moments you specifically ask for it.

## Game State and Memory Management

The context window is the companion's working memory.
Everything it knows about the current session lives there.
This is both a constraint and a useful framing.

For most play sessions, the context window is large enough to hold the full conversation.
Do not over-engineer a memory system before you have actually hit the limit.
The first thing to optimize is keeping individual messages short,
which keeps the context window clean longer.

When you want the companion to remember a specific fact across turns, just tell it:

> Remember: I am playing as Brazil, aiming for a Culture victory, and I have a strong military lead.

The companion holds that in context and refers back to it throughout the session.
No special tool is needed.
This is more reliable than a structured memory system because it stays in the natural conversation flow.

When the session gets long enough that older context starts falling out, the simplest fix
is to ask the companion to summarize the session so far:

> Summarize what we have talked about in this session.

Paste that summary back as a user message at the start of the next session.
It costs one manual step but keeps the companion oriented without any custom state management code.

For game-specific structured state — quest progress, build orders, alliance tracking —
maintain a short plain-text note in Discord or a separate message.
Update it yourself as the session progresses, then paste it in when you want the companion
to use it.
The companion does not need write access to external state to be useful.
What it needs is accurate information at the time of the question.

## Startup Flow

The whole stack needs to start in a specific order, and that order matters.
If something starts in the wrong sequence, diagnosing the problem is annoying.
Make the startup order explicit and repeatable.

Recommended startup sequence:

1. **Windows machine:** start MediaMTX, then launch OBS and begin streaming
2. **macOS machine:** open Chrome and navigate to the stream URL, confirm playback
3. **macOS machine:** start the mlx-audio server if using local TTS
4. **macOS machine:** start the OpenClaw Gateway
5. **macOS machine:** open Discord and confirm the agent is connected

Verify each step before moving to the next.
A stream that is not publishing before OpenClaw starts means OpenClaw will open a blank tab.
An mlx-audio server that is not running before the agent starts means TTS calls will fail silently.

Create a short checklist in a note or sticky, or map it to a script.
Starting the same way every time removes one source of session-opening friction.

### Verifying the Loop Is Live

Once the stack is running, send one message in Discord and wait for a spoken reply.
That single end-to-end test confirms:

1. OBS is publishing
2. MediaMTX is serving
3. Chrome is showing the stream
4. OpenClaw can see the tab
5. Discord is delivering messages to the agent
6. TTS is producing audio

If any of those steps fails, the loop is broken.
Test them in order until you find the break.
Five minutes of startup verification saves a lot of debugging mid-session.

## Operational Stability Over Long Sessions

A session that works for ten minutes is not the same as one that holds up for three hours.
These are the failure modes that surface only with real play time.

**Stream lag accumulating in the browser tab:**
The MediaMTX WebRTC player can accumulate latency in the browser over a long session.
If the screenshot the companion takes looks noticeably behind the game, reload the tab.
The stream reconnects in a few seconds.

**TTS queue stalling:**
If the mlx-audio server stops responding, restart it without shutting down the Gateway.
The Gateway reconnects on the next TTS request.

**Context length approaching limits:**
Ask the companion to summarize the session when replies start losing context from earlier.
Paste the summary back in as a fresh starting point.

**Dictation accuracy degrading:**
macOS dictation accuracy can drop if the session has been running for a long time.
A restart of the dictation feature (disable and re-enable in System Settings) usually fixes it.

**Discord TTS play button moving:**
If Discord's message list scrolls, the play button for older replies moves off-screen.
Use the right pedal only for the most recent reply, or scroll the channel so the current reply is visible before pressing the pedal.

## Links

- [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/)
- [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/)
- [Part 2: OpenClaw Browser Use](/build-your-own-ai-gaming-mate-part-2-openclaw-browser-use/)
- [Part 3: Audio Pipeline](/build-your-own-ai-gaming-mate-part-3-audio-pipeline/)
- [OpenClaw Skills documentation](https://docs.openclaw.ai/skills)
- [OpenClaw configuration reference](https://docs.openclaw.ai/gateway/configuration-reference)
