# Roadmap

## Completed

### Core voice pipeline
- [x] Streaming LLM → TTS — first spoken word in under a second
- [x] Barge-in with VAD — interrupts playback in ~40 ms
- [x] Custom-trained wake word — purpose-built detection model, no generic keyword spotting
- [x] TTS phrase cache — zero synthesis latency for common phrases
- [x] Cloud voice cloning (primary) with automatic local fallback
- [x] Hybrid STT — cloud primary with local fallback
- [x] Bilingual phonetics on the local fallback voice — English words pronounced correctly in Spanish speech

### Intelligence layer
- [x] TieredBrain — five-level LLM with circuit breaker (local, fast cloud, high-throughput cloud, large-context/multimodal cloud, fallback cloud)
- [x] Deterministic intent routing before LLM involvement
- [x] Natural-language classification fallback (local pattern match → fast cloud classification → local model as last resort)
- [x] Session memory — recent sessions injected as context
- [x] Vector memory — semantic recall
- [x] Episodic memory — causal event log, queryable by day/week
- [x] Aggressive local-tier timeouts with circuit breaking, so a slow or cold local model never blocks a response for long

### Operator interface
- [x] Native desktop HUD — live system metrics, system tray integration
- [x] Telegram bot — voice-first (voice notes + text), NL routing
- [x] Wake word active simultaneously with the Telegram bot

### Agentic capabilities
- [x] Autonomous desktop agent — vision + input, up to 25 steps, step verification, hard-allowlisted for safety
- [x] Meeting agent — browser join + virtual mic + live transcription + summary
- [x] Proactive monitor — calendar alerts, email urgency, system resource warnings, security anomalies, morning briefing

### Google Workspace
- [x] Calendar — list, create, natural-language datetime parsing
- [x] Gmail — list unread, send, reply to thread, draft, read full message
- [x] Drive — list, search, read, create text files and Docs

### Expanded skills (v2)
- [x] Resource manager — CPU/RAM priority profiles per activity
- [x] Dev workflow automation — test runner detection, log triage, build, server restart
- [x] Passive security watcher — process/port/USB/auth anomaly detection, alert-only, never acts unilaterally
- [x] Natural-language code generation — spoken request → validated, saved source file
- [x] Home automation — IoT device and scene control by voice
- [x] Personal finance summary — bank notification email parsing

### Safety
- [x] Confirmation gating for irreversible actions (mail, automation, machine power state) with risk levels, expiry, and remote-channel tokens
- [x] Guarded system power control — lock is immediate; suspend/hibernate/shutdown/restart require explicit confirmation
- [x] Hard allowlists for autonomous desktop actions (known-app list, http(s)-only URL opens, pattern-validated key combos)

### Multi-tool exposure
- [x] MCP server exposing core capabilities as tools for external AI clients

### Infrastructure
- [x] Background service for the Telegram bot — auto-starts on login, restarts on failure
- [x] Persistent virtual microphone across reboots
- [x] OAuth2 with automatic token refresh and scope validation
- [x] Gaming-aware resource isolation — every local-model code path (LLM tier, NL classification fallback, memory embeddings) individually disabled during a detected game session, not just the main conversational route

---

## Planned

### Near term

**Wake word fine-calibration**
Environment-specific noise and microphone profiling to tighten detection accuracy for the specific room and hardware in daily use.

**Home automation entity auto-discovery**
Automatic detection of new devices on the local automation hub instead of manual configuration per device.

### Mid term

**Finance agent budgeting integration**
Connect the existing bank-email parsing pipeline to a personal budgeting tool instead of only producing ad hoc summaries.

**Deeper MCP delegation**
Expose more granular tools over the existing MCP server so specialized external agents (research, coding) can be delegated sub-tasks without overloading the main conversational LLM.

### Longer term

**Stronger operator console**
Visual timeline of past sessions, task graph display, and memory browser in the HUD.

---

## Non-Goals for the Public Repository

- open-source release of the private implementation
- exposure of personal workflow automation or local machine details
- publication of prompt assets or behavioral tuning
