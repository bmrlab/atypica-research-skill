# atypica Research Skill

Access atypica.ai's AI-powered business research capabilities through MCP protocol. This skill enables multi-agent research workflows including consumer insights, product R&D, market analysis, and automated report generation.

## Features

- 🤖 **Multi-Agent Research**: Leverage Plan Mode, Study Agent, Fast Insight Agent, Product R&D Agent
- 🗣️ **Interview Simulation**: Conduct 1-on-1 interviews and group discussions with AI personas
- 📊 **Automated Artifacts**: Generate research reports and podcasts
- 🔍 **Semantic Persona Search**: Find and interview relevant AI personas using vector similarity
- 💬 **Interactive Prompts**: Handle dynamic user interactions during research

## Installation

**IMPORTANT**: This skill requires the atypica MCP server to be installed and configured.

### Step 1: Check if MCP server is available

First, verify if the atypica MCP server is already installed by checking if tools starting with `atypica_` are available in your AI assistant.

If atypica tools are not available, proceed to installation.

### Step 2: Get API key

1. Visit **https://atypica.ai/account/settings**
2. Create an API key (format: `atypica_xxx`)
3. **Copy the key immediately** (it will only be shown once)

If you don't have an atypica.ai account, sign up at **https://atypica.ai** first.

### Step 3: Install MCP server

Add the MCP server to your AI assistant's configuration. The exact method varies by tool:

**Example command (syntax varies by MCP client)**:

```bash
# Generic example - adjust for your specific MCP client
mcp add --transport http atypica-research https://atypica.ai/mcp/study \
  --header "Authorization: Bearer YOUR_API_KEY_HERE"
```

**For MCP clients supporting JSON config**:

If your client uses JSON configuration, you can use `mcp-remote` as a proxy:

```json
{
  "mcpServers": {
    "atypica-research": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://atypica.ai/mcp/study",
        "--header",
        "Authorization: Bearer YOUR_API_KEY_HERE"
      ]
    }
  }
}
```

Refer to your AI assistant's documentation for the exact configuration location and syntax.

### Step 4: Restart and verify

1. **Restart your AI assistant** to load the new MCP server
2. Verify tools are available: Check for tools starting with `atypica_`

## Quick Start

```typescript
// 1. Create research session
const result = await callTool("atypica_study_create", {
  content: "Research young people's coffee preferences"
});
const userChatToken = result.structuredContent.token;

// 2. Send message (triggers AI execution)
await callTool("atypica_study_send_message", {
  userChatToken,
  message: {
    role: "user",
    lastPart: { type: "text", text: "Start research" }
  }
});

// 3. Poll for completion
let messages;
do {
  await wait(5000);
  const result = await callTool("atypica_study_get_messages", {
    userChatToken,
    tail: 10  // Get last 10 parts for efficiency
  });
  messages = result.structuredContent;
} while (messages.isRunning);

// 4. Get final report
const reportTool = messages.messages
  .flatMap(m => m.parts)
  .find(p => p.type === "tool-generateReport" && p.output);

if (reportTool?.output?.reportToken) {
  const report = await callTool("atypica_study_get_report", {
    token: reportTool.output.reportToken
  });
}
```

## Documentation

- **[SKILL.md](SKILL.md)** - Complete skill documentation and usage guide
- **[API Reference](references/api-reference.md)** - Detailed API schemas and examples

## Available Tools

### Session Management
- `atypica_study_create` - Create research session
- `atypica_study_send_message` - Send message and execute AI
- `atypica_study_get_messages` - Get conversation history and status
- `atypica_study_list` - List historical sessions

### Artifacts
- `atypica_study_get_report` - Retrieve generated reports
- `atypica_study_get_podcast` - Retrieve generated podcasts

### Personas
- `atypica_persona_search` - Semantic search for AI personas
- `atypica_persona_get` - Get persona details

## Research Types

- `productRnD` - Product research & development
- `fastInsight` - Quick insights with podcast generation
- `insights` - Consumer insights research
- `testing` - Product testing and validation
- `creation` - Creative content generation
- `planning` - Strategic planning
- `misc` - Miscellaneous research

Omit `kind` in `create` to enter **Plan Mode** where AI auto-determines research type.

## Performance Expectations

- Plan Mode: 5-10 seconds
- Fast Insight: 20-40 seconds
- Full Study/Product R&D: 30-120 seconds
- Message/persona retrieval: < 2 seconds

## Support

- **Documentation**: https://atypica.ai/docs/mcp
- **Issues**: https://github.com/bmrlab/atypica-research-skill/issues
- **Website**: https://atypica.ai

## License

This skill is maintained by [atypica.ai](https://atypica.ai) and provided as-is for use with MCP-compatible AI assistants.

---

**Auto-synced from [atypica-llm-app](https://github.com/bmrlab/atypica-llm-app)**
