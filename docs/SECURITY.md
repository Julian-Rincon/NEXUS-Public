# Security and Privacy

## Why the Source Is Not Public

This repository is documentation-only by design.

The private implementation contains:
- local automation flows tied to a real daily workstation
- personal profile, behavioral context, and task history
- Google OAuth credentials and API configuration
- multiple LLM/STT/TTS provider API keys and account-specific model configuration
- Telegram bot token and allowed-user list
- voice model files and TTS tuning parameters
- operational prompts and memory assets

Publishing the source code would expose personal infrastructure, remove the author's competitive advantage in using the tool, and provide a direct blueprint for replication without the effort of building it.

## What Is in This Repository

This repository contains only:
- product and engineering documentation
- architecture description
- technical decision rationale
- roadmap

No code, no configuration, no credentials, no personal data.

## Publishing Guidelines for Demo Material

If screenshots or video demos are added to the `assets/` folder:

- remove usernames, email addresses, device names, and local file paths
- blur or crop any inbox content, calendar entries, or personal reminders
- do not include terminal output showing API keys, token paths, or model paths
- do not include voice samples that contain personal information

## Threat Model

The primary risk this repository guards against is **capability cloning** — someone reading the source and deploying a functionally equivalent system without the original engineering investment.

The secondary risk is **credential exposure** — inadvertently including environment files, OAuth tokens, or provider API keys that would give unauthorized access to personal services or accounts.

Both risks are addressed by publishing documentation only.

## Portfolio Review

For employers or collaborators who want to evaluate the implementation:

- a controlled live walkthrough is available on request
- the author can share selected code sections under NDA if appropriate
- screenshots and capability demonstrations can be provided with sensitive content redacted

Do not request or expect access to the full private repository for general portfolio evaluation.
