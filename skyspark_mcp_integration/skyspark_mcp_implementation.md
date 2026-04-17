# SkySpark MCP Server — Implementation Guide

Based on the Skyforge MCP talk (SkyPosium) and the open-source Skyforge MCP project.

---

## Overview

This guide walks through setting up a local MCP server that exposes your SkySpark data and Axon functions as tools that any MCP-compatible AI client (Claude Desktop, ChatGPT, Cursor, etc.) can call. The server is written in Python, uses the official MCP Python SDK, and communicates with SkySpark via Phable (a Python Haystack client).

**Architecture:**

```
AI Client (Claude Desktop / ChatGPT)
        ↕  MCP protocol
MCP Server (Python, runs locally via uv or Docker)
        ↕  Haystack REST API (Phable)
SkySpark (your AWS server)
```

---

## Prerequisites

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) package manager (recommended)
- Docker (optional for now, required for AWS deployment later)
- SkySpark instance with API access
- An MCP-compatible AI client (Claude Desktop is the easiest starting point)

---

## Step 1 — Install uv

`uv` is a fast Python package manager that handles virtual environments automatically. The speaker uses it throughout.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Verify:

```bash
uv --version
```

---

## Step 2 — Clone the Skyforge MCP repo

```bash
git clone https://github.com/skyforge-labs/skyforge-mcp.git
cd skyforge-mcp
uv sync
```

`uv sync` reads the `pyproject.toml` and installs all dependencies into a managed virtual environment — no manual pip installs needed.

---

## Step 3 — Configure environment variables

Create a `.env` file in the project root:

```env
SKYSPARK_URI=http://your-skyspark-server:8080/api/your-project
SKYSPARK_USERNAME=your_username
SKYSPARK_PASSWORD=your_password
```

> **Security note:** For local development this is fine. When deploying to AWS, use environment variables injected at the container level or AWS Secrets Manager — never commit `.env` to source control.

---

## Step 4 — Start the MCP server

```bash
uv run main.py
```

This starts the server on `http://localhost:8000`. You should see output confirming it's running.

The server uses **stateless HTTP transport**, which means each request is independent — no persistent session state between AI calls.

---

## Step 5 — Start the MCP Inspector

The MCP Inspector is a browser-based UI that lets you browse and test your exposed tools before connecting a real AI client. It's invaluable for debugging.

**Option A — npx (no install required):**

```bash
npx @modelcontextprotocol/inspector
```

**Option B — clone and run:**

```bash
git clone https://github.com/modelcontextprotocol/inspector.git
cd inspector
npm install
npm run start
```

Both options start a proxy server and automatically open your browser. You'll get a URL with a session token.

In the Inspector UI:
- Change the transport to **Streamable HTTP**
- Set the URL to `http://localhost:8000/mcp`
- Click **Connect**

---

## Step 6 — Define tools in SkySpark

This is the key insight from the talk: **tools are defined in SkySpark itself as an Axon function**, not hardcoded in the Python server. The MCP server fetches them dynamically at runtime, so you can add or change tools without restarting the server.

In your SkySpark project, create an Axon function called `fetchMcpTools`:

```axon
fetchMcpTools: () => [
  {
    name: "listSites",
    dis: "List sites",
    help: "Returns all sites in the SkySpark project",
    view: "table"
  },
  {
    name: "listEquip",
    dis: "List equipment",
    help: "Returns all equipment for a given site",
    params: {
      siteRef: {
        type: "ref",
        help: "The site under which to list equipment"
      }
    },
    view: "table"
  },
  {
    name: "siteKpis",
    dis: "Site KPIs",
    help: "Returns key performance indicators for a given site",
    params: {
      siteRef: {
        type: "ref",
        help: "The site to retrieve KPIs for"
      }
    },
    view: "card"
  },
  {
    name: "siteEnergyHis",
    dis: "Site energy history",
    help: "Returns energy history data for a given site",
    params: {
      siteRef: {
        type: "ref",
        help: "The site to retrieve energy history for"
      }
    },
    view: "chart"
  }
]
```

**Tool schema rules:**
- `name` — the Axon function name the MCP server will call (required)
- `dis` — display name shown to the AI (required)
- `help` — description that tells the AI what this tool does (required — write this well)
- `params` — dict of input parameters if the tool takes arguments
- `view` — optional hint for UI rendering (`table`, `chart`, `card`)

For no-argument tools, `params` should be an empty list `[]`. For tools requiring a dict input, `params` should be a dict with keys describing each field.

---

## Step 7 — Implement the corresponding Axon functions

Each tool name in `fetchMcpTools` needs a matching Axon function in your project:

```axon
// listSites — no parameters
listSites: () => readAll(site)

// listEquip — takes a siteRef
listEquip: (siteRef) => readAll(equip and siteRef==siteRef)

// siteKpis — returns a summary dict
siteKpis: (siteRef) => do
  site: readById(siteRef)
  equips: readAll(equip and siteRef==siteRef)
  {
    siteName: site->dis,
    equipCount: equips.size(),
    // add your real KPI calculations here
  }
end

// siteEnergyHis — returns power history
siteEnergyHis: (siteRef) => do
  meters: readAll(meter and siteRef==siteRef)
  // return your actual history query here
  hisRead(meters.first(), yesterday()..today())
end
```

These are bare-bones examples. Replace with your actual SkySpark logic — site summaries, fault counts, energy KPIs, whatever you already have built.

---

## Step 8 — How the server handles a tool call

When an AI client calls a tool, here is what happens under the hood:

1. AI client asks: "what tools do you have?"
2. MCP server calls `fetchMcpTools()` on SkySpark via Phable → gets the tool list back
3. AI picks the right tool and sends inputs as structured JSON
4. MCP server receives the call, looks up the tool name, calls that Axon function on SkySpark with the provided inputs via Phable's `eval` endpoint
5. SkySpark evaluates the Axon, returns a result
6. MCP server packages the result and sends it back to the AI client

The `handle_tool_call` method in the Python server is where step 4 happens — it's essentially a dispatcher that takes the tool name and inputs, evaluates the Axon, and returns the result.

---

## Step 9 — Connect Claude Desktop (local testing)

Claude Desktop natively supports MCP servers. Edit its config file:

**macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "skyspark": {
      "command": "uv",
      "args": ["run", "main.py"],
      "cwd": "/path/to/skyforge-mcp",
      "env": {
        "SKYSPARK_URI": "http://your-skyspark-server:8080/api/your-project",
        "SKYSPARK_USERNAME": "your_username",
        "SKYSPARK_PASSWORD": "your_password"
      }
    }
  }
}
```

Restart Claude Desktop. You should see a tools icon in the chat input indicating the MCP server is connected. You can then ask things like:

- "What sites do I have?"
- "Show me equipment for the Northside Office site"
- "What are the KPIs for site X?"

---

## Step 10 — Expose localhost to the internet (for ChatGPT / web clients)

If you want to test with ChatGPT or any web-based client (not Claude Desktop), your `localhost:8000` needs to be reachable from the internet. The speaker used **ngrok** for this:

```bash
# Install ngrok from https://ngrok.com
ngrok http 8000
```

This gives you a temporary public URL like `https://abc123.ngrok-free.app`. Use that URL in place of `localhost:8000` when configuring your web-based AI client.

> This is for development only. For production on AWS, the server will have a real public or VPC-internal endpoint.

---

## Adding tools for MBCx recommendations

Once the basic connection is working, here are MBCx-specific tools worth adding to `fetchMcpTools`:

```axon
// Active faults for a site
{
  name: "activeFaults",
  dis: "Active faults",
  help: "Returns all active faults for a site, sorted by estimated annual savings",
  params: {siteRef: {type: "ref", help: "The site to query"}},
  view: "table"
},

// Fault detail with context for AI analysis
{
  name: "faultDetail",
  dis: "Fault detail",
  help: "Returns detailed data for a specific fault including equipment history and context",
  params: {faultId: {type: "ref", help: "The fault record to retrieve"}},
  view: "card"
},

// Site energy summary
{
  name: "energySummary",
  dis: "Energy summary",
  help: "Returns energy consumption summary and benchmarking data for a site",
  params: {siteRef: {type: "ref", help: "The site to summarize"}},
  view: "card"
}
```

With these in place, an operator can ask Claude: *"What are the most expensive active faults at Northside Office and what should I do about them?"* — and Claude can call `activeFaults`, then `faultDetail` on the top results, and synthesize a recommendation from real data.

---

## Docker setup (for later AWS deployment)

The repo includes a `docker-compose.yml`. When you're ready to deploy:

```bash
# Build and start
docker-compose up --build

# View logs
docker-compose logs -f

# Restart
docker-compose restart
```

The `.env` file is read by Docker Compose automatically. On AWS, replace `.env` with environment variables set on the container task definition or injected from Secrets Manager.

---

## Key things to know from the talk

**Tools are your SkySpark Axon functions** — the MCP server is just a thin bridge. All the real logic stays in SkySpark where you already know how to build it.

**The `help` string matters a lot** — this is what the AI reads to decide which tool to call. Write it as if you're explaining to a colleague what the function does and when to use it.

**Write actions need confirmation** — any tool that commits data to SkySpark should be flagged appropriately. The speaker showed ChatGPT presenting a confirmation form before the `newSite` tool committed anything. Claude Desktop handles this similarly.

**MCP vs custom agent** — as the speaker noted, MCP is not the only answer. For structured dashboard recommendations, a direct API call from SkySpark (the `dchttp` approach) is simpler. MCP shines when you want operators to ask freeform questions and have the AI decide which data to pull. Both are worth having.

---

## Useful links

- [Skyforge MCP repo](https://github.com/skyforge-labs/skyforge-mcp) — the open source project from the talk
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) — official SDK with copy-paste examples for everything
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector) — browser UI for testing your server
- [Phable](https://github.com/j2inn/pyhaystack) — Python Haystack client used by the server
- [uv docs](https://docs.astral.sh/uv/) — package manager used throughout
- [ngrok](https://ngrok.com) — expose localhost to the internet for web client testing
