# deepseek-web-bridge

Let a **DeepSeek web chat session** do work for any AI client — keeping subagent work off the metered API, while the parent keeps its own model and conversation.

**One setup script.** `node scripts/setup.mjs` generates your token, writes the config, and prints the exact next steps for your chosen path.

## What this is

A browser chat already has a model, a login, and a tool surface. Instead of spending API quota on subagent work — exploration, batch search, parallel sub-tasks, the expensive part — route it to a DeepSeek web session you are already signed into. The parent keeps its own model, context, and tools.

```
 ┌────────────────────────────────┐         ┌──────────────────────────────────┐
 │  Your AI client                │         │  Browser (DeepSeek++ fork)        │
 │  Claude CLI / Codex CLI / dsh  │         │  running the delegate loop        │
 │                                │         │                                  │
 │  calls delegate_to_ds_web      │         │  claims work, runs it in a        │
 │  (blocks until done)           │         │  fresh DS chat session,           │
 └────────────┬───────────────────┘         │  settles the result, deletes it   │
              │                              └──────────────┬───────────────────┘
              │            ┌──────────────────┐           │
              │            │ deepseek-web-mcp │           │
              └───────────▶│  standalone MCP  │◀───────────┘
                           │  one queue,      │
                           │  one token       │
                           └──────────────────┘
```

## Two paths, pick one

### Path 1: CLI only (simplest, no dsh)

Use DeepSeek web as a subagent from **Claude CLI** or any MCP client. ~5 minutes.

```sh
git clone --recursive https://github.com/lisycotana/deepseek-web-bridge.git
cd deepseek-web-bridge
node scripts/setup.mjs     # generates token, writes .env, prints next steps
```

The script tells you exactly what to run. In short:
1. Start the MCP server: `cd deepseek-web-mcp && node index.js`
2. Build + load the browser delegate (the fork): `cd deepseek-pp-delegate && npm i && npm run build:chrome`
3. Add the MCP server to your CLI: `claude mcp add --transport http deepseek-web http://127.0.0.1:8765/mcp --header "Authorization: Bearer <token>"`
4. Delegate: `> 用 delegate_to_ds_web 列出当前目录的文件`

### Path 2: dsh (full integration)

Use DeepSeek web as a dsh subagent via `subagent_deepseek_web`. Same browser side, plus two dsh plugins.

```sh
git clone --recursive https://github.com/lisycotana/deepseek-web-bridge.git
cd deepseek-web-bridge
node scripts/setup.mjs     # pick "dsh" when asked
```

Then install the dsh plugins and configure the profile patch (the script prints the exact lines). See [`dsh-deepseek-plugins/README.md`](dsh-deepseek-plugins/README.md) for the preset row.

## Choosing the model mode

Two choices are independent and each defaults to the safe value.

### DS web model mode (browser side)

When starting the delegate from the sidebar DevTools console, pass `modelType` to pick the DeepSeek web mode:

```javascript
// default (standard chat) — the default when omitted
chrome.runtime.sendMessage({ type: 'START_DELEGATE' }, (r) => console.log(r))

// expert (deep thinking / Reasoner)
chrome.runtime.sendMessage({ type: 'START_DELEGATE', payload: { modelType: 'expert' } }, (r) => console.log(r))

// vision (multimodal)
chrome.runtime.sendMessage({ type: 'START_DELEGATE', payload: { modelType: 'vision' } }, (r) => console.log(r))

// also enable web search for the whole loop
chrome.runtime.sendMessage({ type: 'START_DELEGATE', payload: { searchEnabled: true } }, (r) => console.log(r))
```

| Mode | `modelType` | What it does |
| --- | --- | --- |
| Standard | `default` (or omitted) | normal chat |
| Deep thinking | `expert` | DeepSeek-Reasoner, deeper reasoning (slower) |
| Multimodal | `vision` | image understanding |

Web search is an independent toggle: `searchEnabled: true` lets the delegate use DeepSeek's web search per task.

### dsh agent preset (dsh path only)

The dsh side picks which built-in toolset the **parent** agent gets. Default is `standard` (full toolset). See [Which agent preset to use](dsh-deepseek-plugins/dsh-subagent-deepseek-web/README.md#which-agent-preset-to-use) for the four options (`standard`, `code`, `minimal`, `cordis`) and how to set it.

## Requirements

- **Node.js >= 20** (no other runtime deps — the server is pure built-in modules)
- **A browser** (Chrome or Edge) to load the delegate extension
- **A DeepSeek account** you are signed into at `chat.deepseek.com`

## What's in this repo

This is an **umbrella repo**. The actual code lives in three submodules:

| Submodule | What it is |
| --- | --- |
| [`deepseek-web-mcp`](https://github.com/lisycotana/deepseek-web-mcp) | The core: a standalone MCP server. Any client calls `delegate_to_ds_web`. |
| [`dsh-deepseek-plugins`](https://github.com/lisycotana/dsh-deepseek-plugins) | Two plugins for DeepSeek Harness: expose local tools over MCP, and a dsh subagent provider. Only needed for Path 2. |
| [`deepseek-pp-delegate`](https://github.com/lisycotana/deepseek-pp-delegate) | A fork of DeepSeek++ that drives the browser side: creates a fresh DS chat per task, deletes it when done. |

Clone with `--recursive` to get all three, or run `git submodule update --init --recursive` after cloning.

## Troubleshooting

**`git clone` didn't get the submodules.** Run `git submodule update --init --recursive` in the repo.

**The MCP server won't start (EADDRINUSE).** Another process is on port 8765. Set `DWM_PORT=8766` in `deepseek-web-mcp/.env`.

**Claude CLI says "tools fetch failed".** Check the server is running (`curl http://127.0.0.1:8765/mcp`), the token matches `.env`, and you used `--header "Authorization: Bearer <token>"`.

**The delegate doesn't pick up tasks.** Make sure the fork is loaded (not the store version of DS++), the MCP server is configured in the DS++ sidebar, and you ran `chrome.runtime.sendMessage({ type: "START_DELEGATE" })` from the sidebar DevTools console.

**Tasks time out.** Default is 15 minutes. Set `DWM_TASK_TIMEOUT_MS` in `.env` to raise it.

## CLI compatibility

The server speaks standard MCP Streamable HTTP with Bearer auth. **Claude CLI is tested and confirmed.** Other CLIs (Codex CLI, Cursor, etc.) may differ in how they register HTTP MCP servers or pass auth headers — the server side needs no changes, only the client registration command. Open an issue if your CLI needs a different auth mechanism.

## Acknowledgements

This project builds on open-source platforms. Credit for the platforms belongs to their authors; this project adds the bridge.

- **[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)** (Apache-2.0, DeepSeek AI) — the agent harness the dsh plugins target.
- **[DeepSeek++](https://github.com/zhu1090093659/deepseek-pp)** (Apache-2.0, zhu1090093659) — the browser extension the fork is based on.
- **[agentdock](https://github.com/uvwt/agentdock)** (Apache-2.0, uvwt) — the inspiration: routing browser-chat work to a local runtime over MCP.

Not affiliated with or endorsed by any of the above.

## License

- `deepseek-web-mcp` and the dsh plugins: **MIT**.
- The DS++ fork: **Apache-2.0** (upstream's license).
- Platforms are the property of their respective authors.
