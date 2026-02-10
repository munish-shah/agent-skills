---
name: create-agent
description: Bootstrap a CLI coding agent with OpenRouter SDK callModel() API, Zod tools, and Ink TUI
metadata:
  version: 0.3.0
  homepage: https://openrouter.ai
---

# Build a Modular AI Coding Agent with OpenRouter

This skill helps you create a **terminal-based coding agent**. Unlike simple chatbots, this agent can:

- **Read & Write Files** - Modify your codebase directly
- **Edit Files** - Targeted find-and-replace edits
- **Execute Shell Commands** - Run tests, installs, and builds
- **Browse the Web** - Fetch documentation and search the internet
- **Look Good** - Includes a modern Ink-based Terminal UI (TUI)
- **OpenRouter SDK callModel()** - Automatic tool execution with items-based streaming

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   cli.tsx (TUI)                  │
│         Ink React components + events            │
├─────────────────────────────────────────────────┤
│                  agent.ts (Core)                 │
│  OpenRouter SDK callModel() + getItemsStream()   │
│  EventEmitter for UI hooks                       │
├─────────────────────────────────────────────────┤
│                 tools.ts (Tools)                 │
│  7 Zod-schema tools with callbacks               │
│  list_files, read_file, edit_file, write_file,   │
│  run_command, fetch_web_page, web_search         │
└─────────────────────────────────────────────────┘
```

**How it works:**
1. Agent calls `callModel()` with tools
2. SDK automatically validates args with Zod, executes tools, sends results back to model
3. SDK repeats until the model stops calling tools
4. `getItemsStream()` streams all items (messages, tool calls, results) to the TUI
5. Tool callbacks emit events for the UI's tool indicators

## Prerequisites

> [!IMPORTANT]
> **Node.js 18.x or 20.x required.**

Get an OpenRouter API key at: https://openrouter.ai/settings/keys

> [!CAUTION]
> **Dependency Safety Rules (MUST follow):**
> - Use ONLY the exact dependency versions listed in the package.json below
> - Do NOT add `cheerio`, `undici`, or any package with native bindings — these frequently break across Node versions
> - For HTML parsing, use the built-in regex-based `htmlToText()` helper in tools.ts — no external HTML parser needed
> - After `npm install`, verify zero errors by running `npm start` before considering setup complete

## Project Setup

### Step 1: Initialize Project

```bash
mkdir my-coding-agent && cd my-coding-agent
npm init -y
npm pkg set type="module"
mkdir src
```

### Step 2: Create package.json

Replace your `package.json` with this exact content for guaranteed compatibility:

```json
{
  "name": "my-coding-agent",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "start": "tsx src/cli.tsx",
    "start:headless": "tsx src/headless.ts",
    "dev": "tsx watch src/cli.tsx"
  },
  "dependencies": {
    "@openrouter/sdk": "^0.8.0",
    "dotenv": "^16.4.5",
    "eventemitter3": "^5.0.4",
    "glob": "^10.4.5",
    "ink": "^4.4.1",
    "react": "^18.3.1",
    "zod": "^3.25.0"
  },
  "devDependencies": {
    "@types/node": "^20.14.0",
    "@types/react": "^18.3.28",
    "tsx": "^4.21.0",
    "typescript": "^5.5.0"
  }
}
```

Then install:

```bash
npm install
```

### Step 3: Create .env file

Create a `.env` file in the project root with your API key:

```
OPENROUTER_API_KEY=your-key-here
```

> [!CAUTION]
> Add `.env` to your `.gitignore` to avoid committing secrets!

### Step 4: Create tsconfig.json

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "dist"
  },
  "include": ["src"]
}
```

## File Structure

```
src/
├── agent.ts        # Core: callModel() with automatic tool execution & event emitter
├── tools.ts        # 7 Zod-schema tools with callbacks for UI events
├── cli.tsx         # Ink TUI: ASCII banner, streaming, tool indicators
└── headless.ts     # readline-based CLI mode
```

---

## Implementation

### Step 5: Create src/agent.ts

The agent uses `callModel()` which handles the entire tool execution loop internally. No manual while loop needed — the SDK validates tool args with Zod, executes tools, and re-queries the model automatically. We use `getItemsStream()` to see all items (messages, tool calls, results) as they happen.

```typescript
import { OpenRouter } from '@openrouter/sdk';
import { EventEmitter } from 'eventemitter3';
import { createTools } from './tools.js';

// Message types
export interface Message {
  role: 'user' | 'assistant';
  content: string;
}

// Agent events
export interface AgentEvents {
  'message:user': (message: Message) => void;
  'message:assistant': (message: Message) => void;
  'stream:start': () => void;
  'stream:delta': (delta: string, accumulated: string) => void;
  'stream:end': (fullText: string) => void;
  'tool:call': (name: string, args: unknown) => void;
  'tool:result': (name: string, result: unknown) => void;
  'error': (error: Error) => void;
  'thinking:start': () => void;
  'thinking:end': () => void;
}

// Agent configuration
export interface AgentConfig {
  apiKey: string;
  model?: string;
  instructions?: string;
  maxToolRounds?: number;
}

// The Agent class - uses OpenRouter SDK callModel() with automatic tool execution
export class Agent extends EventEmitter<AgentEvents> {
  private client: OpenRouter;
  private history: Message[] = [];
  private tools: ReturnType<typeof createTools>;
  private config: {
    model: string;
    instructions: string;
    maxToolRounds: number;
  };

  constructor(config: AgentConfig) {
    super();
    this.client = new OpenRouter({ apiKey: config.apiKey });
    this.config = {
      model: config.model ?? 'openai/gpt-4o',
      instructions: config.instructions ?? 'You are a skilled coding assistant.',
      maxToolRounds: config.maxToolRounds ?? 50,
    };
    // Create tools with event callbacks wired to this agent's emitter
    this.tools = createTools({
      onCall: (name, args) => this.emit('tool:call', name, args),
      onResult: (name, result) => this.emit('tool:result', name, result),
    });
  }

  getMessages(): Message[] {
    return [...this.history];
  }

  clearHistory(): void {
    this.history = [];
  }

  setInstructions(instructions: string): void {
    this.config.instructions = instructions;
  }

  async send(content: string): Promise<string> {
    const userMessage: Message = { role: 'user', content };
    this.history.push(userMessage);
    this.emit('message:user', userMessage);
    this.emit('thinking:start');
    this.emit('stream:start');

    try {
      // callModel() handles the entire tool execution loop
      const result = this.client.callModel({
        model: this.config.model,
        instructions: this.config.instructions,
        input: this.history.map(m => ({ role: m.role as 'user' | 'assistant', content: m.content })),
        tools: this.tools,
      });

      // Use getItemsStream() to see ALL items (messages, tool calls, etc.)
      // Items are emitted multiple times with same ID but progressively updated content
      let fullText = '';
      const seenToolCalls = new Set<string>();
      const completedToolCalls = new Set<string>();

      for await (const item of result.getItemsStream()) {
        if (item.type === 'function_call') {
          // Tool call item - emit events for UI indicators
          const callId = item.callId;
          if (!seenToolCalls.has(callId)) {
            seenToolCalls.add(callId);
            // Parse the arguments safely
            let args: unknown = {};
            try {
              args = item.arguments ? JSON.parse(item.arguments) : {};
            } catch {
              args = { raw: item.arguments };
            }
            this.emit('tool:call', item.name, args);
          }
        } else if (item.type === 'function_call_output') {
          // Tool result - mark as complete
          const callId = (item as { callId?: string }).callId;
          if (callId && !completedToolCalls.has(callId)) {
            completedToolCalls.add(callId);
            this.emit('tool:result', 'tool', (item as { output?: string }).output);
          }
        } else if (item.type === 'message') {
          // Message item - extract text content and stream it
          const messageItem = item as { content?: Array<{ type: string; text?: string }> };
          if (messageItem.content) {
            let currentText = '';
            for (const part of messageItem.content) {
              if (part.type === 'output_text' && part.text) {
                currentText += part.text;
              }
            }
            // Emit delta for new text
            if (currentText.length > fullText.length) {
              const delta = currentText.slice(fullText.length);
              fullText = currentText;
              this.emit('stream:delta', delta, fullText);
            }
          }
        }
      }

      this.emit('stream:end', fullText);

      const assistantMessage: Message = { role: 'assistant', content: fullText };
      this.history.push(assistantMessage);
      this.emit('message:assistant', assistantMessage);

      return fullText;
    } catch (err) {
      const error = err instanceof Error ? err : new Error(String(err));
      this.emit('error', error);
      throw error;
    } finally {
      this.emit('thinking:end');
    }
  }
}

// Factory function
export function createAgent(config: AgentConfig): Agent {
  return new Agent(config);
}
```

---

### Step 6: Create src/tools.ts

Tools use Zod schemas for type-safe parameters. The SDK automatically validates args against the schema before calling `execute`. Callbacks notify the UI about tool activity.

```typescript
import { z } from 'zod/v4';
import { tool } from '@openrouter/sdk';
import * as fs from 'fs/promises';
import { exec } from 'child_process';
import { promisify } from 'util';
import { glob } from 'glob';

const execAsync = promisify(exec);

// Callbacks for tool event notifications
export interface ToolCallbacks {
  onCall?: (name: string, args: unknown) => void;
  onResult?: (name: string, result: unknown) => void;
}

// Lightweight HTML-to-text helper (no external dependencies)
function htmlToText(html: string): string {
  return html
    .replace(/<script[\s\S]*?<\/script>/gi, '')
    .replace(/<style[\s\S]*?<\/style>/gi, '')
    .replace(/<nav[\s\S]*?<\/nav>/gi, '')
    .replace(/<footer[\s\S]*?<\/footer>/gi, '')
    .replace(/<header[\s\S]*?<\/header>/gi, '')
    .replace(/<[^>]+>/g, ' ')
    .replace(/&nbsp;/g, ' ')
    .replace(/&amp;/g, '&')
    .replace(/&lt;/g, '<')
    .replace(/&gt;/g, '>')
    .replace(/&quot;/g, '"')
    .replace(/&#39;/g, "'")
    .replace(/\s+/g, ' ')
    .trim();
}

function extractTitle(html: string): string {
  const match = html.match(/<title[^>]*>([\s\S]*?)<\/title>/i);
  return match ? match[1].trim() : '';
}

// Create all tools using the SDK's tool() helper
// Returns properly typed Tool[] for OpenRouter SDK callModel()
export function createTools(callbacks?: ToolCallbacks) {
  const call = (name: string, args: unknown) => callbacks?.onCall?.(name, args);
  const done = <T>(name: string, result: T): T => { callbacks?.onResult?.(name, result); return result; };

  return [
    tool({
      name: 'list_files',
      description: 'List files in a directory to understand project structure',
      inputSchema: z.object({
        path: z.string().default('.').describe('Directory to search'),
        recursive: z.boolean().default(false).describe('List recursively'),
      }),
      execute: async (params) => {
        call('list_files', params);
        try {
          const files = await glob(params.recursive ? '**/*' : '*', {
            cwd: params.path,
            nodir: false,
            ignore: ['**/node_modules/**', '**/.git/**', '**/dist/**'],
          });
          return done('list_files', { files: files.slice(0, 100), count: files.length });
        } catch (e) {
          return done('list_files', { error: e instanceof Error ? e.message : String(e) });
        }
      },
    }),

    tool({
      name: 'read_file',
      description: 'Read the contents of a specific file',
      inputSchema: z.object({
        filepath: z.string().describe('Path to the file to read'),
      }),
      execute: async (params) => {
        call('read_file', params);
        try {
          const content = await fs.readFile(params.filepath, 'utf-8');
          return done('read_file', { filepath: params.filepath, content, length: content.length });
        } catch (e) {
          return done('read_file', { error: `Failed to read ${params.filepath}: ${e instanceof Error ? e.message : e}` });
        }
      },
    }),

    tool({
      name: 'edit_file',
      description: 'Make targeted edits to a file by replacing specific text. Use this instead of write_file when you only need to change part of a file.',
      inputSchema: z.object({
        filepath: z.string().describe('Path to the file to edit'),
        old_string: z.string().describe('The exact text to find and replace (must match exactly)'),
        new_string: z.string().describe('The replacement text'),
        replace_all: z.boolean().default(false).describe('Replace all occurrences instead of just the first'),
      }),
      execute: async (params) => {
        call('edit_file', params);
        try {
          const content = await fs.readFile(params.filepath, 'utf-8');
          if (!content.includes(params.old_string)) {
            return done('edit_file', { error: `old_string not found in ${params.filepath}. Make sure it matches exactly, including whitespace and indentation.` });
          }
          const occurrences = content.split(params.old_string).length - 1;
          if (occurrences > 1 && !params.replace_all) {
            return done('edit_file', { error: `old_string found ${occurrences} times in ${params.filepath}. Provide more context to make it unique, or set replace_all to true.` });
          }
          const updated = params.replace_all
            ? content.replaceAll(params.old_string, params.new_string)
            : content.replace(params.old_string, params.new_string);
          await fs.writeFile(params.filepath, updated, 'utf-8');
          return done('edit_file', { success: true, filepath: params.filepath, replacements: params.replace_all ? occurrences : 1 });
        } catch (e) {
          return done('edit_file', { error: `Failed to edit ${params.filepath}: ${e instanceof Error ? e.message : e}` });
        }
      },
    }),

    tool({
      name: 'write_file',
      description: 'Write content to a file. Overwrites existing content.',
      inputSchema: z.object({
        filepath: z.string().describe('Path to the file to write'),
        content: z.string().describe('Content to write'),
      }),
      execute: async (params) => {
        call('write_file', params);
        try {
          await fs.writeFile(params.filepath, params.content, 'utf-8');
          return done('write_file', { success: true, filepath: params.filepath, bytesWritten: params.content.length });
        } catch (e) {
          return done('write_file', { error: `Failed to write ${params.filepath}: ${e instanceof Error ? e.message : e}` });
        }
      },
    }),

    tool({
      name: 'run_command',
      description: 'Execute a shell command (e.g., git, npm test, ls)',
      inputSchema: z.object({
        command: z.string().describe('The shell command to execute'),
      }),
      execute: async (params) => {
        call('run_command', params);
        try {
          const { stdout, stderr } = await execAsync(params.command);
          return done('run_command', { command: params.command, stdout: stdout.trim().slice(0, 2000), stderr: stderr.trim().slice(0, 500) });
        } catch (e) {
          const err = e as { message?: string; stdout?: string; stderr?: string };
          return done('run_command', { error: 'Command failed', message: err.message, stdout: err.stdout?.slice(0, 1000) });
        }
      },
    }),

    tool({
      name: 'fetch_web_page',
      description: 'Fetch text content from a URL (useful for reading docs)',
      inputSchema: z.object({
        url: z.string().describe('URL to fetch'),
      }),
      execute: async (params) => {
        call('fetch_web_page', params);
        try {
          const res = await fetch(params.url);
          if (!res.ok) throw new Error(`HTTP ${res.status}`);
          const html = await res.text();
          const title = extractTitle(html);
          const text = htmlToText(html).slice(0, 4000);
          return done('fetch_web_page', { url: params.url, title, content: text });
        } catch (e) {
          return done('fetch_web_page', { error: `Failed to fetch ${params.url}: ${e instanceof Error ? e.message : e}` });
        }
      },
    }),

    tool({
      name: 'web_search',
      description: 'Search the internet for information using DuckDuckGo. Returns titles, URLs, and snippets.',
      inputSchema: z.object({
        query: z.string().describe('The search query'),
        num_results: z.number().default(5).describe('Number of results (max 10)'),
      }),
      execute: async (params) => {
        call('web_search', params);
        const numResults = Math.min(params.num_results, 10);
        try {
          const searchUrl = `https://html.duckduckgo.com/html/?q=${encodeURIComponent(params.query)}`;
          const res = await fetch(searchUrl, {
            headers: { 'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36' },
          });
          if (!res.ok) throw new Error(`HTTP ${res.status}`);
          const html = await res.text();
          const results: Array<{ title: string; url: string; snippet: string }> = [];
          const resultBlocks = html.split(/class="result\s/);
          for (let i = 1; i < resultBlocks.length && results.length < numResults; i++) {
            const block = resultBlocks[i];
            const titleMatch = block.match(/class="result__a"[^>]*>([^<]+)</);
            const hrefMatch = block.match(/class="result__a"\s+href="([^"]+)"/);
            const snippetMatch = block.match(/class="result__snippet"[^>]*>([\s\S]*?)<\/a>/);
            const title = titleMatch ? titleMatch[1].trim() : '';
            let url = hrefMatch ? hrefMatch[1] : '';
            if (url.startsWith('//')) url = `https:${url}`;
            const snippet = snippetMatch ? htmlToText(snippetMatch[1]).slice(0, 200) : '';
            if (title && url) results.push({ title, url, snippet });
          }
          return done('web_search', { query: params.query, results, count: results.length });
        } catch (e) {
          return done('web_search', { error: `Search failed: ${e instanceof Error ? e.message : e}` });
        }
      },
    }),
  ];
}
```

---

### Step 7: Create src/cli.tsx

A polished terminal interface with ASCII art banner, streaming text, and tool call indicators.

```typescript
import 'dotenv/config';
import React, { useState, useEffect, useCallback } from 'react';
import { render, Box, Text, useInput, useApp, useStdout } from 'ink';
import { createAgent, type Message } from './agent.js';
// ═══════════════════════════════════════════════════════════════════════════════
// Simple markdown cleaner for terminal display
// ═══════════════════════════════════════════════════════════════════════════════
function formatMarkdown(text: string): string {
  return text
    // Headers: ## Header → ▓ Header
    .replace(/^### (.+)$/gm, '\n░ $1')
    .replace(/^## (.+)$/gm, '\n▓ $1')
    .replace(/^# (.+)$/gm, '\n█ $1 █')
    // Bold: **text** → text (remove markers)
    .replace(/\*\*([^*]+)\*\*/g, '$1')
    // Italic: *text* or _text_ → text
    .replace(/\*([^*]+)\*/g, '$1')
    .replace(/_([^_]+)_/g, '$1')
    // Bullet points: * item or - item → • item
    .replace(/^[\s]*[-*]\s+/gm, '  • ')
    // Code blocks: ```code``` → just the code
    .replace(/```[\w]*\n?([\s\S]*?)```/g, '\n$1\n')
    // Inline code: `code` → code
    .replace(/`([^`]+)`/g, '$1')
    // Links: [text](url) → text
    .replace(/\[([^\]]+)\]\([^)]+\)/g, '$1')
    // Clean up extra newlines
    .replace(/\n{3,}/g, '\n\n')
    .trim();
}

// ═══════════════════════════════════════════════════════════════════════════════
// ASCII Banner
// ═══════════════════════════════════════════════════════════════════════════════
const BANNER = `
 ██████╗ ██████╗ ███████╗███╗   ██╗██████╗  ██████╗ ██╗   ██╗████████╗███████╗██████╗
██╔═══██╗██╔══██╗██╔════╝████╗  ██║██╔══██╗██╔═══██╗██║   ██║╚══██╔══╝██╔════╝██╔══██╗
██║   ██║██████╔╝█████╗  ██╔██╗ ██║██████╔╝██║   ██║██║   ██║   ██║   █████╗  ██████╔╝
██║   ██║██╔═══╝ ██╔══╝  ██║╚██╗██║██╔══██╗██║   ██║██║   ██║   ██║   ██╔══╝  ██╔══██╗
╚██████╔╝██║     ███████╗██║ ╚████║██║  ██║╚██████╔╝╚██████╔╝   ██║   ███████╗██║  ██║
 ╚═════╝ ╚═╝     ╚══════╝╚═╝  ╚═══╝╚═╝  ╚═╝ ╚═════╝  ╚═════╝    ╚═╝   ╚══════╝╚═╝  ╚═╝`;

// ═══════════════════════════════════════════════════════════════════════════════
// Initialization
// ═══════════════════════════════════════════════════════════════════════════════
if (!process.env.OPENROUTER_API_KEY) {
  console.error('\n\x1b[31m✖ Error: OPENROUTER_API_KEY required\x1b[0m');
  console.error('\x1b[90mCreate a .env file with: OPENROUTER_API_KEY=your-key-here\x1b[0m\n');
  process.exit(1);
}

const agent = createAgent({
  apiKey: process.env.OPENROUTER_API_KEY,
  model: 'openai/gpt-4o',
  instructions: `You are an autonomous coding agent. You can read files, write code, run commands, search the web, and read documentation.

IMPORTANT: After using tools, ALWAYS provide a summary of what you found and your insights. Don't just call tools and stop - synthesize the information into a helpful response.

When presenting information:
- Use markdown formatting for readability
- Use bullet points for lists
- Use code blocks for code
- Be concise but thorough`,
  maxToolRounds: 50,
});

// ═══════════════════════════════════════════════════════════════════════════════
// Types
// ═══════════════════════════════════════════════════════════════════════════════
interface ToolCallInfo { name: string; status: 'running' | 'complete'; args?: string; }
interface DisplayMessage { role: 'user' | 'assistant'; content: string; timestamp: Date; }

// ═══════════════════════════════════════════════════════════════════════════════
// Components
// ═══════════════════════════════════════════════════════════════════════════════
function Header() {
  const { stdout } = useStdout();
  const width = stdout?.columns ?? 100;

  if (width >= 95) {
    return (
      <Box flexDirection="column" marginBottom={1}>
        <Text color="cyan">{BANNER}</Text>
        <Box marginTop={1}>
          <Text color="gray">━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━</Text>
        </Box>
        <Box>
          <Text color="gray"> Coding Agent </Text>
          <Text color="magenta">• </Text>
          <Text color="gray">Model: </Text>
          <Text color="cyan">gpt-4o</Text>
          <Text color="magenta"> • </Text>
          <Text color="gray">Press </Text>
          <Text color="yellow">ESC</Text>
          <Text color="gray"> to exit</Text>
        </Box>
      </Box>
    );
  }
  return (
    <Box flexDirection="column" marginBottom={1}>
      <Box><Text bold color="cyan">◆ OPENROUTER</Text><Text color="gray"> Coding Agent</Text></Box>
      <Text color="gray">─────────────────────────────────────────</Text>
    </Box>
  );
}

function InputBox({ value, onChange, onSubmit, disabled }: {
  value: string; onChange: (v: string) => void; onSubmit: () => void; disabled: boolean;
}) {
  useInput((input, key) => {
    if (disabled) return;
    if (key.return) onSubmit();
    else if (key.backspace || key.delete) onChange(value.slice(0, -1));
    else if (input && !key.ctrl && !key.meta) onChange(value + input);
  });
  return (
    <Box borderStyle="round" borderColor={disabled ? 'gray' : 'cyan'} paddingX={1}>
      <Text color={disabled ? 'gray' : 'green'}>❯ </Text>
      <Text>{value}</Text>
      {!disabled && <Text color="cyan">█</Text>}
      {disabled && <Text color="gray"> thinking...</Text>}
    </Box>
  );
}

function ToolCallDisplay({ tools }: { tools: ToolCallInfo[] }) {
  if (!tools.length) return null;
  return (
    <Box flexDirection="column" marginY={1} paddingLeft={2}>
      {tools.map((tc, i) => (
        <Box key={i}>
          <Text color={tc.status === 'complete' ? 'green' : 'yellow'}>
            {tc.status === 'complete' ? '✓' : '⚡'}
          </Text>
          <Text color="gray"> {tc.name}</Text>
          {tc.args && <Text color="gray" dimColor> {tc.args.slice(0, 40)}...</Text>}
        </Box>
      ))}
    </Box>
  );
}

function MessageDisplay({ msg }: { msg: DisplayMessage }) {
  const isUser = msg.role === 'user';
  const time = msg.timestamp.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  // Apply markdown formatting to assistant messages only
  const displayContent = isUser ? msg.content : formatMarkdown(msg.content);
  return (
    <Box flexDirection="column" marginBottom={1}>
      <Box>
        <Text bold color={isUser ? 'blue' : 'green'}>{isUser ? '● You' : '◆ Assistant'}</Text>
        <Text color="gray" dimColor> {time}</Text>
      </Box>
      <Box paddingLeft={2}><Text wrap="wrap">{displayContent}</Text></Box>
    </Box>
  );
}

function StatusBar({ status }: { status: string }) {
  return <Box marginTop={1}><Text color="gray">─ </Text><Text color="cyan">{status}</Text><Text color="gray"> ─</Text></Box>;
}

// ═══════════════════════════════════════════════════════════════════════════════
// Main App
// ═══════════════════════════════════════════════════════════════════════════════
function App() {
  const { exit } = useApp();
  const [messages, setMessages] = useState<DisplayMessage[]>([]);
  const [input, setInput] = useState('');
  const [isLoading, setIsLoading] = useState(false);
  const [toolCalls, setToolCalls] = useState<ToolCallInfo[]>([]);
  const [streamingText, setStreamingText] = useState('');
  const [error, setError] = useState<string | null>(null);
  const [status, setStatus] = useState('Ready');

  useInput((_, key) => { if (key.escape) exit(); });

  useEffect(() => {
    const onStart = () => { setIsLoading(true); setToolCalls([]); setStreamingText(''); setError(null); setStatus('Processing...'); };
    const onDelta = (_d: string, acc: string) => { setStreamingText(acc); setStatus('Streaming...'); };
    const onToolCall = (name: string, args: unknown) => {
      setToolCalls(prev => [...prev, { name, status: 'running', args: JSON.stringify(args) }]);
      setStatus(`Running ${name}...`);
    };
    const onToolResult = () => setToolCalls(prev => prev.map(tc => tc.status === 'running' ? { ...tc, status: 'complete' as const } : tc));
    const onDone = () => {
      setMessages(agent.getMessages().map(m => ({ role: m.role as 'user' | 'assistant', content: m.content, timestamp: new Date() })));
      setToolCalls([]); setStreamingText(''); setIsLoading(false); setStatus('Ready');
    };
    const onError = (err: Error) => { setError(err.message); setIsLoading(false); setStatus('Error'); };

    agent.on('thinking:start', onStart);
    agent.on('stream:delta', onDelta);
    agent.on('tool:call', onToolCall);
    agent.on('tool:result', onToolResult);
    agent.on('message:assistant', onDone);
    agent.on('error', onError);
    return () => { agent.off('thinking:start', onStart); agent.off('stream:delta', onDelta); agent.off('tool:call', onToolCall); agent.off('tool:result', onToolResult); agent.off('message:assistant', onDone); agent.off('error', onError); };
  }, []);

  const send = useCallback(async () => {
    if (!input.trim() || isLoading) return;
    const text = input.trim();
    setInput('');
    setMessages(prev => [...prev, { role: 'user', content: text, timestamp: new Date() }]);
    try { await agent.send(text); } catch (e) { setError(e instanceof Error ? e.message : String(e)); setIsLoading(false); }
  }, [input, isLoading]);

  return (
    <Box flexDirection="column" paddingX={1} paddingY={1}>
      <Header />
      <Box flexDirection="column" flexGrow={1}>
        {messages.map((msg, i) => <MessageDisplay key={i} msg={msg} />)}
        <ToolCallDisplay tools={toolCalls} />
        {streamingText && isLoading && (
          <Box flexDirection="column" marginBottom={1}>
            <Box><Text bold color="green">◆ Assistant</Text><Text color="yellow"> (streaming...)</Text></Box>
            <Box paddingLeft={2}><Text wrap="wrap">{streamingText}</Text></Box>
          </Box>
        )}
        {isLoading && !streamingText && !toolCalls.length && <Box marginY={1} paddingLeft={2}><Text color="cyan">◌ </Text><Text color="gray">Thinking...</Text></Box>}
        {error && <Box marginY={1} paddingLeft={2}><Text color="red">✖ Error: {error}</Text></Box>}
      </Box>
      <StatusBar status={status} />
      <Box marginTop={1}><InputBox value={input} onChange={setInput} onSubmit={send} disabled={isLoading} /></Box>
    </Box>
  );
}

render(<App />);
```

---

### Step 8: Create src/headless.ts

Useful for CI/CD pipelines or API integration.

```typescript
import 'dotenv/config';
import { createAgent } from './agent.js';
import * as readline from 'readline';

async function main() {
  if (!process.env.OPENROUTER_API_KEY) {
    console.error('Error: OPENROUTER_API_KEY environment variable is required');
    process.exit(1);
  }

  const agent = createAgent({
    apiKey: process.env.OPENROUTER_API_KEY,
    model: 'openai/gpt-4o',
    instructions: 'You are a capable coding agent. You can inspect files, write code, run tests, and browse the web.',
    maxToolRounds: 50,
  });

  agent.on('thinking:start', () => console.log('\n🤔 Thinking...'));
  agent.on('tool:call', (name, args) => console.log(`🔧 Using ${name}:`, JSON.stringify(args)));
  agent.on('tool:result', (name) => console.log(`  ✅ Result from ${name}`));
  agent.on('stream:delta', (delta) => process.stdout.write(delta));
  agent.on('stream:end', () => console.log('\n'));
  agent.on('error', (err) => console.error('❌ Error:', err.message));

  const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
  console.log('🤖 OpenRouter Coding Agent (Headless Mode)\nType your message (Ctrl+C to exit):\n');

  const prompt = () => {
    rl.question('You: ', async (input) => {
      if (!input.trim()) { prompt(); return; }
      try { await agent.send(input); } catch (error) { console.error('Error:', error); }
      prompt();
    });
  };
  prompt();
}

main().catch(console.error);
```

---

## Running the Agent

1. **Add your API key to `.env`:**
   ```
   OPENROUTER_API_KEY=sk-or-v1-xxxxx
   ```

2. **Run the TUI:**
   ```bash
   npm start
   ```

3. **Or run in headless mode:**
   ```bash
   npm run start:headless
   ```
