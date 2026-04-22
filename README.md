# AMD Advancing AI Day 2026 OpenClaw Demo
## Social Showdown: The Live Leaderboard! Prizes are at stake!

A demonstration of **OpenClaw**, a distributed multi-agent orchestration framework, showcasing how to coordinate specialized AI agents for social media intelligence gathering. Built for AMD's Advancing AI Day 2026. Several OpenClaw agents will be deployed to track Social Media (Linkedin and Twitter/X) based on a hashtag, e.g #ROCmClaw, and display a live leaderboard.

---

## What is OpenClaw?

OpenClaw is a framework for defining, running, and coordinating multiple AI agents; each agent has its own identity, personality, capabilities, and isolated workspace. Rather than building one large monolithic agent, OpenClaw lets you compose specialized agents and route tasks to the right one automatically.

This demo uses social media monitoring as the use case: a master orchestrator receives requests and delegates them to specialist agents that know how to query Twitter/X or LinkedIn.

---

## Architecture

```
User Request
     │
     ▼
social-media-tracker  (orchestrator)
   ├── Twitter/X request? → twitter-x agent (Xpoz MCP)
   ├── LinkedIn request?  → linkedin agent (Apify CLI)
   └── Generic hashtag?   → both agents, results merged
```

### Agents

| Agent | Role | Tools |
|---|---|---|
| `social-media-tracker` | Master orchestrator - routes requests | `sessions_spawn` |
| `twitter-x` | Twitter/X specialist | Xpoz MCP server (`xpoz-social-search` skill) |
| `linkedin` | LinkedIn specialist | Apify CLI (`harvestapi/linkedin-post-search`) |

Each agent is defined through three Markdown files in its directory:

- **`SOUL.md`** - values, personality, and operating principles
- **`IDENTITY.md`** - role, name, and what the agent is
- **`AGENTS.md`** - task specifications and execution protocol

---

## How It Works

### 1. Configuration (`openclaw.json`)

The root `openclaw.json` wires everything together:

```json
{
  "gateway": { "mode": "local", "auth": { "token": "..." } },
  "mcp": {
    "servers": {
      "xpoz":  { "url": "https://mcp.xpoz.ai/mcp", ... },
      "apify": { "command": "npx @apify/actors-mcp-server", ... }
    }
  },
  "models": {
    "providers": {
      "vllm": { "baseUrl": "http://localhost:8001/v1" }
    }
  },
  "agents": { "list": [ ... ] }
}
```

- **Gateway**: Runs locally over WebSocket at `ws://127.0.0.1:18789`
- **MCP servers**: Xpoz (HTTP) for Twitter, Apify (CLI) for LinkedIn
- **Model**: Local vLLM inference running `google/gemma-4-31B-it`

### 2. Task Routing

The `social-media-tracker` agent reads the incoming request and routes it:

- Detects Twitter/X keywords → spawns a `twitter-x` session
- Detects LinkedIn keywords → spawns a `linkedin` session
- Generic hashtag (e.g. `#AMD`) → runs both in sequence, merges results

### 3. Twitter/X Agent

Uses the **Xpoz MCP server** to search Twitter by hashtag:

1. Calls `xpoz.getTwitterPostsByKeywords({ hashtag, fields, limit })`
2. Polls `xpoz.checkOperationStatus()` until results are ready
3. Exports data as CSV: `id, post, likes, retweets`

Workspace lives in `workspace-twitter-x/` with its own memory, config, and state.

### 4. LinkedIn Agent

Uses the **Apify CLI** to scrape LinkedIn posts:

```bash
apify call harvestapi/linkedin-post-search \
  --input '{"searchQueries": ["#AMD"], "maxPosts": 10, "sortBy": "date_posted"}'
```

Output is a JSON dataset fetched via `apify datasets get-items`, then formatted into a Markdown table and CSV.

### 5. Memory System

Each agent maintains a three-tier memory:

| Tier | File | Purpose |
|---|---|---|
| Daily | `memory/YYYY-MM-DD.md` | Raw session logs |
| Long-term | `MEMORY.md` | Curated, important facts |
| State | `.openclaw/workspace-state.json` | Setup completion flags |

---

## Project Structure

```
.
├── openclaw.json                        # Master configuration
├── subagent-twitter.txt                 # Twitter agent session transcript (demo)
├── agents/
│   ├── social-media-tracker/            # Orchestrator agent definition
│   │   ├── AGENTS.md                    # Routing logic
│   │   ├── IDENTITY.md
│   │   ├── SOUL.md
│   │   └── ai_tweets_12378.csv          # Sample output
│   └── linkedin/                        # LinkedIn agent definition
│       ├── AGENTS.md                    # Apify execution protocol
│       ├── IDENTITY.md
│       ├── SOUL.md
│       └── results/result_linkedin.csv  # Sample output
└── workspace-twitter-x/                   # Active twitter-x workspace
    ├── AGENTS.md                        # Workspace guidelines
    ├── IDENTITY.md / SOUL.md / USER.md
    ├── TOOLS.md                         # Available local tools
    ├── HEARTBEAT.md                     # Periodic health check template
    ├── config/mcporter.json             # MCP transport config (Xpoz)
    ├── memory/                          # Session memory files
    │   ├── 2026-04-22-ai-tweets.md
    │   ├── 2026-04-22-agentic-ai-tweets.md
    │   └── 2026-04-22-xpoz-setup.md
    ├── .openclaw/workspace-state.json   # Workspace state
    └── ai_tweets_*.csv                  # Generated tweet exports
```

---

## Getting Started

### Prerequisites

| Requirement | Notes |
|---|---|
| **OpenClaw** | Install the OpenClaw CLI - this is the runtime that loads `openclaw.json` and manages agents |
| **Node.js ≥ 18** | Required for `npx @apify/actors-mcp-server` |
| **Apify CLI** | `npm install -g apify-cli` - used by the LinkedIn agent |
| **vLLM** | A local inference server serving `google/gemma-4-31B-it` on port `8001` |
| **Xpoz account** | Sign up at [xpoz.ai](https://xpoz.ai) to get a Bearer token for Twitter/X search |
| **Apify account** | Sign up at [apify.com](https://apify.com) to get an API token |
| **LinkedIn session cookie** | A valid `li_at` session cookie from a logged-in LinkedIn browser session |

---

### 1. Clone the Repository

```bash
git clone https://github.com/your-org/amd-advancing-ai-day-2026-openclaw-demo.git
cd amd-advancing-ai-day-2026-openclaw-demo
```

---

### 2. Install OpenClaw

Follow the installation instructions in the [OpenClaw repository (multi-engine branch)](https://github.com/Mahdi-CV/openclaw-amd-sglang/tree/multi-engine).

---

### 3. Configure `openclaw.json`

Open `openclaw.json` and fill in the three placeholder values:

```json
"mcp": {
  "servers": {
    "xpoz": {
      "url": "https://mcp.xpoz.ai/mcp",
      "headers": {
        "Authorization": "Bearer <YOUR_XPOZ_TOKEN>"   // ← replace this
      }
    },
    "apify": {
      "env": {
        "APIFY_TOKEN": "<YOUR_APIFY_TOKEN>",           // ← replace this
        "LINKEDIN_COOKIE": "<YOUR_LI_AT_COOKIE>"       // ← replace this
      }
    }
  }
}
```

**Getting your LinkedIn cookie:**
1. Log into LinkedIn in Chrome/Firefox
2. Open DevTools → Application → Cookies → `www.linkedin.com`
3. Copy the value of the `li_at` cookie

---

### 4. Start the Local vLLM Inference Server on your AMD Dev Cloud GPU instance

The agents use `google/gemma-4-31B-it` served locally over the OpenAI-compatible API. Start vLLM on port `8001`:

```bash
export HF_TOKEN="your_huggingface_token_here"
vllm serve google/gemma-4-31B-it \
  --port 8001 \
  --api-key local-vllm
```

> AMD Instinct MI300X GPUs offer a whopping 192GB of VRAM. This model will load nicely on one MI300X GPU. Confirm whether the server is up at `http://localhost:8001/v1/models` before proceeding.

---

### 5. Authenticate with Xpoz (Twitter/X)

Xpoz uses OAuth 2.1 with PKCE, which requires a one-time browser-based authorization. When you first run the `twitter-x` agent it will print an authorization URL like:

```
https://mcp.xpoz.ai/oauth/authorize?response_type=code&...
```

1. Open that URL in a browser
2. Sign in with your Google account
3. Copy the authorization code returned by Xpoz
4. Paste it back into the OpenClaw TUI when prompted

The token is then saved into `workspace-twitter-x/config/mcporter.json` for subsequent runs.

---

### 6. Start OpenClaw

```bash
openclaw start
```

This reads `openclaw.json`, registers the three agents, connects the MCP servers, and opens the TUI on the local WebSocket gateway at `ws://127.0.0.1:18789`.

---

### 7. Run the Demo

In the OpenClaw TUI, select the `social-media-tracker` agent and send a message:

**Fetch tweets only:**
```
Fetch 10 recent tweets with #ROCmClaw
```

**Fetch LinkedIn posts only:**
```
Fetch the top 10 LinkedIn posts about #ROCmClaw from the last week
```

**Full leaderboard (both platforms):**
```
Fetch posts on #ROCmClaw
```

The orchestrator will route to the appropriate subagent(s), collect results, and return a merged report. Output CSVs land in `workspace-twitter-x/` and `agents/linkedin/results/`.

---

### 8. Placing Agent Definition Files

The agents look for their `SOUL.md`, `IDENTITY.md`, and `AGENTS.md` files at startup. The paths in `openclaw.json` are:

| Agent | Workspace path |
|---|---|
| `social-media-tracker` | `~/.openclaw/agents/social-media-tracker` |
| `linkedin` | `~/.openclaw/agents/linkedin` |
| `twitter-x` | `workspace-twitter-x/` (this repo directory) |

Copy the files from `agents/` into the corresponding workspace paths before running:

```bash
cp -r agents/social-media-tracker ~/.openclaw/agents/social-media-tracker
cp -r agents/linkedin ~/.openclaw/agents/linkedin
```

---

### Known Issues

| Issue | Cause | Workaround |
|---|---|---|
| Xpoz OAuth timeout | The 60-second PKCE flow times out in headless/server environments | Run the OAuth step from a desktop session with browser access |
| HTTP 429 from Xpoz | Rate limiting triggered by repeated auth retries | Wait ~60s before retrying after an OAuth failure |
| LinkedIn scraper requires cookie | Apify's `harvestapi/linkedin-post-search` needs an authenticated session | Refresh the `li_at` cookie if scraping stops returning results (cookies expire) |

---

## Key Design Decisions

**Markdown-defined agents** - Agent behavior is declared in plain Markdown files (`SOUL.md`, `AGENTS.md`, `IDENTITY.md`), not code. This makes agents human-readable, diff-friendly, and modifiable without touching any codebase.

**Workspace isolation** - Every agent gets its own directory with independent memory, config, and state. This prevents context pollution and enables parallel execution.

**MCP for tool integration** - External APIs (Xpoz, Apify) are exposed through the Model Context Protocol rather than embedded directly. Adding a new data source means registering a new MCP server, not changing agent code.

**Skill-based capability model** - Agents declare their skills (e.g. `xpoz-social-search`). The orchestrator reads these before delegating, enabling capability-aware routing.

**Local-first inference** - Default model is served via a local vLLM instance (`http://localhost:8001/v1`), keeping data on-premises. Remote models can be added as additional providers.

**Fine-grained tool permissions** - Each agent only receives access to what it needs (`group:fs`, `exec`, `mcp:apify`), with exec security level configurable per agent.

---

## Sample Output

**Twitter (via Xpoz):**

| id | post | likes | retweets |
|---|---|---|---|
| 1234567 | LLMs are evolving fast with #AgenticAI... | 42 | 8 |

**LinkedIn (via Apify):**

| Author | Post | Likes | Reposts |
|---|---|---|---|
| Jane Doe | Excited about what #AMD is doing in AI inference... | 341 | 27 |

---

## Next Steps 
A third Agent is coming up - it's task is to take info from the Twitter and Linkedin agents and display the results in a nice live dashboard. 

## Technologies

- **OpenClaw** - Agent orchestration framework
- **Model Context Protocol (MCP)** - Standardized tool integration layer
- **vLLM** - Local LLM inference server
- **Google Gemma-4-31B-it** - Primary language model
- **Xpoz** - Twitter/X social media search API (`https://mcp.xpoz.ai/mcp`)
- **Apify** - LinkedIn web scraping platform (`harvestapi/linkedin-post-search`)
- **mcporter** - CLI interface for calling MCP servers

---

## License

Apache 2.0 - see [LICENSE](LICENSE).
