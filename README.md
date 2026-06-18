# JARVIS Ironman

> **Portfolio showcase** — public documentation of a private project.  
> Source code, credentials, and operational details are intentionally excluded.

A local-first, production-grade personal AI assistant built for real daily use on Linux (Fedora / KDE Plasma). Inspired by the JARVIS interaction model from Iron Man — voice-driven, autonomous, and always running.

---

## What It Does

JARVIS is used daily as an actual productivity layer, not a demo. It handles:

- **Voice conversation** with sub-second response (streaming TTS, barge-in interrupt)
- **Natural language routing** — any phrasing in Spanish works; no fixed command syntax required
- **Autonomous desktop control** — opens apps, clicks, types, reads the screen
- **Meeting participation** — joins Meet/Zoom/Teams, transcribes in real time, and can speak inside the meeting through a virtual microphone
- **Google Workspace** — sends emails, creates calendar events in natural language, reads Drive files
- **Proactive alerts** — notifies when a calendar event is approaching, urgent email arrives, or system resources are stressed
- **Telegram remote control** — voice-note-first responses from anywhere
- **Iron Man HUD** — a web-based operator console with animated arc reactor, live system metrics, and real-time conversation bus
- **Daily OS layer** — morning briefing, end-of-day summary, productivity metrics, daily priorities by voice
- **Gaming mode** — automatically frees GPU VRAM and switches thermal profiles when a game is detected

---

## Capability Overview

| Capability | Status |
|---|---|
| Voice conversation (streaming) | Active — first spoken word in ~0.5 s |
| Barge-in interrupt | Active — VAD-based, ~40 ms reaction |
| Wake word (passive listening) | Active |
| Spanish voice (Bond profile) | Active — local TTS, bilingual phonetics |
| Natural language routing | Active — any phrasing, 15 NL categories, voice and Telegram |
| Iron Man HUD (web) | Active — arc reactor UI, WebSocket, live metrics |
| Skills system | Active — 16 auto-discovered skills (media, focus, GPU monitor, notes, weather…) |
| Autonomous desktop agent | Active — vision + input, up to 25 steps |
| Meeting agent | Active — join + virtual mic + live transcript + summary |
| Proactive monitor | Active — calendar, email, system resources, morning briefing |
| Telegram (voice-first) | Active — replies with voice note + text |
| Google Calendar | Active — natural language event creation and query |
| Gmail | Active — send, reply, draft |
| Google Drive | Active — list, read, create |
| Session memory | Active — recent context injected automatically |
| Vector memory | Active — semantic recall (ChromaDB + nomic-embed-text) |
| Episodic memory | Active — causal event log, queryable by day/week |
| Task tracker | Active — deadlines, priorities, search (SQLite) |
| Daily OS | Active — morning briefing, day close, metrics, priorities by voice |
| Gaming mode | Active — GPU-aware VRAM management, HP OMEN thermal profiles |
| Auto-start on login | Active — systemd user service |

---

## Iron Man HUD

The operator console is a web-based HUD that opens in the browser as a PWA-style window.

**Visual elements:**
- Animated arc reactor SVG with three independently rotating rings and a pulsing hex core
- Hexagonal grid background with holographic glow
- Boot sequence animation on startup
- Real-time conversation bus with typewriter effect for JARVIS responses
- CPU / RAM / disk gauges (circular SVG + linear bars, live via WebSocket)
- STT / LLM / TTS latency indicators with color coding
- API status badges (Groq, Gemini, Ollama, Telegram)
- Pending action confirmation panel
- Scanline overlay for the Iron Man aesthetic

---

## Skills System

16 capabilities are packaged as self-contained skills, auto-discovered at startup with no manual registration. Each skill declares its own triggers and handles its own execution context.

Examples:

| Skill | What it does |
|---|---|
| Media control | Play, pause, volume, track info — any phrasing works |
| Focus mode | Timed focus sessions with progress tracking |
| System monitor | CPU, RAM, disk, temperature, GPU VRAM |
| Reminders | Natural language reminders with delivery via voice |
| Notes | Voice-dictated notes, search, listing |
| Weather | Current conditions for any city |

The same skill responds correctly to "súbele a la música", "sube el volumen", "dale más volumen" — all routed without hardcoded strings.

---

## Daily OS Layer

Beyond task execution, JARVIS functions as a daily operating layer:

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

A background monitor checks every 60 seconds for active game sessions (Wine/Proton processes + GPU VRAM usage). When a game is detected:

1. The local LLM model is unloaded from VRAM (~5 GB freed for the game)
2. Queries are routed directly to cloud backends (Groq/Gemini) — JARVIS stays responsive
3. The HP OMEN thermal profile switches to `performance` automatically
4. On game exit, everything restores — Ollama reloads on the next local query, thermal profile returns to `balanced`

No configuration required. Fully automatic.

---

## TieredBrain — LLM Routing

Three backends with automatic failover and shared conversation history:

```
User message
    │
    ├── IntentRouter (deterministic — no LLM cost)
    │       └── Exact, prefix, contains, NL-normalized matches
    │
    └── TieredBrain
            ├── Level 1: Ollama (local llama3.1:8b) — simple conversation
            ├── Level 2: Groq (llama-3.3-70b-versatile) — complex tasks
            └── Level 3: Gemini (gemini-2.5-flash-lite) — fallback
```

Circuit breakers pause failing backends silently and recover after 5 minutes. All three backends share the same conversation history — no context loss on tier switches.

**Streaming TTS** runs in parallel with LLM generation: the first sentence is synthesized and played while the model is still producing tokens.

```
LLM token stream
    │
    ├── sentence boundary detected
    │
Piper TTS synthesis ──► paplay ──► Speaker
    │
    └── next chunk starts synthesis immediately

Streaming TTS ← first sentence synthesized while LLM still generates
    │
    ▼
Barge-in VAD ──► interrupt if user speaks (~40 ms, pre-roll capture)
    │
    ▼
Speaker
```

**Key performance properties:**
- First spoken word delivered in ~0.5 s from end of user speech
- Circuit breaker pauses failing backends, recovers silently
- Common phrases pre-synthesized at startup — zero synthesis latency on cache hit
- English technical words pronounced correctly inside Spanish speech (phonetic substitution, same voice model)

---

## Meeting Agent

JARVIS can fully participate in online meetings:

1. Opens the browser and joins Meet, Zoom, or Teams from a URL
2. Records and transcribes the meeting audio in real time (incremental, every 15 s)
3. Produces a summary and logs a calendar event when the meeting ends
4. Speaks **inside** the meeting through a PipeWire virtual microphone — participants see "JARVIS Virtual Mic" as the input device

This allows JARVIS to relay information, read documents, or answer questions during a live call without the user typing anything.

---

## Autonomous Desktop Agent

A vision-guided agent that operates the Linux desktop:

1. Plans a sequence of steps using a cloud vision model
2. Executes each step (click, type, scroll, key press, focus app)
3. Takes a screenshot after each action and verifies the result before proceeding
4. Abandons safely if it detects an unexpected state

Maximum depth: 25 autonomous steps per task.

---

## Telegram Remote Control

Every reply is sent as an OGG voice note synthesized with the local TTS engine, followed by the text. JARVIS sounds like itself on mobile.

**Natural language routing (no commands needed):**
- Paste a Meet/Zoom/Teams URL → joins the meeting automatically
- "send an email to x@y.com about Z" → routes to Gmail
- "say in the meeting: [message]" → speaks through the virtual mic
- Anything else → conversational response with voice + text

The wake word continues listening locally in the background while the Telegram bot is active — both channels run simultaneously.

---

## Proactive Monitor

A background daemon polls every 5 minutes:

- **Calendar**: announces approaching events via voice + desktop notification
- **Email**: scans subjects for urgency keywords
- **System resources**: warns on CPU / RAM / disk pressure
- **Morning briefing**: summarizes agenda and unread mail automatically after 7:00 AM

---

## Architecture

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full breakdown.

| Module area | Purpose |
|---|---|
| Assistant | Orchestrates STT → LLM → actions → TTS, manages state |
| TieredBrain | Three-level LLM with circuit breaker and token streaming |
| Voice | Capture, barge-in VAD, TTS with bilingual phonetics |
| Skills | 16 auto-discovered skill plugins — media, focus, GPU monitor, notes, weather… |
| Actions | 2600+ line router — NL normalization, Daily OS, Google, meetings, system |
| Gaming mode | GPU-aware VRAM manager + HP OMEN thermal profile switching |
| HUD (web) | Local web server + WebSocket + Iron Man frontend |
| Telegram | Voice-first bot with NL routing |
| MeetingAgent | Browser join + PipeWire virtual mic + live transcript |
| DesktopAgent | Vision + input automation, step verification |
| Google | Calendar NL + Gmail + Drive via OAuth2 |
| Memory | Session log + ChromaDB vector store + episodic SQLite |
| Task tracker | Persistent tasks with deadline and priority (SQLite) |
| ProactiveMonitor | Background daemon for alerts, briefings, and gaming detection |

---

## Technical Decisions

See [docs/DECISIONS.md](docs/DECISIONS.md). The most consequential:

- **Local-first LLM** — default path runs fully offline; cloud activates only when needed
- **Streaming TTS** — LLM output chunked at sentence boundaries, synthesized in parallel with generation
- **Deterministic routing before LLM** — structured requests bypass the model entirely, reducing hallucination risk and latency
- **NL normalization layer** — a single static method maps any natural phrasing to a canonical trigger before routing; no LLM needed for intent classification
- **Bilingual phonetics** — English words in Spanish text are substituted phonetically before synthesis; same voice model, no engine switch
- **Web HUD over desktop framework** — HTML/CSS/JS served by a local server; visually flexible, no native framework dependency
- **Virtual mic via PipeWire** — audio synthesized and routed through a null-sink loopback device, appearing as a real mic to meeting software
- **GPU-aware gaming mode** — proactive VRAM release and thermal management; JARVIS stays useful during gaming sessions without competing for resources

---

## Roadmap

See [docs/ROADMAP.md](docs/ROADMAP.md).

**Completed (mid 2026):**
- Streaming pipeline with barge-in interrupt
- TieredBrain with circuit breaker
- Iron Man web HUD
- 16-skill plugin system with auto-discovery
- NL normalization — any Spanish phrasing routes correctly
- Meeting agent with virtual mic and live transcription
- Telegram voice-first with NL routing
- Google Workspace (Calendar NL + Gmail send/reply/draft + Drive)
- Autonomous desktop vision agent
- Proactive monitor (calendar, email, system)
- Session memory + ChromaDB vector memory (semantic recall)
- Episodic memory — queryable daily/weekly event history
- Task tracker — deadlines, priorities, SQLite persistence
- Daily OS — morning briefing, day close, metrics, voice priorities
- Gaming mode — automatic VRAM management and thermal profiles
- systemd auto-start on login

**Planned:**
- Wake word fine-calibration — environment-specific noise profiling
- Home Assistant integration — IoT and smart home device control
- Finance agent — parse bank notification emails, weekly spending summary
- Multi-agent orchestration — expose modules as MCP tools for delegation

---

## Security and Privacy

See [docs/SECURITY.md](docs/SECURITY.md).

This repository contains no source code, credentials, personal data, or operational automation details. The private implementation handles sensitive personal and professional context — that is exactly why the source is not published.

---

## For Employers and Reviewers

This project demonstrates:

- **Systems integration** across voice, LLM, vision, browser automation, PipeWire audio, and the Linux desktop
- **Production discipline** — not a demo; a tool used and maintained daily for real tasks
- **LLM engineering** — multi-tier routing, streaming, circuit breakers, prompt context management, NL normalization without LLM dependency
- **Plugin architecture** — 16 self-contained skills with auto-discovery; adding a new capability requires one file
- **Product thinking** — the Iron Man HUD, the Daily OS layer, and the gaming mode are deliberate UX choices, not cosmetic additions
- **Security awareness** — explicit about what is and is not published, and why

A live guided walkthrough of the private implementation is available on request.

---

## License

All rights reserved. No license is granted to copy, redistribute, or create derivative works from the private implementation. See [NOTICE.md](NOTICE.md).
