# PROSPECTOR — Per-language pattern tables

Companion to [`SKILL.md`](SKILL.md). Universal anti-patterns + the language-specific sink/dynamic-execution tables. **All patterns are documentation of vulnerabilities to *find and fix* — never copy into real code.**

## Universal (any language)

| Anti-pattern | Why it's dangerous | Control |
|---|---|---|
| Secret in prompt/template/config file | Prompts and configs leak; not a boundary | C01 |
| User text interpolated into system prompt unmarked | Injection merges instructions+data | C06/C08 |
| Model output → sink without validation | Model is attacker-reachable via RAG/web | C04 |
| Tool with shared/admin credential | Confused deputy | C02/C15 |
| Unpinned model/pkg/version | Supply chain implant | C05 |
| No output caps / no per-identity quota | DoW/DoS | C16 |
| Instruction-shaped text in agent config files | Agent poisoning (Rules File Backdoor) | C05 |

## Output sinks (where model output must never land unvalidated)

| Language | Critical sinks | Dangerous pattern shape |
|---|---|---|
| **Python** | `eval()`, `exec()`, `os.system()`, `subprocess.*(..., shell=True)`, `pickle.loads()`, `yaml.load()` (no Loader), `cursor.execute(query_string)` | `db.execute(f"SELECT … {llm_output}")` |
| **Node/JS/TS** | `eval()`, `new Function()`, `child_process.exec/spawn(shell option)`, `vm.runInContext`, `innerHTML/insertAdjacentHTML/dangerouslySetInnerHTML`, `document.write`, Mongo `$where`/`eval` | ``res.send(`<div>${llmOutput}</div>`)`` |
| **Go** | `os/exec.Command` with interpolated string, `text/template` (vs `html/template`), `reflect` invoked with model strings, cgo bridges | `exec.Command("sh", "-c", llmOutput)` |
| **Java/Kotlin** | `Runtime.exec`, `ProcessBuilder` string concat, `ScriptEngine.eval`, `ObjectInputStream.readObject`, JEXL/MVEL/OGNL, raw JDBC string concat | `engine.eval(llmOutput)` |
| **C#/.NET** | `BinaryFormatter.Deserialize`, `Process.Start` with shell, Roslyn `CSharpScript.EvaluateAsync`, Razor `@Html.Raw(model)`, `SqlCommand` concat | `Html.Raw(llmOutput)` |
| **PHP** | `eval`, `system/shell_exec/exec/passthru/backticks`, `unserialize` (with classes), `preg_replace /e`, raw SQL concat, `include $var` | `system("curl " . $llmUrl)` |
| **Ruby** | `eval`, `send`/`constantize` on model strings, `YAML.load` (vs `safe_load`), backticks/Open3 with concat, `ERB` without sanitization | `eval(llm_output)` |
| **Rust** | `std::process::Command` with shell string, `format!`-built SQL into query runners, unsafe FFI with model data (rare) | `Command::new("sh").arg("-c").arg(llm)` |

## Dynamic execution & deserialization quick table

| Language | Flag on sight |
|---|---|
| Python | `eval(`, `exec(`, `pickle.load`, `yaml.load(` without `SafeLoader`, `marshal.loads`, `subprocess` with `shell=True` |
| Node | `eval(`, `new Function(`, `child_process` with `shell: true` or command concat, `require(` of dynamic path from model output |
| Java | `readObject(`, `XMLDecoder`, `ScriptEngine`, `XStream` without security framework |
| .NET | `BinaryFormatter`, `TypeNameHandling.All` (Json.NET), `Process.Start` shell |
| PHP | `unserialize(` without `allowed_classes: false`, `eval(`, `assert(` |
| Ruby | `YAML.load` without safe flag, `eval`, `Marshal.load` on external data |
| Go/Rust | shell-outs with string interpolation; template injection via non-contextual engines |

## LLM-SDK call-shape fingerprints (helps SAST find the LLM surface)

| Stack | Markers that "this code calls a model" |
|---|---|
| Python | `openai.ChatCompletion`, `openai( )` client, `anthropic.Anthropic`, `langchain`, `litellm`, `transformers.pipeline`, `ollama` client, `requests.post(…/v1/chat/completions)` |
| Node/TS | `OpenAI(`, `@anthropic-ai/sdk`, `langchain`, `ai` SDK (`generateText`, `streamText`), `ollama`, `ChatCompletion` |
| Go | `langchaingo`, raw HTTP to `/v1/chat/completions`, `go-openai` |
| Java | `langchain4j`, `openai-java`, Spring AI `ChatClient` |
| .NET | `Azure.AI.OpenAI`, `Semantic Kernel`, `Microsoft.Extensions.AI` |
| Universal | env vars `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, base-URL strings for inference providers |

PROSPECTOR uses these fingerprints in CA-3 to locate the LLM call graph fast, in any of these stacks.

## Semgrep rule skeletons (drop into `.semgrep/` and adapt)

> These are *defensive detection rules* for finding unsafe patterns in your own code.

```yaml
# rule: model-output-to-shell (Python)
rules:
  - id: sentinel-llm-output-to-shell
    languages: [python]
    message: "LLM output flows to shell sink — OWASP LLM05 (C04). Validate/allow-list before use."
    severity: ERROR
    patterns:
      - pattern-either:
          - pattern: subprocess.$CMD($MODEL_OUTPUT, shell=True, ...)
          - pattern: os.system($MODEL_OUTPUT)
      - pattern-inside: |
          $RESP = $CLIENT.chat.completions.create(...)
          ...
```

```yaml
# rule: user-input-into-system-prompt (Python)
rules:
  - id: sentinel-llm-user-input-in-system-prompt
    languages: [python]
    message: "Untrusted input concatenated into system prompt (C06/C08). Spotlight/delimit and keep policy in code."
    severity: WARNING
    patterns:
      - pattern-either:
          - pattern: f"...{$USER_INPUT}..."
          - pattern: "...".format(...)
      - pattern-inside: |
          $MSGS = [{"role": "system", "content": ...}, ...]
          ...
```

```yaml
# rule: secret-in-agent-config (universal, regex class)
rules:
  - id: sentinel-secret-in-agent-config
    languages: [generic]
    message: "Possible credential in AI agent instruction file — these files are read by agents and committed (C01)."
    severity: ERROR
    paths:
      include:
        - "**/CLAUDE.md"
        - "**/AGENTS.md"
        - "**/.cursorrules"
        - "**/.cursor/rules/**"
        - "**/.github/copilot-instructions.md"
    patterns:
      - pattern-regex: "(?i)(api[_-]?key|secret|token|password)\\s*[:=]\\s*['\"]?[A-Za-z0-9_\\-]{12,}"
```

CI wiring (feeds C17): run `semgrep ci --config .semgrep/ gitleaks detect osv-scanner --recursive .` as a merge gate; findings map to control IDs via the `message:` field so PROSPECTOR/MARSHAL can ingest tool output directly.
