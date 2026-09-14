# Sentinel-LLM Roadmap

## Vision
Guidelines (v0.x) → **runnable toolkit** (v1.0): a small, dependency-light open-source repo that lets any team *check, test, and enforce* the 20 controls — with AISI Inspect compatibility for evaluations.

## Milestones

### M1 — v0.1 Guidelines release (current)
- [x] Research compendium (`docs/RESEARCH.md`) with citations
- [x] 20-control lightweight guidelines (`docs/GUIDELINES.md`)
- [x] README + roadmap + MIT license
- [x] **Skills suite** (`skills/`) — 10 named testing disciplines with canary safety, finding schema, RoE template, grading
- [x] **PROSPECTOR** (`skills/audit-codebase/`) — static, language-agnostic codebase audit (per-language sink tables, Semgrep rule skeletons, slopsquatting/agent-config checks)
- [x] **Defender Charter** (`SAFETY.md`) — binding offline/local-only/zero-telemetry guarantees + governance
- [x] **MARSHAL orchestrator spec** — prompt-driven interface (audit / test / summarize / retest), presets, JSON outputs for CI
- [x] **Final blueprint** (`docs/BLUEPRINT.md`) — full system report with worked example and safety architecture
- [ ] Community feedback round ( threat-model review, control numbering/stability )
- [ ] CONTRIBUTING.md, SECURITY.md, Code of Conduct

### M2 — v0.2 Checklist & scoring
- [ ] `checklist.yaml` machine-readable form of the 20 controls (id, level, test, effort, refs)
- [ ] Coverage matrix generator (controls ↔ OWASP LLM10/ASI ↔ MITRE ATLAS)
- [ ] Self-assessment scorer (per-feature security grade A–F)

### M3 — v0.4 Reference implementations
- [ ] Level 0/1 snippets: spotlighting preprocessor, egress allow-list proxy, PII redaction hooks
- [ ] Dual-LLM (Action-Selector first) agent harness example in Python
- [ ] PDP/PEP authorization demo with policy matrix
- [ ] Memory-contract validator for agent memory stores

### M4 — v0.6 Testkit
- [ ] CI-ready red-team presets (promptfoo/garak) mapped to our 20 controls
- [ ] AgentDojo scenarios as regression tests for agent harnesses
- [ ] Inspect-compatible task wrappers so controls run as AISI-style evals
- [ ] Reporting: bypass-rate dashboard, trend over releases
- [ ] **Skills automation**: executable probe scripts + findings JSON emitters for the 10 skills in `skills/` (PROSPECTOR, LOCKPICK, TROJAN, X-RAY, ARCHIVIST, DEPUTY, SMUGGLER, CUSTOMS, WATCHTOWER, MARSHAL)
- [ ] **Optional probe runners**: executable helpers an agent can invoke (gitleaks/trufflehog, semgrep OWASP-LLM ruleset, osv-scanner, modelscan, slopsquat verification, agent-config unicode scan) → findings JSON + repo grade — automation support, not a new interface

### M5 — v1.0 Open-source launch
- [ ] Docs site + quickstart under 30 minutes
- [ ] **Release test: full suite runs with networking disabled (firewall block-all) — charter verified, not just promised**
- [ ] Case-study: apply full checklist to one reference agent, publish results
- [ ] Governance: maintainers, release cadence, security-response process (charter §7)
- [ ] Announce (HN, r/netsec, OWASP GenAI project channels)

## Non-goals
- Building a commercial guardrails platform
- Model training or fine-tuning safety research (we link out instead)
- Promising "prompt injection solved" — we contain, not cure
