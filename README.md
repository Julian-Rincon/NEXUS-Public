# JARVIS Ironman

> **Portfolio showcase** — public documentation of a private project.  
> Source code, credentials, and operational details are intentionally excluded.

A local-first, production-grade personal AI assistant built for real daily use on Linux (Fedora / KDE Plasma). Inspired by the JARVIS interaction model from Iron Man — voice-driven, autonomous, and always running.

---

## What It Does

JARVIS is used daily as an actual productivity layer, not a demo. It handles:

- **Voice conversation** with sub-second response (streaming TTS, barge-in interrupt)
- **Autonomous desktop control** — opens apps, clicks, types, reads the screen
- **Meeting participation** — joins Meet/Zoom/Teams, transcribes in real time, and can speak inside the meeting through a virtual microphone
- **Google Workspace** — sends emails, creates calendar events in natural language, reads Drive files
- **Proactive alerts** — notifies when a calendar event is approaching, urgent email arrives, or system resources are stressed
- **Telegram remote control** — voice-note-first responses from anywhere
- **Iron Man HUD** — a web-based operator console with animated arc reactor, live system metrics, and real-time conversation bus

---

## Capability Overview

| Capability | Status |
|---|---|
| Voice conversation (streaming) | Active — first spoken word in ~0.5 s |
| Barge-in interrupt | Active — VAD-based, ~40 ms reaction |
| Wake word (passive listening) | Active |
| Spanish voice (Bond profile) | Active — local TTS, bilingual phonetics |
| Iron Man HUD (web) | Active — arc reactor UI, WebSocket, live metrics |
| Autonomous desktop agent | Active — vision + input, up to 25 steps |
| Meeting agent | Active — join + virtual mic + live transcript + summary |
| Proactive monitor | Active — calendar, email, system resources |
| Telegram (voice-first) | Active — replies with voice note + text |
| Google Calendar | Active — natural language event creation and query |
| Gmail | Active — send, reply, draft |
| Google Drive | Active — list, read, create |
| Session memory | Active — recent context injected automatically |
| Vector memory | Active — semantic recall (ChromaDB) |
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

The HUD communicates with the assistant over a local WebSocket. All metrics update every 4 seconds without page refresh.

---

## Voice Pipeline

```
Microphone
    │
    ▼
Wake word detection  ←── passive background thread
    │
    ▼
Speech-to-text  ←── fallback chain (cloud primary → local)
    │
    ▼
Intent router ──► deterministic action (no LLM needed)
    │              open app / screenshot / calendar read / etc.
    ▼
Tiered LLM brain
  Level 1: local model (Ollama)  — fast, fully private
  Level 2: cloud premium (Groq)  — complex reasoning
  Level 3: fallback (Gemini)     — circuit breaker recovery
    │
    ▼
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
| Actions | Router for 30+ deterministic and agentic operations |
| HUD (web) | Local web server + WebSocket + Iron Man frontend |
| Telegram | Voice-first bot with NL routing |
| MeetingAgent | Browser join + PipeWire virtual mic + live transcript |
| DesktopAgent | Vision + input automation, step verification |
| Google | Calendar NL + Gmail + Drive via OAuth2 |
| Memory | Session log + ChromaDB vector store |
| ProactiveMonitor | Background daemon for alerts and briefings |

---

## Technical Decisions

See [docs/DECISIONS.md](docs/DECISIONS.md). The most consequential:

- **Local-first LLM** — default path runs fully offline; cloud activates only when needed
- **Streaming TTS** — LLM output chunked at sentence boundaries, synthesized in parallel with generation
- **Deterministic routing before LLM** — structured requests bypass the model entirely, reducing hallucination risk
- **Bilingual phonetics** — English words in Spanish text are substituted phonetically before synthesis; same voice model, no engine switch
- **Web HUD over desktop framework** — HTML/CSS/JS served by a local server; visually flexible, no native framework dependency
- **Virtual mic via PipeWire** — audio synthesized and routed through a null-sink loopback device, appearing as a real mic to meeting software

---

## Roadmap

See [docs/ROADMAP.md](docs/ROADMAP.md).

**Completed (mid 2026):**
- Streaming pipeline with barge-in interrupt
- TieredBrain with circuit breaker
- Iron Man web HUD
- Meeting agent with virtual mic and live transcription
- Telegram voice-first with NL routing
- Google Workspace (Calendar NL + Gmail send/reply/draft + Drive)
- Autonomous desktop vision agent
- Proactive monitor (calendar, email, system)
- systemd auto-start on login

**Planned:**
- Episodic memory — causal log of events and decisions across sessions
- Life OS task tracker — persistent projects with follow-up nudges
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
- **LLM engineering** — multi-tier routing, streaming, circuit breakers, prompt context management
- **Product thinking** — the Iron Man HUD is a deliberate UX choice, not a cosmetic addition
- **Security awareness** — explicit about what is and is not published, and why

A live guided walkthrough of the private implementation is available on request.

---

## License

All rights reserved. No license is granted to copy, redistribute, or create derivative works from the private implementation. See [NOTICE.md](NOTICE.md).
