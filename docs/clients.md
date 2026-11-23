# Client setup

[README](../README.md) · [Configuration](configuration.md) · [Troubleshooting](troubleshooting.md)

Use Node.js **≥22.13.0** and a client that launches MCP servers over **stdio**.
`mcp-web-tools` is a local process, not an HTTP MCP endpoint. No global npm
installation is required.

Examples here are **illustrative configuration**, checked against the linked
client documentation on 2026-11-20. They are not records of live client/model
tests. Tool use depends on the selected model, client, permissions, and context
window; no model compatibility or token-efficiency claim is implied.

## Generic stdio recipe

Configure your client with:

- Command: `npx`
- Arguments: `["-y", "mcp-web-tools"]`
- Environment: `WEBTOOLS_SEARCH_PROVIDER=tavily`, `TAVILY_API_KEY=<your key>`

The extraction provider defaults to Tavily when search uses Tavily. For other
combinations, use the [configuration reference](configuration.md).

`npx` obtains/runs the npm package and may access the registry on first use.
For a reproducible published setup, select an available release and put its
exact version after `mcp-web-tools@` in the arguments. Changes made after the
latest release are not included in the published package until the next release.

The client starts the process and exchanges JSON-RPC on stdin/stdout. Launching
the command in a terminal without an MCP client may appear to do nothing; there
is no interactive prompt or web UI. Logs use stderr.

## LM Studio

[LM Studio's official MCP guide](https://lmstudio.ai/docs/app/mcp) documents MCP
hosting from version 0.3.17, with local and remote servers supported from
0.3.17 build 10. Use a current version with MCP support.

1. Open the **Program** tab in the right-hand sidebar.
2. Select **Install → Edit mcp.json**.
3. Add this entry under the existing `mcpServers` object. The example below is
   a complete file for a setup with no other servers.
4. Replace the key placeholder locally and save. Check the server/tool status
   in the app before asking the model to search.

```json
{
  "mcpServers": {
    "mcp-web-tools": {
      "command": "npx",
      "args": ["-y", "mcp-web-tools"],
      "env": {
        "WEBTOOLS_SEARCH_PROVIDER": "tavily",
        "TAVILY_API_KEY": "tvly-REPLACE_WITH_YOUR_KEY"
      }
    }
  }
}
```

LM Studio follows `mcp.json` notation. Do not paste a second top-level
`mcpServers` object inside an existing one. Keep credential-bearing configuration
private; this example deliberately uses a placeholder, not a secret.

A local model does not make this workflow offline: the search/extraction APIs
still receive queries and URLs. LM Studio's own guide also notes MCP context
usage; read fewer pages at once when working with a small context window.

## OpenCode V2

Sources: [MCP servers](https://opencode.ai/docs/mcp-servers) and
[configuration](https://opencode.ai/docs/config).

Use `opencode.jsonc` in the project root, or
`~/.config/opencode/opencode.jsonc` for a user-wide setup. The alternative project
location `.opencode/opencode.jsonc` is also supported. `.opencode.json` is not
the documented V2 project filename.

Make `TAVILY_API_KEY` available to the OpenCode process through your environment.
Merge this complete file shape with existing settings:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "mcp-web-tools": {
        "type": "local",
        "command": ["npx", "-y", "mcp-web-tools"],
        "environment": {
          "WEBTOOLS_SEARCH_PROVIDER": "tavily",
          "TAVILY_API_KEY": "{env:TAVILY_API_KEY}"
        }
      }
    }
  }
}
```

V2 nests named servers under **`mcp.servers`** and uses **`environment`** for
local-server variables. `{env:TAVILY_API_KEY}` is OpenCode's interpolation
syntax; `$TAVILY_API_KEY` inside JSON is just a literal string. An unset
environment reference becomes empty and the Tavily configuration will fail.

OpenCode watches configuration and reloads MCP servers after a save. Run
`opencode mcp list` or use `/mcps` to inspect the connection; `/reload` forces a
full configuration reload if needed. No OpenCode restart is required just to
apply a saved config change.

OpenCode may prefix tool names with the server name or expose them through
Code Mode. Look for the six [underlying tool names](tools.md), rather than
assuming the client displays them without a namespace.

## Other clients

For clients using `mcpServers`/`command`/`args`/`env`, the LM Studio example
shows the process configuration. Follow your client's documentation for its
configuration location and how to reconnect it. Do not copy OpenCode's
`mcp.servers` shape into these clients.

If your client cannot find `npx` or `node`, inspect its launch environment.
Desktop apps do not necessarily inherit the PATH or secrets of an interactive
shell. Use an absolute executable path if needed. Avoid shell wrappers that
print banners to stdout, which is reserved for MCP messages.

## Run this checkout

To run the source tree instead of a published npm version, run these from
the repository root:

```sh
npm ci
npm run build
```

Then configure the client to launch the compiled entry point. Illustrative
`mcpServers` file; replace both filesystem paths and the key:

```json
{
  "mcpServers": {
    "mcp-web-tools": {
      "command": "/absolute/path/to/node",
      "args": ["/absolute/path/to/mcp-web-tools/build/index.js"],
      "env": {
        "WEBTOOLS_SEARCH_PROVIDER": "tavily",
        "TAVILY_API_KEY": "tvly-REPLACE_WITH_YOUR_KEY"
      }
    }
  }
}
```

For OpenCode, replace its `command` array with
`["/absolute/path/to/node", "/absolute/path/to/mcp-web-tools/build/index.js"]`
and keep the `environment` object. Rebuild and reconnect this MCP process after
source changes. [Contributor checks](../CONTRIBUTING.md) exercise local fixtures
without provider keys.

## Verify the connection and workflow

First inspect the tool list. All six tools should be present; explicitly
disabling extraction leaves five and omits only `web_read`. Tool discovery alone
does not verify a provider key, upstream availability, or model tool use.

For an optional live check, ask the assistant:

> Search for official documentation on a topic I choose, with at most 3 results.
> Read one selected source and answer with its URL. Tell me if either tool fails.

This is an **illustrative prompt**, not a recorded result. Running it sends data
to providers and may consume credits. Review the tool calls, returned source
URLs, and any per-URL failures rather than treating a fluent model answer as
proof of success.
