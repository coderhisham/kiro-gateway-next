# Using Claude Code with Kiro Gateway

This guide shows how to route [Claude Code](https://docs.anthropic.com/en/docs/claude-code) through your existing Kiro subscription using Kiro Gateway. No separate Anthropic API subscription is required — Claude Code speaks the standard Anthropic API protocol, and the gateway translates it to Kiro.

> 📖 Prefer the story version with the reasoning behind each step? Read the blog post: [*I Taught Claude Code to Speak Kiro*](blog/claude-code-on-kiro.md).

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
