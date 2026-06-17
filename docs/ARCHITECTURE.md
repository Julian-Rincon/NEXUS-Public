# Architecture

## Design Intent

JARVIS is a local-first personal assistant built for real daily use on Linux. The design prioritizes:

- low latency in the voice interaction loop
- privacy (most processing runs locally)
- reliability under real-world conditions (audio devices, background noise, network variability)
- autonomous capability without requiring the user to be at the keyboard

---

## High-Level Flow

```
Microphone
    │
    ▼
[Wake word detection]  ←── always-on background thread
    │
    ▼
[Speech-to-text]  ←── fallback chain (cloud → local)
    │
    ▼
[Intent router]
    ├──► Deterministic action  (no LLM — fast, reliable)
    │
    └──► TieredBrain
              Level 1: Ollama (local)
              Level 2: Groq (cloud premium)
              Level 3: Gemini (fallback)
                  │
                  ▼ token stream
          [Streaming TTS]
              sentence boundary chunking
              parallel synthesis + playback
                  │
              [Barge-in VAD] ◄── microphone (always listening)
                  │
                  ▼
              Speaker
```

Additional entry points: HUD web, Telegram, text/debug mode.

---

## Module Areas

### Assistant

Orchestrates every turn in the conversation:
- receives transcribed input from STT or text
- routes to intent router or TieredBrain
- manages session memory injection
- triggers TTS and action execution
- handles barge-in state transitions

### TieredBrain

Three-level LLM with automatic routing and circuit breaker:

| Level | Backend | When |
|---|---|---|
| 1 | Ollama (local) | Simple conversational requests |
| 2 | Groq (cloud) | Complex reasoning, code, analysis |
| 3 | Gemini (cloud) | Circuit breaker fallback |

The circuit breaker pauses a backend after repeated failures and retries it after a cool-down period. All three backends share the same conversation history and context.

### Voice

Three sub-concerns handled independently:

**Capture**: microphone input with configurable device selection.

**STT**: primary cloud model + local model fallback. The local model handles offline and latency-sensitive scenarios. Both paths run through the same interface.

**TTS**: local Piper synthesis with a Spanish voice profile tuned for a characterful, low-pitched spoken response. A phonetic substitution pass handles English technical words before synthesis, keeping the same voice model and avoiding engine switching.

**Barge-in**: a VAD model processes microphone frames continuously while JARVIS speaks. If speech is detected, playback stops within ~40 ms and the captured audio (including pre-roll from before the interrupt) feeds into the next STT round.

### Actions

An intent router classifies user input before involving the LLM. Deterministic routes include:

- application control (open, focus, close)
- screenshot and screen analysis
- file and folder operations
- Google Calendar, Gmail, Drive queries
- system diagnostics
- meeting control (join, transcript, speak)

For requests that do not match a deterministic route, the LLM produces a response and the output may trigger tool calls back into the action layer.

### HUD (Web)

A local web server (HTTP + WebSocket on the same port) serves a single-file Iron Man–style interface. The frontend connects via WebSocket and receives:

- chat messages (user and JARVIS turns)
- system metrics (CPU, RAM, disk — updated every 4 s)
- latency readings (STT, LLM, TTS)
- API status for each configured backend
- pending action confirmations

The interface opens automatically in the browser as a PWA window on startup.

### Telegram

The bot responds voice-first: every reply is synthesized locally to an OGG audio file using the same TTS pipeline and sent as a voice note, followed by the text. Natural language routing handles meeting URLs, email intents, and meeting speech without explicit commands.

A wake word listener runs in a background thread simultaneously with the bot — the local voice interface and Telegram both remain active.

### MeetingAgent

Manages the full lifecycle of a remote meeting:

1. Browser automation joins the meeting (Playwright / Chromium)
2. PipeWire captures the meeting audio stream
3. A background thread transcribes the audio incrementally every 15 seconds
4. On meeting end: full STT pass + LLM summary + Calendar event

The virtual microphone is a PipeWire null-sink with a loopback source. TTS output is routed through this device, which appears to meeting software as a real microphone input.

### DesktopAgent

A vision-guided autonomous agent for the Linux desktop:

1. Receives a natural language task description
2. Plans a step sequence using a cloud vision model
3. Executes steps through the input layer (xdotool for XWayland; AT-SPI2 accessibility API as fallback)
4. After each action: screenshot → cloud vision verification → proceed or abort

Maximum: 25 steps per task. The agent abandons if it detects an unexpected state rather than continuing blindly.

### Google

OAuth2 integration (Calendar + Gmail + Drive) with automatic token refresh and scope validation. If a stored token is missing required scopes, it is discarded and re-authorization is triggered on next use.

Capabilities:
- Calendar: list, create, natural-language datetime parsing
- Gmail: list unread, send, reply, draft, read full message
- Drive: list files, search, read content (exports Docs as plain text), create text files and Docs

### Memory

**Session memory**: previous sessions are summarized to a JSONL file. At startup, recent summaries are injected into the conversation history as a context block. Maximum depth: 50 sessions, last 5 injected.

**Vector memory**: conversation turns are embedded and stored in ChromaDB. Relevant past context is retrieved by semantic similarity on each new user message.

### ProactiveMonitor

A background daemon that polls every 5 minutes:
- Checks calendar for upcoming events; alerts when an event is within 5–15 minutes
- Scans Gmail for urgency keywords in unread message subjects
- Monitors CPU, RAM, and disk usage; alerts on sustained high load
- Delivers a morning briefing (agenda + unread mail) automatically after 07:00

---

## Reliability Design

### Process isolation

The Telegram bot runs as a systemd user service and restarts automatically on failure. The voice pipeline and HUD are separate processes from the bot.

### Startup safety

Background services start only after the main process is stable. A safe-mode flag allows starting the UI without activating secondary services — useful for diagnostics.

### Graceful degradation

Every component has a fallback path:
- STT: cloud → local model
- LLM: Ollama → Groq → Gemini (circuit breaker per level)
- Screenshots: spectacle → scrot → PIL screenshot
- TTS playback: paplay → aplay → sounddevice

---

## Future Direction

The architecture is evolving toward:
- episodic memory with causal chains across sessions
- MCP tool exposure for multi-agent delegation
- Home Assistant integration for physical device control
- a finance agent layer reading bank notification emails
