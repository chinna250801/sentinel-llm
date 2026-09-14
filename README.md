# Sentinel-LLM

> **Motto: assume the model is exploitable — build the system so it doesn't matter.**

**Sentinel-LLM** is an open-source, lightweight playbook for securing LLM applications and AI agents. It distills the 2024–2026 state of AI-cybersecurity research (OWASP GenAI, MITRE ATLAS, NIST AI RMF, UK AISI/Inspect, EU AI Act) into **20 concrete, testable controls** — no platform purchase required.

> 🛡️ **Defender-only. Locally run. Zero data leaves your machine.** Everything here is for finding and fixing weaknesses in *your own* systems — no offensive tooling, no telemetry, no phone-home, works air-gapped. This is binding: read the [Defender Charter](SAFETY.md).

## Why this exists

Models *are* exploitable:

- **Prompt injection** remains unsolved at the model level (OWASP LLM01; OpenAI has said it may never be fully fixed).
- Real-world **zero-click agent exploits** shipped in 2025: [EchoLeak (CVE-2025-32711) in M365 Copilot](https://arxiv.org/html/2509.10540v1), ShadowLeak in ChatGPT connectors.
- **MCP tool poisoning**, RAG poisoning, and agent **memory implanting** turn one-shot attacks into persistent ones.
- AI cyber capability is measurably accelerating (UK AISI: ~4.2-month doubling on software tasks).

So the defense must live in the *system around the model*: least privilege, gated actions, controlled egress, provenance, and continuous testing. That's what this repo gives you.

## What's inside

| File | Purpose |
|---|---|
| [`SAFETY.md`](SAFETY.md) | **The Defender Charter** — offline guarantee, privacy, acceptable use (binding) |
| [`docs/GUIDELINES.md`](docs/GUIDELINES.md) | **The 20 controls** (5 levels, threat → evidence → implement → test) — start here |
| [`docs/RESEARCH.md`](docs/RESEARCH.md) | Full research compendium: attack taxonomies, incidents, standards, defenses — every claim cited |
| [`skills/`](skills/README.md) | **The skills suite** — 10 named testing skills (red team / introspection / pentest / audit / code audit) that probe your LLM app, agent, website, or codebase and score it against the controls |
| [`docs/BLUEPRINT.md`](docs/BLUEPRINT.md) | **The final report** — how everything fits together, worked end-to-end example, safety architecture |
| [`docs/ROADMAP.md`](docs/ROADMAP.md) | Path from guidelines to a runnable open-source toolkit (v1.0) |

## Quick start (1 day, Level 0)

1. Remove all secrets from every prompt; enforce policy in code, not prompts (**C01**).
2. Give each tool its own scoped, short-lived credentials (**C02**).
3. Put human approval behind an explicit dangerous-action list with exact-effect previews (**C03**).
4. Proxy agent network calls through a domain allow-list (**C04**).
5. Prefer safetensors, pin + verify models, vet MCP servers like npm packages (**C05**).

Level 0 alone blunts the majority of realistic agent attacks. Levels 1–4 add prompt hardening, data/memory integrity, CaMeL-style agentic containment, and continuous assurance.

## Test your app with the skills suite

Run [**MARSHAL**](skills/sentinel-assessor/SKILL.md) — the orchestrator and front door. Its whole interface:

```bash
sentinel scan ./my-repo                          # static codebase audit — one command
sentinel audit http://localhost:8080 --preset agentic   # live app you own
sentinel retest runs/latest/                     # after fixes — findings close only with proof
```

It scopes your app, sequences the right specialist skills, and produces a graded report:

| Skill | Codename | What it does |
|---|---|---|
| `sentinel-assessor` | **MARSHAL** | Orchestrates the suite, consolidates findings, computes the A–F grade |
| `redteam-jailbreak` | **LOCKPICK** | Jailbreaks: roleplay, Crescendo, Bad Likert Judge, encoding, smuggling |
| `redteam-injection` | **TROJAN** | Direct + indirect prompt injection (EchoLeak-pattern chains) |
| `introspect-leakage` | **X-RAY** | System-prompt extraction, secret discovery, PII regurgitation |
| `introspect-memory-rag` | **ARCHIVIST** | RAG poisoning + agent memory implants (persistence testing) |
| `pentest-toolchain` | **DEPUTY** | Confused deputy: tool misuse, missing authorization, approval bypass |
| `pentest-egress` | **SMUGGLER** | Exfil chains via canary sink, egress control, denial of wallet |
| `pentest-supply-chain` | **CUSTOMS** | Model/MCP/plugin supply chain: pinning, signatures, tool poisoning |
| `audit-ops` | **WATCHTOWER** | Would your SOC have seen it? Telemetry, IR readiness, governance |
| `audit-codebase` | **PROSPECTOR** | **Static, offline audit of the codebase itself — any language** (Python, Node/TS, Go, Java, .NET, PHP, Ruby, Rust): secrets, agent-config poisoning, unsafe output sinks, slopsquatting, pickle weights, MCP code flaws. No live target needed |

Every probe uses canary strings (never real data), emits machine-readable findings JSON mapped to control IDs, and records negative results too. Safety rules: signed rules-of-engagement for non-local targets, harmless metaprompt-style probes only — see [`skills/_shared/conventions.md`](skills/_shared/conventions.md).

### Just have code, not a running app?

Point [**PROSPECTOR**](skills/audit-codebase/SKILL.md) at any repository — Python, Node/TS, Go, Java, PHP, Ruby, .NET, Rust. It is **static and read-only**: no probes are sent, nothing is executed, no RoE needed. It finds the same control violations in source — hardcoded secrets (including in `CLAUDE.md`/`.cursorrules`), LLM output flowing into shell/SQL/HTML sinks, unpinned or nonexistent (slopsquat) packages, pickle model weights, MCP tool-description poisoning — and emits a graded findings report plus a ready-to-wire CI recipe. Why it matters: ~45% of AI-generated code samples introduce OWASP Top 10 flaws (Veracode 2025), and agent config files are now a credential-leak channel (Radware).

## Design principles

1. **Contain the blast radius** — you can't fix the model; you can make exploitation worthless.
2. **Evidence over vibes** — every control cites research, a standard, or an incident.
3. **Lightweight** — ~20 controls, implementable with code you already have; testable with free/open tooling (promptfoo, garak, PyRIT, AgentDojo, Inspect).
4. **AISI-aligned** — we frame controls as runnable evaluations so they plug into [Inspect](https://inspect.aisi.org.uk/)-style harnesses.

## Status

`v0.1` — guidelines + research. Roadmap to a runnable checker/testkit: see [ROADMAP](docs/ROADMAP.md).

## License

MIT — see [LICENSE](LICENSE).
