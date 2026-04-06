---
layout: page
title: "Build Your Own AI Gaming Mate Part 3: Audio Pipeline"
permalink: /build-your-own-ai-gaming-mate-part-3-audio-pipeline/
description: |
  Add voice input, TTS options, and hands-free controls using macOS dictation,
  Discord, system or hosted TTS, Qwen3-TTS with MLX Audio, and a 3-key deck pedal.
category: build-your-own-ai-gaming-mate
---

## Introduction

This post covers the voice and control layer around the gaming mate.
The main distinction here is not whether audio exists at all, but whether you self-host it.
You can use built-in TTS or an online provider if that is good enough.
I use a local Qwen3-TTS setup when I want tighter control over the voice itself,
and I use macOS dictation plus a foot pedal to make the interaction less awkward during play.

For an overview of the full system, see [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/).

## What This Part Does

At the end of this post, you should have:

1. A low-friction way to speak naturally during gameplay
2. A clear picture of your TTS options, from built-in voices to a self-hosted local server
3. Hands-free shortcut control using a 3-key deck pedal
4. A clear understanding of when local Qwen3-TTS is worth the extra setup

## TTS Options

There are a few reasonable ways to do voice output:

1. Built-in system TTS if you want the fastest path
2. An online TTS provider if you want better voice quality with less local setup
3. Qwen3-TTS plus `mlx-audio` if you want a self-hosted voice you can customize more deeply

The reason I keep the local Qwen3-TTS path is not that other options are unusable.
It is that local hosting gives me more control over the exact voice I want to hear.

## Why This Setup

The combination of macOS dictation, Discord, and a foot pedal looks unconventional, but each piece earns its place.

macOS dictation is built in.
It requires no additional installation and already handles a wide vocabulary reasonably well.
The downside is that it needs a keyboard shortcut to activate, which is why the foot pedal matters.

Discord is the interaction transport.
It gives me a typed message thread where I can see the exact words the agent received, which makes it easy to spot bad transcriptions.
It also means the input path is the same whether I am typing or speaking: a Discord message goes to the agent either way.
Keeping the transport uniform makes the rest of the agent logic simpler.

The deck pedal removes the last awkward moment in the loop.
Without it, activating dictation means taking a hand off the controller or keyboard.
With it, I press a pedal with my foot, speak, and press again to send — without touching anything else.

That combination is why this setup feels usable during real gameplay rather than just during a demo.
None of the three pieces alone is enough.
Together, they make the voice input path feel as fast and low-friction as typing.

## Architecture for This Step

```text
Audio input
  -> press pedal key 1 (activate macOS dictation)
  -> speak
  -> press pedal key 2 (stop dictation and copy text)
  -> press pedal key 3 (paste into Discord and send)
  -> Discord message
  -> local agent

Audio output
  -> local agent text
  -> system TTS, online TTS, or Qwen3-TTS via mlx-audio
  -> spoken reply
```

## Prerequisites

**macOS dictation:**

- macOS Ventura or later
- Enable dictation in System Settings → Keyboard → Dictation
- Set a shortcut that does not conflict with your games or applications
- Enable "Auto-punctuation" if you want cleaner transcriptions
- Microphone access must be granted to the system

**Discord:**

- A Discord server and channel dedicated to the gaming mate session
- The agent connected to that channel (covered in previous posts)
- Discord desktop app, not the browser version — focus behavior is more predictable

**TTS (built-in path):**

- macOS `say` command is available out of the box
- No additional setup required

**TTS (online provider path):**

- An API key from your chosen provider (ElevenLabs, OpenAI TTS, or similar)
- Network access during play

**TTS (local Qwen3-TTS path):**

- An Apple Silicon Mac with enough memory to run the model
- [mlx-audio](https://github.com/Blaizzy/mlx-audio) installed via pip

```shell
pip install mlx-audio
```

- Qwen3-TTS weights downloaded through mlx-audio

**Foot pedal:**

- A 3-key USB deck pedal (any programmable HID pedal will work)
- The pedal driver or macOS keyboard shortcut mapping tool

**Audio routing:**

- If the game runs on a separate Windows machine, audio from the TTS plays on macOS, so there is no routing conflict
- If both run on the same machine, use a virtual audio device or lower the game audio when a reply plays

## Input Path

The voice input side is a three-step loop: activate dictation, speak, send.

### Why Discord Is Still in the Loop

Even with voice input enabled, Discord stays as the transport.
The agent receives a typed message regardless of whether you dictated or typed it.
That means you get a visible transcript of everything you said, which is useful for two reasons.

First, you can see exactly what the agent received.
If it gives an odd response, you can check whether the transcript is correct before assuming the agent is wrong.
Second, you can scroll back through the session and see the full conversation, which is easier to review than an audio log.

### Push-to-Activate Behavior

macOS dictation works in two modes: a manual toggle or a double-press shortcut.
I use the manual toggle bound to the foot pedal rather than the double-press.
Double-press can fire accidentally during gameplay, especially with fast keyboard activity.
A dedicated foot pedal key is harder to trigger by mistake.

The activation shortcut in macOS puts the dictation overlay in whatever app currently has focus.
If Discord is in the background, the shortcut will activate dictation in the wrong app.
Solve this by bringing Discord to focus with a second shortcut before activating dictation,
or by keeping Discord always visible in a corner of the screen.

### Message Length

Keep messages short.
Dictation accuracy drops on long sentences, and the agent response is also longer when the question is longer.
The sweet spot is one to three sentences: ask one thing, get one answer, continue playing.

If you have a longer question, type it.
The foot pedal workflow is optimized for quick back-and-forth, not for dictating paragraphs.

### Recovering From Bad Transcription

When dictation produces something garbled, the simplest fix is to say "ignore that" and repeat the message.
The agent handles it cleanly because it treats the conversation as a dialogue, not a command sequence.
You can also type a correction directly into Discord if the error is significant.

Game-specific terms cause the most trouble.
Hero names, ability names, and place names that are not common English words will often come out wrong.
Build up a short list of corrections for the terms you use most and use Discord's message edit to fix them before sending.

### Staying Usable During Intense Gameplay

The pedal workflow interrupts only for the moment you are speaking.
One foot press starts dictation, one press stops it.
After that, the message is sent automatically and you are back in the game before the agent starts replying.

During intense moments, it is fine to say nothing.
The agent does not interrupt unless you speak first.
Save the conversation for breaks between fights, loading screens, and menu navigation.

## Output Path

The voice output side is where the tradeoffs between convenience and control are most visible.

### When Built-in TTS Is Enough

macOS `say` is available immediately with no configuration.
The voice quality is functional but not particularly characterful.
For validating the end-to-end loop, it is more than sufficient.

To make the agent use `say`, configure it to pipe reply text through the command:

```shell
say -v Samantha "your reply text here"
```

Choose a voice that is easy to understand at medium volume with game audio in the background.
The built-in `Samantha` or `Tom` voices are clear at conversational pace.
Avoid voices with heavy accent styling if your game audio is busy.

The main limitation of built-in TTS is that the voice does not feel like a specific character.
It sounds like a utility, not a companion.
For longer play sessions, that gap becomes noticeable.

### When an Online Provider Is a Better Tradeoff

An online TTS provider gives you better voice quality with a small amount of latency and a recurring cost.
ElevenLabs and OpenAI TTS are both easy to integrate.
The voice library is wide, and you can find one that suits the companion personality you want.

The tradeoff is privacy and latency.
Every reply text is sent to the provider's API before it is spoken.
Network latency adds roughly 300–800 ms depending on the provider and region.
For casual conversation this is not noticeable, but for short reactive replies it can feel slow.

If the extra latency is acceptable and you do not want the local setup cost, an online provider is a reasonable default.

### Why Local Qwen3-TTS Is Worth the Extra Setup

`mlx-audio` wraps Qwen3-TTS so it runs on Apple Silicon without additional tools.
The setup cost is real: you need to install the package and download the weights.
But once running, the voice is customizable in ways that built-in and online TTS are not.

You can adjust the speaking style through prompt engineering.
Qwen3-TTS accepts a description of the speaker's persona alongside the text,
which means you can tune tone, pace, and affect to match the companion character you want.

Start `mlx-audio` as a local server:

```shell
python -m mlx_audio.server
```

The server exposes an OpenAI-compatible TTS endpoint at `http://localhost:8000`.
Configure the agent to send reply text to that endpoint instead of calling `say` directly.

Response latency is lower than an online provider for short replies on Apple Silicon M-series hardware.
Long replies take longer, which is one reason to keep the agent replies short.

### Response Length Limits

Set a maximum reply length in the agent prompt.
Two to four sentences is the right range for voice output during gameplay.
Longer replies take more time to generate and more time to speak.
By the time a ten-sentence reply finishes, the moment in the game has already passed.

If the agent tends to give longer answers, add an explicit instruction:
`Keep replies to three sentences or fewer. Shorter is better.`

### Streaming Versus Full-Buffer Playback

Full-buffer playback waits for the entire reply text to arrive before speaking.
This adds generation latency but produces a more natural delivery because the TTS can phrase the whole sentence correctly.

Streaming playback starts speaking as soon as the first chunk of text is ready.
Latency is lower, but sentence-level phrasing can sound fragmented if the TTS processes partial sentences.

For a gaming companion, full-buffer playback is the better default.
The generation time for short replies is small, and the improvement in delivery is worth the wait.

### Interruptibility

If you speak again while the agent is replying, kill the current TTS playback before starting the new one.
The simplest implementation is to track the TTS process ID and kill it when a new message comes in.

```shell
kill $TTS_PID 2>/dev/null
```

Without interruption support, overlapping replies make the session feel unresponsive.
This is one of the details that separates a usable companion from a demo.

### Audio Routing While the Game Is Already Producing Sound

If TTS audio and game audio both play through the same output, replies can be drowned out during loud moments.
Three options that work without complex routing:

1. Use headphones with separate left/right balance: game audio in one ear, TTS in the other
2. Duck the game audio briefly when a TTS reply starts, then restore it
3. Use a virtual audio device like [BlackHole](https://existential.audio/blackhole/) to keep TTS on a separate channel

The simplest option that works is just to turn the game volume down slightly in the macOS volume mixer and trust that TTS at higher volume will be audible.
Full audio routing is not worth the complexity unless the gaming experience is being streamed.

## Foot Pedal Workflow

The deck pedal turns a three-action keyboard sequence into three single presses with the left foot.

### Which Key Each Pedal Triggers

The mapping I use:

- **Left pedal:** Bring Discord to front and activate macOS dictation (`Cmd+Tab` to Discord, then dictation shortcut)
- **Middle pedal:** Stop dictation and select all dictated text (`Esc` or dictation toggle, then `Cmd+A`)
- **Right pedal:** Copy and send (`Cmd+C`, then paste into Discord message box, then `Return`)

The exact shortcut sequence depends on how you map the pedals in your driver software.
Most programmable pedals support macro sequences, so you can bind each pedal to a multi-step shortcut.

The sequence is not instant — there is about 200 ms of focus switching between steps — but it is fast enough that the total time from "press left pedal" to "message sent" is under two seconds for a short dictated sentence.

### Why Feet Are Better Than Hands for This Interaction

During gameplay, hands are on the controller, keyboard, or mouse.
Any input that requires lifting a hand breaks the game flow.

The foot pedal removes that cost entirely.
Left foot handles all three steps.
Right foot stays on the floor or can be used for a second pedal set if needed.

The mental overhead is also low.
Three pedals in sequence for one complete interaction is easier to remember than reaching for a specific keyboard shortcut.
After a few sessions it becomes automatic.

### How the Pedal Changes Keyboard, Mouse, and Controller Play

**Keyboard and mouse play:** The pedal is unobtrusive.
The left foot is already idle in most keyboard-and-mouse setups.
Activating the pedal does not interfere with WASD movement or mouse aiming.

**Controller play:** Similarly low impact.
Both thumbs stay on the controller during the entire dictation flow.
The only interruption is the brief moment of speaking.

**What still feels awkward:**
The middle step — stopping dictation and selecting the text — sometimes takes an extra moment if macOS dictation is slow to finish processing.
If the pedal fires the second step too quickly, the selected text may be incomplete.
Adding a 300 ms delay after the first pedal press before triggering the second action fixes this.

**What feels natural:**
Once the timing is tuned, the loop feels close to hands-free.
The only perceptible pause is the time you spend speaking.

## Reliability Notes

A few things break more often than others, and it helps to know the pattern before it happens during a session.

**Dictation focus problems** are the most common failure.
If Discord is not the active app when you press the pedal, dictation activates in the wrong window.
Fix this by keeping a consistent window arrangement: Discord in a fixed position, game in the other window.
If using a separate monitor, put Discord on the secondary monitor at all times.

**TTS playback interruption** is the second most common issue.
If a new reply arrives before the previous one has finished, you can end up with two overlapping audio streams.
Ensure the agent sends a signal or PID file when TTS starts so the next reply can kill the previous process cleanly.

**mlx-audio server restarts** happen occasionally after macOS sleep or when the process is idle for a long time.
Add a health check to the agent: before sending reply text to the local TTS endpoint, do a quick ping and restart the server if it is not responding.

**Dictation vocabulary gaps** are a permanent background issue.
For games with unusual terminology, keep a text fallback ready.
The foot pedal workflow is for quick remarks; anything that requires precise terminology is faster to type.

## Failure Modes

1. **Dictation misses game-specific terms** — proper nouns, hero names, and ability names do not survive dictation well. Type these or add them to macOS's learned vocabulary through correction.
2. **Discord focus problems** — if the dictation shortcut fires while Chrome or the game is in focus, dictation goes to the wrong window. Keep Discord visible and include a focus step in the pedal macro.
3. **Built-in TTS is too generic for the companion style** — `say` works for validation but sounds like a utility tool. Upgrade to an online provider or local Qwen3-TTS if voice character matters.
4. **Online TTS adds too much latency** — if the provider is slow or your connection is inconsistent, replies feel delayed. Switch to a lower-latency provider or move to local TTS.
5. **Local TTS quality is good but setup cost is too high** — Qwen3-TTS with mlx-audio takes time to install and tune. If you are not sure the voice difference is worth it, start with built-in or online TTS and migrate later.
6. **Audio routing glitches** — TTS and game audio overlapping is disorienting. Use macOS volume mixer to balance them, and consider headphones as the simplest fix.
7. **Pedal shortcut conflicts** — some games intercept foot pedal input as a generic HID device. Test the pedal in the game before relying on it.
8. **Interaction loop feels too slow** — if the total time from speaking to hearing a reply is more than a few seconds, check each stage in order: dictation speed, Discord message delivery, agent generation time, TTS latency. The bottleneck is usually TTS if you are using an online provider with a slow connection.

## Links

- [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/)
- [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/)
- [Part 2: OpenClaw Browser Use](/build-your-own-ai-gaming-mate-part-2-openclaw-browser-use/)
- [mlx-audio on GitHub](https://github.com/Blaizzy/mlx-audio)
- [BlackHole virtual audio device](https://existential.audio/blackhole/)
