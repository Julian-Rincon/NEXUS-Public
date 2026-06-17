# Technical Decisions

## Local-First LLM

**Decision**: run a local model as the default LLM path; use cloud models only when the task requires it or the local model is unavailable.

**Why**: privacy, offline capability, and no per-query cost for daily conversational use.

**Tradeoff**: local models are less capable for complex reasoning. The tiered routing handles this by escalating to cloud backends selectively.

---

## Streaming TTS

**Decision**: do not wait for the full LLM response before starting synthesis. Chunk the token stream at sentence boundaries and synthesize each chunk in a background thread while the next one generates.

**Why**: the latency difference between "first word" and "complete response" is 3–5 seconds on typical responses. Most users perceive the interaction as near-instant when the first sentence plays within 0.5 s.

**Tradeoff**: if the LLM changes direction mid-response, the first sentence may already be playing. In practice this is rare for well-prompted responses and the barge-in mechanism handles corrections.

---

## Deterministic Routing Before LLM

**Decision**: classify user input before sending it to any model. A large set of request patterns map directly to code paths — opening an app, reading the calendar, taking a screenshot — and execute without LLM involvement.

**Why**: models hallucinate when asked to perform operations they cannot actually execute. Deterministic paths are faster, more reliable, and do not consume tokens.

**Tradeoff**: the routing layer must be maintained as new capabilities are added. Missing a pattern sends the request to the LLM instead, which may respond correctly but with higher latency.

---

## Tiered LLM with Circuit Breaker

**Decision**: three levels of LLM (local, cloud-fast, cloud-fallback) with automatic promotion when a lower level fails. A circuit breaker pauses a failing backend after repeated errors and retries after a cool-down.

**Why**: cloud model availability and quota limits are unpredictable in daily use. A tiered system with fallback means the assistant continues functioning even when individual backends are unavailable.

**Tradeoff**: the routing heuristic that decides which tier to use is not perfect. Some tasks go to the local model that would benefit from cloud reasoning, and vice versa.

---

## Bilingual Phonetics — Same Voice Model

**Decision**: substitute English technical words with Spanish phonetic equivalents before synthesis, rather than switching to a different TTS engine or model for English segments.

**Why**: switching TTS engines mid-sentence produces audible discontinuities in voice character, pitch, and speaking rate. Spanish TTS models also produce incorrect phoneme sequences for English words.

**Tradeoff**: the substitution table must be maintained manually. New English words need to be added when the assistant starts mispronouncing them.

---

## Web HUD Over Desktop Framework

**Decision**: the Iron Man operator console is a single HTML/CSS/JS file served by a local web server rather than a native GUI framework (tkinter, Qt).

**Why**: the web rendering engine handles animations, SVG graphics, and layout with far less code than native alternatives. Iterating on the visual design is significantly faster. The result runs in the browser as a PWA window and requires no GUI framework dependency.

**Tradeoff**: the HUD requires a browser to be installed. This is not a real constraint on the target platform but would matter in embedded or headless scenarios.

---

## Virtual Microphone via PipeWire Null-Sink

**Decision**: route JARVIS voice output through a PipeWire null-sink loopback device rather than injecting audio directly into the meeting's audio stream.

**Why**: null-sink loopback is the correct PipeWire abstraction for this use case. Meeting software (Meet, Zoom, Teams) sees the device as a standard microphone input. No browser or application patching is needed.

**Tradeoff**: the user must manually select "JARVIS Virtual Mic" in the meeting software's audio settings. This is a one-time setup per meeting application.

---

## Voice-First Telegram

**Decision**: all Telegram bot replies are sent as OGG voice notes (synthesized locally) before the text message.

**Why**: the assistant has a specific voice character. Receiving text-only responses on mobile makes it feel like a chatbot. Receiving a voice note first maintains the JARVIS identity across channels.

**Tradeoff**: voice notes add a small processing delay (TTS synthesis + ffmpeg OGG conversion) before the reply is visible. For very short replies, this overhead is noticeable.

---

## Confirmation for Sensitive Operations

**Decision**: actions that modify external state — sending email, creating calendar events, executing automation — require explicit user confirmation before execution.

**Why**: avoiding accidental sends and irreversible side effects. The assistant handles real communication channels and the cost of an unintended action is high.

**Tradeoff**: confirmation adds friction for frequently repeated operations. High-trust, low-risk actions (like reading the calendar) bypass confirmation entirely.

---

## systemd User Service for Telegram Bot

**Decision**: run the Telegram bot as a systemd user service with automatic restart on failure, rather than as a subprocess of the desktop HUD.

**Why**: the bot should remain available even when the desktop HUD is not running (e.g., after a crash, or when operating headlessly). systemd handles restart logic, log rotation, and session lifecycle cleanly.

**Tradeoff**: the bot process and the HUD process are now decoupled. Shared state (e.g., active meeting) must be coordinated explicitly rather than being in the same process.
