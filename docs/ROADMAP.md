# Roadmap

## Completed (mid 2026)

### Core voice pipeline
- [x] Streaming LLM → TTS — first spoken word in ~0.5 s
- [x] Barge-in with VAD — interrupts playback in ~40 ms
- [x] Wake word (passive background detection)
- [x] TTS phrase cache — zero synthesis latency for common phrases
- [x] Hybrid STT — cloud primary with local fallback
- [x] Bilingual phonetics — English words pronounced correctly in Spanish speech, same voice model

### Intelligence layer
- [x] TieredBrain — three-level LLM with circuit breaker
- [x] Deterministic intent routing before LLM involvement
- [x] Session memory — recent sessions injected as context
- [x] Vector memory — ChromaDB semantic recall

### Operator interface
- [x] Iron Man web HUD — arc reactor SVG, WebSocket, live system metrics
- [x] tkinter desktop HUD (legacy, still maintained)
- [x] Telegram bot — voice-first (OGG notes + text), NL routing
- [x] Wake word active simultaneously with Telegram bot

### Agentic capabilities
- [x] Autonomous desktop agent — vision + input, up to 25 steps, step verification
- [x] Meeting agent — browser join + PipeWire virtual mic + live transcription + summary
- [x] Proactive monitor — calendar alerts, email urgency, system resource warnings, morning briefing

### Google Workspace
- [x] Calendar — list, create, natural-language datetime parsing
- [x] Gmail — list unread, send, reply to thread, draft, read full message
- [x] Drive — list, search, read, create text files and Docs

### Infrastructure
- [x] systemd user service — Telegram bot auto-starts on login
- [x] PipeWire virtual microphone — persistent across reboots
- [x] Google OAuth2 — automatic token refresh and scope validation

---

## Planned

### Near term

**Episodic memory**  
A timestamped causal log of what happened, what was decided, and what was said — distinct from semantic vector recall. Enables references like "last Tuesday you said you'd call Luis — did that happen?" without a prompt.

**Life OS task tracker**  
Persistent project and task graph with deadlines, blockers, and automated follow-up nudges. The assistant closes the loop rather than just recording tasks.

### Mid term

**Home Assistant integration**  
Control lights, locks, sensors, and HVAC through natural language. Local API, no cloud required. Bridge to 3,000+ compatible devices.

**Finance agent**  
Parse bank notification emails from Gmail, categorize transactions, and deliver a weekly spending summary with alerts on unusual activity.

**Passive life logging**  
Background process that logs application activity, meeting attendance, and document interactions. Periodic synthesis pass identifies patterns and surfaces insights proactively.

### Longer term

**Multi-agent architecture (MCP)**  
Expose JARVIS capabilities as Model Context Protocol tools. Specialized sub-agents (research, coding, finance) can be delegated tasks without overloading the main conversational brain.

**Stronger operator console**  
Visual timeline of past sessions, task graph display, and memory browser in the HUD.

---

## Non-Goals for the Public Repository

- open-source release of the private implementation
- exposure of personal workflow automation or local machine details
- publication of prompt assets or behavioral tuning
