# OpenClaw × Google Gemini 2.5 Flash Setup Guide

A step-by-step recipe for wiring [OpenClaw](https://www.npmjs.com/package/openclaw) to **Google Gemini 2.5 Flash** via Google's native Generative Language API. Covers the working configuration, the `gemini-proxy` companion plugin (and when you actually need it), provider fallback chains, and the gotchas worth knowing before you start.

> This is a guide repo, not a fork. It documents the configuration that successfully runs Gemini 2.5 Flash inside OpenClaw on Windows 11 — same approach works on macOS and Linux.

---

## Why Gemini 2.5 Flash for OpenClaw?

OpenClaw routes every agent turn through a configured provider. Gemini 2.5 Flash is a strong fit because:

- **Big context** — 1M token window means the full agent persona, tools catalog, and conversation history all fit comfortably without compaction
- **Cheap** — Flash pricing is significantly under Pro tier, fine for high-frequency agent loops
- **Tool calling works cleanly** — no thinking-tag stripping or chat-template fragility you hit with self-hosted Qwen3 / DeepSeek
- **Fast first token** — sub-second cold start for short prompts, ideal for chat-style TUI sessions

It's also a great fallback layer behind a local llama-server when you want zero-cost first-pass and remote handoff for hard turns.

---

## Surface comparison

Google exposes Gemini via three surfaces. OpenClaw supports all three; this guide uses option 1.

| Surface | Endpoint | Auth | Cost | OpenClaw `api` field |
|---|---|---|---|---|
| **Generative Language (native)** | `generativelanguage.googleapis.com` | API key | Free quota + paid | `google-generative-ai` ← **this guide** |
| Generative Language (OpenAI-compat) | `generativelanguage.googleapis.com/v1beta/openai` | API key | Same | `openai-completions` (needs `gemini-proxy` plugin) |
| Cloud Code Assist | `cloudcode-pa.googleapis.com` | gemini-cli OAuth | Free tier | Not directly supported in OpenClaw — use the native path with a Studio key |

The native path is the simplest and most reliable for OpenClaw because OpenClaw has a first-class `google-generative-ai` adapter that handles streaming, tool calls, and reasoning blocks correctly without any compatibility shims.

---

## Prerequisites

1. **Node.js 20+** — OpenClaw runs on Node
2. **OpenClaw installed** — `npm i -g openclaw` (verify with `openclaw --version`)
3. **A Google AI Studio API key** — free to mint at <https://aistudio.google.com/app/apikey>
4. **Initialized OpenClaw config** — run `openclaw configure` once if you've never set it up; this creates `~/.openclaw/openclaw.json`

---

## TL;DR (for the impatient)

Add this provider block to `~/.openclaw/openclaw.json` under `models.providers`:

```json
"gemini-api": {
  "api": "google-generative-ai",
  "baseUrl": "https://generativelanguage.googleapis.com",
  "apiKey": "YOUR_GOOGLE_AI_STUDIO_KEY",
  "models": [
    { "id": "gemini-2.5-flash", "name": "Gemini 2.5 Flash" }
  ]
}
```

Then in your TUI run:

```
/model gemini-api/gemini-2.5-flash
```

You're done. Skip the rest if you don't need fallback chains, provider routing, or an explanation of the `gemini-proxy` plugin.

---

## Full setup

### 1. Mint a Google AI Studio key

Go to <https://aistudio.google.com/app/apikey> → **Create API key** → choose or create a Google Cloud project (a free default one is fine) → copy the key.

The key starts with `AIza`. Treat it as a secret; never commit it.

### 2. Wire it into OpenClaw

Edit `~/.openclaw/openclaw.json`. Under the top-level `models.providers` map, add:

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "gemini-api": {
        "api": "google-generative-ai",
        "baseUrl": "https://generativelanguage.googleapis.com",
        "apiKey": "AIza...your_key_here...",
        "models": [
          {
            "id": "gemini-2.5-flash",
            "name": "Gemini 2.5 Flash"
          }
        ]
      }
    }
  }
}
```

**Field notes:**

- `api: "google-generative-ai"` — tells OpenClaw to use the native Gemini adapter (not OpenAI-compat). This is the part most people get wrong.
- `baseUrl` — Google's public Generative Language endpoint. No proxy required.
- `apiKey` — your Studio key. OpenClaw also reads `GEMINI_API_KEY` from the environment if you'd rather not put it in the file; in that case set `"apiKey": ""` or omit the field and export the env var.
- `models[].id` — must match a real Gemini model id. As of this writing: `gemini-2.5-flash`, `gemini-2.5-flash-lite`, `gemini-2.5-pro`, `gemini-2.0-flash`.

### 3. Pick where Gemini fits in your model lineup

Three common patterns:

**Pattern A — Gemini as primary**

```json
"agents": {
  "defaults": {
    "model": {
      "primary": "gemini-api/gemini-2.5-flash",
      "fallbacks": ["gemini-api/gemini-2.5-pro"]
    }
  }
}
```

Pure cloud, zero local dependencies. Fastest to set up.

**Pattern B — Local primary, Gemini fallback**

```json
"agents": {
  "defaults": {
    "model": {
      "primary": "llama-server/your-local-model",
      "fallbacks": [
        "gemini-api/gemini-2.5-flash",
        "llama-server/your-other-local-model"
      ]
    }
  }
}
```

Free first pass on a self-hosted model; Gemini takes over when llama-server times out or hits a context overflow. This is what produced the working synthshark setup that this guide came out of.

**Pattern C — Per-agent override**

```json
"agents": {
  "list": [
    {
      "id": "synthshark",
      "model": {
        "primary": "gemini-api/gemini-2.5-flash",
        "fallbacks": ["llama-server/local-fallback"]
      }
    }
  ]
}
```

Different agents, different routing. The agent-level `model` block overrides `defaults`.

### 4. Restart and test

```bash
openclaw restart    # or whatever restart command your install uses
openclaw            # launch the TUI
```

In the TUI, switch to Gemini:

```
/model gemini-api/gemini-2.5-flash
```

Send any message. First token should arrive in under a second. The status bar shows `gemini-api/gemini-2.5-flash` and the token meter starts ticking against your Gemini quota.

---

## The `gemini-proxy` plugin — when do you need it?

OpenClaw ships with a bundled plugin called `gemini-proxy` (look in `~/.openclaw/extensions/gemini-proxy/`). It listens on `127.0.0.1:9999` and strips fields like `store`, `metadata`, and `user` from outgoing requests.

**You need it only if** you're using Gemini's **OpenAI-compatible** endpoint with `api: "openai-completions"` — Gemini's OpenAI gateway rejects those fields and OpenClaw sends them by default.

**You do NOT need it** when using the native `api: "google-generative-ai"` path described in this guide. The native adapter never sends those fields.

If your OpenClaw startup logs show `[gemini-proxy] stripping store, metadata, user on :9999` and you're only ever using the native path, you can safely disable it:

```bash
# edit ~/.openclaw/extensions/gemini-proxy/openclaw.plugin.json
# change: "enabledByDefault": true → "enabledByDefault": false
```

The line disappears from boot, and you reclaim port 9999.

---

## Verifying it works

A clean response from Gemini in the TUI looks like:

```
🎹🦈 Test confirmed — systems green, running on the full identity. What's up?
 connected | idle
 agent <yours> | session tui-... | gemini-api/gemini-2.5-flash | tokens 3.5k/1.0m
```

Look for:

- **Sub-second first token** — Gemini Flash is fast on short prompts
- **Context meter shows `/1.0m`** — confirms the 1M-token window is being respected
- **Provider field reads `gemini-api/gemini-2.5-flash`** — not a fallback model

If the provider field shows your fallback instead, something's wrong with the Gemini path. Check the gateway log:

```bash
tail -50 ~/.openclaw/logs/gateway-service.log | grep -i gemini
```

Common errors:

- `403 PERMISSION_DENIED` — key invalid or restricted to a different API
- `429 RESOURCE_EXHAUSTED` — free quota burned for the day; either wait, switch project, or enable billing
- `model not found` — typo in `models[].id`; it must match a real Gemini model id exactly

---

## Cost notes

Gemini 2.5 Flash pricing as of writing (check <https://ai.google.dev/pricing> for current rates):

- Input: ~$0.075 per 1M tokens
- Output: ~$0.30 per 1M tokens
- Free tier: generous daily quota for personal use, but rate-limited

For an agent that holds a fat ~25k-token system prompt across hundreds of turns per day, expect single-digit dollars per month at most. If your system prompt is leaner (a couple thousand tokens), you'll likely stay inside the free tier.

The OpenClaw token meter tracks usage per session. Watch the per-turn input cost — if your system prompt grows large (workspace markdown auto-injection, big tool catalog), you pay it on every turn. Slimming workspace files often pays for itself fast.

---

## Troubleshooting

**Gemini works but my agent's persona doesn't show up**
The `AGENTS.md` file in your agent's workspace is the only file OpenClaw auto-injects into the system prompt. Other workspace `.md` files (`BOOTSTRAP.md`, `IDENTITY.md`, etc.) are not auto-loaded — they're either referenced from `AGENTS.md` or read on demand. Inline what you need into `AGENTS.md`.

**Responses come back as `NO_REPLY`**
This is OpenClaw's silent-reply token, not a Gemini issue. It happens when group activation is set to `"always"` and the model decides no response is warranted. Either flip `requireMention: true` for the channel or add an explicit override in your `AGENTS.md`. Common when the upstream message is wrapped with `Sender (untrusted metadata)` and the model interprets it as bot spam.

**Gemini ignores formatting rules after a few turns**
Gemini occasionally drops mandatory prefix/suffix instructions. Re-state the rule once mid-conversation and it usually re-locks. Or pin the rule in `AGENTS.md` at the top, not buried mid-document.

**Streaming cuts off mid-response**
Check `~/.openclaw/logs/gateway-service.log` for `Connection error` — usually a network blip or hitting a per-minute rate limit. OpenClaw's failover kicks in automatically if you have a fallback configured.

**The `gemini-proxy` plugin keeps trying to bind port 9999**
You probably never use the OpenAI-compat path. Disable the plugin (see the section above).

---

## Background: why the native API path?

OpenClaw treats `google-generative-ai` as a first-class API type alongside `openai-completions`, `anthropic-messages`, and `ollama`. The native adapter:

- Handles Gemini's request/response schema directly (no field translation overhead)
- Properly maps Gemini's `parts[]` content blocks to OpenClaw's internal message format
- Streams via SSE the way Gemini natively delivers it
- Supports Gemini-specific features like inline reasoning blocks without compatibility shims

The OpenAI-compat path (`generativelanguage.googleapis.com/v1beta/openai`) exists for clients that can't speak native Gemini, but it loses fidelity on tool calls, streaming chunk boundaries, and thinking-mode metadata. Use it only if you have to.

---

## License

MIT.
