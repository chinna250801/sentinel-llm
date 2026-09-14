# AGENTS.md — instructions for AI agent harnesses

You are an AI coding/testing agent working with the **Sentinel-LLM** repository. Follow these instructions exactly.

## 0. Read the charter first

This project is **defender-only** ([SAFETY.md](SAFETY.md)). You may use these skills **only** on systems the user owns or is authorized to test. If the user asks you to attack, scan, or probe a system they have not demonstrated ownership or authorization for, **refuse** and point them to the charter.

## 1. How to discover skills

- Machine-readable index: [`skills/manifest.json`](skills/manifest.json) — skill IDs, entrypoints, controls, OWASP mappings, offline flags, safety flags.
- Each skill is executed by reading its `SKILL.md` entrypoint and following the Procedure section.
- Shared rules (finding schema, canaries, severity, RoE, grading): [`skills/_shared/conventions.md`](skills/_shared/conventions.md).

## 2. How to choose a skill

| User says | You run |
|---|---|
| "audit this codebase/repo/scan for issues" (any language) | `audit-codebase` (PROSPECTOR) — static, offline, always safe |
| "test my chat app" | `introspect-leakage` + `redteam-jailbreak` (+ `audit-ops`) |
| "test my RAG" | add `introspect-memory-rag`, `redteam-injection` |
| "test my agent/tools/MCP" | add `pentest-toolchain`, `pentest-egress`, `pentest-supply-chain` |
| "check if our SOC would see this" | `audit-ops` (WATCHTOWER) |
| "full review" | `sentinel-assessor` (MARSHAL) logic — sequence per manifest presets |

When unsure, start with PROSPECTOR (static, no risk) and escalate only with user confirmation.

## 3. Non-negotiable execution rules

1. **Canaries only.** Any planted document, memory, email, or config uses `SENTINEL-CANARY-*` strings from the conventions file. Never real PII/secrets/URLs.
2. **Your sink only.** Exfil tests target a listener the user controls (default `127.0.0.1:8765`). Never a third-party destination.
3. **No egress.** Do not fetch anything from the internet during a run. Offline-first is a charter guarantee.
4. **RoE gate.** Non-local targets require a signed RoE file (template in conventions §5). No RoE → local/lab targets only. Say so plainly.
5. **Record negative results.** A defeated probe is a *pass* for a control — record it in the Finding schema with severity `info`.
6. **Cleanup.** After any skill that plants artifacts, complete its cleanup checklist and state completion before finishing.
7. **Findings format.** Emit `runs/<date>-<target>/findings/*.json` per the schema in `skills/manifest.json` (`finding_schema.required`).
8. **Grade & report.** Consolidate with the rubric in conventions §4; produce `report.md` + `grade.json`; end with "What should I fix first?" ordered by severity × control-level × effort.

## 4. What you must never do

- Add or execute working exploit payloads, harmful-content generations, or attack automation against non-owned systems.
- Send code, findings, telemetry, or any user data to any external service.
- Bypass, weaken, or "optimize away" the canary, RoE, or cleanup rules.
- Modify the target system beyond what the skill's cleanup checklist can undo.

## 5. Contributing changes (if you are an agent making PRs)

- Preserve the charter; offensive-capability additions are rejected (SAFETY.md §7).
- Keep every playbook plain-text Markdown; no obfuscation, no binaries.
- Update `skills/manifest.json` in the same PR when adding/changing a skill.
