# Model Context Protocol (MCP) — Complete Guide

> From zero to production: a team reference for understanding, building, and deploying MCP servers and clients.

---

## Table of Contents

1. [What is MCP?](#1-what-is-mcp)
2. [Why MCP Matters](#2-why-mcp-matters)
3. [Ecosystem Support](#3-ecosystem-support)
4. [Core Architecture](#4-core-architecture)
5. [Core Primitives](#5-core-primitives)
6. [Data Layer Protocol](#6-data-layer-protocol)
7. [Transport Layer](#7-transport-layer)
8. [Lifecycle: Step-by-Step Walkthrough](#8-lifecycle-step-by-step-walkthrough)
9. [Available SDKs](#9-available-sdks)
10. [Building Your First MCP Server (Python)](#10-building-your-first-mcp-server-python)
11. [Connecting to Claude Desktop](#11-connecting-to-claude-desktop)
12. [Multi-Server Patterns](#12-multi-server-patterns)
13. [Client Concepts (Advanced)](#13-client-concepts-advanced)
14. [Security and Authorization](#14-security-and-authorization)
15. [Debugging and Tooling](#15-debugging-and-tooling)
16. [Key Best Practices](#16-key-best-practices)
17. [Further Reading](#17-further-reading)

---

## 1. What is MCP?

**Model Context Protocol (MCP)** is an open-source standard for connecting AI applications to external systems — data sources, tools, APIs, databases, and workflows.

> Think of MCP like a **USB-C port for AI applications**. Just as USB-C standardizes how electronic devices connect, MCP standardizes how AI applications connect to external systems.

### What MCP enables

| Example | What happens |
|---|---|
| AI assistant with Google Calendar + Notion | Acts as a personalized AI scheduler |
| Claude Code + Figma design | Generates an entire web app from a design file |
| Enterprise chatbot + multiple databases | Users analyze data across the org via chat |
| AI model + Blender + 3D printer | Creates and prints 3D models end-to-end |

---

## 2. Why MCP Matters

| Stakeholder | Benefit |
|---|---|
| **Developers** | Reduces time and complexity when building or integrating AI applications/agents |
| **AI apps / agents** | Access to an ecosystem of data sources, tools and apps that enhance capabilities |
| **End-users** | More capable AI that can access your data and take actions on your behalf |

**Before MCP:** every AI app had to build its own one-off integration with each data source or tool.  
**After MCP:** build once, integrate everywhere — any MCP-compatible client can use any MCP server.

---

## 3. Ecosystem Support

MCP is an open protocol with broad industry adoption:

- **AI Assistants:** Claude, ChatGPT
- **Development tools:** VS Code (GitHub Copilot), Cursor, JetBrains, MCPJam
- **Official spec:** [modelcontextprotocol.io/specification](https://modelcontextprotocol.io/specification/latest)
- **Reference servers:** [github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)

---

## 4. Core Architecture

### The Three Participants

```
┌─────────────────────────────────────┐
│     MCP Host (AI Application)        │
│  ┌──────────┐  ┌──────────┐         │
│  │MCP Client│  │MCP Client│  ...    │
│  └────┬─────┘  └────┬─────┘         │
└───────┼─────────────┼───────────────┘
        │             │
   dedicated      dedicated
   connection     connection
        │             │
   ┌────▼─────┐  ┌────▼──────────────┐
   │MCP Server│  │MCP Server         │
   │(Local)   │  │(Remote via HTTP)  │
   └──────────┘  └───────────────────┘
```

| Participant | Role |
|---|---|
| **MCP Host** | The AI application (e.g., Claude Desktop, VS Code) that manages one or more MCP clients |
| **MCP Client** | A component inside the host that maintains a **dedicated connection** to one MCP server |
| **MCP Server** | A program (local or remote) that exposes tools, resources, and prompts to clients |

**Key rule:** One MCP client per MCP server. The host instantiates a new client for each server it connects to.

### Two Layers

| Layer | Responsibility |
|---|---|
| **Data Layer** | JSON-RPC 2.0 protocol: message structure, lifecycle, primitives (tools/resources/prompts), notifications |
| **Transport Layer** | Communication channel: STDIO for local, Streamable HTTP for remote |

> MCP focuses solely on the protocol for context exchange — it does **not** dictate how AI applications use LLMs or manage context internally.

---

## 5. Core Primitives

Primitives are the most important concept in MCP. They define **what servers and clients can offer each other**.

### Server-side Primitives

| Primitive | What it is | Who controls it | Examples |
|---|---|---|---|
| **Tools** | Executable functions the LLM can invoke | **Model** (AI decides when to call) | Search flights, send messages, create events |
| **Resources** | Read-only data sources providing context | **Application** | File contents, DB schemas, API docs |
| **Prompts** | Reusable parameterized templates | **User** (explicit invocation) | "Plan a vacation", "Summarize my meetings" |

### Tools — Deep Dive

Tools are schema-defined interfaces using **JSON Schema** for input validation. The LLM decides when to call them.

```typescript
{
  name: "searchFlights",
  description: "Search for available flights",
  inputSchema: {
    type: "object",
    properties: {
      origin:      { type: "string", description: "Departure city" },
      destination: { type: "string", description: "Arrival city" },
      date:        { type: "string", format: "date", description: "Travel date" }
    },
    required: ["origin", "destination", "date"]
  }
}
```

Protocol operations:

| Method | Purpose |
|---|---|
| `tools/list` | Discover available tools |
| `tools/call` | Execute a specific tool |

**Human oversight:** Applications can show approval dialogs before tool execution, display activity logs, or pre-approve safe operations.

### Resources — Deep Dive

Resources expose data with a **unique URI** and a MIME type. Two discovery patterns:

- **Direct resources** — fixed URIs: `calendar://events/2024`
- **Resource templates** — dynamic URIs with parameters: `weather://forecast/{city}/{date}`

Protocol operations:

| Method | Purpose |
|---|---|
| `resources/list` | List available direct resources |
| `resources/templates/list` | Discover resource templates |
| `resources/read` | Retrieve resource contents |
| `resources/subscribe` | Monitor resource changes |

Templates support **parameter completion** — typing "Par" for `weather://forecast/{city}` might suggest "Paris" or "Park City".

### Prompts — Deep Dive

Prompts are structured templates with defined arguments. They are **user-controlled** — never triggered automatically.

```json
{
  "name": "plan-vacation",
  "title": "Plan a vacation",
  "description": "Guide through vacation planning process",
  "arguments": [
    { "name": "destination", "type": "string", "required": true },
    { "name": "duration",    "type": "number", "description": "days" },
    { "name": "budget",      "type": "number", "required": false },
    { "name": "interests",   "type": "array",  "items": { "type": "string" } }
  ]
}
```

Typical UI patterns: slash commands (`/plan-vacation`), command palettes, context menus.

### Client-side Primitives

Servers can also leverage capabilities exposed by clients:

| Primitive | Purpose |
|---|---|
| **Sampling** | Server requests an LLM completion from the client's AI app — stays model-independent |
| **Elicitation** | Server requests additional input or confirmation from the user |
| **Logging** | Server sends log messages to client for debugging/monitoring |

### Cross-cutting Utility Primitives

| Primitive | Purpose |
|---|---|
| **Tasks** *(Experimental)* | Durable wrappers for deferred/async operations (batch processing, long workflows) |

---

## 6. Data Layer Protocol

MCP uses **JSON-RPC 2.0** as its underlying RPC protocol.

### Message types

| Type | Description |
|---|---|
| **Request** | Has an `id`, expects a response |
| **Response** | Matches an `id` from a prior request |
| **Notification** | No `id`, no response expected (fire-and-forget) |

### Lifecycle Management

MCP is a **stateful protocol**. Connections go through:

1. **Initialize** — capability negotiation handshake
2. **Ready** — client sends `notifications/initialized`
3. **Active** — normal operation (list/call/read/subscribe)
4. **Termination** — clean shutdown

Capabilities declared during initialization tell each side what primitives and features are available. If protocol versions can't be negotiated, the connection is terminated.

### Notifications

Real-time updates without polling. Example: when a server's tool list changes, it sends:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

The client then re-fetches `tools/list` to refresh its registry.

---

## 7. Transport Layer

| Transport | Use case | Auth support |
|---|---|---|
| **STDIO** | Local processes on the same machine; optimal performance, no network overhead | N/A (process-level trust) |
| **Streamable HTTP** | Remote servers; uses HTTP POST for requests, optional Server-Sent Events for streaming | Bearer tokens, API keys, custom headers, **OAuth 2.1 recommended** |

The transport layer is abstracted away — the same JSON-RPC 2.0 messages flow over both transports.

---

## 8. Lifecycle: Step-by-Step Walkthrough

### Step 1 — Initialization

Client sends:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-06-18",
    "capabilities": { "elicitation": {} },
    "clientInfo": { "name": "example-client", "version": "1.0.0" }
  }
}
```

Server responds:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2025-06-18",
    "capabilities": {
      "tools": { "listChanged": true },
      "resources": {}
    },
    "serverInfo": { "name": "example-server", "version": "1.0.0" }
  }
}
```

Client signals ready:

```json
{ "jsonrpc": "2.0", "method": "notifications/initialized" }
```

### Step 2 — Tool Discovery

```json
{ "jsonrpc": "2.0", "id": 2, "method": "tools/list" }
```

Returns array of tool definitions with names, descriptions, and JSON Schema input specs.

### Step 3 — Tool Execution

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "weather_current",
    "arguments": { "location": "San Francisco", "units": "imperial" }
  }
}
```

Response:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "content": [{ "type": "text", "text": "Current weather in San Francisco: 68°F, partly cloudy..." }]
  }
}
```

### Step 4 — Real-time Updates

Server notifies when its tool list changes (only if it declared `"listChanged": true` during init):

```json
{ "jsonrpc": "2.0", "method": "notifications/tools/list_changed" }
```

Client re-fetches `tools/list` to stay in sync.

---

## 9. Available SDKs

| Language | Tier | Repository |
|---|---|---|
| **TypeScript** | Tier 1 | [modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk) |
| **Python** | Tier 1 | [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk) |
| **C#** | Tier 1 | [modelcontextprotocol/csharp-sdk](https://github.com/modelcontextprotocol/csharp-sdk) |
| **Go** | Tier 1 | [modelcontextprotocol/go-sdk](https://github.com/modelcontextprotocol/go-sdk) |
| **Java** | Tier 2 | [modelcontextprotocol/java-sdk](https://github.com/modelcontextprotocol/java-sdk) |
| **Rust** | Tier 2 | [modelcontextprotocol/rust-sdk](https://github.com/modelcontextprotocol/rust-sdk) |
| **Swift** | Tier 3 | [modelcontextprotocol/swift-sdk](https://github.com/modelcontextprotocol/swift-sdk) |
| **Ruby** | Tier 3 | [modelcontextprotocol/ruby-sdk](https://github.com/modelcontextprotocol/ruby-sdk) |
| **PHP** | Tier 3 | [modelcontextprotocol/php-sdk](https://github.com/modelcontextprotocol/php-sdk) |
| **Kotlin** | Tier 3 | [modelcontextprotocol/kotlin-sdk](https://github.com/modelcontextprotocol/kotlin-sdk) |

**Tier 1** = full feature completeness, official protocol support, active maintenance.  
All SDKs support: creating servers with tools/resources/prompts, building clients, STDIO and HTTP transports.

---

## 10. Building Your First MCP Server (Python)

This walkthrough builds a weather server with two tools: `get_alerts` and `get_forecast`.

### System Requirements

- Python 3.10 or higher
- `mcp` Python SDK ≥ 1.2.0

### Step 1 — Set up the project

```bash
# Install uv (Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create and set up the project
uv init weather
cd weather
uv venv
source .venv/bin/activate        # On Windows: .venv\Scripts\activate

# Install dependencies
uv add "mcp[cli]" httpx

# Create server file
touch weather.py
```

### Step 2 — Write the server

```python
# weather.py
from typing import Any
import httpx
from mcp.server.fastmcp import FastMCP

# Initialize FastMCP server
mcp = FastMCP("weather")

NWS_API_BASE = "https://api.weather.gov"
USER_AGENT = "weather-app/1.0"


async def make_nws_request(url: str) -> dict[str, Any] | None:
    headers = {"User-Agent": USER_AGENT, "Accept": "application/geo+json"}
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, headers=headers, timeout=30.0)
            response.raise_for_status()
            return response.json()
        except Exception:
            return None


def format_alert(feature: dict) -> str:
    props = feature["properties"]
    return (
        f"Event: {props.get('event', 'Unknown')}\n"
        f"Area: {props.get('areaDesc', 'Unknown')}\n"
        f"Severity: {props.get('severity', 'Unknown')}\n"
        f"Description: {props.get('description', 'No description available')}\n"
    )


@mcp.tool()
async def get_alerts(state: str) -> str:
    """Get weather alerts for a US state.

    Args:
        state: Two-letter US state code (e.g. CA, NY)
    """
    url = f"{NWS_API_BASE}/alerts/active/area/{state}"
    data = await make_nws_request(url)

    if not data or "features" not in data:
        return "Unable to fetch alerts or no alerts found."
    if not data["features"]:
        return "No active alerts for this state."

    alerts = [format_alert(feature) for feature in data["features"]]
    return "\n---\n".join(alerts)


@mcp.tool()
async def get_forecast(latitude: float, longitude: float) -> str:
    """Get weather forecast for a location.

    Args:
        latitude: Latitude of the location
        longitude: Longitude of the location
    """
    points_url = f"{NWS_API_BASE}/points/{latitude},{longitude}"
    points_data = await make_nws_request(points_url)

    if not points_data:
        return "Unable to fetch forecast data for this location."

    forecast_url = points_data["properties"]["forecast"]
    forecast_data = await make_nws_request(forecast_url)

    if not forecast_data:
        return "Unable to fetch detailed forecast."

    periods = forecast_data["properties"]["periods"]
    forecasts = []
    for period in periods[:5]:
        forecasts.append(
            f"{period['name']}:\n"
            f"Temperature: {period['temperature']}°{period['temperatureUnit']}\n"
            f"Wind: {period['windSpeed']} {period['windDirection']}\n"
            f"Forecast: {period['detailedForecast']}"
        )
    return "\n---\n".join(forecasts)


def main():
    mcp.run(transport="stdio")


if __name__ == "__main__":
    main()
```

### Step 3 — Run the server

```bash
uv run weather.py
```

The server now listens for JSON-RPC messages from any MCP host.

### Logging Warning for STDIO Servers

> **Never write to stdout in a STDIO server.** It corrupts the JSON-RPC stream.

```python
import sys

# Bad
print("Processing request")

# Good
print("Processing request", file=sys.stderr)

# Also good
import logging
logging.info("Processing request")  # writes to stderr by default
```

---

## 11. Connecting to Claude Desktop

### Step 1 — Install Claude Desktop

Download from [claude.ai/download](https://claude.ai/download). Keep it updated to the latest version.

> **Note:** Claude Desktop is not yet available on Linux. Linux users should build an MCP client instead.

### Step 2 — Edit the config file

Open (create if missing):

- **macOS/Linux:** `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows:** `%AppData%\Claude\claude_desktop_config.json`

```bash
# macOS/Linux
code ~/Library/Application\ Support/Claude/claude_desktop_config.json

# Windows
code $env:AppData\Claude\claude_desktop_config.json
```

### Step 3 — Register your server

```json
{
  "mcpServers": {
    "weather": {
      "command": "uv",
      "args": [
        "--directory",
        "/ABSOLUTE/PATH/TO/weather",
        "run",
        "weather.py"
      ]
    }
  }
}
```

- Replace `/ABSOLUTE/PATH/TO/weather` with the actual full path to your project folder.
- Restart Claude Desktop after saving.
- The MCP UI elements (hammer icon) appear only when at least one server is properly configured.

---

## 12. Multi-Server Patterns

MCP's real power emerges when multiple servers work together. A single AI host can connect to many servers simultaneously, combining their specialized capabilities.

### Example: AI Travel Planner

Three servers working together:

| Server | Provides |
|---|---|
| **Travel Server** | Flights, hotels, itineraries |
| **Weather Server** | Climate data, forecasts |
| **Calendar/Email Server** | Schedules, communications |

**Complete flow:**

1. User invokes a `plan-vacation` prompt with structured args (destination, dates, budget)
2. User selects resources: `calendar://my-calendar/June-2024`, `travel://preferences/europe`
3. AI reads resources to gather context (availability, preferences, past trips)
4. AI calls tools in sequence:
   - `searchFlights()` → queries airlines
   - `checkWeather()` → retrieves forecasts
   - `bookHotel()` → finds hotels in budget *(with user approval)*
   - `createCalendarEvent()` → adds trip to calendar
   - `sendEmail()` → sends confirmation

**Result:** a task that could take hours, done in minutes, with full context awareness across all systems.

---

## 13. Client Concepts (Advanced)

These are capabilities that **clients expose to servers** — allowing server authors to build richer interactions.

### Sampling

Allows a server to request an LLM completion from the client's AI application. The server stays model-independent — it doesn't need its own LLM SDK.

```json
{
  "method": "sampling/createMessage",
  "params": {
    "messages": [{ "role": "user", "content": "Summarize this document: ..." }],
    "maxTokens": 500
  }
}
```

Use case: a code analysis server that needs to reason over large files without bundling an LLM.

### Elicitation

Allows a server to request additional input from the user — useful for confirmations before destructive actions.

```json
{ "method": "elicitation/create", "params": { "message": "Are you sure you want to delete all records?" } }
```

### Roots

Allows clients to expose file system roots to servers, scoping what the server can access.

---

## 14. Security and Authorization

### For Remote Servers (Streamable HTTP)

- MCP **recommends OAuth 2.1** for obtaining auth tokens
- Supported auth mechanisms: bearer tokens, API keys, custom headers
- See: [Understanding Authorization in MCP](https://modelcontextprotocol.io/docs/concepts/auth)

### General Security Best Practices

| Practice | Reason |
|---|---|
| Implement user approval for tool calls | Maintains human oversight over AI actions |
| Log all tool executions | Auditability and debugging |
| Scope resource access with Roots | Prevents servers from accessing unintended files |
| Validate all tool inputs with JSON Schema | Prevents injection attacks |
| Use HTTPS for remote transport | Encrypts data in transit |
| Rotate auth tokens | Limits blast radius of leaked credentials |

### For Local STDIO Servers

- Trust is granted at the process level (the host launches the server)
- Only install servers from trusted sources
- Review what tools a server exposes before configuring it

---

## 15. Debugging and Tooling

### MCP Inspector

The official interactive debugging tool for MCP servers.

```bash
npx @modelcontextprotocol/inspector
```

- Connects to any MCP server (local or remote)
- Lists all available tools, resources, and prompts
- Lets you call tools manually and inspect JSON-RPC messages
- GitHub: [modelcontextprotocol/inspector](https://github.com/modelcontextprotocol/inspector)

### Common Issues

| Symptom | Likely cause | Fix |
|---|---|---|
| Server not appearing in Claude Desktop | Config file path wrong or JSON syntax error | Validate JSON, use absolute paths |
| Tool calls silently fail | Writing to stdout in STDIO server | Use `sys.stderr` or a logging library |
| Protocol version mismatch | Client and server on different protocol versions | Update SDK to latest version |
| Auth failures on remote server | Token expired or wrong scope | Refresh token, check OAuth scopes |

### Logging Checklist for STDIO Servers

- Never use `print()` without `file=sys.stderr`
- Use Python `logging` module (it goes to stderr by default)
- For HTTP-based servers, standard stdout logging is fine

---

## 16. Key Best Practices

### Server Design

- **One tool, one responsibility** — tools should do one thing clearly
- **Descriptive names and descriptions** — the LLM reads these to decide when to call tools; be precise
- **JSON Schema validation** — always define `inputSchema`; mark required fields explicitly
- **Graceful error handling** — return error messages as tool content, don't crash the server
- **Idempotent tools when possible** — reduces risk when the LLM retries

### Resource Design

- **Use meaningful URIs** — e.g., `calendar://events/2024-06` not `resource://1`
- **Declare MIME types** — lets clients handle content appropriately
- **Use templates for dynamic data** — e.g., `weather://forecast/{city}/{date}`
- **Support subscriptions** for frequently changing data

### Prompt Design

- **Parameterize everything variable** — avoid hardcoded values in templates
- **Mark required vs optional** clearly
- **Include examples** in descriptions so users know what to input

### Performance

- Use `notifications/tools/list_changed` instead of forcing clients to poll
- For expensive operations, consider the **Tasks** extension for async execution
- STDIO transport has zero network overhead — prefer it for local servers

---

## 17. Further Reading

| Resource | URL |
|---|---|
| Official MCP site | https://modelcontextprotocol.io |
| Full specification | https://modelcontextprotocol.io/specification/latest |
| Architecture overview | https://modelcontextprotocol.io/docs/learn/architecture |
| Server concepts | https://modelcontextprotocol.io/docs/learn/server-concepts |
| Build a server | https://modelcontextprotocol.io/docs/develop/build-server |
| Build a client | https://modelcontextprotocol.io/docs/develop/build-client |
| SDK list | https://modelcontextprotocol.io/docs/sdk |
| MCP Inspector | https://github.com/modelcontextprotocol/inspector |
| Reference servers | https://github.com/modelcontextprotocol/servers |
| Authorization guide | https://modelcontextprotocol.io/docs/concepts/auth |
| Security best practices | https://modelcontextprotocol.io/docs/concepts/security |
| MCP Registry | https://modelcontextprotocol.io/registry |

---

*Generated from official MCP documentation at modelcontextprotocol.io — June 2026.*
