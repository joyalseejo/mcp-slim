# mcp-slim

> Reduce MCP context overhead by 98%. A TypeScript proxy that replaces your MCP server tool manifest with 3 lazy-loading meta-tools — works with any MCP client.

```bash
npx mcp-slim --config mcp-slim.config.json
```

---

## The problem

Every MCP server dumps its full tool schema into your context window at startup. One server with 70 tools costs ~55,000 tokens before you type a single prompt. At 10 servers, you're burning 40–60% of your context window on tool definitions you'll never use in that session.

```
Before mcp-slim:

  startup → loads 200 tool schemas → 73,000 tokens used
  ░░░░░░░░░░░░░░░░░░░░░░████████████████████████████████████
  ^--- your work                              ^--- tool schemas
  ~127K tokens left for actual work

After mcp-slim:

  startup → loads 3 meta-tools → 847 tokens used
  ░█
  ^- 847 tokens
  ~199K tokens left for actual work
```

---

## Benchmark

| Setup | Tokens at startup | Tokens when tool needed |
|---|---|---|
| Direct connection (10 servers, ~200 tools) | **73,000** | 73,000 (already loaded) |
| Via mcp-slim | **847** | ~1,200 (loaded on demand) |

**97% reduction in startup token cost.**

> Baseline measured using Azure DevOps MCP server (70 tools) + GitHub MCP server (35 tools) + 8 other common servers. Token counts measured using the Anthropic tokenizer. Numbers will vary based on your specific servers and tool count.

When the model needs a tool, it pays ~350 additional tokens (search result + schema). That's still 98% cheaper than loading everything upfront — and only for tools it actually uses.

---

## How it works

mcp-slim acts as a single MCP server to your client. Behind the scenes it connects to all your real servers and builds an index of every tool available. Instead of flooding your context with all those schemas, it exposes 3 meta-tools:

| Tool | What it does |
|---|---|
| `search_tools(query)` | Find tools by keyword. Returns names, descriptions, and which server provides them. |
| `get_tool_details(tool_name)` | Get the full parameter schema for one specific tool. |
| `execute_tool(tool_name, params)` | Run the tool. mcp-slim routes it to the right server automatically. |

```
Your MCP Client
      │
      │  sees 3 meta-tools (~847 tokens)
      ▼
  ┌─────────────┐
  │  mcp-slim   │  ← single endpoint
  │   proxy     │
  └──────┬──────┘
         │
         ├──► GitHub MCP Server      (35 tools, loaded on demand)
         ├──► Linear MCP Server      (28 tools, loaded on demand)
         ├──► Azure DevOps Server    (70 tools, loaded on demand)
         ├──► Filesystem Server      (12 tools, loaded on demand)
         └──► ...any other servers
```

**Schemas cache for the session.** The first time the model calls a tool, its schema loads (~200 tokens). Every subsequent call to that same tool is instant — already cached.

### Typical conversation flow

```
Model:  search_tools("create github issue")
Proxy:  → [create_issue, create_pull_request, create_branch]

Model:  get_tool_details("create_issue")
Proxy:  → { inputSchema: { title, body, labels, assignees... } }

Model:  execute_tool("create_issue", { title: "Fix login bug", body: "..." })
Proxy:  → routes to GitHub MCP server → returns result
```

---

## A note for Claude Code users

**If you're using Claude Code: you don't need this.**

Anthropic shipped native MCP Tool Search into Claude Code on January 14, 2026. It automatically defers tool loading when your servers exceed ~10K tokens — no configuration required.

mcp-slim is for:
- Custom MCP clients built in Node.js / TypeScript
- LangChain.js and Mastra integrations  
- Cursor users with large multi-server setups (Cursor doesn't have native lazy loading)
- Any framework or agent runner that implements MCP but doesn't handle context bloat natively

---

## Installation

```bash
# Use without installing (recommended for first try)
npx mcp-slim --config mcp-slim.config.json

# Install globally
npm install -g mcp-slim
mcp-slim --config mcp-slim.config.json

# Install as project dependency
npm install mcp-slim
```

**Requirements:** Node.js 18+

---

## Configuration

Create `mcp-slim.config.json` in your project root:

```json
{
  "servers": [
    {
      "name": "github",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_your_token_here"
      },
      "transport": "stdio"
    },
    {
      "name": "linear",
      "command": "npx",
      "args": ["-y", "@linear/mcp-server"],
      "env": {
        "LINEAR_API_KEY": "lin_api_your_key_here"
      },
      "transport": "stdio"
    },
    {
      "name": "filesystem",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/you/projects"],
      "transport": "stdio"
    }
  ],
  "options": {
    "cacheSchemas": true,
    "searchMaxResults": 5,
    "logLevel": "info"
  }
}
```

> ⚠️ **Add `mcp-slim.config.json` to your `.gitignore`** — it contains API tokens.

Use `mcp-slim.config.example.json` (included in this repo) as a template to commit instead.

### Config reference

| Field | Type | Required | Description |
|---|---|---|---|
| `servers` | array | ✓ | List of MCP servers to proxy |
| `servers[].name` | string | ✓ | Unique name for this server (used in search results) |
| `servers[].command` | string | ✓ | Command to start the server (e.g. `npx`, `node`) |
| `servers[].args` | string[] | | Arguments passed to the command |
| `servers[].env` | object | | Environment variables for this server process |
| `servers[].transport` | `stdio` \| `sse` | ✓ | Transport type |
| `servers[].url` | string | SSE only | URL for SSE transport |
| `options.cacheSchemas` | boolean | | Cache schemas in session. Default: `true` |
| `options.searchMaxResults` | number | | Max results from `search_tools`. Default: `5` |
| `options.logLevel` | `info` \| `debug` \| `silent` | | Logging verbosity. Default: `info` |

---

## Usage

### With Cursor

In your Cursor MCP config (`~/.cursor/mcp.json` or project `.cursor/mcp.json`), replace your individual server entries with a single mcp-slim entry:

```json
{
  "mcpServers": {
    "mcp-slim": {
      "command": "npx",
      "args": [
        "mcp-slim",
        "--config",
        "/absolute/path/to/mcp-slim.config.json"
      ]
    }
  }
}
```

> Use the absolute path to your config file — relative paths can break depending on how Cursor launches the process.

### With a custom Node.js MCP client

```typescript
import { Client } from '@modelcontextprotocol/sdk/client/index.js'
import { StdioClientTransport } from '@modelcontextprotocol/sdk/client/stdio.js'

const transport = new StdioClientTransport({
  command: 'npx',
  args: ['mcp-slim', '--config', './mcp-slim.config.json']
})

const client = new Client({ name: 'my-agent', version: '1.0.0' }, {})
await client.connect(transport)

// Your client now sees 3 tools instead of hundreds
const { tools } = await client.listTools()
console.log(tools.map(t => t.name))
// → ['search_tools', 'get_tool_details', 'execute_tool']
```

### With LangChain.js

```typescript
import { MCP } from '@langchain/mcp'

const mcp = new MCP({
  command: 'npx',
  args: ['mcp-slim', '--config', './mcp-slim.config.json']
})

await mcp.initialize()
// LangChain now loads only 3 tool schemas at startup
```

---

## What happens when a server fails to start?

mcp-slim handles server failures gracefully. If one of your configured servers fails to connect (wrong command, bad token, server not installed), mcp-slim logs the error and continues with the remaining servers. You won't get a crash — you'll get a reduced tool set and a clear error message in stderr telling you which server failed and why.

---

## When NOT to use mcp-slim

- **You're on Claude Code** — native Tool Search already handles this
- **Fewer than 5 servers and they're lightweight** — if your total tool schema overhead is under 5,000 tokens, the added complexity isn't worth it
- **You need deterministic tool visibility** — if your agent logic depends on knowing the full tool list at startup, lazy loading will break that assumption
- **Production agentic pipelines with strict latency requirements** — the on-demand schema fetch adds a small round-trip on first tool use per session

---

## Troubleshooting

**`spawn npx ENOENT`**  
npx isn't on the PATH that mcp-slim sees. Use the full path: `"command": "/usr/local/bin/npx"` or run `which npx` to find it.

**`Tool 'X' not found`**  
The server that provides this tool may have failed to connect at startup. Check stderr output for connection errors.

**`search_tools` returns nothing**  
Try `search_tools` with query `"*"` to list all indexed tools and confirm the index built correctly.

**Schemas aren't updating between sessions**  
By design — schemas cache per session only, not across restarts. Each new session re-fetches from the live servers.

---

## Contributing

Issues and PRs are welcome.

```bash
git clone https://github.com/your-username/mcp-slim
cd mcp-slim
npm install
npm run build
npm test
```

Please run `npm test` before submitting a PR. If you're adding a new transport type or a significant feature, open an issue first to discuss it.

---

## License

MIT — see [LICENSE](./LICENSE)

---

*Built to solve a real problem. If you're on Claude Code, Anthropic already solved this for you in January 2026. If you're not — this is for you.*
