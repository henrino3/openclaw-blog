# Anthropic's MCP v3 Ships With Built-in Agent Discovery — Here's Why It Matters

**Published:** 2026-02-17
**Author:** Jake Morrison
**Type:** News Brief
**Section:** News & Announcements

---

Anthropic dropped MCP v3 at 9pm Pacific last night, and the headline feature sounds boring: "automatic tool discovery." But here's why it's actually a big deal.

Before v3, if you wanted an AI agent to use your internal API, you had to manually register that tool, write a JSON schema describing every endpoint, and pray the agent didn't hallucinate parameters. It worked, but it was tedious and error-prone.

MCP v3 changes this. The protocol now supports discovery — an agent can ask "what tools are available?" and get back a live manifest. Your API can respond with "I have these 5 endpoints, here's what they do, here's what parameters they take." The agent picks what it needs and calls it directly.

This matters for three reasons.

First, it makes agent integrations self-service. Previously, integrating a new internal system into an OpenClaw workflow required engineering work to define tools. With v3, you just run an MCP-compliant discovery endpoint. Your HRIS system, your CRM, your ticketing platform — they all expose their capabilities automatically. Agents find them and use them.

Second, it solves the stale schema problem. In v2, if you added a new endpoint to your API, you had to manually update your MCP tool definitions. Agents kept calling the old endpoints until someone remembered to update the config. In v3, discovery is live. Add a new endpoint? Agents see it immediately.

Third, it enables dynamic agent teams. Agent A can discover Agent B's capabilities at runtime and decide whether to delegate. You don't need to pre-define every possible agent collaboration. They figure it out themselves.

The tradeoff is security. Automatic discovery means any agent with network access can probe your systems for capabilities. Anthropic added an optional auth layer — discovery endpoints can require a token. But you actually have to configure it.

What this means for you: if you're building internal agent workflows, look into upgrading to MCP v3. The learning curve is shallow, and it removes a ton of boilerplate. If you're building products that integrate with agents, add an MCP v3 discovery endpoint. It makes your product agent-ready by default.

The release notes are 47 pages long and include three breaking changes from v2. Most people won't care, but if you're deep in the weeds, check the migration guide. The biggest gotcha: tools now use snake_case parameter names instead of camelCase. This is to align with OpenClaw's existing conventions.

MCP v3 is live in Anthropic's SDKs now. OpenClaw added support in last week's update. If you're on version 2026.2.13 or later, you're good to go.

---

**Related reading:**
- "MCP Protocol: How AI Agents Are Learning to Talk to Each Other" (Dr. Elena Vasquez)
- "Building a Multi-Agent Workflow in OpenClaw" (Sam Okafor)
