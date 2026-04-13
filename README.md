# Dali

<p align="center">
  <img src="image.png" alt="Dali" width="400">
</p>

An [MCP](https://modelcontextprotocol.io) server that gives Claude Code persistent long-term memory across sessions. Stores memories with automatic vector embeddings for semantic search, plus a full audit trail of tool invocations.

## Features

- **Semantic search** -- 768-dimensional vector embeddings via Ollama (`nomic-embed-text`) with cosine similarity ranking and project-context boosting
- **Memory types** -- conversation, decision, file_change, tool_usage -- each with optional metadata, tags, and project scoping
- **Audit logging** -- every tool invocation is recorded with timestamps, inputs, and truncated outputs for later querying
- **Graceful degradation** -- if Ollama is unavailable, memories are still stored and text-based search works; embeddings are backfilled automatically when Ollama comes back online
- **Soft delete** -- archive memories or permanently delete them with cascading cleanup of embeddings and vector mappings

## Architecture

```
src/
  index.ts           -- Entry point, stdio MCP transport
  server.ts          -- MCP tool and resource definitions (7 tools, 2 resources)
  types.ts           -- Zod schemas and TypeScript types
  db/
    client.ts        -- SQLite wrapper (WAL mode, sqlite-vec extension)
    schema.ts        -- DDL, migrations, 6 tables + 10 indexes
  embeddings/
    ollama.ts        -- Ollama HTTP client, batch embedding (chunks of 32)
  memory/
    store.ts         -- CRUD operations, embedding backfill on startup
    search.ts        -- Vector KNN search with CTE, text fallback
  audit/
    logger.ts        -- Audit log recording (10KB output truncation)
    query.ts         -- Paginated audit queries with date/filter support
```

### MCP Tools

| Tool | Description |
|------|-------------|
| `store_memory` | Store a memory with automatic vector embedding |
| `search_memory` | Semantic vector search (cosine similarity, configurable min relevance) |
| `search_by_type` | Filter memories by type with optional text query |
| `get_audit_log` | Paginated audit trail queries with date ranges and filters |
| `get_session_summary` | Session details with all associated memories |
| `list_recent_memories` | Chronological listing with type/project filters |
| `forget_memory` | Soft delete (archive) or permanent delete with cascade |

### MCP Resources

| Resource | Description |
|----------|-------------|
| `dali://memories/recent` | 10 most recent memories |
| `dali://sessions/{sessionId}/memories` | All memories for a session |

## Installation

### 1. Prerequisites

- **Node.js** 18+
- **Ollama** running locally with the `nomic-embed-text` model (optional but recommended for vector search)

### 2. Install Ollama (optional, for vector search)

```bash
# macOS
brew install ollama

# Pull the embedding model
ollama pull nomic-embed-text

# Start Ollama (if not already running)
ollama serve
```

Without Ollama, Dali still works -- memories are stored and text-based search is available. Vector embeddings are backfilled automatically when Ollama becomes available.

### 3. Clone, install, and build

```bash
git clone https://github.com/linnaxis/dali-memory.git
cd dali-memory
npm install
npm run build
```

### 4. Configure Claude Code

Add Dali as an MCP server in your Claude Code config. Edit `~/.claude/mcp.json` (or your project's `.mcp.json`):

```json
{
  "mcpServers": {
    "dali": {
      "command": "node",
      "args": ["/absolute/path/to/dali-memory/dist/index.js"],
      "env": {
        "OLLAMA_BASE_URL": "http://localhost:11434",
        "OLLAMA_MODEL": "nomic-embed-text",
        "DALI_DB_PATH": "~/.claude/dali/dali.db"
      }
    }
  }
}
```

Replace `/absolute/path/to/dali-memory` with the actual path where you cloned the repo.

### 5. Verify

Restart Claude Code. You should see Dali's 7 tools available in your MCP tool list. Try asking Claude to "store a memory" or "search memories" to confirm it's working.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DALI_DB_PATH` | `~/.claude/dali/dali.db` | Path to the SQLite database file |
| `DALI_PROJECT` | Current directory name | Default project name for memory scoping |
| `OLLAMA_BASE_URL` | `http://localhost:11434` | Ollama API endpoint |
| `OLLAMA_MODEL` | `nomic-embed-text` | Ollama model for generating embeddings |

## Run

```bash
# Production
npm start

# Development (auto-rebuild on changes)
npm run dev
```

The server communicates over stdio (MCP standard) -- it's meant to be launched by Claude Code, not run standalone in a terminal.

## Database

Dali uses SQLite with WAL mode and the [sqlite-vec](https://github.com/asg017/sqlite-vec) extension for vector search. The database is created automatically on first run at the path specified by `DALI_DB_PATH`.

### Schema

- `memories` -- core storage (type, content, summary, tags, project, session)
- `memory_vec_map` -- bridge table mapping UUIDs to integer rowids for sqlite-vec
- `memory_embeddings` -- vec0 virtual table with 768-dimensional float vectors
- `audit_log` -- tool invocation audit trail
- `sessions` -- session metadata
- `file_changes` -- file change details linked to memories

## Tech Stack

- TypeScript (ESM, strict mode)
- [MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk) v1.12
- SQLite via [better-sqlite3](https://github.com/WiseLibs/better-sqlite3)
- [sqlite-vec](https://github.com/asg017/sqlite-vec) for vector similarity search
- [Ollama](https://ollama.com) for local embedding generation
- [Zod](https://github.com/colinhacks/zod) for input validation

## License

MIT
