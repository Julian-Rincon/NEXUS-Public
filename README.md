<div align="center">

<img src="assets/banner.svg" alt="NEXUS — personal AI assistant" width="100%" />

<sub>Public showcase of a private project. Source code, credentials and personal data are intentionally excluded.<br/>Every screenshot and terminal capture below comes from the running system.</sub>

<br/><br/>

![Version](https://img.shields.io/badge/version-v2%20·%20Oct%202026-ff1616?style=for-the-badge&labelColor=050001)
![Platform](https://img.shields.io/badge/runs%20on-Phantom%20·%20Fedora%20KDE-ff1616?style=for-the-badge&labelColor=050001)
![Tests](https://img.shields.io/badge/tests-1%2C963%20NEXUS%20·%2013k%2B%20Phantom-ff1616?style=for-the-badge&labelColor=050001)
![Root](https://img.shields.io/badge/root%20access-none-ff1616?style=for-the-badge&labelColor=050001)

<p align="center"><a href="https://julian-rincon.github.io/assets/videos/nexus.mp4"><img src="https://julian-rincon.github.io/assets/videos/nexus.jpg" width="720" alt="▶ Watch the video (~1 min, subtitled)"></a><br><sub>▶ Watch the video (~1 min, subtitled)</sub></p>

</div>

---

## In one paragraph

NEXUS is my personal AI assistant. It plans my day, talks with a voice of its own, remembers what matters across every AI agent I use, and watches all my code repositories so a broken test gets fixed, reviewed and merged without me asking. It is not a standalone app: it is a personal layer (~22,000 lines of Python, 1,963 tests) on top of **[Phantom](https://github.com/Julian-Rincon/phantom)**, my multi-agent workbench that runs Claude Code, OpenCode and Hermes side by side. Phantom provides the agents, models, UI and voice. NEXUS provides context, memory and initiative.

<div align="center">
<img src="assets/v2/nexus-in-phantom.png" alt="NEXUS answering inside its own workspace in Phantom" width="900" />
<br/><sub>NEXUS answering in its own Phantom workspace: identity, a model chosen from measured data, and a live service check. The 🔊 icon reads any answer aloud in the agent's voice.</sub>
</div>

---

## What it does

| Capability | How it works |
|---|---|
| **Daily briefing** | Every morning a Phantom automation reads my calendar, inbox, tasks and services and sends three priorities to Telegram and the desktop. Two `systemd` timers guarantee delivery: if the run fails (for example, a free model runs out of quota), it is re-run on another agent. |
| **Talk to it** | A global shortcut (Meta+N) records, transcribes locally (Whisper on the GPU) and answers out loud. Pressing it again stops the answer, so two replies never overlap. Voice also works in the app on the phone and through Telegram voice notes. |
| **A voice per agent** | NEXUS speaks with its own voice, generated locally with Chatterbox. The other agents each have a distinct voice. English technical terms inside Spanish sentences are pronounced correctly. |
| **Shared memory** | One memory store (SQLite with full-text search) exposed over MCP, so Claude, OpenCode and Hermes all read and write the same facts. |
| **Project copilot** | Watches **every** repository I have, including new ones, automatically. When a commit breaks the tests, an agent fixes it in an isolated git worktree, Claude reviews the diff and re-runs the tests, and only then is it merged. |
| **Model routing by evidence** | NEXUS never hard-codes a provider. It asks Phantom's scorecard, which ranks agent/model pairs by their measured error rates on my own history. |
| **Always-on watchdog** | A dependency-free Python service on a free-tier cloud VM. It alerts me when a service goes down, and when my PC is off it sends the briefing and answers basic Telegram commands. |
| **Meeting listener** | When a class or meeting is on the calendar and a call app is playing, it transcribes locally (Whisper on the GPU, no API quota). It records only the call app, so music in another app is ignored, and the microphone separately, so each line says who spoke: *Them* or *Me*. It stops one minute after the call ends. |
| **Google Workspace, directly** | NEXUS reads and edits the calendar (find, create, move, delete), searches and reads Gmail and leaves **drafts only** (sending does not exist in the code), and searches and reads Drive. It uses its own OAuth token, so it keeps working even when the chat client's connectors are down. |
| **Phone access** | Phantom (chat, voice and agents) is reachable from my phone over a private Tailscale network with HTTPS. Nothing is exposed to the public internet. |

---

## Evidence

### The copilot fixing a real failing test

A repository with a broken function. One commit later, an agent fixed it in a separate worktree, Claude reviewed and approved it, the tests went green and the fix landed on `main`. That took 45 seconds, and no build artifacts were committed.

<div align="center"><img src="assets/v2/copilot.png" alt="Terminal: copilot fixes a failing test and merges after review" width="760" /></div>

### Memory shared across different agents

To prove the agents really share memory (and that one isn't just guessing), a random number is generated. One agent (Claude Code) stores it, and a different agent (Hermes, a different model from a different provider) retrieves it.

<div align="center"><img src="assets/v2/shared-memory.png" alt="Terminal: Claude stores a random code and Hermes retrieves it" width="760" /></div>

### Native to the desktop

NEXUS notifications show up as a first-class app in KDE Plasma, so they can be silenced or configured like any other app.

<div align="center">
<img src="assets/v2/notification.png" alt="Native KDE Plasma notification from NEXUS" width="420" />
</div>

### Meetings transcribed by speaker

A demo meeting: the call app and the microphone are captured separately, and the music playing in another app never reaches the transcript. *(Demo data.)*

<div align="center"><img src="assets/v2/meeting-listener.png" alt="Terminal: meeting transcript with lines labeled Ellos and Tú" width="760" /></div>

### Calendar, mail and Drive without connectors

Moving an event, finding an email and leaving a reply as a draft in the same thread, then finding a document. Every write to Gmail is a draft that I review and send myself. *(Demo data.)*

<div align="center"><img src="assets/v2/google-workspace.png" alt="Terminal: NEXUS moves a calendar event, drafts an email reply and finds a Drive file" width="760" /></div>

### Every deploy verifies itself

The deploy script installs, restarts and then checks twelve things on its own: services, Telegram, model routing, all four voices (including that NEXUS uses its own voice and not the fallback) and the test suite.

<div align="center"><img src="assets/v2/deploy-check.png" alt="Terminal: deploy script verification, 12 of 12 checks passing" width="640" /></div>

---

## Architecture

```mermaid
flowchart LR
    subgraph Interfaces
        A[Desktop app]
        C[Telegram]
        D[Phone · Tailscale]
        E[Meta+N voice]
    end
    subgraph Phantom["Phantom (multi-agent workbench)"]
        F[Claude Code · OpenCode · Hermes]
        G[Model scorecard]
        H[Automations · worktrees · permissions]
        I[Local voice: Whisper + Chatterbox]
    end
    subgraph NEXUS["NEXUS (personal layer)"]
        J[nexus-mcp tools]
        K[(Shared memory)]
        L[Project copilot]
        M[Briefing guarantee timers]
    end
    N[Free-tier cloud watchdog]
    A & B & C & D & E --> Phantom
    F -- MCP --> J
    J --> K
    L -- git hooks --> H
    M --> H
    N -. PC off .-> C
```

**Design principles**
- **Compose, don't rebuild.** Agents, UI, scheduling and voice already exist in Phantom; NEXUS only adds what is personal.
- **Nothing invasive.** No root, no kernel tweaks, no self-installed autostart. On the desktop everything is event-driven (git hooks, path watchers, timers) instead of polling; every piece is a user-level service that is removed by deleting it.
- **Read-only where it can be.** Calendar and mail are read-only, and from mail NEXUS reads headers only, never message bodies.
- **Human-gated where it matters.** The copilot never pushes and never force-merges. If I have unsaved changes in the same files, or a merge conflicts, it stops and tells me.
- **Tests never touch the real machine.** Every fake raises on any call it did not expect, so a test can't silently send a real message or open a real browser.

---

## Why v2 exists

The first NEXUS was a 47,000-line monolith that rebuilt everything itself: a router across five LLM providers, an always-listening wake word, a desktop HUD and a gaming mode that tuned the kernel with root helpers. It taught me a lot, but it was never reliable enough to depend on: free model endpoints kept expiring, the always-on microphone was fragile, and it fought the operating system instead of working with it.

v2 applies that lesson. The same goals now take a fraction of the code, run with no root access, and come with a test suite and self-verifying deploys. The v1 design is archived in [`docs/v1/`](docs/v1/).

| | v1 | v2 |
|---|---|---|
| Size | ~47,000 lines | ~22,000 lines + 1,963 tests |
| Model choice | Hand-written 5-provider router | Measured scorecard |
| Voice | Always-on wake word | Push-to-talk, voice per agent |
| System tuning | Root helpers, kernel settings | None (native desktop tools) |
| Reach | Desktop + one bot | Desktop, phone, Telegram, cloud watchdog |

---

## How it's measured

The code is private, so these commands run against the private repository's checkout, not this one. This section documents the method and the criteria so each figure can be checked on a walkthrough or re-run by anyone granted access. Measured on 2026-11-23 (Python 3.14; the project requires 3.12+).

| Metric | Value | Command (from the private repo root) | Measured |
|---|---|---|---|
| Tests | 1,963 passed, 0 failed, 1 skipped (in ~50 s) | `pip install -e ".[dev]" && python -m pytest -q` | 2026-11-23 |
| Python lines in `src/`, physical | 22,334 (86 files), shown as "~22,000" | `find src -name '*.py' \| xargs cat \| wc -l` | 2026-11-23 |
| Same, without blank lines | 18,929 | `find src -name '*.py' \| xargs cat \| grep -c -v '^\s*$'` | 2026-11-23 |
| Same, without blank or comment-only lines | 18,452 | `find src -name '*.py' \| xargs cat \| grep -c -v -E '^\s*(#.*)?$'` | 2026-11-23 |

**Criteria.**

- A test counts only if it **passes**: 1,963 is the `passed` figure `pytest` prints, not the number collected (1,964). The one skipped test is marked `slow`: it calls a real local model service and only runs when `FORGE_VISION_REAL=1` is set.
- Lines are counted over `src/` only, which holds the application code. Tests, scripts, deploy files, docs and the legacy data are excluded.
- The headline "~22,000" is the **physical** count: it includes blank lines and comments. Counting only lines that carry code gives about 18,500. No line counter other than `wc` and `grep` is used, so the numbers are reproducible on any Linux machine.
- Figures are tied to a date because both grow with every commit.

---

## What this project demonstrates

- **Systems design with restraint.** Replacing a large custom system with a small layer on top of solid tools, and being able to explain what the first version taught me.
- **LLM engineering grounded in measurement.** Routing by measured error rates, fallbacks for provider quotas, and verification instead of trusting what an agent reports.
- **Native Linux integration.** MCP, `systemd` user units and path watchers, KDE global shortcuts over D-Bus, desktop entries and Plasma notifications.
- **Operational discipline.** Real failures found while building were root-caused and turned into tests and guards: a free tier running out of quota, a model-id mismatch, overlapping voices, build artifacts in commits.

A live walkthrough of the private implementation is available on request.

<sub>Retired v1 interface: <a href="assets/v1_hud.png">screenshot</a> · Not licensed for reuse; see <a href="NOTICE.md">NOTICE</a>.</sub>

---

<!-- julian-rincon-footer -->
<p align="center"><sub>Built by <b>Julián Rincón</b> · <a href="https://julian-rincon.github.io/#p-nexus">Portfolio</a> · <a href="https://www.linkedin.com/in/julian-esteban-rinc%C3%B3n-rodriguez-1a05501b7/">LinkedIn</a> · <a href="https://github.com/Julian-Rincon">GitHub</a></sub></p>
