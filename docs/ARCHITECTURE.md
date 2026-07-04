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

Every backend maintains live telemetry — call count, failure count, latency as an exponential moving average, and circuit-breaker state — surfaced as live chips in the desktop HUD and queryable by voice. Local-model GPU residency is also bounded: the model's keep-alive window and context allocation are capped so an idle assistant never creeps toward VRAM exhaustion (see Reliability Design below).

### ConnectionSentinel

A dedicated health-check layer for every external dependency: all LLM tiers, STT, TTS (including remaining-quota checks), the messaging channel, and OAuth token validity. Each dependency resolves to one of four states (ok / warn / fail / off). The sentinel sweeps on a regular interval and alerts **only on state transitions**, never on every check — so a persistent degradation is reported once, not every fifteen minutes.

It is designed to catch quiet failures that produce no exception: a provider silently deprecating a model, an API key that still works for synthesis but has lost account-info scope (distinguished from an invalid key), or the local inference runtime falling back to CPU after a system package update — a 30x latency regression with no error message.

### Voice

Three sub-concerns handled independently:

**Capture**: microphone input with configurable device selection.

**STT**: primary cloud model + local model fallback. The local model handles offline and latency-sensitive scenarios. Both paths run through the same interface.

**TTS**: cloud-based voice cloning as the primary path (low-latency streaming synthesis), with an automatic local fallback if the primary is unavailable for a given call — the system retries the primary on the next turn rather than staying downgraded. A phonetic substitution pass handles English technical words on the local fallback voice before synthesis, keeping the same voice model and avoiding engine switching.

**Wake word**: a purpose-trained detection model runs always-on. Because a lone detection model tuned loose enough to never miss also fires on sneezes and door slams, activation requires three things in sequence: a detection score above threshold, the score *sustained* across consecutive audio frames (transient spikes are rejected), and a final STT double-check — the audio around the trigger is transcribed locally and fuzzy-matched against the wake word before NEXUS actually wakes.

**Barge-in**: a VAD model processes microphone frames continuously while NEXUS speaks. If speech is detected, playback stops within ~40 ms and the captured audio (including pre-roll from before the interrupt) feeds into the next STT round.

**TTS phrase cache**: common phrases are cached to disk keyed by voice configuration (backend + voice + model), so repeating a phrase costs nothing. The cache only ever *persists* audio produced by the primary voice: if a synthesis call fell back to the local engine, that audio is played but never stored, so a temporary outage can't permanently entrench the fallback voice for a cached phrase. Short responses are cached opportunistically as they occur, which materially reduces paid synthesis volume in daily use.

### Actions

An intent router classifies user input before involving the LLM. Deterministic routes include:

- application control (open, focus, close)
- screenshot and screen analysis
- file and folder operations
- Google Calendar, Gmail, Drive queries
- system diagnostics and resource profile switching
- meeting control (join, transcript, speak)
- guarded system power control (lock, suspend, hibernate, shutdown, restart)

For requests that do not match a deterministic route, a natural-language classifier (local pattern matching → fast cloud classification → local model as a last resort) attempts to map the phrase to a known capability before falling back to a full conversational LLM response. Informational questions ("what is…", "how does… work", "who is…") are detected at this stage and routed to live web research — a multi-engine search with LLM synthesis — instead of letting a model answer from stale training data or refuse.

### SkillSelector

With 38 skills, describing every skill to the LLM on every turn degrades response quality (context rot) and wastes tokens. A retrieval layer scores skills against the query — semantic similarity when an embedding function is available, a deterministic lexical mode otherwise, with declared trigger phrases weighted double — and surfaces only the top-K relevant skills. The full registry still handles deterministic trigger matching; retrieval only shapes what the *LLM* sees.

### Heartbeat — Proactivity Judgment

Detection and delivery are separate layers by design. Monitors (calendar, email, resources, security, connections) detect everything and stay simple; the Heartbeat sits at the delivery boundary and decides what actually reaches the user:

- duplicate alerts within a time window are collapsed
- quiet hours pass only genuine urgencies
- gaming and focus sessions suppress all non-urgent alerts
- a rate limit caps normal alerts per hour

Nothing suppressed is dropped: held alerts accumulate into a digest, delivered once when the suppressing mode ends or with the next briefing. Alert policy therefore lives in exactly one place — adding a new monitor requires no thought about etiquette.

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

When the user does decide to act, two defensive actions are available on request — kill a process, or isolate the machine from the network — and both are treated as high-risk: they queue behind the same explicit-confirmation mechanism as power control, and process termination is additionally checked against a whitelist of protected system processes that can never be targeted. Restoring the network is immediate (reversal of a defensive action should never have friction).

### Resource Manager

Switches CPU/RAM scheduling priority between profiles (work, streaming, focus, normal) depending on what the user is doing, using standard OS priority mechanisms. Manual per-application prioritization is also available.

### HUD (Desktop)

A single-panel control center, styled after the void-black/blood-red identity used throughout, with an animated red "digital rain" background behind the panel:

- title, tagline, and a small tag line naming the active stack
- a live backend bar (LLM tier count, STT/TTS models, wake model, skill count, GPU status) read from configuration and hardware at runtime
- one full-width log combining a real boot sequence with the ongoing conversation
- a command input at the bottom

Operational controls (voice round, Telegram start/stop, wake word restart, self-check, settings) live in the system tray menu, not as buttons in the main window — the panel stays focused on what NEXUS is doing. Detailed system resource usage (CPU, RAM, disk, GPU) is queried on demand through voice or Telegram rather than rendered as a permanent HUD gauge.

The HUD is resource-disciplined: the animated background drops its frame rate when the window is unfocused and pauses entirely when hidden, and each status source polls at an interval matched to how fast it actually changes rather than a single aggressive timer. A profiling pass across these paths cut the HUD's idle CPU use by roughly 75%.

Independent of the HUD window, a small animated **speaking indicator** appears on screen whenever NEXUS is talking. It is driven by a cross-process signal (any NEXUS process that speaks can raise it, with a staleness window so a crash can't leave it stuck), rendered click-through, and pinned to the compositor's overlay layer so it remains visible above fullscreen applications — including games.

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

**Representative mode**: NEXUS can attend a meeting as the user, not just alongside them. It joins with an authenticated real-browser profile (meeting platforms reject anonymous automation), presents an animated visual identity through a virtual camera device, speaks with the cloned voice, transcribes the room audio continuously, and generates a spoken reply when addressed by name. Two hard rules from operational experience: the system's default audio input is never modified (the virtual mic is created per-meeting, selected only inside the meeting app, and torn down afterwards), and every cloned-voice utterance goes to an audit log — when, where, and what was said, with sensitive content redacted.

### MeetingEmailWatcher

The meeting pipeline starts at the inbox. A watcher monitors unread mail through an email classifier trained on the user's real inbox — categories like recruiting-process mail, meeting invitations, job alerts, finance, development, and promotional noise, tuned against real data rather than keyword lists (a lesson from that data: job-board notifications are signal, not spam). High-value categories skip the prefilter entirely; noise can never create a calendar event.

For a genuine invitation, the watcher extracts structured details with the LLM, creates a calendar event carrying the email's context, and records it in memory. Before creating anything it checks for an existing same-day event with a similar title or overlapping time — an update to an already-scheduled meeting produces a heads-up referencing the existing event instead of a duplicate. Confirmed events can trigger scheduled auto-attendance, idempotently, inside a defined time window.

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

### MCP — Server and Client

**Server**: exposes a subset of NEXUS's own capabilities (query the LLM, check the calendar, list unread mail, check system status, list Drive files, run a text command) as tools over the Model Context Protocol, so external AI clients can call into the system directly.

**Client**: the reverse direction — NEXUS consumes external MCP tool servers declared in a configuration file, listing and calling their tools through the protocol with no per-integration wrapper code. A synchronous facade wraps the protocol's async API, and the connection layer is isolated behind a single seam so integrations are testable without live servers.

### Cast (TV Presence)

NEXUS can speak through the living-room TV: it synthesizes the announcement with the active TTS pipeline, serves the audio over a short-lived local HTTP endpoint, and casts it to the device — confirming the device actually fetched the audio and retrying across candidate local network interfaces if not (multi-homed machines make "which IP can the TV reach" a real question). Media playback, volume control, pause, and status are also available. Cast commands require an explicit device word ("…on the TV") so the skill never captures ordinary media commands intended for the desktop.

### Memory

**Session memory**: previous sessions are summarized to a log file. At startup, recent summaries are injected into the conversation history as a context block.

**Vector memory**: conversation turns are embedded and stored for semantic retrieval. Relevant past context is retrieved by similarity on each new user message — except during gaming mode, when embedding calls are skipped entirely and retrieval falls back to recent-message history only, so the local embedding model does not compete with the game for CPU.

**Episodic memory**: a separate, causal event log queryable by day or week, distinct from semantic similarity search.

**Managed long-term memory**: after each session, an LLM pass extracts durable facts from the conversation and promotes them: stable preferences and environment facts go to a **core** store injected into the system prompt every session; situational facts go to an **archival** store retrieved on demand. Lexical dedup prevents the stores from accumulating restatements, and writes are atomic.

**Redaction before persistence**: everything headed for any persistent store — session summaries, extracted facts, audit entries — passes through a redaction layer that scrubs API keys, tokens, JWTs, PEM blocks, and credential-shaped strings first. Memory is useful precisely because it is long-lived, which is exactly why secrets must never enter it.

### ProactiveMonitor

A background daemon that polls at regular intervals:
- Checks calendar for upcoming events; alerts ahead of time
- Classifies incoming mail by urgency and importance, using the inbox-trained classifier
- Monitors CPU, RAM, and disk usage; alerts on sustained pressure
- Surfaces provider health changes from the ConnectionSentinel
- Surfaces anomalies from the SecurityWatcher
- Tails the active project's logs for new errors (LogWatcher), tied to the last known work context
- Organizes downloads and scratch files by type, and warns on low disk space (FileJanitor)
- Delivers a morning briefing automatically

Every alert produced here passes through the Heartbeat judgment layer before reaching the user (see above) — the monitor detects, the Heartbeat decides.

---

## Reliability Design

### Process isolation

The Telegram bot runs as a background service and restarts automatically on failure. The voice pipeline and HUD are separate processes from the bot. The remote-bot connection is additionally a **process-level singleton**: an exclusive OS-level lock guarantees only one process polls the messaging API at a time, because two competing pollers produce confusing split responses and API conflicts (a failure mode observed in production, not hypothesized).

### VRAM as a first-class resource

A real incident shaped this: with a game, the local LLM, and the desktop graphics stack all resident, VRAM exhaustion froze the machine — the GPU driver does not degrade gracefully when memory runs out. The permanent guards that came out of the post-mortem:

- **VramGuard** — a monitor that tracks GPU memory headroom and proactively evicts the local model before pressure becomes exhaustion
- **Capped local-model residency** — the local model's keep-alive window and context allocation are bounded so idle residency never grows unbounded
- **Per-game VRAM budgets** — games that overcommit are capped at the API level so their streaming planners behave (see Gaming-Aware Resource Isolation)

The incident became test-covered guards, not a note to be careful.

### Startup safety

Background services start only after the main process is stable. A safe-mode flag allows starting the UI without activating secondary services — useful for diagnostics.

### Graceful degradation

Every component has a fallback path:
- STT: cloud → local model
- TTS: cloud voice clone → local synthesis (retried on the primary again next turn, never permanently downgraded)
- LLM: local → fast cloud → high-throughput cloud → large-context cloud → fallback cloud (circuit breaker per level)
- Screenshots: multiple capture tools tried in order
- TTS playback: multiple playback backends tried in order

### Gaming-Aware Resource Isolation (v2)

**Entry is event-driven.** A universal launch wrapper, set once as the launch command for every game in the library, writes a signal file at the moment a game process starts — carrying the game's literal name resolved from the store's local manifest. Gaming mode enters at second zero with the game identified by name, for any game, with no hardcoded list. The previous heuristic (process signature + GPU memory usage, up to a minute of detection latency) remains as a fallback for games launched outside the wrapper, and a hook in the OS game-optimization daemon covers third-party launchers.

**The wrapper also applies a per-game profile before the game starts**: a persistent unlimited shader cache for all titles, and optional per-game environment tweaks from a small profile file — including a cap on the VRAM the game *sees*, so its asset streaming budgets with headroom instead of exhausting the GPU (see the incident note under Reliability Design).

On entering gaming mode:

1. The local LLM is unloaded from GPU memory
2. The LLM router, the natural-language classification fallback, and the semantic-memory embedding path all stop calling the local model for the duration of the session — every one of those paths is individually gated, not just the main conversational route
3. A per-game GPU power limit is applied through a narrow privileged helper (validated wattage range, no arbitrary arguments) — heavy titles get full power, light ones none
4. The system thermal profile switches to a performance mode
5. Kernel swap behavior is tuned down to reduce memory-latency spikes
6. Background scans are paused, and NEXUS's own background service lowers its CPU and IO scheduling weight — it only yields under contention, costing nothing when the system is idle
7. Non-urgent alerts are suppressed by the Heartbeat and digested for after the session

Everything restores automatically when the game session ends, and NEXUS announces the session summary — game name and duration.

---

## Future Direction

- Automatic entity discovery for the home-automation integration
- Budgeting-tool integration for the finance agent
- Global dictation — hotkey → STT → typed into whatever has focus
- A deep research agent for long-running investigations
- Deeper multi-agent delegation over the MCP-exposed tool surface
