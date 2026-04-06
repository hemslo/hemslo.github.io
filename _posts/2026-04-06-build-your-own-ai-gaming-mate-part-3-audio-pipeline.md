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

1. Discord's built-in TTS if you want the fastest path with zero configuration
2. An online TTS provider if you want better voice quality with less local setup
3. `mlx-audio` running Qwen3-TTS locally if you want a self-hosted voice on Apple Silicon

The reason to run `mlx-audio` locally is not that the other options are unusable.
It is that local hosting gives you lower latency and more control over the voice without any recurring cost or privacy tradeoff.

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
  -> press pedal key 1 (double-click left Command to activate macOS dictation)
  -> speak
  -> press pedal key 2 (Return to send Discord message)
  -> Discord message
  -> local agent

Audio output
  -> local agent text
  -> Discord built-in TTS, online TTS, or Qwen3-TTS via mlx-audio
  -> press pedal key 3 (mouse click via cliclick to play Discord TTS audio)
  -> spoken reply through 3.5mm cable into Windows line-in
```

## Prerequisites

**macOS dictation:**

- macOS Ventura or later
- Enable dictation in System Settings → Keyboard → Dictation
- Set the shortcut to double-click the left Command key
- Enable "Auto-punctuation" if you want cleaner transcriptions
- Microphone access must be granted to the system
- macOS handles only Discord, browser, and OpenClaw; the game itself runs on Windows

**Discord:**

- A Discord server and channel dedicated to the gaming mate session
- The agent connected to that channel (covered in previous posts)
- Discord desktop app, not the browser version — focus behavior is more predictable
- Discord's built-in TTS enabled (the `/tts` prefix or a bot that sends TTS messages)

**TTS (online provider path):**

- An API key from your chosen provider (ElevenLabs, OpenAI TTS, or similar)
- Network access during play

**TTS (local Qwen3-TTS path):**

- An Apple Silicon Mac with enough memory to run the model
- [mlx-audio](https://github.com/Blaizzy/mlx-audio) installed via pip — the upstream package already exposes an OpenAI-compatible TTS API

```shell
pip install mlx-audio
```

- Qwen3-TTS weights downloaded through mlx-audio
- (Optional) A fork of mlx-audio with voice upload and voice-prompt caching for better performance — works like the vllm Qwen3-TTS serving example where the reference audio is uploaded once and reused across all requests instead of being re-encoded on every call

```shell
pip install git+https://github.com/hemslo/mlx-audio.git
```

**Foot pedal:**

- A 3-key USB deck pedal (any programmable HID pedal will work)
- The pedal driver or macOS keyboard shortcut mapping tool
- [cliclick](https://github.com/BlueM/cliclick) installed on macOS (used via an Automator application for the click action)

```shell
brew install cliclick
```

**Audio routing:**

- A 3.5mm audio cable connecting the macOS headphone output to the Windows PC line-in port
- Windows built-in volume mixer to balance game audio and the incoming macOS TTS audio

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

macOS dictation activates on a double-click of the left Command key.
This is the shortcut I use because it does not conflict with any game shortcuts — the game is running on Windows, not macOS.
macOS runs only Discord, the browser, and OpenClaw, so there is very little risk of a shortcut collision.

The activation shortcut puts the dictation overlay in whatever app currently has focus.
If Discord is in the background, the shortcut will activate dictation in the wrong app.
Solve this by keeping Discord always in focus on macOS, or by including a click on the Discord window as the first step of the pedal macro.

### Message Length

Dictation handles both short and long inputs well.
For quick remarks, one sentence is enough.
For longer questions or complex situations, dictate the full message — there is no need to switch to typing.
The foot pedal workflow supports whatever length you need.

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

Discord has a built-in TTS feature.
Any message sent with the `/tts` command, or sent by a bot that uses TTS output, will be read aloud by Discord using the system voice.
This requires zero additional setup: it works out of the box once Discord TTS is enabled in user settings under Notifications.

The voice quality is functional but generic.
It sounds like a system utility rather than a specific character.
For validating the end-to-end loop and getting a feel for the interaction rhythm, Discord TTS is more than sufficient.

The main limitation is control.
You cannot change the voice style, pace, or persona through Discord TTS.
For longer play sessions, the generic voice becomes noticeable.
That is when moving to an online provider or local Qwen3-TTS makes sense.

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

`mlx-audio` runs Qwen3-TTS on Apple Silicon and already exposes an OpenAI-compatible TTS API out of the box.
That means you can point OpenClaw — or any other client that speaks the OpenAI TTS API — straight at the local server without any adapter code.

Start the local TTS server:

```shell
python -m mlx_audio.server
```

The server exposes an OpenAI-compatible TTS endpoint at `http://localhost:8000/v1`.
Configure OpenClaw (or your agent) to send reply text to that endpoint.

If you want voice cloning with better performance, the optional fork adds voice upload and caches the loaded voice prompt across requests.
This works the same way as the [vllm Qwen3-TTS serving example](https://docs.vllm.ai/projects/vllm-omni/en/latest/user_guide/examples/online_serving/qwen3_tts/):
upload a reference audio file once, and every subsequent TTS request reuses the cached voice prompt instead of re-encoding it from scratch.
The result is faster first-token latency for short replies.

Response latency for short replies on Apple Silicon M-series hardware is lower than most online providers.
Long replies take more time to generate, which is one reason to keep agent replies short.

### Configuring OpenClaw to Use the Local TTS Server

OpenClaw's TTS settings live under `messages.tts` in `openclaw.json`.
To point it at the local mlx-audio server, use the `openai` provider and set `baseUrl` to `http://localhost:8000/v1`:

```json
{
  "messages": {
    "tts": {
      "auto": "always",
      "provider": "openai",
      "providers": {
        "openai": {
          "baseUrl": "http://localhost:8000/v1",
          "model": "lucasnewman/f5-tts-mlx",
          "voice": "alloy"
        }
      }
    }
  }
}
```

Set `auto` to `"always"` to have every agent reply spoken automatically, or `"tagged"` to speak only replies that carry a specific tag.
See the [OpenClaw TTS configuration reference](https://docs.openclaw.ai/gateway/configuration-reference#tts-text-to-speech) for the full list of options including per-provider voice settings and fallback providers.

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

### Audio Routing While the Game Is Already Producing Sound

The macOS machine handles Discord, browser, and OpenClaw.
The Windows machine runs the game.
A 3.5mm audio cable connects the macOS headphone output to the Windows PC line-in port.
This merges macOS audio — including Discord TTS and local TTS playback — with the game audio directly in hardware.

Windows sees the macOS audio as a line-in source.
Use the Windows built-in volume mixer to balance the two: raise or lower the line-in level relative to the game audio until the companion voice is comfortably audible over the game sound.
No virtual audio devices or complex routing software are required.

## Foot Pedal Workflow

The deck pedal turns a three-action keyboard sequence into three single presses with the left foot.

### Which Key Each Pedal Triggers

The mapping I use:

- **Left pedal:** Activate macOS dictation (double-click the left Command key)
- **Middle pedal:** Send the Discord message (Return)
- **Right pedal:** Play the Discord TTS audio reply (mouse left click via `cliclick c:.` wrapped in an Automator application)

For the right pedal, the workflow is: hover the mouse over the Discord TTS play button in the reply, then press the right pedal.
The Automator application runs `cliclick c:.` which clicks at the current mouse cursor position without moving it.
This means you position the cursor over the play button once and then trigger the click with your foot.

The exact shortcut sequence depends on how you map the pedals in your driver software.
Most programmable pedals support macro sequences or application bindings, so you can bind each pedal to a key or script.

The sequence is fast: dictate, press Return, wait for the reply to appear, hover over the play button, press the right pedal.
Total hands-free time from speaking to hearing the reply is typically under three seconds for short replies.

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
The right pedal click requires the mouse to already be hovering over the Discord play button.
If the cursor drifted during gameplay, you need to move it back before pressing the pedal.
This is the one moment where a small hand movement is still needed.
Keeping the Discord window at a consistent screen position makes the cursor placement faster over time.

**What feels natural:**
Once the timing is tuned, the loop feels close to hands-free.
The only perceptible pause is the time you spend speaking.

## Links

- [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/)
- [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/)
- [Part 2: OpenClaw Browser Use](/build-your-own-ai-gaming-mate-part-2-openclaw-browser-use/)
- [mlx-audio on GitHub](https://github.com/Blaizzy/mlx-audio)
- [OpenClaw TTS configuration reference](https://docs.openclaw.ai/gateway/configuration-reference#tts-text-to-speech)
- [cliclick on GitHub](https://github.com/BlueM/cliclick)
