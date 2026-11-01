# OPUS 67

A skill-orchestration runtime and MCP server for Claude Code: it picks the skill files, operating mode and model that fit a task and serves them to Claude inside a token budget. Claude Code calls its tools; nothing is injected automatically.

OPUS 67 is not a separate model. It is a library of Markdown skill definitions with a keyword-matching loader, a complexity-based mode selector, an MCP server that exposes both to Claude Code, and an optional HTTP and WebSocket "brain" service that adds model routing, a multi-model council, memory and a pattern-detection loop. It is for developers who want domain context (Solana, React, DevOps, security and more) loaded on demand instead of pasted into every prompt.

Part of [gICM](https://github.com/0motionguy/gICM), where the current copy of this package lives under `packages/opus67`. Notes from the previous README: [docs/DEVELOPMENT.md](docs/DEVELOPMENT.md).

## What it does

- **Skill registry and loader.** `skills/registry.yaml` registers 141 skills, each with a tier, a token cost and `auto_load_when` triggers (keywords, file types, directories, task patterns). `src/skills/matcher.ts` matches a request against them and `src/skill-loader.ts` returns the best fits within a token budget. Skill bodies are Markdown files in `skills/definitions/`.
- **Modes.** `modes/registry.yaml` defines ten core modes (ultra, think, build, vibe, light, creative, data, audit, swarm, auto) with token budgets, trigger keywords and model hints; `src/modes` scores task complexity from 1 to 10 to choose one.
- **MCP server.** `opus67-mcp` serves 15 tools over stdio: detect, get and list skills, modes and MCP connections, context bundles, status, memory queries and writes, and tool metrics. `opus67-legacy init` registers it with Claude Code.
- **Brain service.** `opus67-server` starts a Fastify server (port 3100 by default) with `GET /health`, `GET /api/brain/{boot,status,metrics,history,mode}`, `POST /api/brain/{query,mode,evolution,deliberate}` and a WebSocket at `/api/brain/ws`.
- **Model router and council.** `src/models/router.ts` maps task types to Gemini, DeepSeek or Claude tiers with fallbacks and cost tracking. `src/council` runs a three-stage deliberation: independent answers, peer ranking, chairman synthesis.
- **Memory.** `src/memory` combines hierarchical memory, a local cache and a Graphiti-style temporal graph that uses Neo4j when configured and local storage otherwise; `src/intelligence` can use Qdrant for skill search.
- **Evolution loop.** `src/evolution` detects usage patterns and proposes improvements. It defaults to dry-run with auto-apply off.
- **Connection catalogue.** `mcp/connections.yaml` lists about 80 MCP and API connections (Helius, Jupiter, GitHub, Supabase and others) as definitions. Nothing connects automatically; you run the servers and supply the keys.

## What a skill entry looks like

Skills are registered in YAML and written in Markdown. This is the shape of an entry in `skills/registry.yaml`:

```yaml
- id: solana-anchor-expert
  name: Solana Anchor Expert
  tier: 1
  token_cost: 15000
  auto_load_when:
    keywords: [anchor, solana, pda, cpi, program, instruction, devnet]
    file_types: [.rs]
    directories: [programs/, anchor/]
  mcp_connections: [helius, jupiter]
```

The loader sorts matches by tier, then by token cost, and stops at the session limits set in the registry header (5 skills and a 50,000-token budget).

## How it's built

TypeScript (ESM) bundled with tsup and tested with vitest. Fastify and `@fastify/websocket` serve the brain API, `@modelcontextprotocol/sdk` the MCP server, the Anthropic SDK the Claude calls, and `yaml` and `zod` the registries and config. Skills, modes and connections are data files, so adding a skill needs no code change.

```text
Claude Code --stdio--> opus67-mcp --> skill loader + mode selector --> skills/*.md
Client --HTTP/WS--> opus67-server --> model router --> Claude / Gemini / DeepSeek
                                  |-> council (3 stages)
                                  |-> memory (local, optional Neo4j and Qdrant)
                                  `-> evolution loop (dry-run by default)
```

## Repository layout

- `src/`: TypeScript sources (`brain`, `council`, `models`, `memory`, `evolution`, `intelligence`, `modes`, `skills`, `mcp-server`, `sdk`, `benchmark`, `tests`)
- `skills/`: `registry.yaml`, `definitions/` (Markdown bodies), `metadata/`, `fragments/`
- `modes/`, `registry/`, `config/`: mode registry, `MASTER.yaml` component index, council, endpoint and model YAML
- `mcp/`: `connections.yaml` and a sample `anchor-mcp` server
- `agents/`: two system agent definitions (`skills-navigator`, `learning-observer`)
- `docker/`, `Dockerfile`, `docker-compose.yml`: container setup (assumes the gICM monorepo layout, so build the image from the hub)
- `docs/` and the root `*.md` reports: design notes, benchmark and comparison reports

## Run it locally

```bash
pnpm install     # npm install also works; no lockfile is shipped
pnpm build       # tsup
pnpm start       # brain service on http://localhost:3100
pnpm test        # vitest, 17 test files
```

With the service running, a quick check of the API:

```bash
curl http://localhost:3100/health
curl -X POST http://localhost:3100/api/brain/query \
  -H "Content-Type: application/json" \
  -d '{"query": "Build a REST API with authentication"}'
curl -X POST http://localhost:3100/api/brain/mode \
  -H "Content-Type: application/json" -d '{"mode": "build"}'
```

Set keys in the environment (no `.env.example` is shipped): `ANTHROPIC_API_KEY` for Claude calls, `GEMINI_API_KEY` (or `GOOGLE_AI_API_KEY`) and `DEEPSEEK_API_KEY` for the other router tiers. Optional: `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `QDRANT_URL`, `ACONTEXT_API_URL`, `ACONTEXT_API_KEY`, and `PORT`, `HOST`, `LOG_LEVEL`, `CORS_ORIGIN` for the server.

## Status

Beta. A February 2026 export of the hub's `packages/opus67`; the hub copy has moved on since. The skill loader, MCP server and brain API are implemented and covered by tests. The registry counts (141 skills, 107 agents, 82 MCPs) are metadata: only two system agents ship here, version strings disagree (`package.json` 6.3.0, `src/version.ts` 5.1.14), and the benchmark and comparison reports in the repo root are self-reported and have not been independently reproduced.

MIT licence, see LICENSE.
