---
name: atypica-research
description: Access atypica.ai's AI-powered business research capabilities through MCP protocol. Use when users need to conduct consumer research, market analysis, product testing, or generate insights about business questions. Supports creating research sessions, sending messages with AI execution, handling user interactions (question prompts and plan confirmations), retrieving conversation history, checking research status, accessing generated reports/podcasts, and searching AI personas for interviews.
---

# atypica Research

Access atypica.ai's multi-agent research framework for understanding consumer emotions, market perceptions, and decision preferences.

## Prerequisites

**IMPORTANT**: This skill requires the atypica MCP server to be installed and configured.

### Step 1: Check if MCP server is available

First, verify if the atypica MCP server is already installed by checking available tools:

```typescript
// Check for atypica tools
const hasAtypica = /* check if tools starting with "atypica_" are available */;
```

If atypica tools are not available, proceed to installation.

### Step 2: Guide user to get API key

Instruct the user to:

1. Visit **https://atypica.ai/account/settings**
2. Create an API key (format: `atypica_xxx`)
3. **Copy the key immediately** (it will only be shown once)

If the user doesn't have an atypica.ai account, direct them to **https://atypica.ai** to sign up first.

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

Once the MCP server is installed:

```typescript
// 1. Create research session
const result = await callTool("atypica_study_create", {
  content: "Research young people's coffee preferences"
});
const userChatToken = result.structuredContent.token;

// 2. Send message (triggers AI - may take 10-120s)
await callTool("atypica_study_send_message", {
  userChatToken,
  message: {
    role: "user",
    lastPart: { type: "text", text: "Start research" }
  }
});

// 3. Poll for research progress
let result;
do {
  await wait(5000);  // Wait 5 seconds
  result = await callTool("atypica_study_get_messages", {
    userChatToken,
    tail: 10  // Get last 10 parts for efficiency
  });
} while (result.structuredContent.isRunning);

// 4. Handle pending interactions or get final report
const { messages } = result.structuredContent;
const reportTool = messages
  .flatMap(m => m.parts)
  .find(p => p.type === "tool-generateReport" && p.output);

if (reportTool?.output?.reportToken) {
  const report = await callTool("atypica_study_get_report", {
    token: reportTool.output.reportToken
  });
} else {
  // If stopped without report, you can continue:
  await callTool("atypica_study_send_message", {
    userChatToken,
    message: {
      role: "user",
      lastPart: { type: "text", text: "Please continue the research" }
    }
  });
  // Then poll again...
}
```

## Core Workflow

1. **Create** research session with initial query
2. **Send** messages to drive research forward (AI executes synchronously)
3. **Poll** for pending interactions that require user input
4. **Handle** interactions by submitting tool results
5. **Monitor** progress and retrieve artifacts (reports/podcasts)

## Available Tools

### Session Management

**atypica_study_create** - Create research session
- Input: `{ content: string }`
- Returns: `{ token, studyId, status }`

**atypica_study_send_message** - Send message and execute AI (10-120s)
- Two input types:
  - User text: `{ userChatToken, message: { role: "user", lastPart: { type: "text", text } } }`
  - Tool result: See "User Interactions" section
- Returns: `{ messageId, role, status, attachmentCount }`

**atypica_study_get_messages** - Retrieve conversation history and execution status
- Input: `{ userChatToken, tail? }`
- Returns: `{ isRunning, messages: [{ messageId, role, parts, createdAt }] }`
- **Critical**:
  - `isRunning: true` → AI is executing, wait and poll again later
  - `isRunning: false` → Can interact, check for pending tool calls in `parts`
  - `tail` (optional): Limit to last N parts across all messages

**atypica_study_list** - List historical research sessions
- Input: `{ kind?, page?, pageSize? }`
- Returns: `{ data: [...], pagination }`

### Artifacts

**atypica_study_get_report** - Get research report
- Input: `{ token }`
- Returns: `{ title, description, content (HTML), coverUrl, ... }`

**atypica_study_get_podcast** - Get podcast content
- Input: `{ token }`
- Returns: `{ script, audioUrl, coverUrl, metadata, ... }`

### Personas

**atypica_persona_search** - Semantic search for AI personas
- Input: `{ query?, tier?, limit? }`
- Uses embedding similarity for semantic matching
- Returns: `{ data: [{ personaId, token, name, source, tier, tags }] }`

**atypica_persona_get** - Get persona details
- Input: `{ personaId }`
- Returns: Full persona including `prompt` field

## Understanding Research State from Messages

**All research state is in the messages** - you don't need a separate status API. After calling `getMessages`, follow this pattern:

### 1. Check if AI is executing

```typescript
const { isRunning, messages } = result.structuredContent;

if (isRunning) {
  // AI is working in background, cannot interact now
  // Wait 5-10 seconds and poll again
  return "Research is running, please wait...";
}
```

### 2. Check for pending interactions

Scan the last assistant message for tool calls needing user input:

```typescript
const lastMsg = messages[messages.length - 1];
if (lastMsg.role === "assistant") {
  for (const part of lastMsg.parts) {
    if (part.type.startsWith("tool-") && part.state === "input-available") {
      // Handle this pending tool call (see User Interactions section)
    }
  }
}
```

### 3. Understand research progress

Look at recent tool calls to see what's happening:

| Tool Call | Meaning |
|-----------|---------|
| `makeStudyPlan` | In Plan Mode, AI is clarifying intent |
| `interviewChat`, `discussionChat` | Conducting interviews/discussions |
| `webSearch`, `webFetch` | Gathering information |
| `reasoningThinking` | Deep analysis in progress |
| `generateReport` | Generating final report |
| `generatePodcast` | Generating podcast |

### 4. Check if research is complete

Look for `generateReport` or `generatePodcast` tool call with output:

```typescript
const reportTool = messages
  .flatMap(m => m.parts)
  .find(p => p.type === "tool-generateReport" && p.state === "output-available");

if (reportTool?.output?.reportToken) {
  // Research complete! Get the report
  const report = await callTool("atypica_study_get_report", {
    token: reportTool.output.reportToken
  });
}
```

### 5. Continue a stopped research

**When to use**: If research stops (`isRunning: false`) but no report/podcast was generated, you can continue it.

**How to continue**:
```typescript
// Check if research stopped without completing
const { isRunning, messages } = result.structuredContent;
const hasReport = messages.some(m =>
  m.parts.some(p =>
    (p.type === "tool-generateReport" || p.type === "tool-generatePodcast") &&
    p.state === "output-available"
  )
);

if (!isRunning && !hasReport) {
  // Research stopped but not complete - you can continue it
  await callTool("atypica_study_send_message", {
    userChatToken,
    message: {
      role: "user",
      lastPart: {
        type: "text",
        text: "Please continue the research"  // or "[CONTINUE ASSISTANT STEPS]"
      }
    }
  });

  // Then poll again to check progress
}
```

**Important**:
- The AI will resume from where it stopped, not restart from scratch
- It may retry the last interrupted tool or continue with the next step
- After sending continue message, poll `getMessages` again to track progress

## User Interactions

Two tools require user interaction. Check `getMessages` response for pending calls with `state === "input-available"`.

### requestInteraction - User Choice/Input

**Detect**:
```json
{
  "type": "tool-requestInteraction",
  "state": "input-available",
  "toolCallId": "call_abc",
  "input": {
    "question": "Which age group to focus on?",
    "options": ["18-22", "23-28", "29-35"],
    "maxSelect": 1  // 1=single, 2+=multi with limit, undefined=unlimited
  }
}
```

**Submit** via `sendMessage`:
```json
{
  "userChatToken": "...",
  "message": {
    "id": "msg_2",  // Original message ID
    "role": "assistant",
    "lastPart": {
      "type": "tool-requestInteraction",
      "toolCallId": "call_abc",
      "state": "output-available",
      "input": { /* copy from above */ },
      "output": {
        "answer": "23-28",  // string for single, string[] for multi
        "plainText": "User selected: 23-28"
      }
    }
  }
}
```

### makeStudyPlan - Plan Confirmation

**Detect**:
```json
{
  "type": "tool-makeStudyPlan",
  "state": "input-available",
  "toolCallId": "call_xyz",
  "input": {
    "locale": "zh-CN",
    "kind": "insights",
    "role": "Market Researcher",
    "topic": "Coffee preferences study",
    "planContent": "# Research Plan\n\n## Goals\n...\n## Methods\n..."
  }
}
```

**Submit** via `sendMessage`:
```json
{
  "userChatToken": "...",
  "message": {
    "id": "msg_2",
    "role": "assistant",
    "lastPart": {
      "type": "tool-makeStudyPlan",
      "toolCallId": "call_xyz",
      "state": "output-available",
      "input": { /* copy from above */ },
      "output": {
        "confirmed": true,  // boolean: true to confirm, false to cancel
        "plainText": "User confirmed research plan"
      }
    }
  }
}
```

## Best Practices

1. **Async execution**: Call `sendMessage` asynchronously (10-120s duration)
2. **Poll for interactions**: After each `sendMessage`, check `getMessages` for pending tool calls
3. **Handle timeouts**: If `sendMessage` times out, use `getStatus` to check progress
4. **Error handling**: Check `status` field: `"completed" | "saved_no_ai" | "ai_failed"`
5. **Semantic search**: Use natural language queries in `persona_search` (embedding-based)

## Research Types (kind)

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

## Reference Documentation

See [references/api-reference.md](references/api-reference.md) for complete API documentation including:
- Detailed input/output schemas for all tools
- Error codes and troubleshooting
- Complete workflow examples
- Security and limitations
