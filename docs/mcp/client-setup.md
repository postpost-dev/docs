---
title: "MCP Client Setup Guide"
description: "Detailed setup instructions for connecting PostPost MCP to different AI clients."
---

Detailed setup instructions for connecting PostPost MCP to different AI clients.

## Prerequisites

1. **PostPost account** — Sign up at [postpost.dev](https://postpost.dev)
2. **API key** — Get from **API** in sidebar (starts with `sk_`)
3. **Connected social account** — At least one platform connected

> **Auth headers:** `x-api-key: sk_...` is the required header for direct REST API calls to `api.postpost.dev`. The `Authorization: Bearer sk_...` format is only supported through the MCP proxy server, which translates it to `x-api-key` internally. When making direct REST API calls, always use `x-api-key`. The examples below use `Authorization: Bearer` for MCP client configurations, where it is the standard format.

> **MCP client header:** The MCP server automatically sends an `x-postpost-client: "mcp"` header on internal API calls. This identifies MCP traffic server-side. Your account must have `mcpAccess` enabled — Starter plan users do not have API or MCP access and will receive a `403` error. Upgrade to Pro or Premium for MCP access.

---

## Claude Code (CLI)

Claude Code is Anthropic's command-line interface for Claude.

### Option 1: CLI Command

```bash
claude mcp add postpost --transport http https://mcp.postpost.dev --header "Authorization: Bearer sk_YOUR_API_KEY"
```

### Option 2: Global Configuration

Edit `~/.claude.json`:

```json
{
  "mcpServers": {
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

### Option 3: Project Configuration

Create `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

### Verification

After restarting Claude Code:

```bash
# Check connected MCP servers
/mcp

# Test PostPost connection
"Show my connected social accounts"
```

---

## Claude Desktop

Claude Desktop is Anthropic's desktop application for Claude.

### Setup Steps

1. Open Claude Desktop
2. Go to **Settings** → **Developer** → **Edit Config**
3. Add the PostPost configuration:

```json
{
  "mcpServers": {
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

4. Save the file
5. Restart Claude Desktop completely

### Verification

Start a new conversation and ask:

```text
"What MCP tools do you have available?"
```

Claude should list the 18 active PostPost tools (3 additional tools — `linkedin_posts`, `linkedin_post_comments`, `linkedin_post_reactions` — are temporarily disabled pending LinkedIn approval).

---

## Cursor

Cursor is an AI-powered code editor with MCP support.

### Project Configuration

Create `.cursor/mcp.json` in your project:

```json
{
  "mcpServers": {
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

### Global Configuration

For all projects, edit `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

### Verification

1. Restart Cursor
2. Open the AI chat
3. Ask: "List my PostPost connections"

---

## VS Code with Continue

Continue is an open-source AI coding assistant for VS Code.

### Configuration

Add to your Continue configuration:

```json
{
  "mcpServers": {
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

---

## Windsurf

Windsurf is an AI-powered IDE by Codeium.

### Configuration

Create or edit `.windsurf/mcp.json`:

```json
{
  "mcpServers": {
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

---

## OpenClaw / mcporter CLI

[mcporter](https://github.com/openclaw/mcporter) is a CLI tool for connecting MCP servers to OpenClaw and other agents.

### List Available Tools

```bash
mcporter list --http-url https://mcp.postpost.dev --name postpost
```

### With Authentication

```bash
mcporter list \
  --http-url https://mcp.postpost.dev \
  --name postpost \
  --header "Authorization: Bearer sk_YOUR_API_KEY"
```

### Persist Configuration

```bash
mcporter list \
  --http-url https://mcp.postpost.dev \
  --name postpost \
  --header "Authorization: Bearer sk_YOUR_API_KEY" \
  --persist config/mcporter.json
```

### Configuration File

Create `config/mcporter.json`:

```json
{
  "servers": {
    "postpost": {
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

See [OpenClaw Integration Guide](./openclaw.md) for complete autonomous agent examples.

---

## Multiple MCP Servers

You can use PostPost alongside other MCP servers. They don't conflict — tools are merged.

### Example: PostPost + Context7

```json
{
  "mcpServers": {
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    },
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

### Example: PostPost + GitHub + Filesystem

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "ghp_YOUR_TOKEN"
      }
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/files"]
    },
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer sk_YOUR_API_KEY"
      }
    }
  }
}
```

---

## Programmatic Access (Python)

For testing or custom integrations, use Python:

### Installation

```bash
pip install mcp httpx
```

### Basic Example

```python
import asyncio
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async def main():
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.postpost.dev", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # List available tools
            tools = await session.list_tools()
            print(f"Available tools: {len(tools.tools)}")
            for tool in tools.tools:
                print(f"  - {tool.name}: {tool.description}")

            # Get connections
            result = await session.call_tool("list_connections", {})
            print(result.content[0].text)

asyncio.run(main())
```

### Full Workflow Example

```python
import asyncio
from datetime import datetime, timedelta
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async def schedule_post_workflow():
    """Complete workflow: check connections, create post, verify."""
    headers = {"Authorization": "Bearer sk_YOUR_API_KEY"}

    async with streamablehttp_client("https://mcp.postpost.dev", headers=headers) as (read, write, _):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # Step 1: Get connections
            print("Getting connections...")
            connections = await session.call_tool("list_connections", {})
            print(connections.content[0].text)

            # Step 2: Schedule a post
            tomorrow = (datetime.utcnow() + timedelta(days=1)).replace(
                hour=9, minute=0, second=0, microsecond=0
            ).isoformat() + "Z"

            print(f"\nScheduling post for {tomorrow}...")
            result = await session.call_tool("create_post", {
                "content": "Hello from Python MCP client!",
                "platforms": ["linkedin-YOUR_PLATFORM_ID"],
                "scheduledTime": tomorrow
            })
            print(result.content[0].text)

            # Step 3: List scheduled posts
            print("\nListing scheduled posts...")
            posts = await session.call_tool("list_posts", {
                "status": "scheduled"
            })
            print(posts.content[0].text)

asyncio.run(schedule_post_workflow())
```

---

## Programmatic Access (TypeScript)

### Installation

```bash
npm install @modelcontextprotocol/sdk
```

### Basic Example

```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

async function main() {
  const transport = new StreamableHTTPClientTransport(
    new URL("https://mcp.postpost.dev"),
    {
      headers: {
        Authorization: "Bearer sk_YOUR_API_KEY",
      },
    }
  );

  const client = new Client({
    name: "postpost-client",
    version: "1.0.0",
  });

  await client.connect(transport);

  // List tools
  const tools = await client.listTools();
  console.log(`Available tools: ${tools.tools.length}`);

  // Get connections
  const result = await client.callTool({
    name: "list_connections",
    arguments: {},
  });
  console.log(result.content[0]);

  await client.close();
}

main();
```

---

## cURL Testing

Test the MCP server directly with cURL:

### Health Check

```bash
curl https://mcp.postpost.dev/health
# {"status":"ok","service":"postpost-mcp"}
```

### Initialize Session

```bash
curl -X POST https://mcp.postpost.dev \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer sk_YOUR_API_KEY" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {
        "name": "curl-test",
        "version": "1.0.0"
      }
    }
  }'
```

### List Tools

```bash
curl -X POST https://mcp.postpost.dev \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -H "Authorization: Bearer sk_YOUR_API_KEY" \
  -H "mcp-session-id: YOUR_SESSION_ID" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list",
    "params": {}
  }'
```

---

## Environment Variables

For security, store your API key in environment variables:

### Shell

```bash
export PUBLORA_API_KEY="sk_YOUR_API_KEY"
```

### Configuration with env variable

Some clients support environment variable interpolation:

```json
{
  "mcpServers": {
    "postpost": {
      "type": "http",
      "url": "https://mcp.postpost.dev",
      "headers": {
        "Authorization": "Bearer ${PUBLORA_API_KEY}"
      }
    }
  }
}
```

---

## Troubleshooting Setup

### Tools not appearing

1. **Restart your client** — MCP servers load on startup
2. **Check JSON syntax** — Validate at [jsonlint.com](https://jsonlint.com)
3. **Verify type is "http"** — Not "url" or "sse"
4. **Check the URL** — Use `https://mcp.postpost.dev` (recommended) or `https://mcp.postpost.dev/mcp` (both work)

### Authentication errors

1. **Include "Bearer" prefix** — `Authorization: Bearer sk_...`
2. **Check key format** — Should start with `sk_`
3. **Generate new key** — At postpost.dev → **API** in sidebar

> **Note:** `x-api-key: sk_...` is the required header for direct REST API calls to `api.postpost.dev`. The `Authorization: Bearer sk_...` format is only supported through the MCP proxy server, which translates it to `x-api-key` internally. For MCP client configurations, use the `Authorization: Bearer` format.

### Connection refused

1. **Check internet** — Verify connectivity
2. **Test health endpoint** — `curl https://mcp.postpost.dev/health`
3. **Check firewall** — Ensure HTTPS is allowed

---

## Next Steps

- [Tools Reference](./tools-reference.md) — All 18 active tools with parameters
- [Examples](./examples.md) — Real-world conversation examples
- [Troubleshooting](./troubleshooting.md) — Common issues and solutions
