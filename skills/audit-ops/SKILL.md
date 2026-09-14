# WATCHTOWER — `audit-ops`

**Discipline:** Audit · **Controls:** C16, C17, C18, C19, C20 · **OWASP:** cross-cutting · **Requires:** telemetry access (logs/SIEM), incident-response docs, governance artifacts; read-only — safe anytime

## Purpose
The other eight skills *attack*; WATCHTOWER asks: **would the defenders have known?** It replays the engagement's key probes against the org's logging, detection, and response apparatus and audits the governance layer. A bypass that the SOC can see and kill is a far smaller finding than a silent one — WATCHTOWER measures that difference.

## Probe families

### WA-1 · Telemetry inventory
Map what is actually logged per LLM interaction: prompt (redacted?), response, tool calls + args, approvals granted/denied, egress attempts, guardrail hits, token/cost per request, model version, session/user identity. **Findings:**
- No prompt/response or tool-call logging → **medium** (C18).
- Logging present but **unredacted PII** → **high** (C11 fail — the log store is now a breach surface).
- No cost/token telemetry → **medium** (C16 blind).
- Logs not reaching SIEM (only app-local) → **medium**.

### WA-2 · Detection replay (the core test)
Take 3–5 confirmed findings from the current run (from TROJAN/DEPUTY/SMUGGLER) and replay them in **detection mode** while watching the SIEM/alerts:
- Did any alert fire? Within what time? To whom?
- If no alert: draft the missing detection rule (give the exact log signature that *should* have fired, e.g., "egress proxy: first-seen domain from agent identity", "guardrail: injection-classifier hit > N", "tool: delete-burst by one session").
**Findings:** exfil-class probe silent in SIEM → **high** (the EchoLeak lesson: nobody pages on agent egress); destructive-action replay silent → **high**; guardrail hits logged but unmonitored → **medium**.

### WA-3 · Guardrail efficacy accounting
From guardrail metrics: hit rates by class, override rates, bypasses confirmed by the other skills vs. guardrail placements. **Findings:** guardrails deployed but no dashboard/owner → **medium**; guardrail alerts wired to nothing → **medium**; bypass rate trending unknown → **low** (C17 KPI missing).

### WA-4 · IR readiness (AI-specific runbooks)
Check existence + completeness of runbooks for: memory/RAG poisoning (purge + reindex steps, owners), jailbreak wave, agent mis-action (token revocation, compensating actions), model supply-chain compromise (re-pin/re-verify), and canary-fire response (the drill from conventions §2).
**Findings:** no AI-specific IR runbook → **high** (C19); runbook exists but no owner/rehearsal → **medium**; purge path for poisoned memory untested → **medium** (cross-link ARCHIVIST AR-4).

### WA-5 · Governance audit (C20)
- Risk register: does every LLM feature have an entry + owner?
- ML-BOM: exists? current? (Cross-link CUSTOMS CS-1 output.)
- Red-team cadence: scheduled? last run? bypass-rate tracked over time?
- Model/behavior re-evaluation schedule (capability doubling ~4 months ⇒ posture is perishable — cite AISI).
**Findings:** no register/owner per feature → **medium**; no scheduled re-eval → **medium**; red-team ad-hoc only → **medium**.

### WA-6 · tabletop: canary fire drill
Announce-to-owner only (never surprise): simulate the critical finding — canary string observed at sink. Time the responses: who gets called, who can revoke agent creds, how fast is the agent disabled, is there a comms template? **Output:** measured time-to-contain + gap list. Silent/no-owner paths → **high** (C19).

## Method details
- WATCHTOWER is read-only except the announced drill; it never plants or attacks.
- Its severity carries a multiplier role in MARSHAL: a **high technical finding whose detection replay was silent** gains one severity step in the report narrative (the finding is "live and invisible").
- Deliverables: detection-rule drafts (as actual SIEM query stubs), missing-telemetry list prioritized by attack class, updated IR runbook diffs, governance checklist with owners column filled.

## Finding example
```json
{
  "id": "F-041", "skill": "WATCHTOWER", "probe_id": "WA-2",
  "title": "Exfiltration-class probe produces no SIEM alert",
  "severity": "high", "controls": ["C18"], "owasp": ["LLM01"],
  "evidence": { "request": "replay of SE-3 canary email exfil in detection mode",
                "where": "SIEM query for agent egress + email tool: zero events; guardrail hit logged but unalerted" },
  "impact": "A real EchoLeak-class chain would run unobserved for weeks.",
  "recommendation": "Alert on first-seen egress destination per agent identity + guardrail-hit thresholds (C18); add canary-fire runbook (C19).",
  "status": "open"
}
```

## Negative controls
- Known-good activity (normal user traffic) generates no false alerts during replay window.
- Existing detections verified firing on their own test events (you didn't audit a dead SIEM).

## Cleanup
Export SIEM screenshots/query results into run evidence; delete any temporary detection-rule test events; deliver runbook diffs and governance checklist to the fix backlog (C19/C20).
