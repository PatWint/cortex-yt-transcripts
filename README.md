# Cortex YT transcripts — Claude connector

This connector gives Claude access to the Cortex YT transcript corpus — a private library of investing-podcast transcripts captured via the YT Transcript Downloader Chrome extension.

It's an MCP server hosted at `https://yt-transcript-api.alphacortex.dev/mcp/`. Once connected, Claude can search the corpus, pull a full transcript for one video, or list what channels are available — directly from any conversation.

**Access is invite-only.** You need a `CORTEX_YT_API_KEY` from the developer before any of the install flows below will work.

---

## What Claude gets

Three tools, all read-only:

| Tool | What it does |
|---|---|
| `fetch_transcript(video_id)` | Full transcript + metadata for one YT video (by 11-char ID). |
| `list_transcripts(query?, channel?, sort?, order?, date_from?, date_to?, limit?)` | List / filter / sort transcripts. Defaults return the 20 newest by publish date across all channels. Pass a `query` for text search, `channel` to restrict, `sort` / `order` to reorder, or `date_from` / `date_to` to window. |
| `list_channels()` | Every channel in the corpus with its transcript count. |

---

## Prerequisites

- A `CORTEX_YT_API_KEY` (handed over in person by the developer — not emailed, not in chat).
- Your Claude plan must support custom connectors. As of 2026-04-19:
  - **Claude Code**: any plan.
  - **claude.ai / desktop app**: Pro, Max, Team, or Enterprise.
  - **Messages API**: any API account.
  - **Free tier**: custom connectors not available.

---

## Install — by surface

Pick the surfaces you use. You can add the connector to multiple surfaces with the same key.

### 1. Claude Code

One-liner in your terminal. Replace `YOUR_KEY` with the key the developer gave you:

```bash
claude mcp add cortex-yt-transcripts \
  --transport http \
  --header "Authorization: Bearer YOUR_KEY" \
  -- https://yt-transcript-api.alphacortex.dev/mcp/
```

Restart Claude Code. The three tools appear under the `cortex-yt-transcripts` namespace.

### 2. claude.ai (web)

1. Open [claude.ai](https://claude.ai) → avatar (bottom-left) → **Settings** → **Connectors**.
2. Click **Add custom connector**.
3. Fill in:

   | Field | Value |
   |---|---|
   | Name | `Cortex YT transcripts` |
   | URL | `https://yt-transcript-api.alphacortex.dev/mcp/` ← **trailing slash required** |
   | Authentication | Bearer token |
   | Token | your `CORTEX_YT_API_KEY` |

4. **Save**, then toggle the connector **on** for the chats/projects where you want it.
5. Start a new chat — three tools show up.

### 3. Claude desktop app

Same UI as claude.ai web. Settings → Connectors → Add custom connector → paste URL + token.

Connectors you add in one place (web or desktop) sync to the other on the same account.

### 4. Claude mobile (iOS / Android)

Custom-connector support on mobile is rolling out gradually. If you don't see a Connectors section in the mobile app's settings yet, add the connector from claude.ai web — once Anthropic enables mobile for your account, it'll appear automatically.

### 5. Anthropic Messages API

Pass the server in the `mcp_servers` array of your request, with the beta header:

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: mcp-client-2025-11-20" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-opus-4-7",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "List the channels available in the Cortex corpus."}
    ],
    "mcp_servers": [
      {
        "type": "url",
        "url": "https://yt-transcript-api.alphacortex.dev/mcp/",
        "name": "cortex-yt-transcripts",
        "authorization_token": "YOUR_KEY"
      }
    ],
    "tools": [
      { "type": "mcp_toolset", "mcp_server_name": "cortex-yt-transcripts" }
    ]
  }'
```

Python / TypeScript / Go / Ruby / C# / Java / PHP SDKs all support the same shape — see [Anthropic's MCP connector docs](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector) for idiomatic snippets.

### 6. Other MCP-aware clients (Cursor, Zed, Roo, …)

Any client that supports remote MCP servers with streamable-HTTP transport can connect. Use:

| Field | Value |
|---|---|
| URL | `https://yt-transcript-api.alphacortex.dev/mcp/` |
| Transport | Streamable HTTP |
| Auth header | `Authorization: Bearer YOUR_KEY` |

Consult your client's docs for exactly where to paste these.

---

## Verify it's working

In a fresh chat, ask Claude:

> *"Using the Cortex transcript tools, list the top 5 channels by transcript count."*

Claude should invoke `list_channels` and return something like:

```
AlphaInsightsPod     — 54
MarketFrontiers      — 48
CapitalChronicles    — 42
PortfolioDigest      — 35
LongHorizonPod       — 28
```

If you see an authentication error, re-check the token value. If you see "tool not found" or the tools don't appear, the connector didn't wire up — toggle it off and on, or restart the client.

---

## Troubleshooting

**Trailing slash matters.** The URL must end in `/mcp/` (not `/mcp`). Without it, the server issues a redirect that drops the `Authorization` header, producing a spurious 401.

**401 on valid-looking requests.** If your token starts with `Bearer ` already, strip the prefix — some UIs add it for you.

**"Tool not found" in Messages API.** Make sure you included the beta header `anthropic-beta: mcp-client-2025-11-20` and an `mcp_toolset` entry in `tools` referencing the same `name` as the server.

**Slow first response.** The server sleeps after idle periods, so the first call in a fresh conversation can take a few seconds. Subsequent calls are fast.

