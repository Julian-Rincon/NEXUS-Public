# NEXUS

> **Portfolio showcase** — public documentation of a private project.
> Source code, credentials, and operational details are intentionally excluded.

A local-first, production-grade personal AI system built for real daily use on Linux (Fedora / KDE Plasma). Voice-driven, autonomous, and always running — it manages the machine, not just conversation.

---

## What It Does

NEXUS is used daily as an actual productivity and system-management layer, not a demo. It handles:

- **Voice conversation** with sub-second response (streaming TTS, barge-in interrupt, custom-trained wake word)
- **Natural language routing** — any phrasing in Spanish works; no fixed command syntax required
- **Autonomous desktop control** — opens apps, clicks, types, reads the screen
- **Meeting participation** — joins Meet/Zoom/Teams, transcribes in real time, and can speak inside the meeting through a virtual microphone
- **Google Workspace** — sends emails, creates calendar events in natural language, reads Drive files
- **Home automation** — controls IoT devices (lights, climate, scripts) by voice
- **Personal finance** — summarizes spending from bank notification emails
- **Proactive alerts** — notifies when a calendar event is approaching, urgent email arrives, or system resources are stressed
- **Passive security monitoring** — watches for unknown high-CPU processes, new open ports, new USB devices, and failed SSH auth attempts; detects only, never acts unilaterally
- **System resource management** — CPU/RAM priority profiles for work, streaming, or focus sessions
- **Development workflow automation** — runs test suites, reads logs, builds, restarts dev servers
- **Natural-language code generation** — turns a spoken request into a validated Python file
- **Telegram remote control** — voice-note-first responses from anywhere
- **Desktop HUD** — a native control center with subsystem status cards and a real-time conversation log
- **Daily OS layer** — morning briefing, end-of-day summary, productivity metrics, daily priorities by voice
- **Gaming mode** — automatically frees GPU VRAM and switches thermal profiles when a game is detected
- **Guarded system power control** — can lock, suspend, hibernate, shut down, or restart the machine by voice, gated behind an explicit confirmation step

---

## Capability Overview

| Capability | Status |
|---|---|
| Voice conversation (streaming) | Active — first spoken word in ~0.5 s |
| Barge-in interrupt | Active — VAD-based, ~40 ms reaction |
| Custom wake word | Active — purpose-trained detection model, no cloud dependency |
| Cloned voice (primary) + local fallback | Active — low-latency streaming synthesis, automatic local fallback |
| Natural language routing | Active — any phrasing, 15+ NL categories, voice and Telegram |
| Desktop HUD | Active — native control center, subsystem status cards, system tray integration |
| Skills system | Active — 36 auto-discovered skills (media, focus, GPU monitor, notes, weather, resource manager, dev workflow, security monitor, code generator, personal automation…) |
| Autonomous desktop agent | Active — vision + input, up to 25 steps, allowlisted for safety |
| Meeting agent | Active — join + virtual mic + live transcript + summary |
| Proactive monitor | Active — calendar, email, system resources, morning briefing |
| Passive security watcher | Active — process/port/USB/auth anomaly detection, alert-only |
| Resource manager | Active — CPU/RAM priority profiles (work / streaming / focus / normal) |
| Dev workflow automation | Active — test runner detection, log triage, build, server restart |
| Natural-language code generation | Active — spoken request → validated Python file |
| Guarded system power control | Active — lock is immediate; suspend/hibernate/shutdown/restart require explicit confirmation |
| Telegram (voice-first) | Active — replies with voice note + text |
| Google Calendar | Active — natural language event creation and query |
| Gmail | Active — send, reply, draft |
| Google Drive | Active — list, read, create |
| Home automation | Active — IoT device and scene control by voice |
| Personal finance summary | Active — bank notification email parsing |
| Multi-tool exposure (MCP) | Active — exposes core capabilities as tools for external AI clients |
| Session memory | Active — recent context injected automatically |
| Vector memory | Active — semantic recall |
| Episodic memory | Active — causal event log, queryable by day/week |
| Task tracker | Active — deadlines, priorities, search |
| Daily OS | Active — morning briefing, day close, metrics, priorities by voice |
| Gaming mode | Active — GPU-aware VRAM management, thermal profile switching |
| Auto-start on login | Active — background service, starts with the desktop session |

---

## Desktop HUD

The operator console is a native desktop control center that runs alongside the voice pipeline — void-black background, blood-red accents, an animated red "digital rain" effect behind the panels.

![NEXUS desktop HUD](assets/nexus_hud.png)

**What it shows:**
- Real-time conversation log (user and NEXUS turns)
- A live backend bar — active LLM tier count, current STT and TTS models, wake word model, number of loaded skills, GPU status — read directly from configuration and hardware at runtime, not hardcoded
- An animated status core for the local pipeline
- Status cards for each subsystem — wake word listener, Telegram bot, browser automation, desktop automation, audio devices, Google credentials
- System tray integration for quick access without keeping a window open

Detailed system resource usage (CPU/RAM/disk/GPU) is queried on demand through voice or Telegram (`estado del sistema`, `cómo está el equipo`) rather than rendered as a permanent gauge in the HUD.

---

## Skills System

36 capabilities are packaged as self-contained skills, auto-discovered at startup with no manual registration. Each skill declares its own triggers and handles its own execution context.

Examples:

| Skill | What it does |
|---|---|
| Media control | Play, pause, volume, track info — any phrasing works |
| Focus mode | Timed focus sessions with progress tracking |
| System monitor | CPU, RAM, disk, temperature, GPU VRAM |
| Resource manager | Switch CPU/RAM priority profile for work, streaming, or focus |
| Dev workflow | Run tests, check logs, build, restart the active dev server |
| Security monitor | Query open ports, suspicious processes, whitelist management |
| Code generator | Turn a natural-language request into a validated, saved Python file |
| Reminders | Natural language reminders with delivery via voice |
| Notes | Voice-dictated notes, search, listing |
| Weather | Current conditions for any city |

The same skill responds correctly to "súbele a la música", "sube el volumen", "dale más volumen" — all routed without hardcoded strings.

---

## Daily OS Layer

Beyond task execution, NEXUS functions as a daily operating layer:

**Morning briefing** (`día de hoy`):
Date, time, weather, today's calendar events, top unread emails, last saved work context, and pending daily priorities — in a single voice response.

**Day close** (`cierra el día`):
Session stats (interactions, LLM vs direct), priority completion rate, and a Markdown diary note saved automatically.

**Productivity metrics** (`mis métricas`):
Interaction count for today and the past 7 days, derived from a persistent activity log.

**Daily priorities** (by voice):
Add, list, complete, and clear priorities — stored in the user profile, surfaced in the morning briefing.

---

## Gaming Mode

A background monitor checks continuously for active game sessions (Wine/Proton processes + GPU VRAM usage). When a game is detected:

1. The local LLM model is unloaded from VRAM (~5 GB freed for the game)
2. Queries are routed directly to cloud backends — NEXUS stays responsive without competing for GPU
3. The system thermal profile switches to performance mode automatically
4. Kernel swap behavior is tuned to reduce memory-latency spikes during play
5. Background scans that compete for CPU are paused for the session

On game exit, everything restores automatically — no configuration required.

---

## TieredBrain — LLM Routing

Five backends with automatic failover and shared conversation history:

```
User message
    │
    ├── IntentRouter (deterministic — no LLM cost)
    │       └── Exact, prefix, contains, NL-normalized matches
    │
    └── TieredBrain
            ├── Level 1: local model — simple conversation, offline
            ├── Level 2: cloud (fast, general-purpose) — complex tasks
            ├── Level 3: cloud (high-throughput) — long responses, low latency
            ├── Level 4: cloud (large context, multimodal) — long-context and vision-capable tasks
            └── Level 5: cloud (fallback) — final safety net
```

Circuit breakers pause failing backends silently and recover after a cool-down. All five backends share the same conversation history — no context loss on tier switches. During gaming mode, the local tier is bypassed entirely so responses stay fast without contending with the GPU.

**Streaming TTS** runs in parallel with LLM generation: the first sentence is synthesized and played while the model is still producing tokens.

```
LLM token stream
    │
    ├── sentence boundary detected
    │
Primary TTS (cloud, low-latency) ──► playback
    │                                    ▲
    └── local fallback if primary unavailable
                    │
              Barge-in VAD ◄── interrupt if user speaks (~40 ms, pre-roll capture)
                    │
                    ▼
                 Speaker
```

**Key performance properties:**
- First spoken word delivered in under a second from end of user speech
- Circuit breaker pauses failing backends, recovers silently, no manual intervention
- Common phrases pre-synthesized at startup — zero synthesis latency on cache hit
- English technical words pronounced correctly inside Spanish speech on the local fallback voice

---

## Meeting Agent

NEXUS can fully participate in online meetings:

1. Opens the browser and joins Meet, Zoom, or Teams from a URL
2. Records and transcribes the meeting audio in real time (incremental)
3. Produces a summary and logs a calendar event when the meeting ends
4. Speaks **inside** the meeting through a virtual microphone — participants see a dedicated virtual input device

This allows NEXUS to relay information, read documents, or answer questions during a live call without the user typing anything.

---

## Autonomous Desktop Agent

A vision-guided agent that operates the Linux desktop:

1. Plans a sequence of steps using a cloud vision model
2. Executes each step (click, type, scroll, key press, focus app) through an allowlisted action layer
3. Takes a screenshot after each action and verifies the result before proceeding
4. Abandons safely if it detects an unexpected state

Hard safety limits: application launches are restricted to a known-app allowlist, URL opens are restricted to `http(s)`, and keyboard shortcuts are pattern-validated — none of these are left to unchecked model output. Maximum depth: 25 autonomous steps per task.

---

## Guarded System Power Control

NEXUS can lock the screen, suspend, hibernate, shut down, or restart the machine by voice or from Telegram.

- **Lock** executes immediately — it is reversible and carries no risk of lost work.
- **Suspend, hibernate, shutdown, and restart** are treated as high-risk actions: they require an explicit spoken or typed confirmation before anything executes, using the same confirmation mechanism as other state-changing operations (see [docs/DECISIONS.md](docs/DECISIONS.md)). A pending request expires automatically if not confirmed within a short window.

This mirrors the general design principle across the system: irreversible or disruptive actions never fire on a single ambiguous phrase.

---

## Telegram Remote Control

Every reply is sent as a voice note synthesized with the active TTS engine, followed by the text. NEXUS sounds like itself on mobile.

**Natural language routing (no commands needed):**
- Paste a Meet/Zoom/Teams URL → joins the meeting automatically
- "send an email to x@y.com about Z" → routes to Gmail
- "say in the meeting: [message]" → speaks through the virtual mic
- Anything else → conversational response with voice + text

The wake word continues listening locally in the background while the Telegram bot is active — both channels run simultaneously.

---

## Proactive Monitor

A background daemon polls at regular intervals:

- **Calendar**: announces approaching events via voice + desktop notification
- **Email**: classifies incoming mail (important / spam / normal) and surfaces what matters
- **System resources**: warns on CPU / RAM / disk pressure
- **Security**: surfaces anomalies detected by the passive security watcher
- **Log watcher**: tails the active project's logs and alerts on new errors, tied to the last known work context
- **File organization**: sorts downloads and scratch files into folders by type, and warns when disk space runs low
- **Morning briefing**: summarizes agenda and unread mail automatically

---

## Architecture

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full breakdown.

| Module area | Purpose |
|---|---|
| Assistant | Orchestrates STT → LLM → actions → TTS, manages state |
| TieredBrain | Five-level LLM with circuit breaker and token streaming |
| Voice | Capture, barge-in VAD, custom wake word, primary + fallback TTS |
| Skills | 36 auto-discovered skill plugins |
| Actions | Central router — NL normalization, Daily OS, Google, meetings, system, guarded power control |
| SecurityWatcher | Passive anomaly detection — processes, ports, USB, auth failures |
| Resource manager | CPU/RAM priority profiles per activity |
| Gaming mode | GPU-aware VRAM manager + thermal profile switching |
| HUD (desktop) | Native control center with subsystem status cards and system tray |
| Telegram | Voice-first bot with NL routing |
| MeetingAgent | Browser join + virtual mic + live transcript |
| DesktopAgent | Vision + input automation, allowlisted, step verification |
| Google | Calendar NL + Gmail + Drive via OAuth2 |
| Home automation | IoT device and scene control |
| Finance | Bank notification parsing and spending summary |
| MCP server | Exposes core capabilities as tools for external AI clients |
| Memory | Session log + vector store + episodic event log |
| Task tracker | Persistent tasks with deadline and priority |
| ProactiveMonitor | Background daemon for alerts, briefings, and gaming detection |

---

## Technical Decisions

See [docs/DECISIONS.md](docs/DECISIONS.md). The most consequential:

- **Local-first LLM** — default path can run fully offline; cloud activates only when needed or during gaming, when the local model is deliberately bypassed
- **Multi-provider tiering** — five LLM backends instead of one or two, chosen for complementary strengths (offline capability, throughput, context length, cost ceiling) rather than redundancy alone
- **Streaming TTS** — LLM output chunked at sentence boundaries, synthesized in parallel with generation
- **Deterministic routing before LLM** — structured requests bypass the model entirely, reducing hallucination risk and latency
- **Confirmation gating for irreversible actions** — any action with real-world side effects (sending mail, executing automation, changing machine power state) requires an explicit confirmation step; nothing destructive fires on ambiguous phrasing alone
- **Native desktop HUD over a web frontend** — a native control surface that integrates with the system tray and window manager, rather than a browser-hosted interface
- **Virtual mic via system audio routing** — audio synthesized and routed through a loopback device, appearing as a real mic to meeting software
- **GPU-aware gaming mode** — proactive VRAM release and thermal management; NEXUS stays useful during gaming sessions without competing for resources

---

## Roadmap

See [docs/ROADMAP.md](docs/ROADMAP.md).

**Completed:**
- Streaming pipeline with barge-in interrupt and a custom-trained wake word
- Five-tier TieredBrain with circuit breaker
- Cloned-voice TTS with automatic local fallback
- Native desktop HUD with subsystem status cards and system tray
- 26-skill plugin system with auto-discovery
- NL normalization — any Spanish phrasing routes correctly
- Meeting agent with virtual mic and live transcription
- Telegram voice-first with NL routing
- Google Workspace (Calendar NL + Gmail send/reply/draft + Drive)
- Home automation and personal finance summarization
- Autonomous desktop vision agent with hard safety allowlists
- Passive security watcher (process / port / USB / auth anomalies)
- Resource manager, dev workflow automation, natural-language code generation
- Guarded system power control with confirmation gating
- Multi-tool exposure via MCP for external AI clients
- Proactive monitor (calendar, email, system, security)
- Session memory + vector memory + episodic memory
- Task tracker — deadlines, priorities, persistent storage
- Daily OS — morning briefing, day close, metrics, voice priorities
- Gaming mode — automatic VRAM management and thermal profiles
- Auto-start on login

**Planned:**
- Wake word fine-calibration for the specific audio environment
- Home automation entity auto-discovery
- Finance agent integration with a personal budgeting tool
- Deeper multi-agent delegation over the exposed MCP tools

---

## Security and Privacy

See [docs/SECURITY.md](docs/SECURITY.md).

This repository contains no source code, credentials, personal data, or operational automation details. The private implementation handles sensitive personal and professional context — that is exactly why the source is not published.

---

## For Employers and Reviewers

This project demonstrates:

- **Systems integration** across voice, multi-provider LLM routing, vision, browser automation, system audio, and the Linux desktop
- **Production discipline** — not a demo; a tool used and maintained daily for real tasks, including running as an always-on background service
- **LLM engineering** — five-tier routing, streaming, circuit breakers, prompt/context management, NL normalization without LLM dependency
- **Plugin architecture** — 26 self-contained skills with auto-discovery; adding a new capability requires one file
- **Safety-conscious design** — confirmation gating for irreversible actions, allowlisted automation, passive-only security monitoring that never acts unilaterally
- **Product thinking** — the desktop HUD, the Daily OS layer, and the gaming mode are deliberate UX choices, not cosmetic additions
- **Security awareness** — explicit about what is and is not published, and why

A live guided walkthrough of the private implementation is available on request.

---

## License

All rights reserved. No license is granted to copy, redistribute, or create derivative works from the private implementation. See [NOTICE.md](NOTICE.md).
