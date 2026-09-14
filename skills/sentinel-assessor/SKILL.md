# MARSHAL — `sentinel-assessor`

**Discipline:** Orchestrator · **Controls:** all C01–C20 · **Offline:** fully (unless you point it at your own target URL) · **Requires:** nothing to start

## Purpose
The coordinator skill — and the **front door**. MARSHAL scopes the target, picks and sequences the specialist skills, consolidates their findings into one graded report, and drives the retest loop. Everything downstream is a skill; everything upstream is one command.

> **Charter note (SAFETY.md):** MARSHAL runs **locally and offline by default**. It opens network connections only to a target *you explicitly name* (and only the live-skill presets), and never sends your code, findings, or telemetry anywhere. Findings stay on your disk.

## The user experience (this is the whole interface)

```bash
# 1. The one-liner: audit a repo (any language), grade it, get a fix list.
sentinel scan ./my-repo                 # offline. PROSPECTOR + all static checks.

# 2. One command for a running app you own.
sentinel audit http://localhost:8080 --preset agentic   # live skills, chosen for you

# 3. Everything, decided for you:
sentinel audit ./my-repo http://localhost:8080 --preset full
sentinel report runs/latest/            # re-render the graded report, any time

# 4. Re-run after fixes — only against what was failing:
sentinel retest runs/latest/
```

That's the product. No config file required to start; sensible defaults everywhere; `--help` explains everything; every command ends with a human-readable **"What should I fix first?"** summary.

## Interface contract (automation-friendly)

| Command | Input | Output | Notes |
|---|---|---|nice|
| `sentinel scan <repo>` | path | `runs/<id>/report.md` + `findings/*.json` + `grade.json` | offline-only; static |
| `sentinel audit <target>` | URL/path | same, plus live-skill evidence dirs | target must be yours (RoE prompt if non-local) |
| `sentinel report <run>` | run dir | rendered report | idempotent re-render |
| `sentinel retest <run>` | run dir | delta report | only re-probes failing findings |

All outputs follow the shared Finding schema (`_shared/conventions.md`); `report.md` is always accompanied by `grade.json` and a "top 3 fixes" block. `sentinel --json` gives stable machine output for CI.

## MARSHAL's decision procedure

1. **Classify** (auto-detected, override with `--preset`):
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
7. **Retest loop:** `sentinel retest` re-runs only the failed probes; a finding closes only when its probe yields a negative result (defense verified working).

## Aborting (always safe to stop)
Any canary outside the declared sink, any real (non-canary) PII surfaced, any 5xx storm, or signs of shared production data → run halts, evidence preserved, cleanup checklist printed.

## Report (the "super cool thing" the user sees)
- **Grade A–F** with the exact rubric trace (which findings capped it).
- **Top 3 fixes** in plain language with control IDs and file:line where applicable.
- Findings table (severity → control → effort), KPI section (bypass rate, negative results = defenses proven working), and the per-language/tool evidence links.
- Always ends with: *"Re-run `sentinel retest` after fixes — findings close only with proof."*

## Automation hooks (CI/agent harnesses)
- Exit codes: `0` clean, `1` findings (grade B+), `2` findings (grade ≤ C), `3` abort, `4` RoE missing — CI-merge-gate ready.
- `--json` for stable machine output; run dirs are self-contained; the report renderer is deterministic.
- Everything is a readable playbook: an AI agent harness can execute any skill by following its SKILL.md — the orchestrator is a convenience, not a requirement.

## Cleanup
MARSHAL plants nothing itself; it verifies every specialist skill's cleanup checklist is complete before closing a run.
