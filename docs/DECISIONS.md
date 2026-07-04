# Technical Decisions

## Local-First LLM

**Decision**: run a local model as the default LLM path; use cloud models only when the task requires it, the local model is unavailable, or the system is under a competing workload (see "Gaming-Aware Resource Isolation" below).

**Why**: privacy, offline capability, and no per-query cost for daily conversational use.

**Tradeoff**: local models are less capable for complex reasoning, and local inference performance depends entirely on the host's own hardware and driver stack. The tiered routing handles the capability gap by escalating to cloud backends selectively; short timeouts on the local tier handle the performance variability so a slow local backend degrades gracefully instead of blocking the response.

---

## Multi-Provider Tiering, Not Just Redundancy

**Decision**: five LLM backends instead of one or two — chosen for complementary strengths (offline capability, raw throughput, context length and multimodal support, cost ceiling) rather than as interchangeable redundant copies of the same capability.

**Why**: different request shapes benefit from different backends. A one-size-fits-all single cloud provider either overpays for simple requests or underperforms on demanding ones. A tiered system lets simple requests stay cheap and local while still having headroom for long-context or high-throughput needs without a rewrite.

**Tradeoff**: more moving parts to keep healthy — five sets of credentials, five failure modes, and a circuit breaker per tier instead of one. Provider-side changes (model deprecations, catalog changes) need to be caught and adapted to individually.

---

## Streaming TTS

**Decision**: do not wait for the full LLM response before starting synthesis. Chunk the token stream at sentence boundaries and synthesize each chunk while the next one generates.

**Why**: the latency difference between "first word" and "complete response" is several seconds on typical responses. Most users perceive the interaction as near-instant when the first sentence plays within a second.

**Tradeoff**: if the LLM changes direction mid-response, the first sentence may already be playing. In practice this is rare for well-prompted responses, and the barge-in mechanism handles corrections.

---

## Deterministic Routing Before LLM

**Decision**: classify user input before sending it to any model. A large set of request patterns map directly to code paths — opening an app, reading the calendar, taking a screenshot, switching a resource profile — and execute without LLM involvement.

**Why**: models hallucinate when asked to perform operations they cannot actually execute. Deterministic paths are faster, more reliable, and do not consume tokens.

**Tradeoff**: the routing layer must be maintained as new capabilities are added. Missing a pattern sends the request to the LLM instead, which may respond correctly but with higher latency.

---

## Tiered LLM with Circuit Breaker

**Decision**: multiple levels of LLM (local, then several cloud tiers) with automatic promotion when a lower level fails or is too slow. A circuit breaker pauses a failing backend after repeated errors and retries after a cool-down.

**Why**: cloud model availability, quota limits, and local hardware performance are all unpredictable in daily use. A tiered system with fallback and short local timeouts means the assistant keeps responding quickly even when individual backends are degraded.

**Tradeoff**: the routing heuristic that decides which tier to use is not perfect, and an aggressively short local timeout means an occasionally-slow-but-working local response gets preempted in favor of a cloud tier more often than a very patient timeout would allow. In practice, felt responsiveness matters more than maximizing local-tier usage.

---

## Confirmation Gating for Irreversible Actions

**Decision**: actions that modify external state or system state — sending email, creating calendar events, executing generated automation, changing machine power state — require explicit user confirmation before execution, with a risk level, an expiry window, and (on remote channels) a short confirmation token.

**Why**: natural-language matching against free text is inherently approximate. A phrase can superficially resemble a destructive command — a reminder to do something later, a question about a past action — without being an actual request to perform it right now. Gating execution behind confirmation means an imprecise match costs a follow-up question instead of an unintended, possibly irreversible, action.

**Tradeoff**: confirmation adds friction for frequently repeated operations. High-trust, low-risk actions (reading the calendar, locking the screen) bypass confirmation entirely so the friction is proportional to the actual risk.

---

## Gaming-Aware Resource Isolation

**Decision**: when a game session is detected, every code path that would otherwise call the local model — the main conversational tier, the natural-language classification fallback, and the semantic-memory embedding calls — is individually disabled for the duration of the session, not just deprioritized.

**Why**: a single "route the main conversation to cloud during gaming" rule is not sufficient if a secondary code path (like a pre-classification step or a memory-retrieval embedding call) still quietly calls the local model on every message. Any of those paths spinning up the local model competes with the game for CPU and, if GPU-resident, for VRAM. The isolation has to be applied at every call site that can reach the local model, not just the obvious one.

**Tradeoff**: semantic memory recall degrades to recent-message history only during a game session — conversation turns are still recorded, but not embedded for similarity search until the session ends. This is an acceptable tradeoff against the alternative of contending with the game for resources.

---

## Bilingual Phonetics on the Local Fallback Voice

**Decision**: on the local fallback TTS voice, substitute English technical words with Spanish phonetic equivalents before synthesis, rather than switching to a different engine or model for English segments.

**Why**: switching TTS engines mid-sentence produces audible discontinuities in voice character, pitch, and speaking rate. The local Spanish voice model also produces incorrect phoneme sequences for English words if left unmodified.

**Tradeoff**: the substitution table must be maintained manually. New English words need to be added when the assistant starts mispronouncing them. The primary cloud voice does not need this — it handles mixed-language text natively.

---

## Native Desktop HUD

**Decision**: the operator console is a native desktop application integrated with the system tray and window manager, rather than a browser-hosted interface.

**Why**: closer integration with the desktop session — tray presence, window management, and startup behavior all work the way a native application is expected to, without depending on a browser being available or a local web server staying up.

**Tradeoff**: visual iteration is slower than it would be with a web-based UI, since layout and animation require the native toolkit rather than CSS/JS.

---

## Virtual Microphone via System Audio Routing

**Decision**: route NEXUS voice output through a loopback audio device rather than injecting audio directly into the meeting's audio stream.

**Why**: the loopback/null-sink pattern is the correct abstraction for this use case at the OS audio layer. Meeting software sees the device as a standard microphone input. No browser or application patching is needed.

**Tradeoff**: the user must manually select the virtual microphone in the meeting software's audio settings. This is a one-time setup per meeting application.

---

## Voice-First Telegram

**Decision**: all Telegram bot replies are sent as synthesized voice notes before the text message.

**Why**: the assistant has a specific voice character. Receiving text-only responses on mobile makes it feel like a generic chatbot. Receiving a voice note first maintains the same identity across channels.

**Tradeoff**: voice notes add a small processing delay (synthesis + audio format conversion) before the reply is visible. For very short replies, this overhead is noticeable.

---

## Background Service for the Telegram Bot

**Decision**: run the Telegram bot as a background service with automatic restart on failure and auto-start on login, rather than as a subprocess of the desktop HUD.

**Why**: the bot should remain available even when the desktop HUD is not running — after a crash, or when operating headlessly. The OS service manager handles restart logic and session lifecycle cleanly, and the assistant behaves as the always-on system it is designed to be rather than an app the user has to remember to launch.

**Tradeoff**: the bot process and the HUD process are decoupled. Shared state (e.g., an active meeting, or gaming-mode status) has to be coordinated explicitly rather than living in one process.

---

## Detection Separated from Delivery (Heartbeat)

**Decision**: proactive monitors detect everything and report freely; a single judgment layer at the delivery boundary decides what actually interrupts the user — deduplication windows, quiet hours, suppression during gaming and focus sessions, and an hourly rate limit, with suppressed alerts accumulated into a digest.

**Why**: putting etiquette logic inside each monitor scatters the same policy across many components and makes it inconsistent the moment one monitor is updated and another is not. It also couples detection correctness to delivery politeness — the stress tests that validate detection should not depend on whether an alert would have been polite to send. One layer, one policy.

**Tradeoff**: urgency classification at the delivery boundary has to understand every alert type that flows through it, and a genuinely urgent alert misclassified as routine will be held during a suppression window. The digest guarantees it is delayed, never lost.

---

## Event-Driven Gaming Signal Over Heuristic Polling

**Decision**: the primary way NEXUS learns a game session started is a signal emitted by the game's own launch command — a universal wrapper configured once per library, which also resolves the game's literal name from the store manifest. The process/VRAM heuristic that previously drove detection was demoted to a fallback for games launched outside the wrapper.

**Why**: polling heuristics discover a game up to a minute late — after the game has already fought the local LLM for VRAM during its most sensitive phase, initial loading. The launcher knows the exact moment and the exact identity of the game; asking it is strictly better than inferring. It also eliminates per-game detection lists: any game launched through the wrapper is recognized by name with zero configuration.

**Tradeoff**: the wrapper must be attached to each launcher (a one-time setup step), and games launched by other paths fall back to the slower heuristic — which is why the heuristic was kept rather than deleted.

---

## Fail-Closed Remote Authorization

**Decision**: the remote channel's user allowlist rejects everyone when it is empty or unconfigured, instead of falling back to "anyone with the bot token may command the system."

**Why**: the remote channel can read email summaries, speak with the user's cloned voice, and request power-state changes. A configuration accident (a missing environment variable, a fresh deployment) must degrade to *nobody gets in*, not *everybody does*. Fail-open authorization is only discovered after someone walks through the open door.

**Tradeoff**: a misconfiguration locks the legitimate user out of the remote channel until fixed — an inconvenience that is loudly visible, which is exactly the failure mode to prefer.

---

## Generated Code Is Never Executed to Validate It

**Decision**: validation of LLM-generated code is purely static — parsing and AST inspection that flags dynamic execution primitives and shell access — and the validation step has no code path that runs the generated program.

**Why**: executing untrusted output in order to decide whether it is trustworthy inverts the trust order; the original import-based validation approach amounted to remote code execution triggered by a voice request. Static analysis answers "is this structurally safe to save?" without granting execution.

**Tradeoff**: static analysis cannot prove runtime behavior — it catches the dangerous constructs, not every dangerous outcome. The generated file is saved for the user to run deliberately, which keeps a human in the execution path.

---

## The TTS Cache Only Trusts the Primary Voice

**Decision**: the phrase cache is keyed by the full voice configuration (backend, voice, model), and only audio synthesized by the primary voice is ever persisted. Fallback audio is played but never cached. Short responses are cached opportunistically as they occur.

**Why**: a cache that stores whatever was played converts a temporary outage into a permanent one — phrases synthesized by the fallback voice during a quota lapse would keep playing in the wrong voice long after the primary recovered. Caching is a cost optimization; it must never be allowed to change what the system sounds like.

**Tradeoff**: during a primary-voice outage every response pays fallback synthesis cost with no cache benefit, and a configuration change invalidates the cache wholesale (a one-time re-synthesis cost). Both are prices worth paying for a cache that can never lie about the voice.

---

## Memory Promotion Instead of Memory Accumulation

**Decision**: after each session, an LLM pass extracts durable facts and *promotes* them into one of two stores with different privileges: a small core store injected into the system prompt every session, and an archival store retrieved on demand. Everything else from the session stays in the ordinary session/vector logs. Secrets are redacted before anything persists.

**Why**: injecting raw history into context does not scale and buries the few facts that actually matter (stable preferences, environment realities) under conversational noise. The valuable memory operation is deciding what deserves permanent, always-present status — that is a promotion decision, not a storage decision.

**Tradeoff**: an extraction pass can promote a misunderstanding, which then colors every future session until corrected — so the core store is kept small, deduplicated, and inspectable on demand ("what do you know about me").
