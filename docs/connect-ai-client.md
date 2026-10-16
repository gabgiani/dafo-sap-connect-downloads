# Connect an AI client

[← Documentation index](../README.md#documentation)

Any AI client that speaks MCP over **Streamable HTTP** and runs **on the same computer** as DAFO SAP Connect can use it. This page explains what you need, how to configure the most common clients and which tools to enable.

## Before you start

1. DAFO SAP Connect is installed and the SAP connection is saved and tested (setup page → connection gear → **Connection**).
2. The tools you want the AI to use exist and are enabled: read tools, published SAP queries and the investigations you selected. See [Organizing SAP information for the AI](sap-specialist-guide.md).
3. You have an access token for this connection (next section).

## 1. Create an access token

1. Open `http://127.0.0.1:50002/` and sign in as administrator.
2. Choose the connection in the top bar and open its **gear** (Edit connection).
3. Open **Access tokens**, enter a name that identifies the client (for example *Swarm – sales team* or *Reporting script*) and choose **Create token**. An expiry date is optional under **Advanced options**.
4. **Copy** the token and paste it into the client. Treat it like a password: never paste it into a chat prompt.

Create one token per client or application. The **Activity** view records, for each call, the token name, the MCP operation, the tool name, the HTTP status and the duration — never the arguments or the returned data. **Revoke** stops a token immediately without affecting the others.

## 2. Find the MCP URL

```text
http://127.0.0.1:<port>/mcp
```

The first connection uses port `50002`. Each additional SAP connection has its own port, shown in its connection settings; its tokens are valid only on that port. `localhost` can be used instead of `127.0.0.1`.

> **Same computer only.** The gateway listens on the loopback interface. Cloud-hosted connectors (for example connectors added in a web chat product) run on the vendor's servers and cannot reach it. Web pages cannot call it either: requests with a browser `Origin` header are rejected. Use a desktop client, a local agent or a server-side application on this computer.

## 3. Configure the client

### DAFO Swarm

In **Add MCP server**:

| Field | Value |
|---|---|
| Server URL | `http://127.0.0.1:50002/mcp` (or the connection's port) |
| Server label | A short name, for example `SAP` |
| Authorization token | The access token, **without** the `Bearer ` prefix |

Choose **Connect and list tools**, select the tools for this chat or agent and save. Repeat **Connect and list tools** whenever the administrator adds, removes or republishes tools.

### Claude Desktop

Claude Desktop's *custom connectors* connect from Anthropic's cloud and cannot reach `127.0.0.1`. Use a local MCP server entry with the [`mcp-remote`](https://github.com/geelen/mcp-remote) adapter (requires Node.js). Merge this into `mcpServers` in `claude_desktop_config.json`, keeping your other entries:

```json
{
  "mcpServers": {
    "dafo-sap": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote@0.14.3",
        "http://127.0.0.1:50002/mcp",
        "--allow-http",
        "--transport", "http-only",
        "--header", "Authorization:${DAFO_SAP_AUTH}"
      ],
      "env": { "DAFO_SAP_AUTH": "Bearer <access token>" }
    }
  }
}
```

The header value is passed through an environment variable so the space after `Bearer` survives on every platform. Restart Claude Desktop; **Settings → Developer** shows the server as running.

### VS Code (GitHub Copilot agent mode)

Add the server to `.vscode/mcp.json` in a workspace, or to your user MCP configuration. VS Code asks for the token once and stores it securely:

```json
{
  "inputs": [
    { "type": "promptString", "id": "dafo-sap-token", "description": "DAFO SAP Connect access token", "password": true }
  ],
  "servers": {
    "dafo-sap": {
      "type": "http",
      "url": "http://127.0.0.1:50002/mcp",
      "headers": { "Authorization": "Bearer ${input:dafo-sap-token}" }
    }
  }
}
```

### Any other MCP client

| Setting | Value |
|---|---|
| Transport | Streamable HTTP (sometimes called *HTTP* or *http-only*); SSE-only clients are not supported |
| URL | `http://127.0.0.1:<port>/mcp` |
| Header | `Authorization: Bearer <access token>` |
| Sessions | Not required: every request is self-contained |

If a client can only start local *stdio* servers, use `mcp-remote` as shown for Claude Desktop.

### Verification status

| Client | Status |
|---|---|
| DAFO Swarm | Used and verified with this gateway |
| Claude Desktop with `mcp-remote` | Verified for connection and tool listing |
| VS Code, other MCP clients | Standard MCP configuration; not yet verified by DAFO |
| Official MCP SDKs (Python, TypeScript, Rust) | See [Use the MCP server from an application](connect-from-an-app.md) |

## 4. Choose which tools the AI gets

Fewer, well-chosen tools give better answers. Every tool's name, description and argument schema is sent to the model on every turn, and a long list makes it more likely to choose the wrong tool or stop after the first read.

| Enable | Why |
|---|---|
| `search_sap_knowledge`, `get_sap_knowledge` | Let the AI fetch only the guidance card it needs, instead of carrying all guidance in the prompt. |
| `describe_sap_tool_fields` | Lets the AI ask what a status code or column means instead of guessing. |
| The `investigate_*` tools of the topics this chat handles | One call runs every related read of the topic; the AI cannot forget a step. |
| The `sap_read_*` tools of those topics | For follow-up questions about a single record. |
| `list_sap_queries` and the `sap_query_*` tools the chat needs | For lists, joins, history and scheduled data. |

The administrator can shrink the catalogue on the server side as well: selecting investigations publishes only the read tools they need. See [Why the AI is not given every endpoint](sap-specialist-guide.md#why-the-ai-is-not-given-every-endpoint).

## 5. Try it

Ask questions that name an identifier the tools can look up:

> Check whether production order 154 can be built: read its components and the stock of each material in the production warehouse. Separate confirmed values from what cannot be concluded.

> Show the open lines of sales order 1215 and the production orders created from it.

The AI should call the tools, quote the SAP values they returned and state the limits it was given (for example, that an empty result is not proof that no record exists).

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| HTTP `401` | Missing, wrong, revoked or expired token, or a token of another connection. Create or copy the token again from this connection's **Access tokens**. |
| HTTP `403` | The request did not use `127.0.0.1:<port>` or `localhost:<port>` as host, or it carried a browser `Origin` header (web page or browser extension). Call from a desktop or server-side client on the same computer. |
| HTTP `404` on `/mcp` | The port does not belong to a running connection. Check the port in the connection settings. |
| Connection refused | DAFO SAP Connect is not running, or the connection's MCP port could not open. Check the setup page. |
| A tool is missing | It is disabled, hidden by the access mode, not needed by the selected investigations, or (DI API) not validated. Refresh the client's tool list after every change. |
| *SAP is not reachable right now* | The gateway could not reach SAP (VPN, network or SAP server). Nothing was read; try again later. |
| *The gateway could not sign in to SAP* | Credentials, company database or permissions of the integration user; check the connection in setup. |
| *Query not available* | A scheduled query has no complete snapshot yet, or its definition changed. Run it from setup with **Run query now**. |
| Rows are empty but data exists in SAP | SAP may hide records from the integration user (authorization groups, data ownership). Setup's **Check read** shows *Read permitted · 0 rows* in that case. |
