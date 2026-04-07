---
layout: page
title: "Build Your Own AI Gaming Mate Part 4: Skills and Operations"
permalink: /build-your-own-ai-gaming-mate-part-4-skills-and-operations/
description: |
  Design game-specific skills, a context file that describes how to play, and a
  lightweight state file that tracks the current session for your AI gaming mate.
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

1. Game-specific skills that make the companion feel aware of the game you are actually playing
2. A per-game context file that describes how to interact and the standing rules for that game
3. A lightweight JSON state file that tracks what is happening in the current session

## Why This Layer Matters

Without this layer, the system may work technically but still feel awkward.
A companion that uses the wrong vocabulary, gives advice that ignores your actual game state,
or loses track of rules you have already established is more distraction than help.

The skill and context layer is where you encode what the companion knows about the game,
how you want it to interact, and what is happening right now in the current session.
It is the most opinionated part of the project and the hardest to copy
from someone else's setup directly, because it depends on the games you play and the
kind of interaction you want.

Think of this post as a set of principles with concrete examples, not a fixed recipe.

## Game-Specific Skill Design

Skills in this setup are game-specific.
There are no generic interactive skills — all interaction conventions live in the context file.
Skills exist to encode knowledge about a particular game.

A game-specific skill carries a specialized prompt and a narrow tool set.
For example, a Civ VI advisor skill knows the victory conditions, evaluates your position
when you give it numbers, and avoids guessing exact tech or civic progress from a screenshot alone.

You do not need to build all skills at once.
Start with a single game and a single skill that covers the questions you ask most.
Add vocabulary by listing proper nouns in the skill's system prompt: leader names, district names,
unit names, and any term that voice transcription gets wrong.
Once the model knows the vocabulary, transcription errors become easier to spot and correct.

For games with publicly available data, you can include a short reference block in the skill.
For example, a tech tree summary, a table of unit costs, or a list of leader abilities.
Keep it short: a few dozen lines covers the facts the companion actually needs.
Anything longer starts to dilute the prompt and slow things down.

## Game State and Memory Management

The key idea is simple: one dedicated Discord channel per game, with two files the agent loads
at the start of each session.

- **Context file** — describes how to play the game: interaction conventions, standing rules, vocabulary
- **State file** — records what is happening right now: current civilization, leader, score, active goals

Create a channel named after the game — for example `game-civilization-6` — and instruct the agent
to load both files when it starts in that channel.
The files live at predictable paths, for example `context/game-civilization-6.md` and
`state/game-civilization-6.json`.

### The Context File: How to Play

The context file describes the setup that stays stable across sessions.
It does not record who you are playing as — that goes in the state file.
What it records is how the companion should behave when playing this game.

A minimal context file for Civ VI:

```markdown
## Interaction

- Say "take a look" to trigger a screenshot
- Keep replies to three sentences unless I ask for a full analysis
- Use correct Civ VI vocabulary: districts, wonders, governors, great people, city-states

## Rules

- Do not guess exact tech or civic progress from a screenshot; ask me for numbers
- When I give you numbers, evaluate position directly without hedging
- Victory routes to consider: Domination, Culture, Science, Diplomatic, Religion
```

Everything in that file applies regardless of which leader you pick or which playthrough you are in.
It is the standing instruction set for the game, not the record of a particular session.

### The State File: The Current Session

The state file is a lightweight JSON document that tracks the current playthrough.
Think of it as a game save that the agent reads and updates as the session progresses.

For example, a Civ VI session playing as Victoria (Age of Steam):

```json
{
  "game": "Civilization VI",
  "leader": "Victoria (Age of Steam)",
  "difficulty": "Deity",
  "victory_target": "Domination",
  "era": "Industrial",
  "turn": 203,
  "scores": {
    "military": 1507,
    "science": 240,
    "culture": 291,
    "gold": 1690,
    "faith": 1313
  },
  "notes": [
    "Colonized three continents; harbor network active",
    "Alliance with Egypt expires turn 160",
    "Japan is the main military threat — eliminate next"
  ]
}
```

When you give the agent new numbers or report a major event, it updates the state file.
At the start of the next session, the agent loads both files and picks up exactly where you left off.
No summary step needed, no pasting context back in.

### Growing Both Files Over Time

The context file grows when you discover rules or patterns that the model does not handle well.
Each time you notice a gap — wrong advice, missed vocabulary, a bad assumption — add a line to the context file.

Some examples of what accumulates in the context file:

- **Leader-specific rules:** "Victoria's Industrial Zones get +4 production from Harbour adjacency; factor that into district placement"
- **Observed model mistakes:** "Do not recommend building Encampments when I already have military dominance; focus on infrastructure"
- **Vocabulary corrections:** "Use 'Suzerain' not 'patron' when referring to city-state relationships"

The state file updates as the game progresses: score changes, new alliances, completed objectives,
next strategic priority.
Keep notes in the state file short and actionable.
They are reminders for the agent, not a full play log.

### Extracting Knowledge Into Skills

Over time, the context file will accumulate enough game-specific logic that it starts to feel like
a reference document.
That is the signal to start extracting reusable pieces as skills.

The extraction pattern:

1. **Start game-specific.** A Civ VI advisor skill knows victory conditions, evaluates position from numbers, and avoids guessing from screenshots. It is narrowly scoped and correct for one game.
2. **Generalize when the pattern repeats.** After building a Civ VI skill, you will notice that other turn-based strategy games share the same core pattern: evaluate position from explicit numbers, advise on win condition priority, track long-term goals. Extract a turn-based strategy skill and make the Civ VI skill a specialization of it.
3. **Keep the context file for interaction conventions and game-specific rules.** Keep the state file for session data. Skills carry forward general game knowledge that is stable and reusable.

This progression — context file and state file first, skill extraction second — lets you start
useful immediately without over-engineering before you understand the game's actual interaction patterns.

## Leveling Up Together

There is a side effect to this process that is worth naming.

Building skills, refining the context file, fixing model mistakes, and updating the state file
after each session is not maintenance work.
It is a progression loop.
Every session surfaces something new: a vocabulary gap, a rule the model bends, a pattern in how
you like to play.
Each fix makes the companion slightly more tuned to you.

It feels a lot like leveling up in an RPG, except the character you are leveling up is the agent.
You are accumulating knowledge, expanding its repertoire, and watching it get better at reading
your style.
That loop is genuinely addictive.
The companion you have after thirty sessions is meaningfully different from the one you started with,
and the difference came from playing together.

That is, in the end, the most addictive part of the whole system.
Not the tech — the progression.

## Links

- [Part 0: Overview](/build-your-own-ai-gaming-mate-part-0-overview/)
- [Part 1: Gameplay Streaming](/build-your-own-ai-gaming-mate-part-1-gameplay-streaming/)
- [Part 2: OpenClaw Browser Use](/build-your-own-ai-gaming-mate-part-2-openclaw-browser-use/)
- [Part 3: Audio Pipeline](/build-your-own-ai-gaming-mate-part-3-audio-pipeline/)
- [OpenClaw Skills documentation](https://docs.openclaw.ai/skills)
- [OpenClaw configuration reference](https://docs.openclaw.ai/gateway/configuration-reference)
