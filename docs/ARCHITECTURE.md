# Architecture

## Design Intent

NEXUS is a local-first personal AI system built for real daily use on Linux. The design prioritizes:

- low latency in the voice interaction loop
- privacy (most processing runs locally by default)
- reliability under real-world conditions (audio devices, background noise, network variability, GPU contention from other workloads)
- autonomous capability without requiring the user to be at the keyboard
- safety by construction — irreversible actions are gated, not just documented as risky

---

## High-Level Flow

```
Microphone
    │
    ▼
[Custom wake word detection]  ←── always-on background thread
    │
    ▼
[Speech-to-text]  ←── fallback chain (cloud primary → local)
    │
    ▼
[Intent router]
    ├──► Deterministic action  (no LLM — fast, reliable)
    │
    └──► TieredBrain
              Level 1: local model (offline)
              Level 2: cloud, fast general-purpose
              Level 3: cloud, high-throughput
              Level 4: cloud, large context / multimodal
              Level 5: cloud, fallback
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

Additional entry points: desktop HUD, Telegram, text/debug mode.

---

## Module Areas

### Assistant

Orchestrates every turn in the conversation:
- receives transcribed input from STT or text
- routes to intent router or TieredBrain
- manages session and semantic memory injection
- triggers TTS and action execution
- handles barge-in state transitions

### TieredBrain

Five-level LLM with automatic routing and circuit breaker:

| Level | Backend | When |
|---|---|---|
| 1 | Local model | Simple conversational requests, fully offline |
| 2 | Cloud (fast, general-purpose) | Complex reasoning, code, analysis |
| 3 | Cloud (high-throughput) | Long responses, instant turnaround |
| 4 | Cloud (large-context, multimodal) | Long-context tasks, vision-capable requests |
| 5 | Cloud (fallback) | Final safety net if everything above is unavailable |

The circuit breaker pauses a backend after repeated failures and retries it after a cool-down. All five backends share the same conversation history and context. Local-tier timeouts are intentionally short: a slow or cold local model degrades to the next cloud tier in seconds rather than making the user wait.

During gaming mode, the local tier is skipped entirely at the routing level — not just deprioritized. This includes the LLM tier, the natural-language classification fallback, and the semantic memory embedding calls, so nothing local competes with the game for CPU or GPU.

### Voice

Three sub-concerns handled independently:

**Capture**: microphone input with configurable device selection.

**STT**: primary cloud model + local model fallback. The local model handles offline and latency-sensitive scenarios. Both paths run through the same interface.

**TTS**: cloud-based voice cloning as the primary path (low-latency streaming synthesis), with an automatic local fallback if the primary is unavailable for a given call — the system retries the primary on the next turn rather than staying downgraded. A phonetic substitution pass handles English technical words on the local fallback voice before synthesis, keeping the same voice model and avoiding engine switching.

**Barge-in**: a VAD model processes microphone frames continuously while NEXUS speaks. If speech is detected, playback stops within ~40 ms and the captured audio (including pre-roll from before the interrupt) feeds into the next STT round.

### Actions

An intent router classifies user input before involving the LLM. Deterministic routes include:

- application control (open, focus, close)
- screenshot and screen analysis
- file and folder operations
- Google Calendar, Gmail, Drive queries
- system diagnostics and resource profile switching
- meeting control (join, transcript, speak)
- guarded system power control (lock, suspend, hibernate, shutdown, restart)

For requests that do not match a deterministic route, a natural-language classifier (local pattern matching → fast cloud classification → local model as a last resort) attempts to map the phrase to a known capability before falling back to a full conversational LLM response.

### Confirmation Gating

Any action that changes external or system state beyond simple reads — sending mail, executing generated automation, changing machine power state — is queued behind an explicit confirmation step rather than executing on the first ambiguous match. A pending request carries a risk level, an expiry window, and (for remote channels) a short confirmation token. Low-risk, easily reversible actions (like locking the screen) remain immediate.

This exists because free-text natural-language matching is inherently approximate: a phrase can resemble a destructive command without being one. Gating the execution — not just the interpretation — is what makes that acceptable.

### SecurityWatcher

A passive background monitor, integrated into the proactive daemon:

- Unknown processes with sustained high CPU usage (whitelist-based)
- Newly opened network ports compared to a startup baseline
- New USB devices
- Failed SSH authentication attempts

It only detects and alerts — it never modifies firewall rules, kills processes, or changes system configuration on its own. The user is always the one who decides what to do about an alert.

### Resource Manager

Switches CPU/RAM scheduling priority between profiles (work, streaming, focus, normal) depending on what the user is doing, using standard OS priority mechanisms. Manual per-application prioritization is also available.

### HUD (Desktop)

A native control center that runs alongside the voice pipeline:

- real-time conversation log
- an animated status core for the local pipeline
- status cards per subsystem (audio, Telegram, wake word, browser automation, desktop automation, Google credentials)
- live STT/LLM/TTS latency tiles, color-coded when a stage exceeds its expected threshold
- system tray integration

Detailed system resource usage (CPU, RAM, disk, GPU) is queried on demand through voice or Telegram rather than rendered as a permanent HUD gauge.

### Telegram

The bot responds voice-first: every reply is synthesized to an audio file using the same TTS pipeline and sent as a voice note, followed by the text. Natural language routing handles meeting URLs, email intents, and meeting speech without explicit commands.

A wake word listener runs in a background thread simultaneously with the bot — the local voice interface and Telegram both remain active at once.

### MeetingAgent

Manages the full lifecycle of a remote meeting:

1. Browser automation joins the meeting
2. The system audio stack captures the meeting audio stream
3. A background thread transcribes the audio incrementally
4. On meeting end: full STT pass + LLM summary + Calendar event

The virtual microphone is a loopback audio device. TTS output is routed through this device, which appears to meeting software as a real microphone input.

### DesktopAgent

A vision-guided autonomous agent for the Linux desktop:

1. Receives a natural language task description
2. Plans a step sequence using a cloud vision model
3. Executes steps through an allowlisted input layer
4. After each action: screenshot → cloud vision verification → proceed or abort

Hard safety constraints, independent of model output: application launches are restricted to a known-app allowlist, URL opens are restricted to `http(s)` schemes, and keyboard shortcut strings are pattern-validated before being sent to the input layer. Maximum: 25 steps per task. The agent abandons if it detects an unexpected state rather than continuing blindly.

### Google

OAuth2 integration (Calendar + Gmail + Drive) with automatic token refresh and scope validation. If a stored token is missing required scopes, it is discarded and re-authorization is triggered on next use.

Capabilities:
- Calendar: list, create, natural-language datetime parsing
- Gmail: list unread, send, reply, draft, read full message
- Drive: list files, search, read content, create text files and Docs

### Home Automation

A client for a local home-automation hub, controlling lights, devices, and scenes via natural language, with no cloud dependency for the control path itself.

### Finance

Parses bank notification emails, extracts transaction amounts and merchants, and produces a spending summary on request.

### MCP Server

Exposes a subset of NEXUS's own capabilities (query the LLM, check the calendar, list unread mail, check system status, list Drive files, run a text command) as tools over the Model Context Protocol, so external AI clients can call into the system directly.

### Memory

**Session memory**: previous sessions are summarized to a log file. At startup, recent summaries are injected into the conversation history as a context block.

**Vector memory**: conversation turns are embedded and stored for semantic retrieval. Relevant past context is retrieved by similarity on each new user message — except during gaming mode, when embedding calls are skipped entirely and retrieval falls back to recent-message history only, so the local embedding model does not compete with the game for CPU.

**Episodic memory**: a separate, causal event log queryable by day or week, distinct from semantic similarity search.

### ProactiveMonitor

A background daemon that polls at regular intervals:
- Checks calendar for upcoming events; alerts ahead of time
- Classifies incoming mail by urgency and importance
- Monitors CPU, RAM, and disk usage; alerts on sustained pressure
- Surfaces anomalies from the SecurityWatcher
- Tails the active project's logs for new errors (LogWatcher), tied to the last known work context
- Organizes downloads and scratch files by type, and warns on low disk space (FileJanitor)
- Delivers a morning briefing automatically

---

## Reliability Design

### Process isolation

The Telegram bot runs as a background service and restarts automatically on failure. The voice pipeline and HUD are separate processes from the bot.

### Startup safety

Background services start only after the main process is stable. A safe-mode flag allows starting the UI without activating secondary services — useful for diagnostics.

### Graceful degradation

Every component has a fallback path:
- STT: cloud → local model
- TTS: cloud voice clone → local synthesis (retried on the primary again next turn, never permanently downgraded)
- LLM: local → fast cloud → high-throughput cloud → large-context cloud → fallback cloud (circuit breaker per level)
- Screenshots: multiple capture tools tried in order
- TTS playback: multiple playback backends tried in order

### Gaming-Aware Resource Isolation

A background monitor detects active game sessions (process signature + GPU memory usage). On detection:

1. The local LLM is unloaded from GPU memory
2. The LLM router, the natural-language classification fallback, and the semantic-memory embedding path all stop calling the local model for the duration of the session — every one of those paths is individually gated, not just the main conversational route
3. The system thermal profile switches to a performance mode
4. Kernel swap behavior is tuned down to reduce memory-latency spikes
5. Background scans that compete for CPU are paused

Everything restores automatically when the game session ends.

---

## Future Direction

- Fine-calibration of the wake word model to the specific microphone and room acoustics
- Automatic entity discovery for the home-automation integration
- Budgeting-tool integration for the finance agent
- Deeper multi-agent delegation over the MCP-exposed tool surface
