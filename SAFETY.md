# The Defender Charter (SAFETY.md)

**Sentinel-LLM is a shield, not a sword.** This charter is binding for every file, script, rule, and PR in this repository. It exists so that any team can adopt this project — and any open-source host can host it — with the confidence that **nothing here is built to attack anyone**.

---

## 1. What this project IS

A **defender's toolkit**: guidelines, audit skills, and (per roadmap) an orchestrator CLI that help teams *find and fix* weaknesses in **their own** LLM applications, agents, and codebases — before someone else finds them.

## 2. What this project IS NOT (and will never ship)

- **No offensive tooling.** No exploit code, no working harmful payloads, no attack automation aimed at systems you don't own. Live-skill probes are metaprompt-style *testing templates* whose only "payloads" are the canary strings defined in `skills/_shared/conventions.md`.
- **No working exfiltration channels.** All exfil tests target a sink **you run on your own machine**. Nothing in this repo points at any third-party sink.
- **No data collection. Ever.** No telemetry, no phone-home, no crash reporting, no analytics, no "check for updates" beacons, no account, no license server. See §3.
- **No cloud dependency.** Everything runs on your machine. See §3.
- **No dual-use drift.** PRs that add attack capability against non-owned systems are out of scope and will be declined per §7.

## 3. The Local-Only Guarantee

Adapted from the [local-first software principles](https://www.inkandswitch.com/essay/local-first/) (Ink & Switch) and hardened for security tooling:

| Guarantee | Meaning | How it's enforced |
|---|---|---|
| **Offline by default** | Every feature works with networking disabled. The full audit (PROSPECTOR) and the live-skill playbooks run air-gapped. | Tools run against local files/ports; CI recipe uses local runners; no step in any skill requires internet |
| **No egress** | The code never opens an outbound connection. (Exception, user-initiated and explicit: *scanning your own* declared local/staging target URL — never any destination the user didn't type.) | Orchestrator default: `--offline` on; egress only to user-supplied target; a network allow-list ships empty by default |
| **Data never leaves the machine** | Findings, evidence, transcripts, logs stay in `runs/` on your disk. | No upload/import/send verbs exist in the codebase; no telemetry SDKs in dependencies |
| **You own your data** | Reports are plain JSON/Markdown on your disk; delete them and they're gone. No copies anywhere else, because none are ever made. | Filesystem-only outputs; no background daemons; no caches outside the run directory |
| **Reproducible & inspectable** | Every probe and check is a readable playbook/script; nothing obfuscated, nothing packed, nothing that resists review | Plain-text SKILL.md files; small, reviewable scripts; no binaries |

**Verifiable claim:** run any command with your firewall set to block-all — everything still works. This is a release test in the roadmap, not a promise.

## 4. Scope of use — the healthy-purpose rule

This toolkit may only be used on systems **you own or have written authorization to test** (the RoE template exists exactly for this). It is built for:

- Securing your production LLM apps and agents
- Auditing your own codebases before ship
- Teaching teams what attacks look like *so they can defend*
- Feeding your SIEM detections and your CI gates

Using findings from this toolkit against systems you don't own is not enabled by it, not endorsed by it, and — nothing in the repo makes it easier: there are no exploit tools here, only *tests* whose evidence is designed for remediation tickets.

## 5. Canary discipline (why our tests can't harm anyone)

- Every "secret", "PII item", and "internal URL" in any probe or planted artifact is a `SENTINEL-CANARY-*` string from the conventions file.
- Every exfil destination is the tester's **own local listener**.
- If a canary is ever observed outside the lab, that event *is the finding* — the toolkit's tests are incapable of moving real data because real data is never an input.

## 6. Attribution & deterrence notes (defender help, not offense)

Where the skills describe attack techniques (e.g., EchoLeak chains, tool poisoning), descriptions are at the *pattern level* with citations to public research, and always paired with the detection rule and the fix. We document attacks the way NIST, OWASP, MITRE, and CERTs do — to build defenses. No technique is described beyond what public advisories already document, and always in remediation form.

## 7. Governance

- **Scope guard for PRs:** contributions adding offensive capability (working exploits, target discovery, brute-force, evasion automation against non-owned systems) are rejected. Defenses, checks, audits, playbooks, and fixes are welcome.
- **Review stance:** maintainers review for charter compliance, not just correctness.
- **Reporting misuse of this repo's name:** open an issue; we'll document and, if needed, revoke releases under our name.
- **License:** MIT, with this charter as the project's stated intent (it's binding on the project itself, even though the license keeps the code free).

## 8. The one-line test

> *If the project vanished tomorrow, could anything in it be used to attack someone?*
> No — it contains tests for your own systems, evidence schemas, fixes, and detection rules. That is the whole toolkit. That is the point.
