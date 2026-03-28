# StatLens — Complete Project Guide

> **Last updated:** 2026-03-26  
> This document explains the entire StatLens project: what it is, how every component works, how they connect together, and the exact steps to start everything from scratch.

---

## Table of Contents

1. [What is StatLens?](#1-what-is-statlens)
2. [Architecture Overview](#2-architecture-overview)
3. [Component Breakdown](#3-component-breakdown)
   - 3.1 [PostgreSQL Database](#31-postgresql-database)
   - 3.2 [Backend API Server (Port 8012)](#32-backend-api-server-port-8012)
   - 3.3 [MCP Server (Model Context Protocol)](#33-mcp-server-model-context-protocol)
   - 3.4 [ChatGPT Widget (Port 5173)](#34-chatgpt-widget-port-5173)
   - 3.5 [Cloudflare Tunnels](#35-cloudflare-tunnels)
4. [How Data Flows End-to-End](#4-how-data-flows-end-to-end)
5. [The .env File — Every Variable Explained](#5-the-env-file--every-variable-explained)
6. [How to Start the Project (Step by Step)](#6-how-to-start-the-project-step-by-step)
7. [CLI Commands Reference](#7-cli-commands-reference)
8. [Troubleshooting Common Issues](#8-troubleshooting-common-issues)

---

## 1. What is StatLens?

StatLens is a **statistical data aggregation and search platform** that connects to 11+ government and international data sources (BEA, BLS, CDC, Census, WHO, World Bank, OECD, StatCan, UNDATA, NCSES, BTS) and provides:

- A **unified search interface** — ask questions in natural language like "US unemployment rate" and get matching time-series data from all sources
- An **LLM-powered query engine** — decomposes complex questions into sub-queries, searches the catalog, resolves parameters, and fetches real observations
- An **MCP server** — exposes `search()`, `fetch()`, and `open_comparison_dashboard()` tools that ChatGPT can call directly
- A **React comparison widget** — renders interactive trend charts inside ChatGPT conversations

**In simple terms:** A user types "unemployment rate" into ChatGPT → ChatGPT calls StatLens MCP tools → StatLens searches its catalog of thousands of government datasets → returns matching series → the widget renders a comparison chart, all within the ChatGPT UI.

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CHATGPT (chatgpt.com)                        │
│                                                                      │
│  User types: "unemployment rate"                                     │
│       ↓                                                              │
│  ChatGPT calls MCP tool: search({intents: [...]})                    │
│       ↓                                                              │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Cloudflare Tunnel (API)                                       │  │
│  │  https://heritage-zinc-programmes-shoulder.trycloudflare.com   │  │
│  │       ↓                                                        │  │
│  │  StatLens API Server (port 8012)                               │  │
│  │  ├── /mcp         → MCP tools (search, fetch, dashboard)      │  │
│  │  ├── /api/v1/ask  → NLWeb query endpoint                      │  │
│  │  ├── /graphql     → GraphQL API                                │  │
│  │  └── /docs        → Swagger UI                                 │  │
│  └────────────────────────────────────────────────────────────────┘  │
│       ↓ (returns search results)                                     │
│                                                                      │
│  ChatGPT calls: open_comparison_dashboard({series_ids: [...]})       │
│       ↓                                                              │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Cloudflare Tunnel (Widget)                                    │  │
│  │  https://adventures-poetry-seattle-windsor.trycloudflare.com   │  │
│  │       ↓                                                        │  │
│  │  ChatGPT Widget (port 5173)                                    │  │
│  │  React app with Recharts trend visualization                   │  │
│  │  Communicates back to API via MCP bridge                       │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  Widget renders interactive chart inside ChatGPT                     │
└──────────────────────────────────────────────────────────────────────┘

                Server Machine (159.65.145.221)
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  PostgreSQL (pgvector)           port 5432                           │
│  ├── Schema: statlens                                                │
│  ├── Extensions: vector, pg_trgm                                     │
│  ├── Tables: sources, slices, tenants, agents, inference_models...   │
│  └── Full-text + vector search indexes                               │
│                                                                      │
│  StatLens API Server             port 8012                           │
│  ├── FastAPI + Uvicorn                                               │
│  ├── MCP (JSON-RPC 2.0)                                             │
│  ├── Strawberry GraphQL                                              │
│  └── Sentence-transformers embeddings (all-MiniLM-L6-v2)            │
│                                                                      │
│  ChatGPT Widget (Vite preview)   port 5173                           │
│  └── Pre-built React app served as static files                      │
│                                                                      │
│  Cloudflare Tunnel #1            localhost:8012 → public URL         │
│  Cloudflare Tunnel #2            localhost:5173 → public URL         │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Breakdown

### 3.1 PostgreSQL Database

**What it is:** A PostgreSQL 18 database with the `pgvector` extension for vector similarity search and `pg_trgm` for trigram text matching.

**How it runs:** Inside a Docker container named `statlens-postgres`.

| Property | Value |
|----------|-------|
| Image | `pgvector/pgvector:pg18` |
| Container name | `statlens-postgres` |
| Host port | `127.0.0.1:5432` |
| Database name | `statlens` |
| Superuser | `postgres` / `postgres` |
| App user | `statlens` / `statlens` |
| Extensions | `vector`, `pg_trgm` |

**What it stores:**
- **Sources** — 11 data source definitions (bea, bls, cdc, uscensus, who, worldbank, oecd, statcan, undata, ncses, bts)
- **Slices** — Indexed dataset fragments with metadata, embeddings, and facet schemas
- **Tenants** — Multi-tenant configuration (e.g., `acme` tenant subscribed to bea, uscensus, cdc)
- **Inference agents** — LLM agent configurations (decompose, select, narrate, etc.)
- **Inference models** — Model settings (gpt-4o-mini, text-embedding-3-small, etc.)
- **Query plan cache** — Cached execution plans (SQLite-backed)
- **System config** — Runtime knobs stored as JSON

**Docker Compose file:** `infra/docker-compose.yml`  
**Init scripts:** `infra/init/01-extensions.sql` (creates pgvector + pg_trgm), `infra/init/02-app-user.sql` (creates statlens user)

---

### 3.2 Backend API Server (Port 8012)

**What it is:** A Python FastAPI application that serves REST, GraphQL, and MCP endpoints.

**Tech stack:**
- **FastAPI** — REST API framework
- **Strawberry GraphQL** — Type-safe GraphQL
- **SQLModel** (SQLAlchemy + Pydantic) — Async ORM
- **asyncpg** — Async PostgreSQL driver
- **Uvicorn** — ASGI server
- **sentence-transformers** — Local embedding model (`all-MiniLM-L6-v2`, 384 dimensions)
- **OpenAI API** — LLM inference (gpt-4o-mini for reasoning, text-embedding-3-small for embeddings)

**Endpoints:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/docs` | GET | Swagger UI (auto-generated API documentation) |
| `/api/v1/ask` | POST | NLWeb query — natural language → structured data |
| `/api/v1/inference/models` | GET | List available LLM models |
| `/api/v1/inference/agents` | GET | List inference agents |
| `/api/v1/admin/tenants/{id}/keys` | POST/GET/DELETE | Manage tenant API keys |
| `/graphql` | POST | GraphQL queries and mutations |
| `/mcp` | POST | MCP (Model Context Protocol) tools |
| `/api/dev/widget-bridge` | POST | Local dev bridge for widget testing |

**Startup process (lifespan):**
1. Run `bootstrap()` — create schema, tables, indexes; seed sources, agents, models, tenant
2. Load sentence-transformers embedding model into memory
3. Verify inference agents exist (decompose, select)
4. Initialize plan cache (SQLite)
5. Start plan warmer background task (refreshes cached plans every hour)
6. Start MCP session manager

---

### 3.3 MCP Server (Model Context Protocol)

**What it is:** An MCP-compliant server that exposes tools ChatGPT can call directly via JSON-RPC 2.0 over HTTP.

**Mount point:** `/mcp` on the API server

**Tools exposed to ChatGPT:**

#### `search(request)`
- **Input:** A list of search intents (topic + measures)
  ```json
  {
    "intents": [
      { "topic": "US unemployment rate", "measures": ["unemployment rate"], "period_hint": "latest" }
    ]
  }
  ```
- **What it does:** Runs parallel keyword + semantic vector search across all subscribed sources
- **Output:** Ranked list of matching series with relevance scores, source, title, unit, temporal scope

#### `fetch(request)`
- **Input:** A `series_id` (e.g., `bea:nipa_t10101`)
- **What it does:** Calls the actual government API to get observations (time-series data points)
- **Output:** Array of `{period, measures, dimensions}` objects

#### `open_comparison_dashboard(request)`
- **Input:** Optional title, query, pre-selected series_ids
- **What it does:** Prepares widget state and returns metadata telling ChatGPT to render the widget
- **Output:** Widget configuration (view, title, query, selected series) + OpenAI Apps SDK metadata

**Widget metadata:** Each MCP tool response includes OpenAI-specific metadata:
- `openai/widgetDomain` — The widget's public URL (Cloudflare tunnel)
- `openai/widgetCSP` — Content Security Policy with allowed domains
- `openai/widgetDescription` — Human-readable tool description

---

### 3.4 ChatGPT Widget (Port 5173)

**What it is:** A React/TypeScript single-page application that renders an interactive statistical comparison dashboard inside ChatGPT conversations.

**Location:** `apps/chatgpt-widget/`

**Tech stack:**
- **React 19** — UI framework
- **Recharts** — SVG chart library for trend visualization
- **Vite** — Build tool and dev server
- **TypeScript** — Type safety

**How it works:**

1. ChatGPT calls `open_comparison_dashboard()` MCP tool
2. ChatGPT renders the widget iframe using the public URL from `STATLENS_APPS_WIDGET_URL`
3. The widget loads and reads its initial state (title, query, pre-selected series)
4. The widget communicates with the API server through **MCP bridge** — JSON-RPC 2.0 messages via `postMessage` to the parent ChatGPT frame

**Widget features:**
- **Search bar** — Enter natural language queries, calls `search()` MCP tool
- **Series selection** — Pick multiple indicators to compare
- **Trend chart** — Interactive time-series visualization with Recharts
- **State persistence** — Uses OpenAI `widgetState` API to remember selections across turns

**Bridge modes (how widget talks to backend):**
| Mode | When | How |
|------|------|-----|
| `mcp-standard` | Running inside ChatGPT | JSON-RPC 2.0 via parent frame postMessage |
| `window-openai` | ChatGPT compatibility layer | `window.openai.callTool()` |
| `local-mock` | Local development | Mock data for testing |

**Build output:** Single JS + CSS bundle in `dist/`:
- `dist/assets/widget.js` (~209 KB)
- `dist/assets/widget.css` (~4.5 KB)
- `dist/assets/TrendChart-*.js` (~335 KB, lazy-loaded chart component)

---

### 3.5 Cloudflare Tunnels

**What they are:** Free temporary HTTPS tunnels from Cloudflare that expose your local services to the public internet. ChatGPT needs public HTTPS URLs to call your MCP tools and render your widget.

**Why needed:** ChatGPT runs in the browser at `chatgpt.com`. It cannot reach `localhost:8012` or `localhost:5173` on your server. Cloudflare tunnels give you random `.trycloudflare.com` URLs that route traffic to your local ports.

**Two tunnels are required:**

| Tunnel | Local Port | Purpose | Example URL |
|--------|-----------|---------|-------------|
| API tunnel | `localhost:8012` | ChatGPT calls MCP tools here | `https://heritage-zinc-programmes-shoulder.trycloudflare.com` |
| Widget tunnel | `localhost:5173` | ChatGPT loads widget iframe from here | `https://adventures-poetry-seattle-windsor.trycloudflare.com` |

**Important:** The tunnel URLs change every time you restart `cloudflared`. You must update the `.env` file with new URLs after each restart.

**How the URLs connect:**
- `STATLENS_APPS_WIDGET_URL` = Widget tunnel URL (where ChatGPT loads the widget from)
- `STATLENS_APPS_WIDGET_DOMAIN` = Same as widget URL (used for CSP domain matching)
- `STATLENS_APPS_WIDGET_CONNECT_DOMAINS` = API tunnel URL (tells the widget where to send MCP requests)

The API server reads these settings and includes them in MCP tool responses as metadata, so ChatGPT knows which domains to allow for iframe loading and API calls.

---

## 4. How Data Flows End-to-End

Here's what happens when a user asks "unemployment rate" in ChatGPT with StatLens connected:

```
Step 1: USER → ChatGPT
        User types: "unemployment rate"

Step 2: ChatGPT → MCP search()
        ChatGPT decides to call the StatLens search tool
        POST https://heritage-zinc-..../mcp
        Body: {"method": "tools/call", "params": {"name": "search", "arguments": {
            "request": {
                "intents": [
                    {"topic": "US unemployment rate", "measures": ["unemployment rate"]},
                    {"topic": "global unemployment rate", "measures": ["unemployment rate"]}
                ]
            }
        }}}

Step 3: Cloudflare Tunnel → localhost:8012/mcp
        Tunnel routes the HTTPS request to your local API server

Step 4: API Server processes search
        a) SliceSearcher runs parallel keyword + semantic searches
        b) Queries PostgreSQL for matching slices (using pg_trgm + pgvector)
        c) Ranks results by combined score (keyword rank + embedding similarity)
        d) Returns top matches per intent

Step 5: ChatGPT receives search results
        Gets back: [{series_id: "bls:...", title: "Unemployment Rate", relevance: 0.95}, ...]

Step 6: ChatGPT → MCP open_comparison_dashboard()
        ChatGPT decides to show the comparison widget
        Calls: open_comparison_dashboard({title: "Unemployment Rate Comparison", series_ids: [...]})

Step 7: Widget renders in ChatGPT
        a) ChatGPT creates an iframe pointing to:
           https://adventures-poetry-seattle-windsor.trycloudflare.com/
        b) Widget loads React app + TrendChart
        c) Widget reads initial state (series_ids from the tool call)
        d) Widget calls fetch() for each series to get actual data points
        e) Renders interactive trend chart

Step 8: User interacts with widget
        - Can search for more indicators
        - Can select/deselect series
        - Widget pushes context back to ChatGPT model
```

**When catalogs are empty (your current situation):**

The warning `search_catalog_empty source=bea — catalog may not have been harvested yet` means the slices table is empty for that source. You need to **harvest** data first:

```bash
uv run statlens harvest bea          # Discover + persist BEA catalog
uv run statlens slice-generate bea   # Generate searchable slices
```

Until you harvest, `search()` will return 0 results for all sources.

---

## 5. The .env File — Every Variable Explained

```env
# ══════════════════════════════════════════════════════════════════════
# DATABASE
# ══════════════════════════════════════════════════════════════════════
STATLENS_DATABASE_URL=postgresql+asyncpg://statlens:statlens@localhost:5432/statlens
#                     ─────────┬────────── ───┬─── ───┬─── ────┬──── ──┬─ ───┬───
#                      driver+dialect    user  pass   host   port  dbname
# ⚠ Port MUST be 5432 (matches docker-compose). Port 5433 will NOT work.

STATLENS_CACHE_BASE=.cache/statlens
# Directory for HTTP caches, embedding caches, and plan cache

# ══════════════════════════════════════════════════════════════════════
# SOURCE API KEYS
# ══════════════════════════════════════════════════════════════════════
BEA_API_KEY="key1,key2,key3,..."
# Comma-separated BEA API keys — automatic round-robin rotation
# Get yours at: https://apps.bea.gov/API/signup/

# ══════════════════════════════════════════════════════════════════════
# LLM INFERENCE
# ══════════════════════════════════════════════════════════════════════
STATLENS_OPENAI_API_KEY=sk-proj-...
# Your OpenAI API key — used for decompose/select/narrate agents

STATLENS_OPENAI_MODEL=gpt-4o-mini
# Main LLM model for query decomposition and curation

STATLENS_EMBEDDING_MODEL=text-embedding-3-small
# OpenAI embedding model (used alongside local sentence-transformers)

STATLENS_EMBEDDING_DIM=1536
# Embedding dimension for the OpenAI model

# ══════════════════════════════════════════════════════════════════════
# SECURITY
# ══════════════════════════════════════════════════════════════════════
STATLENS_ADMIN_SECRET=96f78f7a...
# Bearer token for admin API routes (tenant management, model CRUD)
# Used as: Authorization: Bearer <this-value>

# ══════════════════════════════════════════════════════════════════════
# TENANT CONFIGURATION
# ══════════════════════════════════════════════════════════════════════
STATLENS_TENANT_ID=acme
# Default tenant identifier (seeded on bootstrap)

STATLENS_TENANT_SOURCES=["bea","uscensus","cdc"]
# Sources this tenant is subscribed to (JSON array)
# Only these sources will be searched for this tenant's queries

# ══════════════════════════════════════════════════════════════════════
# CHATGPT WIDGET CONFIGURATION (Cloudflare Tunnel URLs)
# ══════════════════════════════════════════════════════════════════════
STATLENS_APPS_WIDGET_URL=https://adventures-poetry-seattle-windsor.trycloudflare.com
# Public URL where ChatGPT loads the widget iframe from
# This is the Cloudflare tunnel URL for port 5173 (widget)
# ⚠ Changes every time you restart cloudflared — MUST UPDATE

STATLENS_APPS_WIDGET_DOMAIN=https://adventures-poetry-seattle-windsor.trycloudflare.com
# Same as WIDGET_URL — used for CSP domain matching in MCP metadata
# ⚠ Must match WIDGET_URL

STATLENS_APPS_WIDGET_CONNECT_DOMAINS=https://heritage-zinc-programmes-shoulder.trycloudflare.com
# Public URL of the API tunnel (port 8012)
# Tells the widget which domain it's allowed to make MCP requests to
# ⚠ This is the API tunnel URL, NOT the widget tunnel URL
# ⚠ Changes every time you restart cloudflared — MUST UPDATE

STATLENS_APPS_WIDGET_DEV_BRIDGE_ENABLED=true
# Enables /api/dev/widget-bridge endpoint for local testing
# Set to false in production
```

**Relationship between tunnel URLs and .env variables:**

```
cloudflared tunnel --url http://localhost:5173
  → gives you: https://adventures-poetry-..trycloudflare.com
  → put in: STATLENS_APPS_WIDGET_URL and STATLENS_APPS_WIDGET_DOMAIN

cloudflared tunnel --url http://localhost:8012
  → gives you: https://heritage-zinc-..trycloudflare.com
  → put in: STATLENS_APPS_WIDGET_CONNECT_DOMAINS
```

---

## 6. How to Start the Project (Step by Step)

### Prerequisites
- Docker & docker-compose (v1) installed
- Python 3.11+ with `uv` package manager
- Node.js 20+ with npm
- `cloudflared` installed (`sudo dpkg -i cloudflared-linux-amd64.deb`)

### Step-by-step startup:

You need **5 terminal sessions** (or use background processes). Execute these in order:

---

#### Terminal 1: Start PostgreSQL

```bash
cd /mnt/volume_blr1_01/Enconvers/statlens
docker-compose -f infra/docker-compose.yml up -d
```

Verify it's running:
```bash
docker ps --filter name=statlens-postgres
# Should show: statlens-postgres ... Up ... 0.0.0.0:5432->5432/tcp
```

---

#### Terminal 2: Bootstrap & Start API Server

```bash
cd /mnt/volume_blr1_01/Enconvers/statlens

# Bootstrap (creates schema, seeds data, loads models)
uv run statlens bootstrap

# Start API server on port 8012
uv run statlens serve --host 0.0.0.0 --port 8012
```

Verify: Open `http://localhost:8012/docs` — should show Swagger UI.

**Expected bootstrap output:**
```
statlens.core.db db_init schema=statlens
statlens.core.db db_search_indexes_ready
statlens.bootstrap.inference inference_agents_upserted count=9
statlens.bootstrap.tenant tenant_seeded tenant=acme sources=['bea', 'uscensus', 'cdc']
statlens.inference.backends.sentence_transformers st_backend_ready model=sentence-transformers/all-MiniLM-L6-v2
statlens.bootstrap bootstrap_complete
```

---

#### Terminal 3: Build & Serve ChatGPT Widget

```bash
cd /mnt/volume_blr1_01/Enconvers/statlens/apps/chatgpt-widget

# Install dependencies (only needed once)
npm install

# Build the widget
npm run build

# Serve the built widget on port 5173
npm run preview -- --host 0.0.0.0 --port 5173
```

Verify: Open `http://localhost:5173` — should load the widget page.

---

#### Terminal 4: Start Cloudflare Tunnel for API (port 8012)

```bash
cloudflared tunnel --url http://localhost:8012
```

**Copy the generated URL** from the output:
```
Your quick Tunnel has been created! Visit it at:
https://heritage-zinc-programmes-shoulder.trycloudflare.com    ← COPY THIS
```

---

#### Terminal 5: Start Cloudflare Tunnel for Widget (port 5173)

```bash
cloudflared tunnel --url http://localhost:5173
```

**Copy the generated URL** from the output:
```
Your quick Tunnel has been created! Visit it at:
https://adventures-poetry-seattle-windsor.trycloudflare.com    ← COPY THIS
```

---

#### Final Step: Update .env with new tunnel URLs

Edit `/mnt/volume_blr1_01/Enconvers/statlens/.env` and update:

```env
STATLENS_APPS_WIDGET_URL=https://<WIDGET-TUNNEL-URL-FROM-TERMINAL-5>
STATLENS_APPS_WIDGET_DOMAIN=https://<WIDGET-TUNNEL-URL-FROM-TERMINAL-5>
STATLENS_APPS_WIDGET_CONNECT_DOMAINS=https://<API-TUNNEL-URL-FROM-TERMINAL-4>
```

**Then restart the API server** (Terminal 2) — press Ctrl+C and run serve again:
```bash
uv run statlens serve --host 0.0.0.0 --port 8012
```

The server re-reads `.env` on startup and includes the new tunnel URLs in MCP tool metadata.

---

### Quick-Start Cheat Sheet (copy-paste)

```bash
# ── Terminal 1: Database ──
cd /mnt/volume_blr1_01/Enconvers/statlens
docker-compose -f infra/docker-compose.yml up -d

# ── Terminal 2: API Server ──
cd /mnt/volume_blr1_01/Enconvers/statlens
uv run statlens bootstrap
uv run statlens serve --host 0.0.0.0 --port 8012

# ── Terminal 3: Widget ──
cd /mnt/volume_blr1_01/Enconvers/statlens/apps/chatgpt-widget
npm run build && npm run preview -- --host 0.0.0.0 --port 5173

# ── Terminal 4: API Tunnel ──
cloudflared tunnel --url http://localhost:8012

# ── Terminal 5: Widget Tunnel ──
cloudflared tunnel --url http://localhost:5173

# ── Then update .env with new tunnel URLs and restart API server ──
```

---

## 7. CLI Commands Reference

All commands run from `/mnt/volume_blr1_01/Enconvers/statlens/`:

### Core Commands

| Command | Purpose |
|---------|---------|
| `uv run statlens bootstrap` | Init DB schema, seed sources/agents/models, load embeddings |
| `uv run statlens serve --host 0.0.0.0 --port 8012` | Start the API server |
| `uv run statlens sources` | List all seeded data sources |
| `uv run statlens sources -f table` | List sources in table format |
| `uv run statlens agents` | List inference agents |

### Data Pipeline Commands

| Command | Purpose |
|---------|---------|
| `uv run statlens harvest bea` | Discover and persist BEA catalog |
| `uv run statlens harvest bea -f NIPA/T10101` | Harvest specific BEA table |
| `uv run statlens catalog bea` | List BEA catalog entries |
| `uv run statlens slice-generate bea` | Generate searchable slices for BEA |
| `uv run statlens slice-generate bea --dry-run` | Preview slice generation |
| `uv run statlens slices bea` | List BEA slices |
| `uv run statlens slices bea -d bea:NIPA/T10101` | List slices for specific dataset |

### Query Commands

| Command | Purpose |
|---------|---------|
| `uv run statlens search "unemployment rate"` | Search catalog from CLI |
| `uv run statlens ask "What is US GDP?"` | Run full NLWeb query from CLI |
| `uv run statlens fetch <series_id>` | Fetch observations for a series |

### Admin Commands

| Command | Purpose |
|---------|---------|
| `uv run statlens seed` | Re-seed sources from `seeds/sources/*.json` |
| `uv run statlens profile <dataset_id> --slices` | Profile a dataset's slices |
| `uv run statlens validate <source>` | Validate source configuration |

---

## 8. Troubleshooting Common Issues

### "OSError: [Errno 111] Connect call failed ('127.0.0.1', 5432)"

**Cause:** PostgreSQL container is not running.  
**Fix:**
```bash
# Check if container exists
docker ps -a --filter name=statlens-postgres

# If stopped, start it:
docker-compose -f infra/docker-compose.yml up -d

# If gone, recreate:
docker-compose -f infra/docker-compose.yml up -d
```

### "OSError: [Errno 111] Connect call failed ('127.0.0.1', 5433)"

**Cause:** `.env` has wrong port (5433 instead of 5432).  
**Fix:** Edit `.env` and change:
```
STATLENS_DATABASE_URL=postgresql+asyncpg://statlens:statlens@localhost:5432/statlens
```
Port **must** be `5432` (matching `infra/docker-compose.yml`).

### "search_catalog_empty source=bea — catalog may not have been harvested yet"

**Cause:** No data has been harvested yet. The 11 sources are registered but their catalogs are empty.  
**Fix:** Harvest data for the sources you need:
```bash
cd /mnt/volume_blr1_01/Enconvers/statlens

# BEA
uv run statlens harvest bea
uv run statlens slice-generate bea

# US Census
uv run statlens harvest uscensus
uv run statlens slice-generate uscensus

# CDC
uv run statlens harvest cdc
uv run statlens slice-generate cdc
```

### "cloudflared: command not found"

**Cause:** cloudflared not installed.  
**Fix:**
```bash
cd /mnt/volume_blr1_01/Enconvers/statlens
sudo dpkg -i cloudflared-linux-amd64.deb
```
Note: The `.deb` file must be in the current directory. It was downloaded to the `statlens/` folder.

### "winget: command not found"

**Cause:** `winget` is a Windows package manager. This is a Linux server.  
**Fix:** Use `apt` or `dpkg` instead (see cloudflared install above).

### "docker compose: command not found"

**Cause:** Docker Compose v2 plugin is not installed. Only v1 (`docker-compose`) works.  
**Fix:** Use `docker-compose` (with hyphen) instead of `docker compose` (with space).

### Tunnel URLs changed after restart

**Cause:** Free Cloudflare tunnels generate random URLs each time.  
**Fix:**
1. Copy new URLs from cloudflared output
2. Update `.env` with new URLs
3. Restart the API server (Ctrl+C → `uv run statlens serve --host 0.0.0.0 --port 8012`)

### Widget shows blank or errors in ChatGPT

**Cause:** CSP (Content Security Policy) mismatch — the tunnel URLs in `.env` don't match the actual running tunnels.  
**Fix:** Ensure `STATLENS_APPS_WIDGET_URL` matches the widget tunnel and `STATLENS_APPS_WIDGET_CONNECT_DOMAINS` matches the API tunnel, then restart the API server.

---

## Appendix: Project File Structure

```
statlens/
├── .env                        ← Runtime configuration (EDIT THIS)
├── .env.example                ← Template for .env
├── pyproject.toml              ← Python dependencies & metadata
├── uv.lock                     ← Locked dependency versions
├── pytest.ini                  ← Test configuration
│
├── apps/
│   └── chatgpt-widget/         ← React widget for ChatGPT
│       ├── package.json
│       ├── vite.config.ts
│       ├── src/
│       │   ├── App.tsx          ← Main comparison dashboard
│       │   ├── main.tsx         ← Entry point
│       │   ├── components/
│       │   │   └── TrendChart.tsx
│       │   └── lib/
│       │       ├── bridge.ts    ← MCP bridge interface
│       │       └── mcpBridge.ts ← JSON-RPC client
│       └── dist/                ← Built output
│
├── src/statlens/
│   ├── api/                    ← FastAPI server
│   │   ├── app.py              ← App factory + lifespan
│   │   ├── auth.py             ← Authentication
│   │   ├── deps.py             ← Shared dependencies
│   │   ├── graphql/            ← Strawberry GraphQL
│   │   └── routes/             ← REST endpoints
│   │       ├── admin.py        ← Tenant/key management
│   │       ├── query.py        ← POST /api/v1/ask
│   │       ├── inference.py    ← Model CRUD
│   │       └── widget_dev.py   ← Dev bridge
│   │
│   ├── mcp/                    ← MCP server
│   │   ├── server.py           ← FastMCP tools (search, fetch, dashboard)
│   │   └── handlers.py         ← Tool implementations
│   │
│   ├── core/                   ← Core framework
│   │   ├── settings.py         ← Pydantic Settings (env vars)
│   │   ├── db.py               ← Database engine + init
│   │   └── cache/              ← Caching layer
│   │
│   ├── bootstrap/              ← Schema + seed management
│   │   ├── __init__.py         ← Main bootstrap() function
│   │   ├── agents.py           ← Seed inference agents
│   │   ├── inference.py        ← Seed LLM models
│   │   ├── seeds.py            ← Seed data sources
│   │   └── tenant.py           ← Seed default tenant
│   │
│   ├── query/                  ← Query engine
│   │   ├── search.py           ← SliceSearcher (keyword + semantic)
│   │   ├── curator.py          ← QueryCurator (resolve params)
│   │   ├── fetcher.py          ← ObservationFetcher
│   │   ├── plan.py             ← ExecutablePlan
│   │   ├── plan_cache.py       ← Cached plans (SQLite)
│   │   └── plan_warmer.py      ← Background cache warming
│   │
│   ├── harvest/                ← Data pipeline
│   ├── sources/                ← Source adapters (BEA, BLS, etc.)
│   ├── slices/                 ← Slice generation
│   ├── models/                 ← SQLModel domain models
│   ├── storage/                ← Storage abstractions
│   ├── inference/              ← LLM backends
│   └── cli/                    ← Click CLI commands
│
├── seeds/sources/              ← Source definitions (JSON)
├── infra/
│   ├── docker-compose.yml      ← PostgreSQL container
│   └── init/                   ← DB init scripts
├── docs/                       ← Documentation
└── tests/                      ← Test suite
```
