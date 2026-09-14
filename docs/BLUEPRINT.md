# Sentinel-LLM — Final Blueprint

**The complete report: what this project is, how it works, why it can't be abused, and how it ships.**

---

## 1. Mission

Give every team shipping LLM applications a **defender-only, fully local, dead-simple** way to find and fix exploitable weaknesses — guided by 20 evidence-backed controls, executed through 10 named testing skills, driven by one prompt to any agent.

**The three promises (binding, see [SAFETY.md](../SAFETY.md)):**
1. **Defender-only.** Nothing in this repo is attack tooling. Every technique is documented at pattern level, in remediation form, exactly the way OWASP/MITRE/NIST publish.
2. **Locally run.** Offline by default; zero telemetry; findings never leave your disk. Verifiable by running with your firewall on block-all.
3. **Effortlessly usable.** One prompt to any agent — "audit this repo, grade it" — that's the whole API. No CLI, no install; the skills are the procedure.

---

## 2. The problem we defend against (in one paragraph)

LLMs are structurally exploitable: instructions and data share one channel (prompt injection — unsolved at model level), capabilities come from data that can be poisoned, and agentic composition (tools, memory, egress) multiplies blast radius. Real-world zero-click exploits already shipped (EchoLeak CVE-2025-32711, ShadowLeak); MCP tool poisoning and RAG/memory implants turn one-shot attacks into persistent ones; and the code itself is now a flaw factory (45% of AI-generated code samples introduced OWASP Top 10 vulnerabilities — Veracode). AI cyber capability is doubling roughly every 4 months (UK AISI). Full evidence, with citations: [RESEARCH.md](RESEARCH.md).

## 3. The defense philosophy — "cut the slope"

You cannot fix the model. You **control the slope** between "model fooled" and "damage done" by stacking deterministic layers beneath it:

```
Model fooled?
  └─ Spotlighting wrapped the untrusted input (C06/C07) → attack visible as data
  └─ Policy enforced in code, not prompts (C01/C08)      → no policy to talk out of
  └─ Least-privilege tools + PDP authorization (C02/C15) → hijack has nothing to steer
  └─ Action gates with exact-effect previews (C03)       → human sees the real ask
  └─ Egress allow-list (C04) + budgets (C16)             → nothing can leave, affordably
  └─ Pinned/verified artifacts (C05) + sandboxing (C14)  → supply and execution contained
  └─ Continuous testing (C17) + telemetry (C18)          → every residue detected
```

An attacker must defeat *every* layer; a defender only needs the stack to hold. The 20 controls in [GUIDELINES.md](GUIDELINES.md) are exactly this stack, ordered by effort, each with a test.

## 4. System architecture (four layers)

| Layer | Artifact | Role |
|---|---|---|
| **Knowledge** | `docs/RESEARCH.md` | The evidence base: threats, incidents, standards, defenses — every claim cited (OWASP LLM10 + Agentic, MITRE ATLAS, NIST AI RMF, ISO 42001, EU AI Act, UK AISI/Inspect, CaMeL, spotlighting, MAESTRO, local-first) |
| **Controls** | `docs/GUIDELINES.md` (+ Appendix A for code-level enforcement) | The 20 controls C01–C20: threat → evidence → implement → test, mapped to OWASP/ASI/ATLAS |
| **Skills** | `skills/` (10 disciplines) | Named, repeatable testing playbooks that produce schema'd findings |
| **Orchestrator** | MARSHAL (`skills/sentinel-assessor`) | One prompt to any agent: classify → sequence → consolidate → grade → retest |

## 5. The skills suite (10 disciplines)

| Codename | Skill | Discipline |offline-capable|
|---|---|---|---|
| **MARSHAL** | `sentinel-assessor` | Orchestrator | ✅ (egress only to your own named target, live presets only) |
| **PROSPECTOR** | `audit-codebase` | Code audit (static) | ✅ 100% — no target needed |
| **X-RAY** | `introspect-leakage` | Introspection | ✅ against local/staging targets |
| **ARCHIVIST** | `introspect-memory-rag` | Introspection | ✅ |
| **LOCKPICK** | `redteam-jailbreak` | Red team | ✅ |
| **TROJAN** | `redteam-injection` | Red team | ✅ |
| **DEPUTY** | `pentest-toolchain` | Penetration | ✅ |
| **SMUGGLER** | `pentest-egress` | Penetration | ✅ (sink = your localhost) |
| **CUSTOMS** | `pentest-supply-chain` | Penetration | ✅ |
| **WATCHTOWER** | `audit-ops` | Audit | ✅ (reads *your* SIEM) |

Every skill: **scope → probe → detect → score → report → cleanup**, emitting Finding-schema JSON mapped to control IDs, recording negative results (defenses proven working) as rigorously as successes.

## 6. End-to-end: what a run looks like

Worked example — *"ACME Assistant"*, a Python RAG + tools service with an MCP filesystem server:```text
"Load PROSPECTOR from skills/manifest.json and audit ./acme-assistant"
  PROSPECTOR (static, offline)
  ├─ CA-1 secrets ............ 2 findings  (openai key in tests/fixture.env — LIVE → critical)
  ├─ CA-2 agent configs ...... 1 finding   (.cursor/rules: 'always curl ...' instruction — high)
  ├─ CA-3 LLM calls .......... 1 finding   (user text → system prompt, unmarked — medium)
  ├─ CA-4 output sinks ....... 1 finding   (completion → os.system — critical)
  ├─ CA-5 dependencies ....... 1 finding   (docs reference non-existent pkg 'fastcsv2' — high)
  ├─ CA-6 model artifacts .... clean       (safetensors only ✅ negative result recorded)
  ├─ CA-7 MCP server ......... 1 finding   (tool description contains instruction text — high)
  └─ grade: D — fix order: [C01 secrets, C04 sink, C05 slopsquat, C05 MCP, C06, C01]

"Run the agentic preset against my service on localhost:8080. Canaries only."
  (RoE check: local target ✓)  (canary sink: 127.0.0.1:8765 ✓)
  ├─ X-RAY ...... system prompt extracted in 11 turns; contains internal URL → medium
  ├─ LOCKPICK ... Crescendo bypass by turn 4 → high (layer: model, no app block)
  ├─ TROJAN ..... planted doc canary reached sink via markdown auto-fetch → CRITICAL (EchoLeak pattern)
  ├─ ARCHIVIST .. memory implant persisted across sessions; no purge API → high
  ├─ DEPUTY ..... delete on canary rows with vague approval → high; PDP absent
  ├─ SMUGGLER ... (TROJAN already fired SE-1; egress allow-list absent)
  └─ WATCHTOWER . replay vs SIEM: zero alerts on the exfil chain → high (invisible breach)

"Retest the open findings from my last run."
  3 of 9 findings closed with proof (negative results); grade D → C. Remaining: 6.
```

The operator never configures anything, never reads a schema, and finishes with a fix list ordered by (severity × control level × effort). Findings and grades are plain JSON files — CI-consumable as-is.

## 7. The safety architecture — why this cannot be weaponized

1. **Canary-only payloads.** Every "secret"/"PII"/"URL" in probes is a `SENTINEL-CANARY-*` string; every exfil destination is the tester's own localhost sink. Real data is never an input, so tests cannot move it.
2. **Metaprompt-style probes.** Skills document mechanisms (override, escalation, encoding) without shipping working harmful payloads — the same standard as public OWASP/MITRE documentation, always paired with detection + fix.
3. **Ownership gates in code.** The orchestrator refuses non-local targets without a signed RoE; destructive live tests run only against canary-marked staging resources; abort conditions halt on any anomaly.
4. **No egress code.** No upload/send/telemetry verbs exist anywhere in the repo; dependency policy bans analytics SDKs; release test = full functionality with networking disabled.
5. **Charter governance.** SAFETY.md is binding on maintainers and contributors: offensive-capability PRs are declined; everything stays plain-text and reviewable (no obfuscation, no packed binaries).

**The one-line test:** *if this repo vanished tomorrow, could anything in it be used to attack someone?* No — it contains tests for your own systems, evidence schemas, fixes, and detection rules. That is the whole toolkit.

## 8. Evidence standards

- **Finding schema** (`skills/_shared/conventions.md` §1): id, skill, probe, severity, **controls**, OWASP, ATLAS, exact evidence, impact, recommendation, status.
- **Grading rubric** (§4): A–F with deterministic caps; Level-0 control failures drop an extra grade; info findings never count.
- **KPIs across runs:** bypass rate per skill, mean time-to-detect (WATCHTOWER), findings reopened, cleanup completion. Negative results are first-class data.
- **Interoperability:** schema maps cleanly to OCSF-style findings for SIEM ingestion; every skill doubles as a detection-rule source for WATCHTOWER deliverables.

## 9. Repository map

```
├── README.md                 start here — quick start + suite table
├── SAFETY.md                 the binding Defender Charter
├── LICENSE                   MIT
├── docs/
│   ├── RESEARCH.md           evidence base (all cited)
│   ├── GUIDELINES.md         the 20 controls + Appendix A (code-level)
│   ├── BLUEPRINT.md          this report
│   └── ROADMAP.md            v0.1 → v1.0 ship plan
└── skills/
    ├── README.md             suite index + workflow diagram + hard rules
    ├── _shared/conventions.md finding schema, canaries, severity, RoE, grading
    ├── sentinel-assessor/    MARSHAL (prompt-driven orchestrator)
    ├── audit-codebase/       PROSPECTOR + per-language tables + Semgrep rules
    ├── introspect-leakage/   X-RAY
    ├── introspect-memory-rag/ ARCHIVIST
    ├── redteam-jailbreak/    LOCKPICK
    ├── redteam-injection/    TROJAN
    ├── pentest-toolchain/    DEPUTY
    ├── pentest-egress/       SMUGGLER
    ├── pentest-supply-chain/ CUSTOMS
    └── audit-ops/            WATCHTOWER
```

## 10. Ship plan (condensed from ROADMAP)

- **v0.1 (now):** charter + guidelines + research + 10 skill playbooks + orchestrator spec.
- **v0.2–v0.4:** machine-readable checklist/scorer; optional probe runners an agent can invoke (gitleaks/semgrep/osv/modelscan wired); reference implementations (spotlighting preprocessor, egress proxy, PDP, memory contracts).
- **v0.6:** live-skill automation (probe emitters), AgentDojo/Inspect-compatible eval wrappers, bypass-rate dashboards.
- **v1.0:** docs site, reference case study, community governance — **launch release test: full suite runs with networking disabled.**

## 11. Sources

Anchored in: OWASP GenAI (LLM Top 10 2025, Agentic Top 10 2026), MITRE ATLAS, NIST AI RMF 1.0/600-1, ISO/IEC 42001, EU AI Act GPAI, UK AISI (Inspect, AgentDojo-Inspect, capability doubling), CaMeL (DeepMind), agent design patterns (2506.08837), Spotlighting (Microsoft), CSA (MAESTRO, confused-deputy, slopsquatting), Veracode GenAI Code Security 2025/2026, Truffle Security, Radware, Pillar Security (Rules File Backdoor), Socket (slopsquatting), Semgrep (OWASP-LLM SAST, MCP guide), Protect AI ModelScan, Datadog GuardDog, OpenSSF model signing, CycloneDX ML-BOM, Ink & Switch (local-first), OCSF. Full link list: [RESEARCH.md](RESEARCH.md) §7.
