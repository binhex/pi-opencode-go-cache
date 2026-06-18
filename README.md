# pi-opencode-go-cache

Brings OpenCode CLI–equivalent prompt caching to Pi for the
[OpenCode Go](https://opencode.ai/docs/go) provider.

**Source / issues:** [github.com/nnocte/pi-opencode-go-cache](https://github.com/nnocte/pi-opencode-go-cache)

## How it works

The extension hooks Pi's `before_provider_request` event, which fires after
the provider has built its API payload but before the HTTP request is
sent. It mutates the payload in place:

```
   ┌──────────────────────────────────────────────────────────┐
   │  Pi built this payload (e.g. openai-completions body)    │
   │                                                          │
   │  { model, messages: [...], tools: [...], stream, ... }   │
   └──────────────────────────────────────────────────────────┘
                                │
                                ▼
   ┌──────────────────────────────────────────────────────────┐
   │  before_provider_request handler runs                    │
   │                                                          │
   │  1. Skip if provider !== "opencode-go"                   │
   │  2. Strip any stale cache_control from previous turns    │
   │  3. payload.prompt_cache_key       = session id          │
   │  4. payload.prompt_cache_retention  = "24h"              │
   │  5. Stamp cache_control on system + last 2 user/assistant│
   │     messages + last tool (matches OpenCode CLI's 2+2+1)  │
   │  6. Show "cache: <api>" in the TUI footer                │
   └──────────────────────────────────────────────────────────┘
                                │
                                ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Outgoing HTTP request to https://opencode.ai/zen/go/v1  │
   │                                                          │
   │  • System prompt and stable prefix → cache hit           │
   │  • Cache reads are 5–120× cheaper than input             │
   └──────────────────────────────────────────────────────────┘
```

The cache_control markers create **multiple breakpoints** in the
conversation, so the cache stays useful as the conversation grows:

```
   Turn 1                    Turn 2                    Turn 3
   ──────                    ──────                    ──────
   ┌─ system ─✦  ◀── new    ┌─ system ─✦   ◀── hit  ┌─ system ─✦     ◀── hit
   │  user              │    │  user         ◀── hit  │  user          ◀── hit
   └────────────────────┘    │  assistant ✦  ◀── new │  assistant ✦   ◀── hit
                             │  user              │   │  user           ◀── hit
                             └────────────────────┘   │  assistant ✦   ◀── new
                                                      │  user              │
                                                      └────────────────────┘

   ✦ = cache_control breakpoint
   ◀── hit = prefix matched and read from cache (cheap)
   ◀── new = freshly written (one-time, usually free on opencode-go)
```

Each marker tells the gateway "cache everything up to and including this
point". On a long session, the system prompt and earlier turns stay
cached and pay 5–120× less per token than the input price.

## Problem

The OpenCode Go gateway (`opencode.ai/zen/go`) caches the request prefix
automatically, but only with:

- a short ~5 min TTL
- no per-session cache key
- no explicit `cache_control` breakpoints

Pi's built-in `openai-completions` provider never sets `prompt_cache_key`
or `prompt_cache_retention` for `opencode-go`, and adds at most one
`cache_control` marker — so you pay full input price on every call and
lose the cache between long pauses.

## What it does

Hooks `before_provider_request` and, for any `opencode-go/*` model, sets:

- `prompt_cache_key` = clamped session id (so the cache is scoped per-Pi-session)
- `prompt_cache_retention: "24h"` (default is ~5 min)
- `cache_control: {type:"ephemeral", ttl:"1h"}` markers on the system
  prompt, last tool, and last 2 user/assistant messages — matching what
  OpenCode CLI does

Stale markers from previous turns are stripped before re-stamping, so
breakpoints stay correct across the conversation. A compact `cache: <api>`
indicator shows up in the TUI footer so you can confirm it's active.

## Install

### Recommended (npm)


```bash
pi install npm:pi-opencode-go-cache
```

### From GitHub


```bash
pi install git:github.com/nnocte/pi-opencode-go-cache
```

### One-off run (no install)

```bash
pi -e npm:pi-opencode-go-cache
pi -e git:github.com/nnocte/pi-opencode-go-cache
```

## What you save

Every model on the OpenCode Go subscription benefits. Cache-read prices
are 5–120× cheaper than input:

| Model              | API                | Cache ratio | cacheWrite cost |
| ------------------ | ------------------ | ----------- | --------------- |
| deepseek-v4-pro    | openai-completions | 120×        | free            |
| deepseek-v4-flash  | openai-completions | 50×         | free            |
| mimo-v2.5-pro      | openai-completions | 120×        | free            |
| mimo-v2.5          | openai-completions | 50×         | free            |
| qwen3.6-plus       | openai-completions | 10×         | $0.625/M        |
| qwen3.7-plus       | anthropic-messages | 10×         | $0.50/M         |
| qwen3.7-max        | anthropic-messages | 5×          | $3.125/M        |
| kimi-k2.7-code     | openai-completions | 5×          | free            |
| kimi-k2.6          | openai-completions | 5.9×        | free            |
| minimax-m3         | anthropic-messages | 5×          | free            |
| minimax-m2.7       | openai-completions | 5×          | free            |
| glm-5.1            | openai-completions | 5.4×        | free            |
| glm-5              | openai-completions | 5×          | free            |

On a long coding session, the deepseek and mimo models see roughly
80–95 % off the input bill after the first call.

## Why this and not `PI_CACHE_RETENTION=long`?

Setting `PI_CACHE_RETENTION=long` only does two of the three things
OpenCode CLI does to get cheap cache hits on opencode-go:

|                                                                    | `PI_CACHE_RETENTION=long`   | OpenCode CLI  | opencode-go-cache |
| ------------------------------------------------------------------ | :-----------------------:   | :----------:  |  :------------:   |
| `prompt_cache_retention: "24h"` (vs. ~5 min default)               |             ✅             |      ✅       |        ✅        |
| `prompt_cache_key` (per-session, not opportunistic)                |             ✅             |      ✅       |        ✅        |
| `cache_control` markers on system + last 2 messages + last tool    |             ❌             |      ✅       |        ✅        |
| Works for `anthropic-messages` models too (qwen, minimax)          |             ❌             |      ✅       |        ✅        |
| Single source of truth (no env vars, no `models.json` overrides)   |             ❌             |      ✅       |        ✅        |

Pi's `openai-completions` provider already drops `cache_control` markers
for `openai-completions` when `cacheControlFormat: "anthropic"` is set in
`~/.pi/agent/models.json` — but only on the system prompt + last user
message (1 breakpoint) and only for that one API. This extension does the
full OpenCode CLI recipe for every opencode-go model in one place, so
there's nothing else to configure.

## Verification

Tested live against the real gateway. Every one of the 13 opencode-go
models registered in Pi gets `prompt_cache_key=set | retention=24h |
cache_control markers=2–3` in the payload right before the request is
sent.

## Uninstall

```bash
pi remove npm:pi-opencode-go-cache
# or
pi remove git:github.com/nnocte/pi-opencode-go-cache
```
