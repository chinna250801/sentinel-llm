# MARSHAL — `sentinel-assessor`

**Discipline:** Orchestrator · **Controls:** all C01–C20 · **Offline:** fully (unless you point it at your own target URL) · **Requires:** nothing to start

## Purpose
The coordinator skill — and the **front door**. MARSHAL scopes the target, picks and sequences the specialist skills, consolidates their findings into one graded report, and drives the retest loop. Everything downstream is a skill; everything upstream is one prompt to your agent.

> **Charter note (SAFETY.md):** MARSHAL runs **locally and offline by default**. It opens network connections only to a target *you explicitly name* (and only the live-skill presets), and never sends your code, findings, or telemetry anywhere. Findings stay on your disk.

## The user experience (this is the whole interface)

There is **no CLI** — the interface is a prompt to any agent harness. The agent loads [`skills/manifest.json`](../manifest.json), reads the SKILL.md of each chosen skill as its procedure, and executes under [`AGENTS.md`](../../AGENTS.md) rules.

```text
# 1. Audit a repo (any language), grade it, get a fix list — fully offline.
"Load PROSPECTOR from skills/manifest.json and audit this repo.
 Grade it. Show my fix list."

# 2. Test a running app you own — preset chosen for you.
"Run the agentic preset against my service on localhost:8080.
 Canaries only, sink on 127.0.0.1:8765."

# 3. Re-render or retest — one more prompt.
"Summarize the report from my last run."
"Retest the open findings — close only what passes now."
```

That's the product. No config file, no install; sensible defaults everywhere; every run ends with a human-readable **"What should I fix first?"** summary.

## Interface contract (automation-friendly)

| Prompt says | Agent does | Output |
|---|---|---|
| "audit this repo" (static) | PROSPECTOR only | `runs/<id>/report.md` + `findings/*.json` + `grade.json` |
| "test my app at <target>" | preset skills, live | same, plus live-skill evidence dirs (target must be yours — RoE gate if non-local) |
| "summarize / re-render the report" | re-render | idempotent report |
| "retest open findings" | re-probe failures only | delta report; findings close only with proof |

All outputs follow the shared Finding schema (`_shared/conventions.md`); `report.md` is always accompanied by `grade.json` and a "top 3 fixes" block. Agents can emit `--json`-style stable output files for CI when asked.

## MARSHAL's decision procedure

1. **Classify** (auto-detected, override by naming a preset in your prompt):
   - Repo only → `static` preset (PROSPECTOR full).
   - + live chat app → add X-RAY, LOCKPICK.
   - + RAG (retrieval endpoints detected) → add ARCHIVIST.
   - + tools/MCP → add TROJAN, DEPUTY, SMUGGLER.
   - + MCP servers/plugins → CUSTOMS.
   - + SIEM/log access declared → WATCHTOWER.
2. **Phase 0 — static first:** PROSPECTOR runs before anything else; its findings *aim* the live phases (secrets found in prompt templates → prioritize X-RAY XL-2; egress-capable tool found → prioritize SMUGGLER SE-3).
3. **RoE gate:** if target is non-local, MARSHAL asks for the signed RoE file and refuses to proceed without it (per conventions §5). Local targets (localhost/127.0.0.1/ declared staging) skip this.
4. **Canary sink check:** live presets require a local sink address; MARSHAL starts one for you (default `127.0.0.1:8765`) and records it in scope.
5. **Sequence** the chosen skills (introspection → red team → pentest → supply → audit), collecting findings.
6. **Consolidate & grade** per conventions §4: dedupe by control+probe family, compute grade, per-control scores, and the **fix backlog** ordered by (severity × Level-0-first × effort).
7. **Retest loop:** on a "retest" prompt, re-run only the failed probes; a finding closes only when its probe yields a negative result (defense verified working).

## Aborting (always safe to stop)
Any canary outside the declared sink, any real (non-canary) PII surfaced, any 5xx storm, or signs of shared production data → run halts, evidence preserved, cleanup checklist printed.

## Report (the "super cool thing" the user sees)
- **Grade A–F** with the exact rubric trace (which findings capped it).
- **Top 3 fixes** in plain language with control IDs and file:line where applicable.
- Findings table (severity → control → effort), KPI section (bypass rate, negative results = defenses proven working), and the per-language/tool evidence links.
- Always ends with: *"Prompt your agent to retest after fixes — findings close only with proof."*

## Automation hooks (CI/agent harnesses)
- Findings and grade are plain JSON files — ingest them into CI/pipelines as-is.
- Run dirs are self-contained; the report format is deterministic.
- Everything is a readable playbook: an AI agent harness executes any skill by following its SKILL.md — that is the primary interface, not a fallback.

## Cleanup
MARSHAL plants nothing itself; it verifies every specialist skill's cleanup checklist is complete before closing a run.
