# Building Custom MCP Tools: A Developer's Guide

*By Sam Okafor | February 15, 2026*

Last month I built a custom MCP tool that lets my AI agent query our company's internal knowledge base. Took about 3 hours. Now every agent in our stack can search internal docs without any additional integration work.

That's the power of MCP – build once, use everywhere.

Here's exactly how to build your own MCP tools from scratch.

## What Is MCP, Really?

MCP (Model Context Protocol) is a standard way for AI agents to talk to external tools and data sources. Think of it as USB for AI – a universal connector that works across different agent frameworks.

Before MCP, every integration was custom:
- Agent A talks to your database one way
- Agent B needs a different connector
- Agent C requires yet another integration
- You maintain three separate implementations

With MCP:
- Build one MCP server for your database
- Every MCP-compatible agent can use it
- One implementation, infinite reuse

The protocol handles discovery, authentication, request/response formats, and error handling. You just implement your tool's specific logic.

## The Architecture

An MCP setup has three parts:

**MCP Host**: The AI agent or application that needs tools
**MCP Client**: The library that speaks MCP protocol
**MCP Server**: Your custom tool that provides capabilities

The flow:
1. Host asks Client: "What tools are available?"
2. Client asks Server: "What can you do?"
3. Server responds: "I can search_documents, get_user, update_record"
4. Host calls a tool through Client
5. Server executes and returns results

All over a standardized JSON-based protocol.

## Your First MCP Server

Let's build a real tool: a GitHub issue searcher.

### Setup

```bash
mkdir github-issues-mcp
cd github-issues-mcp
npm init -y
npm install @modelcontextprotocol/sdk
```

### Basic Server Structure

```javascript
// server.js
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new Server(
  {
    name: "github-issues",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// Define available tools
server.setRequestHandler("tools/list", async () => ({
  tools: [
    {
      name: "search_issues",
      description: "Search GitHub issues in a repository",
      inputSchema: {
        type: "object",
        properties: {
          repo: {
            type: "string",
            description: "Repository in format owner/repo"
          },
          query: {
            type: "string", 
            description: "Search query"
          },
          state: {
            type: "string",
            enum: ["open", "closed", "all"],
            description: "Issue state filter"
          }
        },
        required: ["repo", "query"]
      }
    }
  ]
}));

// Handle tool calls
server.setRequestHandler("tools/call", async (request) => {
  if (request.params.name === "search_issues") {
    const { repo, query, state = "open" } = request.params.arguments;
    
    const response = await fetch(
      `https://api.github.com/search/issues?q=${encodeURIComponent(query)}+repo:${repo}+state:${state}`,
      {
        headers: {
          "Authorization": `token ${process.env.GITHUB_TOKEN}`,
          "Accept": "application/vnd.github.v3+json"
        }
      }
    );
    
    const data = await response.json();
    
    return {
      content: [
        {
          type: "text",
          text: JSON.stringify(data.items.slice(0, 10), null, 2)
        }
      ]
    };
  }
  
  throw new Error(`Unknown tool: ${request.params.name}`);
});

// Start the server
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Running It

```bash
GITHUB_TOKEN=your_token node server.js
```

That's a working MCP server. Any MCP-compatible agent can now search GitHub issues.

## Adding More Tools

Real MCP servers usually provide multiple related tools. Let's expand:

```javascript
server.setRequestHandler("tools/list", async () => ({
  tools: [
    {
      name: "search_issues",
      description: "Search GitHub issues in a repository",
      inputSchema: { /* ... */ }
    },
    {
      name: "get_issue",
      description: "Get details of a specific issue",
      inputSchema: {
        type: "object",
        properties: {
          repo: { type: "string" },
          issue_number: { type: "number" }
        },
        required: ["repo", "issue_number"]
      }
    },
    {
      name: "create_comment",
      description: "Add a comment to an issue",
      inputSchema: {
        type: "object", 
        properties: {
          repo: { type: "string" },
          issue_number: { type: "number" },
          body: { type: "string" }
        },
        required: ["repo", "issue_number", "body"]
      }
    }
  ]
}));
```

Now your agent can search, read, and comment on issues – all through one MCP server.

## Production Considerations

The basic example works, but production needs more:

### Error Handling

```javascript
server.setRequestHandler("tools/call", async (request) => {
  try {
    // Your tool logic
    const result = await executeToolLogic(request);
    return { content: [{ type: "text", text: JSON.stringify(result) }] };
  } catch (error) {
    // Return error as content, don't throw
    return {
      content: [{ 
        type: "text", 
        text: `Error: ${error.message}` 
      }],
      isError: true
    };
  }
});
```

### Rate Limiting

```javascript
import Bottleneck from "bottleneck";

const limiter = new Bottleneck({
  minTime: 100,  // 10 requests per second max
  maxConcurrent: 5
});

async function rateLimitedFetch(url, options) {
  return limiter.schedule(() => fetch(url, options));
}
```

### Caching

```javascript
import NodeCache from "node-cache";

const cache = new NodeCache({ stdTTL: 300 }); // 5 minute cache

async function getCachedOrFetch(key, fetchFn) {
  const cached = cache.get(key);
  if (cached) return cached;
  
  const result = await fetchFn();
  cache.set(key, result);
  return result;
}
```

### Logging

```javascript
import winston from "winston";

const logger = winston.createLogger({
  level: "info",
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: "mcp-server.log" })
  ]
});

// In your handler
logger.info("Tool called", { 
  tool: request.params.name, 
  arguments: request.params.arguments 
});
```

## Transport Options

The example uses stdio transport (communication via stdin/stdout). Other options:

**HTTP/SSE Transport**: For web-based agents
```javascript
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import express from "express";

const app = express();

app.get("/mcp", async (req, res) => {
  const transport = new SSEServerTransport("/mcp", res);
  await server.connect(transport);
});

app.listen(3000);
```

**WebSocket Transport**: For bidirectional real-time communication
```javascript
// Coming in MCP SDK 1.2
```

Choose transport based on your deployment model. Stdio for local agents, HTTP for remote.

## Testing Your MCP Server

Don't ship untested tools. Here's a basic test setup:

```javascript
// test.js
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import { spawn } from "child_process";

const serverProcess = spawn("node", ["server.js"], {
  env: { ...process.env, GITHUB_TOKEN: "test_token" }
});

const transport = new StdioClientTransport({
  reader: serverProcess.stdout,
  writer: serverProcess.stdin
});

const client = new Client({ name: "test-client", version: "1.0.0" });
await client.connect(transport);

// Test tool listing
const tools = await client.request({ method: "tools/list" });
console.assert(tools.tools.length > 0, "Should have tools");

// Test tool execution
const result = await client.request({
  method: "tools/call",
  params: {
    name: "search_issues",
    arguments: { repo: "anthropics/claude", query: "bug" }
  }
});
console.assert(!result.isError, "Should not error");

serverProcess.kill();
console.log("All tests passed!");
```

## Real-World MCP Servers to Study

Learn from existing implementations:

- **filesystem**: Official file operations server
- **github**: GitHub API integration  
- **postgres**: Database queries
- **brave-search**: Web search capabilities
- **memory**: Persistent knowledge storage

All open source. Read the code, understand the patterns.

## Common Mistakes

Things I got wrong so you don't have to:

**Over-engineering the first version**: Start with one tool. Add complexity later.

**Ignoring input validation**: Invalid inputs crash servers. Validate everything.

**Blocking operations**: Long-running tools should be async. Don't block the server.

**Missing tool descriptions**: Agents use descriptions to decide when to call tools. Be specific.

**Hardcoded configuration**: Use environment variables. Different deployments need different configs.

## What's Next?

Once you've built basic tools, explore:

- **Resources**: Expose data that agents can read (MCP Resources protocol)
- **Prompts**: Provide pre-built prompts for common tasks (MCP Prompts protocol)  
- **Sampling**: Let servers request LLM completions (MCP Sampling protocol)

MCP is more than just tools. It's a complete interface for AI capabilities.

## The Bottom Line

MCP tools are the building blocks of AI infrastructure. Every reusable capability should be an MCP server.

Build once, use everywhere. That's the promise, and it actually delivers.

Your internal APIs, your databases, your custom workflows – wrap them in MCP and every agent in your organization can use them.

Start with something small. A tool your team needs daily. Get it working. Then build the next one.

Three hours to usefulness. That's the MCP developer experience.

---

*Sam Okafor builds AI infrastructure. Code samples and tutorials at [OpenClaw Blog](https://openclaw-blog.vercel.app).*
