# AI/LLM Cybersecurity Research Compendium

> **Scope.** What can go wrong when LLMs are deployed in real systems, and what actually works to reduce that risk. Compiled September 2026 from primary frameworks (OWASP GenAI, MITRE ATLAS, NIST, EU AI Act, UK AISI) plus incident and defense literature. Every claim links to a source so the guidelines can be audited back to evidence.

---

## 1. Why "the model is exploitable" is the correct starting assumption

An LLM is not a deterministic program — it is a probabilistic function trained on untrusted data, invoked with untrusted input, and increasingly connected to tools, memory, and networks. Three structural facts make it exploitable:

1. **Instructions and data share one channel.** The model reads attacker-influenced text with the same machinery it reads your instructions with. There is no hardware-enforced boundary between "what the developer said" and "what the document said." Prompt injection (OWASP LLM01) exploits exactly this and is widely considered *not fully solvable at the model level* ([OpenAI, 2025, cited in CyberDesserts](https://blog.cyberdesserts.com/prompt-injection-attacks/)).
2. **Model behavior is stolen, not authored.** Capabilities come from training corpora and training-time objectives, so poisoning and backdoors persist silently ([OWASP LLM04 Data & Model Poisoning](https://genai.owasp.org/llmrisk/llm04-data-and-model-poisoning/), [MITRE ATLAS](https://atlas.mitre.org/)).
3. **Agentic composition multiplies blast radius.** Once the model can call tools, browse, or act on email/tickets, a single successful injection becomes a confused-deputy attack with real-world permissions ([CSA: Confused Deputy Attacks on Autonomous AI Agents](https://labs.cloudsecurityalliance.org/research/csa-research-note-ai-agent-confused-deputy-prompt-injection/), [Quarkslab](https://blog.quarkslab.com/agentic-ai-the-confused-deputy-problem.html)).

**Consequence for engineering:** you do not "fix" the model; you build a system around it so that *when* it is exploited, nothing valuable is destroyed or leaked. That is the design philosophy of the accompanying [GUIDELINES.md](GUIDELINES.md).

---

## 2. Reference taxonomies

### 2.1 OWASP Top 10 for LLM Applications, 2025 (GenAI Security Project)

| # | Risk | Essence |
|---|------|---------|
| LLM01 | Prompt Injection | Direct, indirect, and multimodal inputs that hijack model behavior ([OWASP LLM01](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)) |
| LLM02 | Sensitive Information Disclosure | PII/credentials/IP leakage via prompts, outputs, or training data ([Pomerium summary](https://www.pomerium.com/blog/the-owasp-top-10-for-llms-and-how-to-defend-against-them)) |
| LLM03 | Supply Chain Vulnerabilities | Compromised models, datasets, plugins, and now MCP servers ([OWASP project page](https://owasp.org/www-project-top-10-for-large-language-model-applications)) |
| LLM04 | Data and Model Poisoning | Training-time and RAG-time corruption of what the model "knows" |
| LLM05 | Improper Output Handling | Unsafe downstream use of model output (SQL/shell/code paths, XSS) |
| LLM06 | Excessive Agency | Tools, permissions, or autonomy beyond what the function requires |
| LLM07 | System Prompt Leakage | Prompt secrets extraction; prompts are not a security boundary ([OWASP LLM07](https://genai.owasp.org/llmrisk/llm07-insecure-plugin-design/)) |
| LLM08 | Vector and Embedding Weaknesses | RAG-specific: retrieval poisoning, cross-context access in multi-tenant stores |
| LLM09 | Misinformation | Hallucination, overreliance, confident wrong output with downstream harm |
| LLM10 | Unbounded Consumption | Variable-length input floods, resource-intensive queries, **Denial of Wallet** ([OWASP LLM10](https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption/)) |

### 2.2 OWASP Top 10 for Agentic Applications (2026)

Extends the LLM list for autonomous agents: **ASI01 Agent Goal Hijack**, **ASI02 Tool Misuse and Exploitation**, **ASI03 Identity & Privilege Abuse**, **ASI04 Resource Overload**, agentic supply chain, memory poisoning, and **rogue agents** ([OWASP announcement](https://genai.owasp.org/2025/12/09/owasp-top-10-for-agentic-applications-the-benchmark-for-agentic-security-in-the-age-of-autonomous-ai/), [Graylog explainer](https://graylog.org/post/what-is-the-owasp-top-10-agentic-ai/), [Zenity analysis](https://zenity.io/blog/the-owasp-top-10-for-agentic-applications)).

### 2.3 MITRE ATLAS

ATT&CK-style knowledge base for attacks on AI systems: currently **16 tactics, 84 techniques, 56 sub-techniques** (reconnaissance through impact), including ML-specific ones like model poisoning, evasion, and extraction ([atlas.mitre.org](https://atlas.mitre.org/), [Vectra summary](https://www.vectra.ai/topics/mitre-atlas)). Useful for mapping your LLM app to a threat matrix that your SOC already understands — attack simulation via [DeepTeam](https://trydeepteam.com/docs/frameworks-mitre-atlas) or [promptfoo's ATLAS plugin](https://www.promptfoo.dev/docs/red-team/mitre-atlas/).

---

## 3. Attack surface deep-dives (with adverse effects)

### 3.1 Prompt injection — the #1 risk
- **Direct:** attacker types an override ("ignore previous instructions…"). **Indirect:** malicious instructions hidden in content the LLM *reads* — emails, web pages, PDFs, tickets, tool outputs, even images ([multimodal prompt injection survey](https://arxiv.org/html/2509.05883v1)).
- **Adverse effects:** policy bypass, data exfiltration, unauthorized actions with the agent's privileges, persistent compromise via memory ([Microsoft MSRC defense-in-depth analysis](https://www.microsoft.com/en-us/msrc/blog/2025/07/how-microsoft-defends-against-indirect-prompt-injection-attacks)).
- **Detection is weak:** LLM-based injection detectors miss large fractions of attacks; treat detection as a layer, never the control ([A-MemGuard finding via mem0](https://mem0.ai/blog/ai-memory-security-best-practices)).

### 3.2 Jailbreaks — bypassing safety training
Taxonomy of techniques ([Promptfoo jailbreak guide](https://www.promptfoo.dev/blog/how-to-jailbreak-llms/), [Vaikora taxonomy](https://vaikora.com/blog/ai-jailbreak-taxonomy-enterprise-defenses)):
- Roleplay/persona ("DAN" variants), hypothetical framing, policy framing.
- **Multi-turn escalation ("Crescendo")** — each turn stays below single-turn refusal thresholds ([kibotu gist](https://gist.github.com/kibotu/c06f54d6fbc4705e886a50fb2e59e6ae)).
- **Bad Likert Judge** — asking the model to rate harmful content on a scale, harvesting the "worst" example ([Unit42](https://unit42.paloaltonetworks.com/multi-turn-technique-jailbreaks-llms/)).
- Encoding (Base64, leetspeak, low-resource languages), token smuggling, cipher attacks.

**Adverse effects:** harmful content generation, reputational/legal exposure, downstream automation of harm.

### 3.3 Agentic exploitation — the 2025–2026 incident record
| Incident | What happened | Lesson |
|---|---|---|
| **EchoLeak (CVE-2025-32711)**, M365 Copilot, Jun 2025 | First real-world **zero-click** LLM exploit: a malicious email planted indirect instructions; Copilot's own context gathered data; exfiltration via a trusted-link bypass with no malware or user click ([arXiv 2509.10540](https://arxiv.org/html/2509.10540v1), [Cato Networks](https://www.catonetworks.com/blog/breaking-down-echoleak/), [Checkmarx](https://checkmarx.com/zero-post/echoleak-cve-2025-32711-show-us-that-ai-security-is-challenging/)) | The AI layer itself is the attack surface; egress control on the agent matters more than malware detection |
| **ShadowLeak**, ChatGPT + connectors, Sep 2025 | Zero-click **service-side** leak: malicious instruction in a fetched email made the cloud-hosted agent send data out; nothing visible on the client ([Radware](https://www.radware.com/security/threat-advisories-and-attack-reports/shadowleak/)) | Cloud-side agents need egress monitoring too; "it never ran on my machine" ≠ safe |
| **ChatGPT Deep Research zero-click**, Sep 2025 | Prompt injection exfiltrated PII through Deep Research; patched by OpenAI ([Malwarebytes](https://www.malwarebytes.com/blog/news/2025/09/chatgpt-deep-research-zero-click-vulnerability-fixed-by-openai)) | Even top vendors get bitten; zero-click agent exploits are now normal |
| **MCP tool poisoning**, Apr 2025 | Malicious instructions embedded in MCP *tool descriptions* hijacked clients; cross-server shadowing possible ([Invariant Labs](https://invariantlabs.ai/blog/mcp-security-notification-tool-poisoning-attacks)) | Tool metadata is untrusted input; pin, review, and sandbox MCP servers |
| **MCP in the wild / IDE auto-exec**, 2025–2026 | Malicious MCP server deployed in the wild; auto-executing IDEs (Cursor etc.) turned tool results into code execution ([authzed timeline](https://authzed.com/blog/timeline-mcp-breaches), [CSA](https://labs.cloudsecurityalliance.org/research/csa-research-note-mcp-tool-poisoning-auto-execution-20260701/)) | Human-in-the-loop before executing tool-driven actions; treat IDE agents as semi-trusted |
| **AI-driven offensive campaigns** | Threat actors combining autonomous AI scanning with manual exploitation across multiple vulns ([Unit 42](https://unit42.paloaltonetworks.com/autonomous-ai-cyber-attack-campaign/)) | Models are also *weapons*; monitor for misuse of your own AI features |
| **Autonomous-attack capability doubling** | AISI-measured cyber capability doubling every ~4.2 months on software tasks ([AISI blog](https://www.aisi.gov.uk/blog/how-fast-is-autonomous-ai-cyber-capability-advancing)) | Today's "safe" autonomy level may not be safe next year; re-evaluate continuously |

### 3.4 RAG & knowledge-base poisoning
- Injecting a few crafted documents into the corpus is enough to bend retrieval and answers ([EMNLP 2025 Findings](https://aclanthology.org/2025.findings-emnlp.1023.pdf), [EmergentMind topic](https://www.emergentmind.com/topics/retrieval-augmented-generation-rag-poisoning)).
- Defenses with evidence: **hybrid BM25 + vector retrieval** blunts gradient-guided poisoning ([arXiv 2603.18034](https://arxiv.org/html/2603.18034v1)); layered frameworks like **RAGuard** (retrieval-level adversarial training + inference patch) ([NeurIPS 2025](https://neurips.cc/virtual/2025/133168)).
- **Adverse effects:** persistent misinformation, targeted defamation, silent policy overrides that survive model updates.

### 3.5 Agent memory poisoning
- Indirect injection can persist into **long-term memory**, turning a one-shot attack into a durable implant; systematic studies show ~50% attack success rates across agents ([arXiv 2606.04329](https://arxiv.org/html/2606.04329v1), [Unit42](https://unit42.paloaltonetworks.com/indirect-prompt-injection-poisons-ai-longterm-memory/), [MINJA/OpenReview](https://openreview.net/forum?id=QINnsnppv8)).
- **Defenses:** provenance-tracked memory writes, memory "contracts" (schemas + validation), TTLs and integrity checks ([Christian Schneider](https://christian-schneider.net/blog/persistent-memory-poisoning-in-ai-agents/), [Hannecke](https://medium.com/@michael.hannecke/agent-memory-poisoning-the-attack-that-waits-9400f806fbd7)).

### 3.6 Supply chain
- **Poisoned model repos:** pickle-based models executing arbitrary code on load ("nullifAI" broken-pickle attacks) ([ReversingLabs](https://www.reversinglabs.com/blog/rl-identifies-malware-ml-model-hosted-on-hugging-face), [JFrog silent backdoor](https://jfrog.com/blog/data-scientists-targeted-by-malicious-hugging-face-ml-models-with-silent-backdoor/), [PickleBall paper](https://arxiv.org/html/2508.15987v1)).
- **Controls with teeth:** prefer safetensors, **sign and verify models** ([Sigstore model-transparency](https://github.com/sigstore/model-transparency), [OpenSSF OMS spec](https://openssf.org/blog/2025/06/25/an-introduction-to-the-openssf-model-signing-oms-specification/)), publish **AI/ML-BOMs** ([CycloneDX ML-BOM](https://cyclonedx.org/capabilities/mlbom/)), pin versions and hashes.
- MCP servers and agent plugins are the newest supply-chain link — threat-model them like npm packages ([NSA/CISA MCP security CSI](https://media.defense.gov/2026/Jun/02/2003943289/-1/-1/0/CSI_MCP_SECURITY.PDF), [MCP security best practices](https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices)).

### 3.7 Privacy: memorization & extraction
- LLMs memorize training data; simple sampling recovers a large share of memorized content, and fine-tuning amplifies leakage ([arunbaby AI security notes](https://www.arunbaby.com/ai-security/0027-what-ai-systems-remember-training-data-extraction-memorization-privacy/), [USENIX Security 2025 PII extraction](https://www.usenix.org/system/files/usenixsecurity25-cheng-shuai.pdf)).
- Post-hoc "unlearning" is unreliable; the dependable control is **redact before training/RAG** ([Philterd](https://philterd.ai/blog/can-an-llm-leak-its-training-data-why-you-cannot-un-train-pii/)); differential privacy during training as a stronger option.

### 3.8 Model theft
- API-based distillation/extraction clones models from ordinary query access ([survey arXiv 2506.22521](https://arxiv.org/html/2506.22521v1), [Praetorian](https://www.praetorian.com/blog/stealing-ai-models-through-the-api-a-practical-model-extraction-attack/), [KDD 2025 tutorial](https://labrai.github.io/KDD2025_Tutorial/)).
- **Defenses:** rate limiting, anomaly detection on query patterns, watermarking/fingerprinting ([IEEE TIFS watermark](https://dl.acm.org/doi/10.1109/TIFS.2025.3530691)), and ToS + monitoring as deterrence.

### 3.9 Resource abuse (LLM10)
- Variable-length input floods, continuous overflow, and resource-intensive queries → **Denial of Service and Denial of Wallet** ([OWASP LLM10](https://genai.owasp.org/llmrisk/llm102025-unbounded-consumption/)).
- Controls: per-identity quotas, token/output caps, cost circuit breakers, queueing, anomaly-based autoscaling.

### 3.10 The codebase itself is an attack surface (static/code-audit evidence)

*Added Sept 2026 — extends the research from runtime behavior to source code in any language.*

- **AI-generated code is a measurable flaw factory:** Veracode's 2025 GenAI Code Security Report found **45% of AI-generated code samples introduced OWASP Top 10 vulnerabilities** across 100+ LLMs and 4 languages; their Spring 2026 update still shows 28–30% ([Veracode](https://www.veracode.com/blog/genai-code-security-report/), [Spring 2026 update](https://www.veracode.com/blog/spring-2026-genai-code-security/)).
- **LLMs teach bad habits:** Truffle Security found most popular LLMs *recommend hardcoding API keys and passwords* ([Truffle blog](https://trufflesecurity.com/blog/llms-are-teaching-developers-to-hardcode-api-keys)).
- **Agent config files are "the new .env":** Radware measured credential leakage in CLAUDE.md / .cursorrules / copilot-instructions files — agents read these every session, and developers paste secrets into them ([Radware](https://www.radware.com/blog/the-new-env-measuring-credential-leakage-in-ai-agent-instruction-files/)); Codacy scanned 34,266 repos and found 1 in 4 orgs with AI-agent config gaps ([Codacy](https://blog.codacy.com/we-scanned-34266-repos.-1-in-4-orgs-showed-gaps-in-ai-agent-config-files)).
- **Rules File Backdoor (Pillar Security, Mar 2025):** hidden instructions in Copilot/Cursor rule files (render-vs-raw markdown discrepancies, unicode tricks) turn the coding agent itself into the attack vector — a supply-chain implant in the developer toolchain ([Pillar](https://www.pillar.security/blog/new-vulnerability-in-github-copilot-and-cursor-how-hackers-can-weaponize-code-agents), [Hidden Layer analysis](https://www.hiddenlayer.com/research/how-hidden-prompt-injections-can-hijack-ai-code-assistants-like-cursor)); 8+ prompt-injection CVEs across Copilot, Claude Code, Cursor, Amazon Q, Codex within 12 months ([yage.ai roundup](https://yage.ai/share/ai-coding-config-injection-en-20260422.html)).
- **Slopsquatting:** LLMs hallucinate plausible package names 5–20% of the time; attackers pre-register those names with malware — a *new* supply-chain attack class created by AI coding assistants ([Socket](https://socket.dev/blog/slopsquatting-how-ai-hallucinations-are-fueling-a-new-class-of-supply-chain-attacks), [CSA research note](https://labs.cloudsecurityalliance.org/research/csa-research-note-slopsquatting-ai-supply-chain-20260419-csa/)).
- **Static analysis for LLM apps is mature enough to gate CI:** Semgrep ships coverage for the OWASP LLM Top 10 (prompt injection, sensitive disclosure, improper output handling, excessive agency) ([Semgrep blog](https://semgrep.dev/blog/2026/getting-ready-for-mythos-with-semgrep)) plus an MCP-specific security guide ([Semgrep MCP guide](https://semgrep.dev/blog/2025/a-security-engineers-guide-to-mcp)); academic toolkits now statically audit MCP servers for over-privileged tools (mcp-sec-audit, [arXiv](https://arxiv.org/html/2603.21641v1)).
- **Artifact scanning tooling:** Protect AI **ModelScan** (pickle/H5/SavedModel serialization attacks) ([GitHub](https://github.com/protectai/modelscan)); promptfoo **ModelAudit** (42+ model formats) ([promptfoo](https://www.promptfoo.dev/blog/open-sourcing-modelaudit/)); Datadog **GuardDog** 3.0 (malicious PyPI/npm/Go/GitHub-Action heuristics) ([Datadog](https://securitylabs.datadoghq.com/articles/guarddog-3-0-release/)).
- **Secrets & dependency scanning are the baseline:** Gitleaks/TruffleHog for secrets (TruffleHog adds live credential verification) ([comparison](https://www.jit.io/resources/appsec-tools/trufflehog-vs-gitleaks-a-detailed-comparison-of-secret-scanning-tools)); OSV-Scanner/npm audit/pip-audit + CycloneDX SBOM for dependencies ([GitLab SBOM scanning](https://docs.gitlab.com/user/application_security/dependency_scanning/dependency_scanning_sbom/), [OWASP dep-scan](https://owasp.org/www-project-dep-scan/)).

**Consequence for the project:** runtime skills (LOCKPICK…WATCHTOWER) test the *running* system; a static, read-only **code audit** discipline (PROSPECTOR) finds the same control violations *in source, in any language, before deployment* — with no live target and no risk. Both share the control IDs and finding schema.

### 3.11 Misinformation & overreliance (LLM09)
- Confident hallucinations + human overtrust produce bad decisions, fake citations, defamation risk ([factuality review, Springer 2026](https://link.springer.com/article/10.1007/s10462-025-11454-w)).
- Mitigations: RAG grounding with *verifiable* citations, self-consistency decoding, uncertainty surfacing, human sign-off for consequential outputs ([RAG grounding tests](https://medium.com/@Nexumo_/rag-grounding-11-tests-that-expose-fake-citations-30d84140831a)).

---

## 4. Standards, laws, and AISI

### 4.1 NIST AI RMF 1.0 (AI 100-1) + Generative AI Profile (AI 600-1)
Govern–Map–Measure–Manage lifecycle, 12 GenAI risks incl. CBRN/info integrity, 400+ actions ([NIST AI 600-1 PDF](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf), [NIST AI RMF hub](https://www.nist.gov/itl/ai-risk-management-framework)). An agentic profile is emerging ([CSA note](https://labs.cloudsecurityalliance.org/agentic/agentic-nist-ai-rmf-profile-v1/)). **Use for:** org-level risk process and vocabulary.

### 4.2 ISO/IEC 42001:2023
Certifiable AI Management System standard: 38 controls, AIMS lifecycle, risk treatment ([ISO page](https://www.iso.org/standard/42001), [Schellman requirements](https://www.schellman.com/blog/iso-certifications/what-are-iso-42001-requirements)). **Use for:** audits and enterprise procurement credibility.

### 4.3 EU AI Act — GPAI
GPAI obligations live since **2 Aug 2025**; GPAI Code of Practice declared adequate 1 Aug 2025; systemic-risk providers have Art. 55 duties (evaluations, incident reporting); enforcement & penalties from **2 Aug 2026** ([AI Act explainer](https://artificialintelligenceact.eu/high-level-summary/), [EC Code of Practice page](https://digital-strategy.ec.europa.eu/en/policies/ai-code-practice), [CSET analysis](https://cset.georgetown.edu/article/eu-ai-code-safety/)). **Use for:** deployment decisions, documentation duties, timeline pressure.

### 4.4 UK AI Security Institute (AISI)
- **Inspect AI** — open-source evaluation framework; Tasks/Datasets/Solvers/Scorers ([inspect.aisi.org.uk](https://inspect.aisi.org.uk/), [GitHub](https://github.com/UKGovernmentBEIS/inspect_ai)); **Inspect Evals** packages their published eval suites ([AISI blog](https://www.aisi.gov.uk/blog/inspect-evals)).
- **AgentDojo-Inspect** — US AISI (CAISI) port of ETH's AgentDojo for agent-hijack research ([NIST entry](https://www.nist.gov/data-publications/agentdojo-inspect)); original [AgentDojo (NeurIPS 2024)](https://arxiv.org/abs/2406.13352) measures both *utility* and *security* under injection.
- **Cyber capability measurement** — 71 cyber evaluations aggregated via IRT-style scoring; capability doubling ~4.2 months ([AISI](https://www.aisi.gov.uk/blog/how-fast-is-autonomous-ai-cyber-capability-advancing)); joint assessments with NIST CAISI (e.g., Kimi K3) ([NIST news](https://www.nist.gov/news-events/news/2026/07/uk-aisi-caisi-preliminary-assessment-kimi-k3s-cyber-capabilities)).
- **Why AISI matters for this project:** Inspect gives us a ready-made, standards-adjacent harness to implement our checklist as *runnable evaluations* instead of static prose.

### 4.5 Layered engineering frameworks
- **Databricks DASF v3.0:** 13 system components, ~97 technical risks mapped to 64+ controls, agentic AI added in 2026 ([DASF 2.0 announcement](https://www.databricks.com/blog/announcing-databricks-ai-security-framework-20), [DASF v3.0](https://www.databricks.com/blog/agentic-ai-security-new-risks-and-controls-databricks-ai-security-framework-dasf-v30)). **Use for:** control coverage checklists.
- **Frontier Model Forum** technical reports on managing offensive-cyber capability risk ([FMF](https://www.frontiermodelforum.org/technical-reports/managing-advanced-cyber-risks-in-frontier-ai-frameworks/)).

---

## 5. Defenses that actually work (evidence-backed)

### 5.1 Architectural (strongest class)
- **CaMeL** (Google DeepMind, "Defeating Prompt Injections by Design"): splits the agent into a **privileged LLM** (never sees untrusted data) and a **quarantined LLM** (never triggers actions), with **capability tracking and provenance** enforced by a system-level interpreter — security holds *even if the model is fooled* ([arXiv 2503.18813](https://groups.google.com/g/cap-talk/c/YmZj1EhY4Uk/m/uwqSO0YTEgAJ), [Simon Willison's analysis](https://simonwillison.net/2025/Apr/11/camel/)).
- **Six agent design patterns** (Debenedetti et al.): Action-Selector, Plan-Then-Execute, Dual LLM, Tagged/Spotlighted inputs, Code-then-Execute, Contextual integrity — with code samples ([paper](https://arxiv.org/html/2506.08837v1), [Willison](https://simonwillison.net/2025/Jun/13/prompt-injection-design-patterns/), [ReversecLabs samples](https://github.com/ReversecLabs/design-patterns-for-securing-llm-agents-code-samples)).
- Guiding principle: *once an agent has ingested untrusted input, it must be constrained so it cannot cause unacceptable harm* ([HN discussion](https://news.ycombinator.com/item?id=44268335)).

### 5.2 Input/output hardening
- **Spotlighting** (Microsoft Research): delimiting, datamarking, encoding of untrusted spans — measurably improves injection resistance ([arXiv 2403.14720](https://arxiv.org/html/2403.14720v1), [Microsoft Learn](https://learn.microsoft.com/en-us/security/zero-trust/sfi/defend-indirect-prompt-injection)).
- **Instruction hierarchy** training and hardened system prompts (OpenAI/MSRC) — layered, not absolute.
- Guardrail stacks: **NeMo Guardrails** (programmable rails), **Llama Guard** (safety classifier), **Guardrails AI** (output validators), **LLM Guard** (fast input/output scanner) ([comparison](https://particula.tech/blog/ai-guardrails-compared-nemo-guardrails-ai-llama-guard), [Unit42 platform guardrail study](https://unit42.paloaltonetworks.com/comparing-llm-guardrails-across-genai-platforms/)). Benchmarks show all are bypassable → layer them, measure them.

### 5.3 Authorization & blast-radius reduction
- **Least privilege per tool** with short-lived, scoped credentials (OAuth scopes per tool, no ambient admin) — the confused-deputy fix is authorization design, not better prompts ([Quarkslab](https://blog.quarkslab.com/agentic-ai-the-confused-deputy-problem.html)).
- **Human-in-the-loop gates** for high-impact actions (payments, deletions, emails, code execution); even with HITL, expect a slip-through rate — design for *fewer* dangerous actions, not just approvals ([MDPI study](https://www.mdpi.com/2079-3197/14/5/98)).
- **Egress control** for agents: allow-list external destinations (this would have blunted EchoLeak/ShadowLeak).
- Sandbox tool execution (containers, no network by default) — MCP auto-exec incidents show why.

### 5.4 Data & retrieval integrity
- Provenance labels on documents; treat retrieved content as untrusted (spotlight it).
- Hybrid retrieval (BM25+vector), deduplication, source vetting, quarantine of user-contributed docs ([RAG poisoning defenses](https://arxiv.org/html/2603.18034v1)).

### 5.5 Red-teaming & evaluation (AISI-aligned)
- **garak** — static probe library of known LLM vulnerabilities; **PyRIT** — orchestrator-driven dynamic attacks; **promptfoo** — app-level CI red-teaming with YAML configs; **DeepTeam** — MITRE ATLAS-mapped agent attacks ([tool comparison](https://qawerk.com/blog/llm-red-teaming-tools/), [Promptfoo vs Garak](https://www.promptfoo.dev/blog/promptfoo-vs-garak/)).
- **AgentDojo / Inspect** for agentic utility+security measurement; **optstop** (AISI) for eval-efficiency science ([AISI Science of Evaluations](https://www.aisi.gov.uk/category/science-of-evaluations)).
- Ops: security telemetry from LLM apps into SIEM, anomaly detection on tool calls and cost, guardrail hit dashboards ([Datadog guardrail best practices](https://www.datadoghq.com/blog/llm-guardrails-best-practices/), [Elastic LLM observability](https://www.elastic.co/observability/llm-monitoring)).

### 5.6 Prompt-injection defense catalogs
- [tldrsec/prompt-injection-defenses](https://github.com/tldrsec/prompt-injection-defenses) — the most complete practical catalog: input preprocessing, blast-radius reduction, detection, training-based, system hardening.

---

## 6. Open problems (be honest about these)
1. Prompt injection has **no complete model-level fix**; architectural containment is the current best answer.
2. Detector-based defenses have poor recall and decay as attacks evolve.
3. Memory and RAG poisoning defenses are early-stage; provenance tracking is promising but rarely deployed.
4. Watermarking for extraction defense is provable only in narrow settings.
5. Agentic security moves faster than standards; expect OWASP ASI and NIST agentic profile churn.
6. Capability curves (AISI: ~4-month doubling) mean **security posture is perishable** — schedule re-evaluations, don't "certify once."

---

## 7. Full source list
(All links inline above; primary anchors:)
- OWASP GenAI Security Project — LLM Top 10 2025, Agentic Top 10 2026, incident round-ups: https://genai.owasp.org
- MITRE ATLAS: https://atlas.mitre.org
- NIST AI RMF / GenAI Profile: https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf
- ISO/IEC 42001: https://www.iso.org/standard/42001
- EU AI Act + GPAI Code of Practice: https://artificialintelligenceact.eu
- UK AISI Inspect: https://inspect.aisi.org.uk • https://github.com/UKGovernmentBEIS/inspect_ai • https://www.aisi.gov.uk
- AgentDojo: https://agentdojo.spylab.ai • https://arxiv.org/abs/2406.13352
- CaMeL: https://arxiv.org/abs/2503.18813 • Design patterns: https://arxiv.org/abs/2506.08837
- Spotlighting: https://arxiv.org/abs/2403.14720
- MCP security: https://modelcontextprotocol.io • NSA/CISA CSI • Invariant Labs
- Incident reports: EchoLeak (CVE-2025-32711), ShadowLeak (Radware), Unit 42, Malwarebytes
- Supply chain: ReversingLabs, JFrog, Sigstore model-transparency, OpenSSF OMS, CycloneDX ML-BOM
- Defense catalogs: https://github.com/tldrsec/prompt-injection-defenses
- Red-team tooling: garak, PyRIT, promptfoo, DeepTeam; comparisons via qawerk/promptfoo blogs
