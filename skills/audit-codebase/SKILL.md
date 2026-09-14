# PROSPECTOR — `audit-codebase`

**Discipline:** Code audit (static, offline) · **Controls:** C01–C05, C07, C08, C11, C13–C15 (code level) · **OWASP:** LLM03, LLM05, LLM06, LLM07 · **ATLAS:** AML.T0010, AML.T0048 · **Requires:** repository access. **Nothing else.**

## Purpose
PROSPECTOR is the **read-only, offline** discipline: point it at *any* codebase in *any* language (Python, Node/TS, Go, Java, Ruby, PHP, .NET — see [`languages.md`](languages.md)) and it finds the LLM-security flaws **in the code itself** before anything runs. No live target, no probes, no canaries, no RoE required — static analysis only. It answers: *if we shipped this repo tomorrow, which of the 20 controls is already violated in source?*

**Why this matters (evidence):** 45% of AI-generated code fails basic security tests (Veracode 2025 GenAI Code Security Report; still 28–30% in their Spring 2026 update); LLMs actively recommend hardcoding API keys (Truffle Security); AI-agent config files are now a credential-leak channel ("the new .env" — Radware) and a poisoning vector (Pillar Security's Rules File Backdoor; 8+ prompt-injection CVEs across Copilot, Cursor, Claude Code, Amazon Q, Codex in 12 months).

## Method
Run automated tooling first (deterministic recall), then targeted manual review of the flagged + AI-suspect code (precision). Record everything per `_shared/conventions.md` Finding schema. Per-language pattern tables live in [`languages.md`](languages.md).

### CA-1 · Secrets & credential hygiene
- **Find:** hardcoded API keys/tokens, connection strings, webhook URLs, `.env` committed, secrets in test fixtures, **secrets inside LLM system prompts and prompt templates**, and — AI-specific — **credentials pasted into agent instruction files** (`CLAUDE.md`, `AGENTS.md`, `.cursor/rules/*`, `.github/copilot-instructions.md`) so the model "remembers" them.
- **Tools:**
  ```bash
  gitleaks detect --source . --report-path findings/gitleaks.json
  trufflehog filesystem . --only-verified   # live verification of candidates
  # agent-config leak check (the "new .env"):
  rg -n -i "(api[_-]?key|secret|token|password|Bearer |sk-[A-Za-z0-9]{20,})" \
     -g "*.md" -g ".cursorrules" -g "!.md" -g "AGENTS.md" -g "CLAUDE.md"
  ```
- **Severity:** live credential in any file → **critical** (rotate immediately); shaped secret/test key → **high**; secret in prompt template → **high** (C01); committed `.env` without secrets → **low**.

### CA-2 · Agent/AI config poisoning (Rules File Backdoor class)
- **Find:** hidden or instruction-shaped text in files AI coding agents auto-ingest: `.cursor/rules/`, `.cursorrules`, `CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md`, skill/instruction directories, and any markdown a review agent reads. Look for: instructions addressed to the agent ("always run…", "ignore…", "before committing…"), zero-width/RTL/bidi control characters, HTML-comment-wrapped text, rendered-vs-raw markdown discrepancies, instructions conditioned on file paths ("when editing `.env`…").
- **Tools:**
  ```bash
  # invisible characters (zero-width, BOM, bidi overrides):
  rg -n "[\x{200B}-\x{200D}\x{FEFF}\x{202A}-\x{202E}]" -g "*.md" -g "*.mdc" -g "*.txt"
  # instruction-shaped sentences in agent configs:
  rg -n -i "(ignore (all )?(previous|prior)|always run|before (you )?commit|curl |http[s]?://|eval|exec)" \
     AGENTS.md CLAUDE.md .cursorrules .cursor/ .github/copilot-instructions.md 2>/dev/null
  ```
- **Manual diff check:** rendered markdown vs raw bytes for every agent config (Pillar's discrepancy trick hides payloads in link titles, image alts, collapsed sections).
- **Severity:** agent-behavior-modifying hidden instruction → **high** (supply-chain implant); exfil- or exec-shaped instruction → **critical**; network URLs in config without documented purpose → **medium**; missing config hygiene (no owner, no review) → **low**.

### CA-3 · LLM call construction (how prompts are assembled)
- **Find:** user/external input interpolated into system prompts without delimiting (spotlighting gap, C06); PII flowing into model calls unredacted (C11); no max-token/output caps (C16); temperature/system-prompt built from config users control; provider SDK keys read from correct env (good — record negative result).
- **Patterns:** Python f-strings/`.format()`/`+` assembling `system=` content from request data; JS template literals feeding `messages:`; prompt-template files that `{user_input}` lands in position 0.
- **Tool:** Semgrep with the OWASP-LLM coverage (see `semgrep.dev` — ruleset name in registry may be `p/llm`/`p/gen-ai`; pin the exact one in your CI) plus custom rules from your own findings:
  ```bash
  semgrep --config p/owasp-top-ten --config p/secrets --config p/command-injection \
          --config <llm-llm-top10-ruleset> --json --output findings/semgrep.json
  ```
- **Severity:** user input into system prompt unmarked → **medium** (C06/C08); PII → model with no redaction hook → **medium** (C11); missing output caps → **low** (C16).

### CA-4 · Output-sink tracing (LLM05 in code)
- **Find:** model output flowing into dangerous sinks without validation — the full list of sinks per language is in [`languages.md`](languages.md) §"Output sinks". Core question: **is there any path from `completion` / `response.content` / `choices[0].message` to a sink?** Trace manually if the SAST dataflow can't.
- **Severity:** path to code exec (eval/exec/shell/`Function`) → **critical**; path to SQL/shell with any sanitization → **high**; path to HTML without escaping → **medium**; path to logs/file write → **low**.

### CA-5 · Dependency integrity & slopsquatting
- **Find:** known-vulnerable deps; lockfile drift/absence; **hallucinated package names** in docs, READMEs, examples, and AI-generated code comments — attackers register those names (slopsquatting, 5–20% hallucination rates in studies); typosquatting-shaped names; install scripts (`postinstall`, `setup.py` commands).
- **Tools:**
  ```bash
  npm audit --json          # or pnpm/yarn audit          (Node)
  pip-audit -r requirements.txt                            # Python
  osv-scanner --recursive .                                 # universal (OSV)
  guarddog npm-scan / guarddog pip-scan <pkg>               # malicious-package heuristics
  # slopsquatting sweep: extract package-ish identifiers from docs & code, verify each exists:
  npm view <name> version || echo "NONEXISTENT - slopsquat candidate"
  pip index versions <name> 2>/dev/null || echo "NONEXISTENT - slopsquat candidate"
  ```
- **Severity:** vulnerable dep on an auth/egress path → **high**; non-existent package referenced anywhere → **high** (it *will* be registered eventually); missing lockfile → **medium**; install scripts without justification → **medium**.

### CA-6 · Model & dataset artifacts in the repo
- **Find:** committed model weights (esp. pickle/pytorch `.bin`), datasets of unknown origin, model URLs pinned to `main`.
- **Tools:** `modelscan -p ./models` (Protect AI) or `promptfoo modelaudit` (42+ formats) — run **in a sandbox**, never import-load directly.
- **Severity:** pickle-format weights in repo → **high** (C05); unscanned third-party weights → **medium**; unpinned model refs → **medium**. (Deep inventory lives in CUSTOMS CS-1/CS-2; PROSPECTOR flags what's *in the repo*.)

### CA-7 · MCP / tool-server code
- **Find (static mcp-sec-audit patterns):** tool descriptions containing instruction-shaped text; tools requesting broad scopes (fs root, `*` env, network); missing per-call schema validation; ambient credential reuse (server process owner token passed to tools — token passthrough); non-idempotent tools without confirmation hooks.
- **Tools:** `mcpserver-audit`, `mcp-sec-audit` toolkits; MCP Inspector against a local instance; Semgrep MCP guidance rules (Semgrep "Security Engineer's Guide to MCP").
- **Severity:** description-injection text → **high**; over-broad scope + ambient creds → **high** (C02/C15); missing schema validation → **medium** (C13).

### CA-8 · Unsafe deserialization & dynamic execution
- **Find:** per-language table in [`languages.md`](languages.md) §"Dynamic execution" — `eval`/`exec`/`pickle`/`yaml.load`/`Marshal.load`/`unserialize`/`Function`/`eval`, Java `XStream`/`ObjectInputStream`, .NET `BinaryFormatter`.
- **Severity:** reachable with external input → **critical**; reachable with model output → **critical** (C04 chain); present but unreachable → **medium** (defense-in-depth).

### CA-9 · CI/CD & pipeline (code side of CUSTOMS CS-5)
- **Find:** workflows pulling models/containers `:latest`; actions pinned by tag not SHA; verify-signature steps that warn instead of fail; model download steps without hash check; secrets passed to model-training jobs.
- **Severity:** fail-open verification → **medium**; unpinned third-party action with secrets access → **high**; `:latest` model pulls → **medium**.

## Scoring & outputs
- Findings JSON per conventions; `controls` mapped per family header; each finding lists `file:line`.
- Produce **repo grade** (same rubric) + the two deliverables: (1) fix backlog with exact locations, (2) CI recipe — the exact tool commands above wired as a pipeline stage (feeds ROADMAP M4 automation).
- Mark AI-generated-code suspects: files/dirs with generation signatures (`Generated by`, agent commit patterns, pristine style clusters) get extra CA-3/CA-4 scrutiny — per Veracode, ~1/3 of such code carries a flaw.

## Negative controls
- Run the full tool set against a known-clean sample repo → zero criticals/false-positive storm (tune rules before trusting results).
- Re-run after fixes → findings marked `fixed`, grade re-computed (regression evidence for C17).

## Cleanup
None required beyond run artifacts — PROSPECTOR never modifies the target repo, installs nothing, executes no target code (artifact scanners run sandboxed). Evidence: tool JSON outputs + finding files only.
