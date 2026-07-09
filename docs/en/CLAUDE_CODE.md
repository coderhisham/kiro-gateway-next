# Using Claude Code with Kiro Gateway

This guide shows how to route [Claude Code](https://docs.anthropic.com/en/docs/claude-code) through your existing Kiro subscription using Kiro Gateway. No separate Anthropic API subscription is required — Claude Code speaks the standard Anthropic API protocol, and the gateway translates it to Kiro.

> 📖 Prefer the story version with the reasoning behind each step? Read the blog post: [_I Taught Claude Code to Speak Kiro_](blog/claude-code-on-kiro.md).

---

## How it works

Claude Code reads two environment variables to decide where to send requests:

- `ANTHROPIC_BASE_URL` — the API endpoint
- `ANTHROPIC_API_KEY` — the key sent with each request

Point `ANTHROPIC_BASE_URL` at a local Kiro Gateway instance and Claude Code believes it is talking to Anthropic. Requests are translated and routed through your Kiro account instead. The API key Claude Code sends is your gateway's `PROXY_API_KEY`, not a real Anthropic key.

```
Claude Code ──Anthropic protocol──▶ Kiro Gateway ──translated──▶ Kiro API
```

---

## Step 1: Install Claude Code

Requires Claude Code version 2.1.29 or later.

```bash
npm install -g @anthropic-ai/claude-code
```

## Step 2: Clone and set up Kiro Gateway

```bash
git clone --depth=1 https://github.com/coderhisham/kiro-gateway-next ~/kiro-gateway
cd ~/kiro-gateway
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

## Step 3: Configure Kiro Gateway

Create `~/kiro-gateway/.env`:

```bash
PROXY_API_KEY="kiro-local-proxy-key"
KIRO_CLI_DB_FILE="/Users/<YOUR_USER>/Library/Application Support/kiro-cli/data.sqlite3"
SERVER_HOST="127.0.0.1"
SERVER_PORT="9000"
```

Notes:

- `PROXY_API_KEY` is a secret you choose. Claude Code will send it as its API key.
- `KIRO_CLI_DB_FILE` points at your kiro-cli auth database. On macOS it lives under `~/Library/Application Support/kiro-cli/data.sqlite3`. Any of the gateway's [authentication options](../../README.md#%EF%B8%8F-configuration) work here.
- Binding to `127.0.0.1` keeps the gateway local-only.

## Step 4: Start Kiro Gateway

```bash
~/kiro-gateway/.venv/bin/python ~/kiro-gateway/main.py --port 9000 &
```

Confirm it is healthy:

```bash
curl http://127.0.0.1:9000/health
```

## Step 5: Point Claude Code at it

Create `~/.claude/settings.json`:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:9000",
    "ANTHROPIC_API_KEY": "kiro-local-proxy-key",
    "ANTHROPIC_MODEL": "claude-sonnet-4-5"
  }
}
```

- `ANTHROPIC_API_KEY` must match the `PROXY_API_KEY` from your `.env`.
- `ANTHROPIC_MODEL` is any model your Kiro account can access. This fork also supports **Claude Opus 4.8** with a 1M-token context window if your plan has access — set `"ANTHROPIC_MODEL": "claude-opus-4-8"`. See [Available Models](../../README.md#-available-models).

## Step 6: Run Claude Code

```bash
claude
```

On the first run, Claude Code asks whether to use the configured API key. Choose **Yes** — that is your local gateway key, and it never leaves your machine.

---

## Model switching and sub-agents

Model selection is a Claude Code decision, not a gateway one. Per Anthropic's docs, `ANTHROPIC_BASE_URL` only changes _where_ requests are sent, not _which_ model answers ([Model configuration](https://docs.anthropic.com/en/docs/claude-code/model-config)). Every request Claude Code sends carries its own `model` field, and the gateway resolves each one independently before translating it to Kiro. That means model switching and sub-agents already work through the gateway with no extra "mode" — you just point each model slot at a model your Kiro plan can serve.

> The gateway is a passthrough, not a gatekeeper. If you request a model your Kiro subscription doesn't include, Kiro rejects it with `INVALID_MODEL_ID` (surfaced as "Invalid model ID or insufficient subscription level to use it."). Pick models your plan actually grants — Sonnet and Haiku are broadly available; Opus is a premium tier.

### The three model slots

Claude Code uses up to three model "slots", each set by an environment variable in `~/.claude/settings.json`:

| Slot         | Env variable                 | Used for                                                               | Suggested Kiro model |
| ------------ | ---------------------------- | ---------------------------------------------------------------------- | -------------------- |
| Main         | `ANTHROPIC_MODEL`            | Primary conversation and tasks                                         | `claude-sonnet-5`    |
| Small / fast | `ANTHROPIC_SMALL_FAST_MODEL` | Background work: compaction, summarization, titles (defaults to Haiku) | `claude-haiku-4-5`   |
| Sub-agent    | `CLAUDE_CODE_SUBAGENT_MODEL` | Default model for spawned sub-agents                                   | `claude-sonnet-5`    |

Latest models (mid-2026): `claude-sonnet-5` is the newest Sonnet — available in Claude Code across all plans, with performance close to Opus 4.8 at lower cost ([Introducing Claude Sonnet 5](https://www.anthropic.com/news/claude-sonnet-5)). `claude-opus-4-8` is the newest Opus (premium tier), and `claude-haiku-4-5` is still the newest small/fast model. Use `claude-opus-4-8` for the main slot only if your Kiro plan includes Opus.

Names use Claude Code's dash format (e.g. `claude-sonnet-5`); the gateway normalizes them to Kiro format (`claude-sonnet-5`, `claude-sonnet-4.5`) and strips any date suffix, so both forms resolve. Whichever model you pick, your Kiro plan must actually serve it.

### Pin tier aliases to models your plan serves (important)

There is one non-obvious gotcha that causes `INVALID_MODEL_ID` **even when the model you want is on your plan**. Claude Code routes sub-agents and the Task tool by _tier alias_ (`opus`, `sonnet`, `haiku`) rather than by full model ID, and the Task tool only accepts those three keywords ([issue #27754](https://github.com/anthropics/claude-code/issues/27754)). Each alias resolves to whatever version Claude Code has wired up for it, which is frequently **not** the latest — for example `sonnet` may resolve to an older Sonnet, and `--model haiku` has even resolved to a Sonnet build ([issue #39701](https://github.com/anthropics/claude-code/issues/39701)).

The gateway forwards whatever resolved ID it receives (it's a passthrough, not a gatekeeper), so if the alias resolves to a version your Kiro plan doesn't include, Kiro rejects it. This is why a sub-agent can fail with "model not available" while your main conversation on the same tier works fine.

The fix is to pin what each alias resolves to, using the `ANTHROPIC_DEFAULT_*_MODEL` variables. Then every alias — main loop, sub-agents, the Task tool, `/model sonnet`, and `opusplan` — maps to a model your plan actually serves ([Model configuration](https://docs.anthropic.com/en/docs/claude-code/model-config)):

| Env variable                     | What it controls                    | Suggested value    |
| -------------------------------- | ----------------------------------- | ------------------ |
| `ANTHROPIC_DEFAULT_OPUS_MODEL`   | What the `opus` alias resolves to   | `claude-opus-4-8`  |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | What the `sonnet` alias resolves to | `claude-sonnet-5`  |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL`  | What the `haiku` alias resolves to  | `claude-haiku-4-5` |

Set these to the exact models your Kiro plan grants. Drop the Opus line (or point it at Sonnet) if your plan doesn't include Opus.

### Recommended `~/.claude/settings.json`

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:9000",
    "ANTHROPIC_API_KEY": "kiro-local-proxy-key",
    "ANTHROPIC_MODEL": "claude-sonnet-5",
    "ANTHROPIC_SMALL_FAST_MODEL": "claude-haiku-4-5",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-8",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-5",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5"
  }
}
```

`ANTHROPIC_API_KEY` must match `PROXY_API_KEY` from your gateway `.env`. These variables are read at launch, so restart Claude Code (or start a fresh session) after editing — a running `/model` switch won't reload them.

### Adaptive setup: default to Opus, let sub-agents follow

If you want Opus as the default and want sub-agents/sub-tasks to track whatever model you're on (switching manually with `/model` when needed), set the main model to Opus and **leave `CLAUDE_CODE_SUBAGENT_MODEL` unset**. With no sub-agent override, sub-agents inherit the main model, and the alias pins above keep any per-invocation alias valid:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:9000",
    "ANTHROPIC_API_KEY": "kiro-local-proxy-key",
    "ANTHROPIC_MODEL": "claude-opus-4-8",
    "ANTHROPIC_SMALL_FAST_MODEL": "claude-haiku-4-5",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "claude-opus-4-8",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "claude-sonnet-5",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "claude-haiku-4-5"
  }
}
```

Setting `CLAUDE_CODE_SUBAGENT_MODEL` to a fixed model instead **pins all sub-agents to that model** and overrides their frontmatter — reliable, but no longer adaptive. Use that only if the alias-pinning above still leaves a sub-agent hitting `INVALID_MODEL_ID` (a known Claude Code bug can make sub-agent alias resolution ignore `ANTHROPIC_DEFAULT_SONNET_MODEL` on some versions, [issue #73556](https://github.com/anthropics/claude-code/issues/73556)); in that case set `"CLAUDE_CODE_SUBAGENT_MODEL": "claude-sonnet-5"`.

This adaptive setup runs most work on Opus, which uses quota fastest. To keep specific workers cheap regardless of the main model, pin them per agent with `model: haiku` in their frontmatter; `ANTHROPIC_SMALL_FAST_MODEL` keeps background chores (compaction, titles) on Haiku either way.

### Switching models mid-session

Use `/model` inside a running session to switch without restarting ([Claude Code model configuration](https://support.claude.com/en/articles/11940350-claude-code-model-configuration)):

- `/model sonnet`, `/model haiku`, `/model opus` — switch the main model to that alias.
- `/model claude-sonnet-4-5` — switch by full model ID.
- `/model opusplan` — plan with Opus (in Plan mode) and execute with Sonnet. Skip this one if your plan doesn't include Opus, since the plan phase will hit `INVALID_MODEL_ID`.

Each switch simply changes the `model` value on subsequent requests, which the gateway resolves per request.

### Sub-agents

Sub-agents are separate Claude Code requests, each tagged with its own model, so they resolve through the gateway exactly like main-conversation requests. Sub-agent definitions are markdown files with YAML frontmatter in `.claude/agents/` (project) or `~/.claude/agents/` (user). The `model` field accepts a Claude Code alias (`sonnet`, `opus`, `haiku`, `fable`), a full model ID (e.g. `claude-opus-4-8`), or `inherit`, and defaults to `inherit` ([Create custom subagents](https://docs.claude.com/en/docs/claude-code/sub-agents)).

Example project sub-agent at `.claude/agents/code-reviewer.md`:

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices. Use proactively after changes.
tools: Read, Glob, Grep
model: claude-sonnet-4-5
---

You are a senior code reviewer. Analyze the diff and report issues by
priority (critical, warning, suggestion) with specific fixes.
```

Claude Code resolves a sub-agent's model in this order (first match wins): the `CLAUDE_CODE_SUBAGENT_MODEL` environment variable, the per-invocation model chosen by Claude, the sub-agent's `model` frontmatter, then the main conversation's model. Whatever it lands on must be a model your Kiro plan serves.

A common cost pattern is routing cheap, high-volume work (searching, log processing) to Haiku while keeping Sonnet for the main thread — set `model: claude-haiku-4-5` on those sub-agents.

### If you want the gateway to pick the model automatically

The gateway intentionally does not choose models for you (see the "gateway, not gatekeeper" principle in `AGENTS.md`). If you want automatic selection, route to Kiro's own `auto` model (exposed as the `auto-kiro` alias to avoid the Cursor IDE conflict) and let Kiro decide, rather than building selection logic into the proxy.

Content in this section was rephrased from Anthropic's documentation for compliance with licensing restrictions.

---

## Optional: AWS Agent Plugins

[Agent Plugins for AWS](https://github.com/awslabs/agent-plugins) are skill packs for Claude Code covering serverless, deployment, SageMaker, and more. They work through the gateway like any other Claude Code feature. Install them from inside Claude Code:

```text
/plugin marketplace add awslabs/agent-plugins
/plugin install deploy-on-aws@agent-plugins-for-aws
/plugin install aws-serverless@agent-plugins-for-aws
```

Restart Claude Code after installing. These plugins bundle skills, MCP servers, hooks, and reference docs that the model uses while working.

---

## Troubleshooting

### "Invalid model ID" / model not available on your plan

If a request fails with "Invalid model ID or insufficient subscription level to use it." (Kiro's `INVALID_MODEL_ID`), the resolved model isn't included in your Kiro subscription tier. This is an upstream entitlement rejection, not a gateway bug.

- **Most common with sub-agents:** if a sub-agent (or `opusplan`, or `/model sonnet`) fails while your main conversation on the same tier works — even though the model _is_ on your plan — a tier alias is resolving to a version you don't have. Pin the aliases with `ANTHROPIC_DEFAULT_*_MODEL`. See [Pin tier aliases to models your plan serves](#pin-tier-aliases-to-models-your-plan-serves-important).
- Switch to a model your plan serves: set `ANTHROPIC_MODEL` to a Sonnet or Haiku your plan grants, or use `/model sonnet` live. See [Model switching and sub-agents](#model-switching-and-sub-agents).
- Opus (including `opusplan`) is a premium tier many plans don't include — avoid it unless your plan grants Opus.
- To capture the exact rejected model ID and reason, enable debug logging with `DEBUG_MODE="errors"` and reproduce; details land in `debug_logs/`.
- Note: `GET /v1/models` won't reliably reflect your plan's entitlements, because the runtime endpoint doesn't expose `/ListAvailableModels` and the list falls back to the gateway's built-in set. Kiro is the final arbiter — the practical test is trying a model.

### `400 Improperly formed request` with many tools / MCP servers

MCP servers often generate long, descriptive tool names. The Kiro API enforces a hard **64-character limit** on tool names, and large tool sets can also push the request past Kiro's payload size limit. Either can surface as the vague `Improperly formed request` error.

What you can do:

- **Shorten tool names** so each stays at or under 64 characters. The gateway validates this and reports exactly which names are too long.
- **Enable payload auto-trim** if you run with 30+ tools. Add to `.env`:

  ```bash
  AUTO_TRIM_PAYLOAD=true
  ```

  This trims the oldest conversation history until the request fits, instead of failing outright. See the `PAYLOAD SIZE GUARD` section in [`.env.example`](../../.env.example).

### Responses feel slower than the Anthropic API

Requests take an extra local hop through the gateway and are translated to Kiro's format, so a small latency increase is expected. This is fine for everyday development and experimentation.

### `aws-iac-mcp` fails to build on Apple Silicon

If an MCP server that depends on a Rust-based Python package fails with a message about the `x86_64-apple-darwin` target not being installed, add the cross-compilation target and restart Claude Code:

```bash
rustup target add x86_64-apple-darwin
```

### Diagnosing other errors

Enable debug logging to capture full request/response details for failing calls:

```bash
DEBUG_MODE="errors"
```

Logs are written to the `debug_logs/` directory. See [Debugging](../../README.md#-debugging).

---

## Cost

No extra cost beyond your existing Kiro subscription. Claude Code uses the standard Anthropic protocol and the gateway translates it to Kiro.

## Credits

- [@jwadow](https://github.com/jwadow) for the original [kiro-gateway](https://github.com/jwadow/kiro-gateway)
- [awslabs](https://github.com/awslabs) for [Agent Plugins for AWS](https://github.com/awslabs/agent-plugins)
