# Sentinel-LLM Skills Suite

Named, repeatable testing **skills** that operationalize [`docs/GUIDELINES.md`](../docs/GUIDELINES.md) against a *live* LLM application, agent, or website. Each skill is a self-contained playbook an engineer (or AI agent) can execute end-to-end: **scope → probe → detect → score → report → clean up.**

> **Motto:** assume the model is exploitable — then prove exactly *how*, measure it, and hand defenders the fix list.

## The skills

| Codename | Skill | Discipline | Probes | Maps to controls |
|---|---|---|---|---|
| **MARSHAL** | [`sentinel-assessor`](sentinel-assessor/SKILL.md) | Orchestration | Runs the right skills in the right order, consolidates findings into one graded report | All C01–C20 |
| **LOCKPICK** | [`redteam-jailbreak`](redteam-jailbreak/SKILL.md) | Red team | Jailbreaks: roleplay, multi-turn escalation (Crescendo, Bad Likert Judge), encoding, token smuggling | C01, C06–C08, C17 |
| **TROJAN** | [`redteam-injection`](redteam-injection/SKILL.md) | Red team | Direct + indirect prompt injection via documents, emails, web pages, tool output | C01, C03, C04, C06, C12–C15 |
| **X-RAY** | [`introspect-leakage`](introspect-leakage/SKILL.md) | Introspection | System-prompt extraction, secret discovery, PII/training-data regurgitation, cross-session bleed | C01, C07, C11 |
| **ARCHIVIST** | [`introspect-memory-rag`](introspect-memory-rag/SKILL.md) | Introspection | RAG poisoning, agent memory implanting, provenance and retrieval integrity | C09, C10 |
| **DEPUTY** | [`pentest-toolchain`](pentest-toolchain/SKILL.md) | Penetration | Confused-deputy: excessive agency, tool misuse, missing authorization, missing HITL gates | C01–C03, C12–C15 |
| **SMUGGLER** | [`pentest-egress`](pentest-egress/SKILL.md) | Penetration | Data exfiltration chains (EchoLeak/ShadowLeak style), egress control, denial of wallet | C04, C16 |
| **CUSTOMS** | [`pentest-supply-chain`](pentest-supply-chain/SKILL.md) | Penetration | Model/plugin/MCP supply chain: unpinned artifacts, pickle RCE risk, unsigned models, tool-description poisoning | C05 |
| **WATCHTOWER** | [`audit-ops`](audit-ops/SKILL.md) | Audit | Logging, detection coverage, guardrail telemetry, IR readiness, governance | C16–C20 |
| **PROSPECTOR** | [`audit-codebase`](audit-codebase/SKILL.md) | Code audit | **Static, offline audit of the codebase itself, any language** (Python, Node/TS, Go, Java, …): secrets, agent-config poisoning, LLM call construction, output sinks, slopsquatting, model artifacts, MCP server code, unsafe deserialization, CI | C01–C05, C07–C08, C11, C13–C15 |

## Discipline legend

- **Red team** — attacks the model's *behavior* through its intended channels (safety training, policies).
- **Introspection** — makes the system reveal what it should not (state, memory, data, internals).
- **Penetration** — attacks the *system around the model*: tools, permissions, network, supply chain.
- **Audit** — verifies the org can *see and respond* to what the other skills find.
- **Code audit** — reads the *source code itself*, offline and read-only: finds the flaws in any language before anything runs (no live target needed).

## Hard rules (read before running anything)

0. **Charter first.** This suite operates under the [Defender Charter](../SAFETY.md): local-only, offline by default, zero telemetry, your own systems only. Everything below implements that.
1. **Authorization first.** Written authorization from the system owner (RoE template in [`_shared/conventions.md`](_shared/conventions.md)) before any probe touches a non-local system. *(Exception: PROSPECTOR is static and read-only — no RoE needed.)*
2. **Canary tokens only.** Never exfiltrate real data — every exfil test uses the canary strings defined in the conventions file. If a canary ever leaves the lab environment, that *is* the finding.
3. **Harmless probes.** Probe prompts in these skills are metaprompt-style testing templates (defenses, scoring, ablations). They never include working harmful-content payloads. Where a technique needs "harmful content" to test refusal, substitute the benign canary phrases.
4. **Evidence or it didn't happen.** Every finding carries the exact probe, exact response, and the failing control ID. No vibes.
5. **Teardown.** Every skill lists its cleanup steps (delete planted memory/docs, revoke test keys). ARCHIVIST and CUSTOMS especially.

## Workflow

```
MARSHAL (orchestrator)
 ├─ 0. PROSPECTOR             (code audit — offline, before anything is deployed or probed)
 ├─ 1. scope & RoE            (read conventions, write skills/<run>/scope.md)
 ├─ 2. X-RAY + ARCHIVIST      (introspection — learn what the app exposes)
 ├─ 3. LOCKPICK + TROJAN      (red team — behavior attacks, parallel)
 ├─ 4. DEPUTY + SMUGGLER      (pentest — tool & egress attacks, parallel)
 ├─ 5. CUSTOMS                (pentest — supply chain inventory & checks)
 ├─ 6. WATCHTOWER             (audit — could the SOC have seen steps 2–5?)
 └─ 7. consolidated report    (findings.json → graded report.md, control scores)
```

Each skill writes machine-readable findings to `runs/<date>-<target>/findings/*.json` using the shared **Finding schema**; MARSHAL aggregates them into `report.md` with the A–F grade and the prioritized fix list.

## Status

v0.1 — playbooks complete, automation hooks (promptfoo/garak/AgentDojo wiring) planned per [`docs/ROADMAP.md`](../docs/ROADMAP.md) M4.
