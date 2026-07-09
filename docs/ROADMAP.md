# Roadmap

## Completed

### Core voice pipeline
- [x] Streaming LLM → TTS — first spoken word in under a second
- [x] Barge-in with VAD — interrupts playback in ~40 ms
- [x] Custom-trained wake word — purpose-built detection model, no generic keyword spotting
- [x] Wake word false-trigger hardening — sustained-score requirement across consecutive frames + local STT double-verification of the trigger audio before waking
- [x] TTS phrase cache — zero synthesis latency for common phrases
- [x] TTS cache credit governor — cache keyed by full voice configuration and only ever persists primary-voice audio, so a temporary fallback never gets entrenched; opportunistic caching of short responses reduces paid synthesis volume
- [x] Cloud voice cloning (primary) with automatic local fallback
- [x] Hybrid STT — cloud primary with local fallback
- [x] Bilingual phonetics on the local fallback voice — English words pronounced correctly in Spanish speech

### Intelligence layer
- [x] TieredBrain — five-level LLM with circuit breaker (local, fast cloud, high-throughput cloud, large-context/multimodal cloud, fallback cloud)
- [x] Live per-backend telemetry — calls, failures, latency EMA, breaker state — in the HUD and by voice
- [x] Connection sentinel — health checks across every external dependency, four-state model, alerts on transitions only
- [x] Deterministic intent routing before LLM involvement
- [x] Natural-language classification fallback (local pattern match → fast cloud classification → local model as last resort)
- [x] Web research routing — informational questions answered from live multi-engine search with synthesis
- [x] Skill retrieval — top-K relevant skills surfaced to the LLM per query (semantic + lexical) instead of the full registry
- [x] Session memory — recent sessions injected as context
- [x] Vector memory — semantic recall
- [x] Episodic memory — causal event log, queryable by day/week
- [x] Managed long-term memory — durable facts extracted post-session, promoted to a core profile (system prompt) or searchable archive, with dedup
- [x] Aggressive local-tier timeouts with circuit breaking, so a slow or cold local model never blocks a response for long

### Operator interface
- [x] Native desktop HUD — live system metrics, system tray integration
- [x] Adaptive HUD performance — background effects scale with focus state, per-source polling intervals (~75% idle CPU reduction)
- [x] On-screen speaking indicator — animated presence whenever NEXUS talks, from any process, visible above fullscreen windows
- [x] TV presence — spoken announcements, media playback, and volume on the living-room cast device, with delivery confirmation and multi-interface retry
- [x] Telegram bot — voice-first (voice notes + text), NL routing
- [x] Wake word active simultaneously with the Telegram bot

### Agentic capabilities
- [x] Autonomous desktop agent — vision + input, up to 25 steps, step verification, hard-allowlisted for safety
- [x] Meeting agent — browser join + virtual mic + live transcription + summary with action items and drafted follow-up
- [x] Meeting representative mode — attends as the user with cloned voice, animated virtual-camera identity, and conversational replies when addressed
- [x] Meeting-email intelligence — inbox classifier trained on real mail, invitation extraction, calendar events with context, dedup against existing events, scheduled auto-attendance
- [x] Code triage — error/deploy diagnosis, severity classification, fix briefing; never executes without permission
- [x] Proactive monitor — calendar alerts, email urgency, system resource warnings, connection health, security anomalies, morning briefing
- [x] Proactivity judgment layer (Heartbeat) — dedup, quiet hours, gaming/focus suppression, hourly rate limit, suppressed alerts delivered as a digest

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
- [x] Fail-closed remote authorization — an empty allowlist on the Telegram channel rejects everyone
- [x] Static-only validation of generated code — LLM output is analyzed, never executed, during validation
- [x] Confirmation-gated active defense — process kill and network isolation behind explicit confirmation, with a protected-process whitelist; network restore is immediate
- [x] Secret redaction before persistence — keys, tokens, and credential-shaped strings scrubbed from everything written to memory
- [x] Cloned-voice usage audit — every use of the cloned voice logged with redacted content

### Multi-tool exposure
- [x] MCP server exposing core capabilities as tools for external AI clients
- [x] MCP client consuming external tool servers from configuration, with no per-integration wrappers

### Infrastructure
- [x] Background service for the Telegram bot — auto-starts on login, restarts on failure
- [x] Process-singleton remote bot — exclusive lock prevents two processes from polling the messaging API simultaneously
- [x] Persistent virtual microphone across reboots
- [x] OAuth2 with automatic token refresh and scope validation
- [x] Gaming-aware resource isolation — every local-model code path (LLM tier, NL classification fallback, memory embeddings) individually disabled during a detected game session, not just the main conversational route
- [x] Gaming mode v2 — event-driven entry from a universal launch wrapper (second-zero detection, literal game name from the store manifest), per-game profiles (VRAM budget, GPU power limit, persistent shader cache), service CPU/IO weight reduction, heuristic fallback, session summary on exit
- [x] VRAM crash resilience — headroom guard with proactive model eviction, capped local-model keep-alive and context residency (from a real exhaustion post-mortem)
- [x] Persistent cloud node for network-dependent subsystems — the messaging bot, email/calendar watcher, and LLM routing now run on a dedicated always-on host on a free-tier cloud provider, decoupled from the desktop machine being powered on; desktop-only capabilities (screen, audio, local model) stay on the desktop
- [x] Automatic cost-safety cutoff — a monitoring guardrail on the cloud host tracks usage against the platform's free-tier limits and automatically suspends the service before any billable threshold is crossed, not just an alert; the cutoff itself retries until it actually succeeds and re-arms every billing cycle, rather than firing once and going silent
- [x] Minimal cross-host state bridge — the one feature that needs a desktop-only capability (screen/audio) after being triggered remotely pulls its confirmation state from the cloud host over an existing, already-authenticated channel: read-only, desktop-initiated only, and adds no new inbound surface on the cloud host
- [x] Live external security validation of the new cloud host — an independent network sweep and unauthorized-access attempt against the live host, not just a review of the firewall configuration
- [x] Full skill-dispatch audit — systematic review of the entire skill registry's phrase-matching logic across every skill; found and fixed several cases where an overly broad skill silently intercepted a command meant for a different, more specific one, returning a plausible-looking but wrong response instead of a visible error
- [x] Over 1250 automated tests kept green across every change

---

## Planned

### Near term

**Home automation entity auto-discovery**
Automatic detection of new devices on the local automation hub instead of manual configuration per device.

**Global dictation**
A system-wide hotkey that captures speech, transcribes it, and types the result into whatever application has focus.

### Mid term

**Finance agent budgeting integration**
Connect the existing bank-email parsing pipeline to a personal budgeting tool instead of only producing ad hoc summaries.

**Deep research agent**
Long-running multi-step investigations that search, read, and synthesize across sources — beyond the current single-pass web research routing.

**Deeper MCP delegation**
Expose more granular tools over the existing MCP server so specialized external agents (research, coding) can be delegated sub-tasks without overloading the main conversational LLM.

### Longer term

**Stronger operator console**
Visual timeline of past sessions, task graph display, and memory browser in the HUD.

**Personal CRM**
A relationship layer over the existing memory stores — who was discussed, when, and in what context.

---

## Non-Goals for the Public Repository

- open-source release of the private implementation
- exposure of personal workflow automation or local machine details
- publication of prompt assets or behavioral tuning
