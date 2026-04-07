---
layout: page
title: "Build Your Own AI Gaming Mate Part 4: Skills and Operations"
permalink: /build-your-own-ai-gaming-mate-part-4-skills-and-operations/
description: |
  Shape agent skills, context management, and game state so your AI gaming mate
  stays aware and improves across play sessions.
category: build-your-own-ai-gaming-mate
---

## Introduction

This post covers the layer that makes the system feel good to use over time:
skills, context management, and game state.
The previous posts set up the video pipeline, observation layer, and audio path.
This one is about shaping the agent behavior that sits on top of all of that.

For an overview of the full system, see [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/).

## What This Part Does

At the end of this post, you should have:

1. A clear interactive skill structure that keeps responses fast and focused
2. Game-specific skills that make the companion feel aware of the game you are actually playing
3. A reply length policy that works for voice output during gameplay
4. A clear sense of when the companion should speak and when it should stay quiet
5. A per-game context file that captures setup, rules, and accumulated knowledge

## Why This Layer Matters

Without this layer, the system may work technically but still feel awkward.
A companion that takes too long to respond, speaks over critical gameplay moments,
or loses track of what you established earlier is more distraction than help.

The skill and context layer is where you shape timing, tone, interruption behavior,
and what the companion carries forward between sessions.
It is the most opinionated part of the project and the hardest to copy
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

The key idea is simple: one dedicated Discord channel per game, one context file per channel.

Create a channel named after the game — for example `game-civilization-6` — and instruct the agent to load the matching context file when it starts in that channel.
The file lives at a predictable path, for example `context/game-civilization-6.md`.

### What Goes in the Context File

The context file is the companion's persistent knowledge about the game.
It starts small and grows as you play.

A minimal starting file covers:

1. **What game is running** — game name, current save or campaign, and your faction or character
2. **How interaction works** — how you speak (voice via dictation, typed messages), what triggers a screenshot, reply length expectations
3. **Generic rules for this game** — victory conditions, pacing, any standing instructions like "never guess exact tech counts from a screenshot"

For example, a starting `context/game-civilization-6.md` might look like:

```markdown
## Game

Civilization VI. Current save: Emperor difficulty, Deity later.
Playing as Brazil. Aiming for Culture victory.

## Interaction

- Say "take a look" to trigger a screenshot
- Keep replies to three sentences unless I ask for a full analysis
- Use the correct Civ VI vocabulary: districts, wonders, governors, great people

## Rules

- Do not guess exact tech or civic progress from a screenshot; ask me for numbers
- When I give you numbers, evaluate position directly without hedging
- Victory routes to watch: Culture, Science, Domination
```

### Growing the Context File Over Time

As the game progresses, edge cases and patterns emerge that the model does not handle well out of the box.
Each time you notice a gap — a bad guess, wrong vocabulary, a missed rule — add a note to the context file.

Some examples of what accumulates:

- **Faction-specific rules:** "Menelik II gets +10 appeal from Holy Sites; factor that into district placement advice"
- **Observed mistakes:** "Stop recommending Theater Squares before Classical era; I already have enough culture output"
- **Session-specific state:** current score, key alliances, which victory route I am committed to

The file is plain text, so you can update it between sessions without any tooling.
The companion loads it fresh at the start of each channel session and treats it as authoritative.

### Extracting Knowledge Into Skills

Over time, the context file will accumulate enough game-specific logic that it starts to feel like a reference document.
That is the signal to start extracting reusable pieces as skills.

The extraction pattern:

1. **Start game-specific.** A Civ VI advisor skill knows victory conditions, evaluates position from numbers, and avoids guessing from screenshots. It is narrowly scoped and correct for one game.
2. **Generalize when the pattern repeats.** After building a Civ VI skill, you will notice that other turn-based strategy games share the same core pattern: evaluate position from explicit numbers, advise on win condition priority, track long-term goals. Extract a turn-based strategy skill and make the Civ VI skill a specialization of it.
3. **Keep the context file for what is session-specific.** Skills carry forward general game knowledge. The context file carries forward the state of the current playthrough. Both are needed; they are not substitutes.

This progression — context file first, skill extraction second — lets you start useful immediately without over-engineering the setup before you understand the game's actual interaction patterns.

## Links

- [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/)
- [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/)
- [Part 2: OpenClaw Browser Use](/build-your-own-ai-gaming-mate-part-2-openclaw-browser-use/)
- [Part 3: Audio Pipeline](/build-your-own-ai-gaming-mate-part-3-audio-pipeline/)
- [OpenClaw Skills documentation](https://docs.openclaw.ai/skills)
- [OpenClaw configuration reference](https://docs.openclaw.ai/gateway/configuration-reference)
