# CLAUDE.md - Proxima Development Guide

## Project Overview

Proxima is a local MCP (Model Context Protocol) server and Multi-AI Gateway that routes requests to ChatGPT, Claude, Gemini, and Perplexity through a single API endpoint. It runs as an Electron desktop app with an embedded MCP server, REST API, and browser automation layer.

**Version:** 3.0.0
**License:** Personal Use (non-commercial)

## Architecture

Proxima uses a multi-process architecture:

```
AI Coding Tools (Cursor, VS Code, Claude Desktop)
    |
    v  MCP Protocol
MCP Server (src/mcp-server-v3.js) -- 44 tools
    |
    v  TCP IPC (port 19222)
Electron Main Process (electron/main-v2.cjs)
    |-- IPC Server (port 19222)
    |-- REST API Server (port 3210, OpenAI-compatible)
    |-- BrowserView per provider (Claude, ChatGPT, Gemini, Perplexity)
    |     |-- Network interceptors (SSE stream parsing)
    |     |-- Anti-detection headers
    |     '-- Response capture logic
    '-- Settings management
```

### Key Components

| Component | File | Purpose |
|-----------|------|---------|
| MCP Server | `src/mcp-server-v3.js` | 44 MCP tools for AI coding tools |
| Electron Main | `electron/main-v2.cjs` | Desktop app, IPC server, browser management |
| REST API | `electron/rest-api.cjs` | OpenAI-compatible `/v1/chat/completions` endpoint |
| Browser Manager | `electron/browser-manager.cjs` | BrowserView lifecycle, network interception, SSE parsing |
| UI | `electron/index-v2.html` | Settings and provider tab interface |
| Preload Scripts | `electron/preload.cjs`, `electron/provider-preload.cjs` | Renderer/provider bridge |
| Python SDK | `sdk/proxima.py` | Python client library |
| JavaScript SDK | `sdk/proxima.js` | Node.js client library |

## Directory Structure

```
proxima/
├── assets/              # App icons, logos, demo video, screenshots
├── electron/            # Electron desktop app (CommonJS)
│   ├── main-v2.cjs      # Main process (~2,846 lines)
│   ├── browser-manager.cjs  # BrowserView management (~41KB)
│   ├── rest-api.cjs     # REST API gateway (~900 lines)
│   ├── index-v2.html    # Application UI (~55KB)
│   ├── preload.cjs      # Renderer preload bridge
│   ├── provider-preload.cjs  # Provider page preload
│   └── package.json     # Electron sub-package (CommonJS)
├── src/                 # Source code (ES Modules)
│   ├── mcp-server-v3.js # MCP server with 44 tools (~1,573 lines)
│   └── enabled-providers.example.json  # Provider config template
├── sdk/                 # Client SDKs
│   ├── proxima.py       # Python SDK
│   └── proxima.js       # JavaScript SDK
├── package.json         # Root config (ESM, scripts, build config)
└── package-lock.json    # Dependency lock
```

## Tech Stack

- **Runtime:** Node.js 18+
- **Desktop Framework:** Electron 33
- **Module System:** ESM (`src/`) + CommonJS (`electron/`)
- **Schema Validation:** Zod 4
- **MCP SDK:** @modelcontextprotocol/sdk 1.25+
- **Database:** sql.js (SQLite in JS)
- **Build:** electron-builder (NSIS installer, Windows x64)

### Dependencies

**Production:** `@modelcontextprotocol/sdk`, `sql.js`, `zod`
**Dev:** `electron`, `electron-builder`, `concurrently`, `http-server`

## Commands

| Command | Description |
|---------|-------------|
| `npm start` | Launch the Electron desktop app |
| `npm run mcp` | Start the standalone MCP server |
| `npm run build` | Build Windows x64 installer |
| `npm run build:installer` | Build NSIS installer variant |

## Module System Convention

- **`src/` directory:** ES Modules (`import`/`export`). The root `package.json` has `"type": "module"`.
- **`electron/` directory:** CommonJS (`.cjs` extension, `require`/`module.exports`). Electron's sub-package has `"type": "commonjs"`.
- **`sdk/` directory:** CommonJS for Node.js compatibility.

Always use `.cjs` extension for new Electron files and standard `.js` for `src/` files.

## Code Patterns and Conventions

### MCP Tool Registration

All 44 tools follow this pattern:

```javascript
server.tool(
    'tool_name',
    {
        param1: z.string().describe('Description'),
        param2: z.array(z.string()).optional().describe('Optional files'),
    },
    async ({ param1, param2 }) => {
        const disabled = checkDisabled('provider_name');
        if (disabled) return disabled;

        try {
            return toolResponse(await provider.action(param1, param2));
        } catch (err) {
            return toolError(err);
        }
    }
);
```

Key conventions:
- Always call `checkDisabled()` before executing provider logic
- Use `toolResponse()` for success, `toolError()` for failures
- Use Zod schemas for parameter validation
- Optional `files` parameter for file attachment support

### Tool Categories (44 total)

| Category | Tools |
|----------|-------|
| Search (8) | `deep_search`, `pro_search`, `youtube_search`, `reddit_search`, `news_search`, `academic_search`, `image_search`, `math_search` |
| Code (7) | `generate_code`, `explain_code`, `debug_code`, `optimize_code`, `review_code`, `verify_code`, `research_fix` |
| AI Providers (6) | `ask_chatgpt`, `ask_claude`, `ask_gemini`, `ask_all_ais`, `compare_ais`, `smart_query` |
| Content (8) | `brainstorm`, `translate`, `fact_check`, `find_stats`, `how_to`, `writing_help`, `summarize_url`, `generate_article` |
| Analysis (5) | `analyze_document`, `analyze_image_url`, `extract_data`, `compare`, `generate_image_prompt` |
| File Analysis (2) | `analyze_file`, `review_code_file` |
| Window Control (4) | `show_window`, `hide_window`, `toggle_window`, `set_headless_mode` |
| Session (2) | `new_conversation`, `clear_cache` |
| Status (2) | `router_stats`, `get_typing_status` |

### Response Capture

Each AI provider has a custom SSE (Server-Sent Event) parser in `browser-manager.cjs`:
- **Claude:** Parses `content_block_delta` events, tracks multiple content blocks, returns longest
- **ChatGPT:** Watches `/backend-api/conversation`, extracts `data.message.content.parts`
- **Gemini:** Extracts text from streaming JSON responses
- **Perplexity:** Generic event-stream format handler

### Error Handling

- Tool-level: wrap in try/catch, return `toolError(err)`
- IPC-level: connection retry with timeout
- REST API: HTTP status codes with JSON error bodies
- Provider-level: typing status checks to prevent desync (5s wait max)

### Caching

- 5-minute TTL per provider
- Message-based cache keys
- Automatic invalidation on expiry

## REST API

**Base URL:** `http://localhost:3210`

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/chat/completions` | POST | Main chat endpoint (OpenAI-compatible) |
| `/v1/models` | GET | List available models |
| `/v1/functions` | GET | Function catalog |
| `/v1/stats` | GET | Response time statistics |
| `/v1/conversations/new` | POST | Start fresh conversation |

### Model Aliases

The REST API accepts these model name aliases:
- **ChatGPT:** `chatgpt`, `gpt`, `gpt-4`, `gpt-4o`, `gpt-4.5`, `openai`
- **Claude:** `claude`, `claude-3`, `claude-3.5`, `claude-4`, `anthropic`, `sonnet`, `opus`, `haiku`
- **Gemini:** `gemini`, `gemini-pro`, `gemini-2`, `gemini-2.5`, `google`, `bard`
- **Perplexity:** `perplexity`, `pplx`, `sonar`
- **Smart Router:** `auto` (auto-pick best), `all` (query all providers)

## Configuration

### Settings File Locations

| Platform | Path |
|----------|------|
| Windows | `%APPDATA%\proxima\settings.json` |
| macOS | `~/Library/Application Support/proxima/settings.json` |
| Linux | `~/.config/proxima/settings.json` |

### Settings Structure

```json
{
  "providers": {
    "perplexity": { "enabled": true, "loggedIn": false },
    "chatgpt": { "enabled": true, "loggedIn": false },
    "claude": { "enabled": false, "loggedIn": false },
    "gemini": { "enabled": true, "loggedIn": false }
  },
  "ipcPort": 19222,
  "theme": "dark",
  "headlessMode": false,
  "startMinimized": false
}
```

### Provider Configuration

`src/enabled-providers.json` controls which providers are available to the MCP server. Copy `src/enabled-providers.example.json` to get started. This file is gitignored.

## Ports

| Port | Service |
|------|---------|
| 3210 | REST API (OpenAI-compatible) |
| 19222 | IPC server (Electron <-> MCP communication) |

## Build and Distribution

- **Target:** Windows 10/11 (x64) via NSIS installer
- **Output:** `dist/Proxima-Setup-3.0.0.exe`
- **Includes:** `electron/`, `src/`, `sdk/`, `assets/`, `node_modules/`
- macOS and Linux paths are defined in code but installers are not actively built

## Testing

No automated testing framework is configured. Testing is done manually through the Electron app and MCP tool invocations. When making changes, verify by:

1. Running `npm start` and testing through the Electron UI
2. Running `npm run mcp` and invoking tools via an MCP client
3. Testing REST API endpoints with curl or the SDKs

## Security Model

### REST API Authentication

The REST API requires an API key on all endpoints except the docs page (`/` and `/docs`).

- **Auto-generated** on first run if not configured; printed to console on startup.
- **Fixed key** via `PROXIMA_API_KEY` environment variable or `apiKey` in settings.
- **Pass as** `Authorization: Bearer <key>` or `X-API-Key: <key>` header.

### CORS Policy

Cross-origin requests are only accepted from `localhost` / `127.0.0.1` origins. All other origins are silently denied (no `Access-Control-Allow-Origin` header). This prevents any external website from accessing the API via the browser.

### IPC Authentication

The IPC server (port 19222) uses a shared secret handshake:
1. Electron writes a random secret to `<userData>/ipc-secret` (file mode `0600`).
2. MCP server reads the secret on connect and sends `{"action":"auth","secret":"..."}`.
3. Unauthenticated connections are rejected and disconnected.
4. Override via `PROXIMA_IPC_SECRET` environment variable.

### File Path Restrictions

The `files` parameter and file analysis tools block reads from:
- SSH keys, GPG keys, AWS/Azure/GCloud credentials
- `.env` files, `.netrc`, `.npmrc`, `.pypirc`
- Browser profile directories (prevents cookie theft)
- Any file named `id_rsa`, `credentials.json`, `service-account.json`, etc.

### Script Execution Guard

The `executeScript` IPC command blocks scripts containing:
`document.cookie`, `localStorage`, `sessionStorage`, `indexedDB`, `navigator.credentials`, `fetch(`, `XMLHttpRequest`, `window.open`, `eval(`

### URL Validation

- `openExternal` only allows `http:` and `https:` protocols (blocks `file:`, `smb:`, custom protocols).
- `navigate` restricts URLs to known provider domains per provider.
- Certificate error bypass uses exact domain suffix matching (not substring).

## Important Notes for AI Assistants

- **No linter or formatter configured** -- follow existing code style (2-space indent, single quotes in JS, descriptive variable names).
- **No TypeScript** -- the project is pure JavaScript. Do not introduce TypeScript files.
- **Dual module system** -- use `.cjs` + `require()` in `electron/`, use `.js` + `import` in `src/`.
- **Anti-detection is critical** -- the Electron app spoofs browser identity. Do not modify user-agent strings or automation flags without understanding the implications.
- **Provider parsers are fragile** -- SSE response parsing in `browser-manager.cjs` is tightly coupled to each provider's API format. Changes to provider websites may break parsing.
- **`enabled-providers.json` is gitignored** -- never commit user provider configurations.
- **The `files` parameter** on MCP tools reads local files and attaches them to prompts. File type detection determines formatting (code blocks vs plain text). Sensitive paths are blocked (see Security Model above).
- **IPC protocol** -- MCP server communicates with Electron via raw TCP JSON messages on port 19222. Messages are newline-delimited JSON. Authentication is required (see Security Model above).
- **Smart Router** (`smart_query` tool) auto-selects the best available provider with retry logic and fallback.
