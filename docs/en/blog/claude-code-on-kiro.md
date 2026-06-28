---
title: "I Taught Claude Code to Speak Kiro"
published: false
description: "Claude Code and Kiro run on the same Claude models. Here's how I wired them together with a small local translator so I stopped paying for two subscriptions."
tags: claude, ai, aws, productivity
canonical_url: https://github.com/coderhisham/kiro-gateway-next/blob/main/docs/en/blog/claude-code-on-kiro.md
cover_image: ""
---

> **TL;DR** — Claude Code sends its requests wherever one environment variable points. Aim that at a small local translator and it runs on the Kiro plan you already pay for. Full setup below, plus the two snags worth knowing about.

# I Taught Claude Code to Speak Kiro

Claude Code and Kiro have something funny in common: underneath, they're powered by the same Claude models. Same brain. They just grew up speaking different dialects, so out of the box they can't hold a conversation.

I noticed this right as I was about to start a second subscription for Claude Code. My Kiro plan was already renewing every month, already serving the exact models Claude Code wanted to charge me for again. Paying twice to talk to the same thing felt absurd.

So instead of buying a second seat, I hired an interpreter. One small program that sits between them, listens to Claude Code, and relays everything to Kiro in a dialect it understands. Here's how to set it up, and what I learned doing it.

---

## Why they can't just talk

Claude Code is more open-minded than people assume. It doesn't hard-code where it sends requests. It reads one environment variable, `ANTHROPIC_BASE_URL`, and ships everything to that address. Normally that's the official endpoint, but it'll happily send its requests anywhere you tell it to.

That's the opening. Point it somewhere local and the whole thing becomes possible.

## Meet the interpreter

The catch is that Claude Code and Kiro phrase things differently. You can't just redirect one at the other and expect them to understand each other. You need a translator fluent in both.

That's [kiro-gateway](https://github.com/coderhisham/kiro-gateway-next): a tiny proxy that runs on your own machine. A request arrives phrased one way and leaves phrased another:

```
Claude Code ──phrases it for Anthropic──▶ kiro-gateway ──rephrases it for Kiro──▶ your Kiro account
```

Claude Code gets a reply in the format it expects. Kiro receives a request it recognizes. The interpreter does the rephrasing in the middle, and the conversation just flows.

## Setting it up, step by step

Six steps. About five minutes. None of it is more complicated than a `git clone`.

**Step 1 — install Claude Code** (version 2.1.29 or newer):

```bash
npm install -g @anthropic-ai/claude-code
```

**Step 2 — bring in the interpreter and give it a clean workspace:**

```bash
git clone --depth=1 https://github.com/coderhisham/kiro-gateway-next ~/kiro-gateway
cd ~/kiro-gateway
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
```

**Step 3 — hand it your details.** Create `~/kiro-gateway/.env`:

```bash
PROXY_API_KEY="kiro-local-proxy-key"
KIRO_CLI_DB_FILE="/Users/<YOUR_USER>/Library/Application Support/kiro-cli/data.sqlite3"
SERVER_HOST="127.0.0.1"
SERVER_PORT="9000"
```

`PROXY_API_KEY` is a passphrase you pick yourself. `KIRO_CLI_DB_FILE` is where kiro-cli keeps your login on macOS.

**Step 4 — start it up:**

```bash
~/kiro-gateway/.venv/bin/python ~/kiro-gateway/main.py --port 9000 &
```

**Step 5 — make sure it's awake:**

```bash
curl http://127.0.0.1:9000/health
```

**Step 6 — tell Claude Code where to send things.** Create `~/.claude/settings.json`:

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://127.0.0.1:9000",
    "ANTHROPIC_API_KEY": "kiro-local-proxy-key",
    "ANTHROPIC_MODEL": "claude-sonnet-4-5"
  }
}
```

That key just has to match the passphrase from Step 3. It's local, and it never leaves your machine. Run `claude`, say yes when it asks about the key, and you're talking to Kiro.

> Worth knowing: if your Kiro plan includes it, switch the model to `claude-opus-4-8` and you get Opus 4.8 with a **1M-token context window**. I gave it a codebase I'd normally have to explain in pieces. It read the whole thing at once.

---

## The two snags nobody warns you about

It didn't go perfectly on the first try. Two things tripped me up, and both cost more time than the actual setup.

**Snag one — names that run too long.** I added a few MCP-heavy plugins and everything stalled with a blunt `400 Improperly formed request`. No explanation, just a wall.

The reason: Kiro won't accept a tool name longer than 64 characters, and MCP servers love generating mouthfuls like `mcp__github__check_if_a_repository_is_starred_by_the_authenticated_user`. Count it. It's over the line.

Keep tool names short. And if it's just the sheer number of tools bloating the request, the gateway has a relief valve:

```bash
AUTO_TRIM_PAYLOAD=true
```

**Snag two — a Rust build that fights Apple Silicon.** One AWS plugin pulls in a Rust dependency that tried to cross-compile for x86 on my M-series Mac and fell over with a missing `x86_64-apple-darwin` target. One line fixes it, then restart Claude Code:

```bash
rustup target add x86_64-apple-darwin
```

Neither snag is the gateway's fault. They're the usual friction of getting two ecosystems to cooperate. Tutorials skip the errors, so consider this the heads-up I wish I'd had.

---

## The one honest trade-off

I'm not going to pretend it's free of cost. There's a little latency. Your request takes a short detour through a local process and gets rephrased on the way, so it lands slightly slower than going direct. By the second hour I'd stopped noticing. If your work is timing-sensitive, run your own test before you rely on it.

And if anything misbehaves, turn on the logs:

```bash
DEBUG_MODE="errors"
```

It captures the full request and response for every failed call. That's how I tracked down the long-name issue instead of guessing.

---

## Was it worth an evening?

Here's the math that settled it. Two subscriptions for one set of models, every month, indefinitely. Or one quiet evening, once, plus a little grumbling at Rust.

I took the evening. The interpreter has been running in the background ever since, my bill hasn't moved, and Claude Code is perfectly happy talking to Kiro without ever realizing the difference.

If you already pay for Kiro and you've been eyeing Claude Code, this is the easiest yes on the menu. Worst case, you lose a night to a tool name that's three characters too long. Best case, you forget the gateway is even there and just enjoy not paying twice for the same Claude.

You don't need a second subscription. You need a translator.

---

*The interpreter is built on [jwadow's kiro-gateway](https://github.com/jwadow/kiro-gateway). AWS plugins come from [awslabs](https://github.com/awslabs/agent-plugins).*
