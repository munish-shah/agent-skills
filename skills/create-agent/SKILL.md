---
name: create-agent
description: Bootstrap a modular AI agent with OpenRouter SDK callModel() API, Zod tools, and Ink TUI
metadata:
  version: 0.4.0
  homepage: https://openrouter.ai
---

# Build a Modular AI Agent with OpenRouter

This skill helps you create a **terminal-based agent**. Unlike simple chatbots, this agent can:

- **Read & Write Files** - Modify your codebase directly
- **Edit Files** - Targeted find-and-replace edits
- **Execute Shell Commands** - Run tests, installs, and builds
- **Browse the Web** - Fetch documentation and search the internet
- **Look Good** - Includes a modern Ink-based Terminal UI (TUI)
- **OpenRouter SDK callModel()** - Automatic tool execution with items-based streaming

## Architecture

```
┌─────────────────────────────────────────────────┐
│       cli.tsx (TUI) / headless.ts (CLI)          │
│         UI layer + user interaction              │
├─────────────────────────────────────────────────┤
│               config.ts (Setup)                  │
│  Env validation, shared agent instance           │
├─────────────────────────────────────────────────┤
│                agent.ts (Core)                   │
│  OpenRouter SDK callModel() + getItemsStream()   │
│  EventEmitter for UI hooks                       │
├─────────────────────────────────────────────────┤
│                tools.ts (Tools)                  │
│  7 Zod-schema tools with callbacks               │
│  list_files, read_file, edit_file, write_file,   │
│  run_command, fetch_web_page, web_search         │
└─────────────────────────────────────────────────┘
```

**How it works:**

1. `config.ts` validates the API key and creates a shared agent instance
2. Agent calls `callModel()` with tools
3. SDK automatically validates args with Zod, executes tools, sends results back to model
4. SDK repeats until the model stops calling tools
5. `getItemsStream()` streams all items (messages, tool calls, results) to the TUI
6. Tool callbacks emit events for the UI's tool indicators

## Prerequisites

> [!IMPORTANT]
> **Node.js LTS (latest stable even-numbered release) required.**
> Run `node -v` to check. If below the current LTS, upgrade via [nodejs.org](https://nodejs.org) or `nvm install --lts && nvm alias default lts/*`.
> All dependencies use unpinned ranges (`npm install` fetches latest compatible versions), so no version numbers in this skill will go stale.

Get an OpenRouter API key at: https://openrouter.ai/settings/keys

> [!CAUTION]
> **Dependency Safety Rules (MUST follow):**
>
> - Use the dependencies listed in the package.json below
> - Do NOT add `cheerio`, `undici`, or any package with native bindings — these frequently break across Node versions
> - For HTML parsing, use the built-in regex-based `htmlToText()` helper in tools.ts — no external HTML parser needed
> - After `npm install`, verify zero errors by running `npm start` before considering setup complete

## Project Setup

### Step 1: Initialize Project

Verify Node.js LTS is available, then scaffold:

```bash
node -v  # must be current LTS (even-numbered: 20, 22, 24, …)
mkdir my-agent && cd my-agent
npm init -y
npm pkg set type="module"
mkdir src
echo -e 'node_modules/\ndist/\n.env\n.DS_Store' > .gitignore
```

### Step 2: Install Dependencies

Install runtime dependencies (no version pinning — always fetches latest compatible):

```bash
npm install @openrouter/sdk dotenv eventemitter3 glob ink react zod
```

| Package           | Purpose                                             | npm                                                                                |
| ----------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `@openrouter/sdk` | LLM API client with streaming & tool-call support   | [npmjs.com/package/@openrouter/sdk](https://www.npmjs.com/package/@openrouter/sdk) |
| `dotenv`          | Loads `.env` vars into `process.env`                | [npmjs.com/package/dotenv](https://www.npmjs.com/package/dotenv)                   |
| `eventemitter3`   | Lightweight event bus for agent ↔ UI communication  | [npmjs.com/package/eventemitter3](https://www.npmjs.com/package/eventemitter3)     |
| `glob`            | File-pattern matching for tool implementations      | [npmjs.com/package/glob](https://www.npmjs.com/package/glob)                       |
| `ink`             | React-based terminal UI framework                   | [npmjs.com/package/ink](https://www.npmjs.com/package/ink)                         |
| `react`           | Required peer dependency for Ink                    | [npmjs.com/package/react](https://www.npmjs.com/package/react)                     |
| `zod`             | Schema validation used for defining tool parameters | [npmjs.com/package/zod](https://www.npmjs.com/package/zod)                         |

Install dev dependencies:

```bash
npm install -D @types/node @types/react tsx typescript
```

| Package        | Purpose                                                  | npm                                                                          |
| -------------- | -------------------------------------------------------- | ---------------------------------------------------------------------------- |
| `@types/node`  | TypeScript types for Node.js APIs                        | [npmjs.com/package/@types/node](https://www.npmjs.com/package/@types/node)   |
| `@types/react` | TypeScript types for React (needed by Ink)               | [npmjs.com/package/@types/react](https://www.npmjs.com/package/@types/react) |
| `tsx`          | TypeScript execution engine (runs `.ts`/`.tsx` directly) | [npmjs.com/package/tsx](https://www.npmjs.com/package/tsx)                   |
| `typescript`   | TypeScript compiler                                      | [npmjs.com/package/typescript](https://www.npmjs.com/package/typescript)     |

Then add the scripts to `package.json`:

```bash
npm pkg set scripts.start="tsx src/cli.tsx"
npm pkg set scripts.start:headless="tsx src/headless.ts"
npm pkg set scripts.dev="tsx watch src/cli.tsx"
```

### Step 3: Create .env file

Create a `.env` file in the project root with your API key and (optionally) a model:

```
OPENROUTER_API_KEY=your-key-here
MODEL=openrouter/auto
```

`MODEL` is optional — it defaults to `openrouter/auto` (smart routing). Set it to any tool-capable model ID from [openrouter.ai/models](https://openrouter.ai/models).

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
├── config.ts       # Shared setup: env validation, agent initialization
├── tools.ts        # 7 Zod-schema tools with callbacks for UI events
├── cli.tsx         # Ink TUI: ASCII banner, streaming, tool indicators
└── headless.ts     # readline-based CLI mode
```

---

## Implementation

### Step 5: Create src/agent.ts

The agent uses `callModel()` which handles the entire tool execution loop internally. No manual while loop needed — the SDK validates tool args with Zod, executes tools, and re-queries the model automatically. We consume `getItemsStream()` to drive the full loop and extract text deltas for streaming. Tool callbacks emit events to the UI during execution.

```typescript
import { OpenRouter, stepCountIs } from '@openrouter/sdk';
import { EventEmitter } from 'eventemitter3';
import { createTools } from './tools.js';

// ─── Types ───────────────────────────────────────────────────────────────────

export interface Message {
  role: 'user' | 'assistant';
  content: string;
}

export interface ModelInfo {
  id: string;
  name: string;
  contextLength: number | null;
  promptPricing: string;
  completionPricing: string;
}

export interface AgentEvents {
  'message:user': (message: Message) => void;
  'message:assistant': (message: Message) => void;
  'stream:start': () => void;
  'stream:delta': (delta: string, accumulated: string) => void;
  'stream:end': (fullText: string) => void;
  'tool:call': (name: string, args: Record<string, unknown>) => void;
  'tool:result': (name: string, result: Record<string, unknown>) => void;
  'model:changed': (modelId: string) => void;
  error: (error: Error) => void;
  'thinking:start': () => void;
  'thinking:end': () => void;
}

export interface AgentConfig {
  apiKey: string;
  /** Model ID from OpenRouter (e.g. "anthropic/claude-sonnet-4"). Defaults to "openrouter/auto". */
  model?: string;
  instructions?: string;
  maxToolRounds?: number;
}

export const DEFAULT_MODEL = 'openrouter/auto';

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
      model: config.model ?? DEFAULT_MODEL,
      instructions: config.instructions ?? 'You are a skilled AI assistant.',
      maxToolRounds: config.maxToolRounds ?? 50,
    };
    // Tool callbacks are the single source of truth for tool events.
    // No duplicate emissions from the stream — callbacks fire during execute().
    this.tools = createTools({
      onCall: (name, args) =>
        this.emit('tool:call', name, args as Record<string, unknown>),
      onResult: (name, result) =>
        this.emit('tool:result', name, result as Record<string, unknown>),
    });
  }

  /** The model ID currently in use (useful for displaying in UI). */
  get model(): string {
    return this.config.model;
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

  setModel(modelId: string): void {
    this.config.model = modelId;
    this.emit('model:changed', modelId);
  }

  async listToolCapableModels(): Promise<ModelInfo[]> {
    const response = await this.client.models.list({
      supportedParameters: 'tools',
    });

    const models = response.data.map((m) => ({
      id: m.id,
      name: m.name,
      contextLength: m.contextLength,
      promptPricing: m.pricing.prompt,
      completionPricing: m.pricing.completion,
    }));

    // Provider priority order (earlier = higher priority)
    const providerOrder = ['anthropic', 'openai', 'google', 'meta-llama', 'x-ai', 'deepseek', 'mistralai'];

    const getProviderPriority = (id: string): number => {
      const provider = id.split('/')[0];
      const index = providerOrder.indexOf(provider);
      return index === -1 ? providerOrder.length : index;
    };

    // Sort by: provider priority first, then alphabetically within each provider
    return models.sort((a, b) => {
      const priorityDiff = getProviderPriority(a.id) - getProviderPriority(b.id);
      if (priorityDiff !== 0) return priorityDiff;
      return a.name.localeCompare(b.name);
    });
  }

  async send(content: string): Promise<string> {
    const userMessage: Message = { role: 'user', content };
    this.history.push(userMessage);
    this.emit('message:user', userMessage);
    this.emit('thinking:start');
    this.emit('stream:start');

    try {
      // callModel() handles the entire tool execution loop automatically.
      // Tool callbacks emit tool:call / tool:result events during execution.
      // We consume getItemsStream() to drive the full loop (including tool rounds)
      // and extract text deltas from 'message' items for streaming to the UI.
      const result = this.client.callModel({
        model: this.config.model,
        instructions: this.config.instructions,
        input: this.history.map((m) => ({
          role: m.role,
          content: m.content,
        })),
        tools: this.tools,
        stopWhen: stepCountIs(this.config.maxToolRounds),
      });

      // getItemsStream() drives the full tool execution loop.
      // We only extract text here — tool events come from callbacks.
      let fullText = '';
      for await (const item of result.getItemsStream()) {
        if (item.type !== 'message') continue;
        const parts = (item as { content?: Array<{ type: string; text?: string }> }).content;
        if (!parts) continue;
        const currentText = parts
          .filter((p) => p.type === 'output_text' && p.text)
          .map((p) => p.text!)
          .join('');
        if (currentText.length <= fullText.length) continue;
        const delta = currentText.slice(fullText.length);
        fullText = currentText;
        this.emit('stream:delta', delta, fullText);
      }

      this.emit('stream:end', fullText);

      const assistantMessage: Message = {
        role: 'assistant',
        content: fullText,
      };
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
import { dirname } from 'path';
import { exec } from 'child_process';
import { promisify } from 'util';
import { glob } from 'glob';

const execAsync = promisify(exec);

const COMMAND_TIMEOUT_MS = 30_000;
const FETCH_TIMEOUT_MS = 15_000;

function toErrorMessage(e: unknown): string {
  return e instanceof Error ? e.message : String(e);
}

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
  const done = <T>(name: string, result: T): T => {
    callbacks?.onResult?.(name, result);
    return result;
  };

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
          return done('list_files', {
            files: files.slice(0, 100),
            count: files.length,
          });
        } catch (e) {
          return done('list_files', { error: toErrorMessage(e) });
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
          return done('read_file', {
            filepath: params.filepath,
            content,
            length: content.length,
          });
        } catch (e) {
          return done('read_file', {
            error: `Failed to read ${params.filepath}: ${toErrorMessage(e)}`,
          });
        }
      },
    }),

    tool({
      name: 'edit_file',
      description:
        'Make targeted edits to a file by replacing specific text. Use this instead of write_file when you only need to change part of a file.',
      inputSchema: z.object({
        filepath: z.string().describe('Path to the file to edit'),
        old_string: z
          .string()
          .describe('The exact text to find and replace (must match exactly)'),
        new_string: z.string().describe('The replacement text'),
        replace_all: z
          .boolean()
          .default(false)
          .describe('Replace all occurrences instead of just the first'),
      }),
      execute: async (params) => {
        call('edit_file', params);
        try {
          const content = await fs.readFile(params.filepath, 'utf-8');
          if (!content.includes(params.old_string)) {
            return done('edit_file', {
              error: `old_string not found in ${params.filepath}. Make sure it matches exactly, including whitespace and indentation.`,
            });
          }
          const occurrences = content.split(params.old_string).length - 1;
          if (occurrences > 1 && !params.replace_all) {
            return done('edit_file', {
              error: `old_string found ${occurrences} times in ${params.filepath}. Provide more context to make it unique, or set replace_all to true.`,
            });
          }
          const updated = params.replace_all
            ? content.replaceAll(params.old_string, params.new_string)
            : content.replace(params.old_string, params.new_string);
          await fs.writeFile(params.filepath, updated, 'utf-8');
          return done('edit_file', {
            success: true,
            filepath: params.filepath,
            replacements: params.replace_all ? occurrences : 1,
          });
        } catch (e) {
          return done('edit_file', {
            error: `Failed to edit ${params.filepath}: ${toErrorMessage(e)}`,
          });
        }
      },
    }),

    tool({
      name: 'write_file',
      description: 'Write content to a file (creates parent directories if needed). Overwrites existing content.',
      inputSchema: z.object({
        filepath: z.string().describe('Path to the file to write'),
        content: z.string().describe('Content to write'),
      }),
      execute: async (params) => {
        call('write_file', params);
        try {
          await fs.mkdir(dirname(params.filepath), { recursive: true });
          await fs.writeFile(params.filepath, params.content, 'utf-8');
          return done('write_file', {
            success: true,
            filepath: params.filepath,
            size: Buffer.byteLength(params.content, 'utf-8'),
          });
        } catch (e) {
          return done('write_file', {
            error: `Failed to write ${params.filepath}: ${toErrorMessage(e)}`,
          });
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
          const { stdout, stderr } = await execAsync(params.command, {
            timeout: COMMAND_TIMEOUT_MS,
          });
          return done('run_command', {
            command: params.command,
            stdout: stdout.trim().slice(0, 2000),
            stderr: stderr.trim().slice(0, 500),
          });
        } catch (e) {
          const err = e as Error & { stdout?: string; stderr?: string };
          return done('run_command', {
            error: 'Command failed',
            message: err.message,
            stdout: err.stdout?.slice(0, 1000),
            stderr: err.stderr?.slice(0, 500),
          });
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
          const controller = new AbortController();
          setTimeout(() => controller.abort(), FETCH_TIMEOUT_MS);
          const res = await fetch(params.url, { signal: controller.signal });
          if (!res.ok) throw new Error(`HTTP ${res.status}`);
          const html = await res.text();
          const title = extractTitle(html);
          const text = htmlToText(html).slice(0, 4000);
          return done('fetch_web_page', {
            url: params.url,
            title,
            content: text,
          });
        } catch (e) {
          return done('fetch_web_page', {
            error: `Failed to fetch ${params.url}: ${toErrorMessage(e)}`,
          });
        }
      },
    }),

    tool({
      name: 'web_search',
      description:
        'Search the internet for information using DuckDuckGo. Returns titles, URLs, and snippets.',
      inputSchema: z.object({
        query: z.string().describe('The search query'),
        num_results: z
          .number()
          .default(5)
          .describe('Number of results (max 10)'),
      }),
      execute: async (params) => {
        call('web_search', params);
        const numResults = Math.min(params.num_results, 10);
        try {
          const searchUrl = `https://html.duckduckgo.com/html/?q=${encodeURIComponent(params.query)}`;
          const controller = new AbortController();
          setTimeout(() => controller.abort(), FETCH_TIMEOUT_MS);
          const res = await fetch(searchUrl, {
            signal: controller.signal,
            headers: {
              'User-Agent':
                'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
            },
          });
          if (!res.ok) throw new Error(`HTTP ${res.status}`);
          const html = await res.text();
          const results: Array<{
            title: string;
            url: string;
            snippet: string;
          }> = [];
          const resultBlocks = html.split(/class="result\s/);
          for (
            let i = 1;
            i < resultBlocks.length && results.length < numResults;
            i++
          ) {
            const block = resultBlocks[i];
            const titleMatch = block.match(/class="result__a"[^>]*>([^<]+)</);
            const hrefMatch = block.match(/class="result__a"\s+href="([^"]+)"/);
            const snippetMatch = block.match(
              /class="result__snippet"[^>]*>([\s\S]*?)<\/a>/,
            );
            const title = titleMatch ? titleMatch[1].trim() : '';
            let url = hrefMatch ? hrefMatch[1] : '';
            if (url.startsWith('//')) url = `https:${url}`;
            const snippet = snippetMatch
              ? htmlToText(snippetMatch[1]).slice(0, 200)
              : '';
            if (title && url) results.push({ title, url, snippet });
          }
          return done('web_search', {
            query: params.query,
            results,
            count: results.length,
          });
        } catch (e) {
          return done('web_search', {
            error: `Search failed: ${toErrorMessage(e)}`,
          });
        }
      },
    }),
  ];
}
```

---

### Step 7: Create src/config.ts

Shared setup module imported by both entry points. This is the single source of truth for API key validation, system instructions, and agent initialization — avoiding duplication between `cli.tsx` and `headless.ts`.

```typescript
import 'dotenv/config';
import { createAgent } from './agent.js';

if (!process.env.OPENROUTER_API_KEY) {
  console.error('Error: OPENROUTER_API_KEY environment variable is required');
  console.error('Create a .env file with: OPENROUTER_API_KEY=your-key-here');
  process.exit(1);
}

// SKILL INSTRUCTION: Replace with a system prompt tailored to the user's agent name and purpose.
export const agent = createAgent({
  apiKey: process.env.OPENROUTER_API_KEY,
  model: process.env.MODEL,
  instructions: `You are a helpful AI Agent with access to tools for working with files, running commands, writing code, and browsing the web. Use these tools when they're helpful for the task at hand, but you're not limited to just using tools - you can also engage in conversation, answer questions, provide advice, or help with any other request.

When using tools, always provide context and synthesis - don't just call tools silently. Explain what you're doing and why.`,
});
```

---

### Step 8: Create src/cli.tsx

A polished terminal interface with ASCII art banner, streaming text, and tool call indicators.

````typescript
import React, { useState, useEffect, useCallback } from 'react';
import { render, Box, Text, useInput, useApp, useStdout } from 'ink';
import { DEFAULT_MODEL, type ModelInfo } from './agent.js';
import { agent } from './config.js';

// ═══════════════════════════════════════════════════════════════════════════════
// Theme — centralised so every muted/secondary colour is easy to tune
// ═══════════════════════════════════════════════════════════════════════════════
const THEME = {
  muted: '#999999',   // secondary text, separators, labels
  mutedDim: '#777777' // even less prominent (tool args, pagination)
} as const;

// ═══════════════════════════════════════════════════════════════════════════════
// Simple markdown cleaner for terminal display
// ═══════════════════════════════════════════════════════════════════════════════
function formatMarkdown(text: string): string {
  return text
    .replace(/^### (.+)$/gm, '\n░ $1')
    .replace(/^## (.+)$/gm, '\n▓ $1')
    .replace(/^# (.+)$/gm, '\n█ $1 █')
    .replace(/\*\*([^*]+)\*\*/g, '$1')
    .replace(/\*([^*]+)\*/g, '$1')
    .replace(/_([^_]+)_/g, '$1')
    .replace(/^[\s]*[-*]\s+/gm, '  • ')
    .replace(/```[\w]*\n?([\s\S]*?)```/g, '\n$1\n')
    .replace(/`([^`]+)`/g, '$1')
    .replace(/\[([^\]]+)\]\([^)]+\)/g, '$1')
    .replace(/\n{3,}/g, '\n\n')
    .trim();
}

// ═══════════════════════════════════════════════════════════════════════════════
// ASCII Banner
// ═══════════════════════════════════════════════════════════════════════════════
// SKILL INSTRUCTION: Ask the user for an ASCII art banner for their agent.
// If they provide one, paste it into the template string below.
// If they don't have one, generate one with: npx -y figlet -f "ANSI Shadow" "AGENT NAME"
// Then paste the output directly into the template literal below (no trimStart).
// The figlet output includes leading spaces for alignment — preserve them exactly.
// If figlet is unavailable, use a simple text fallback:
//   const BANNER = `◆  MY AGENT`;
const BANNER = `<FIGLET ASCII BANNER — see instructions above>`;

// ═══════════════════════════════════════════════════════════════════════════════
// Types
// ═══════════════════════════════════════════════════════════════════════════════
interface ToolCallInfo { name: string; status: 'running' | 'complete'; args?: string; }
interface DisplayMessage { role: 'user' | 'assistant'; content: string; timestamp: Date; }

// ═══════════════════════════════════════════════════════════════════════════════
// Components
// ═══════════════════════════════════════════════════════════════════════════════
// SKILL INSTRUCTION: Replace "MY AGENT" with the user's agent name in Header and MessageDisplay.
function Header({ currentModel }: { currentModel: string }) {
  const { stdout } = useStdout();
  const width = stdout?.columns ?? 100;
  const bannerWidth = BANNER.split('\n').reduce((max, line) => Math.max(max, line.length), 0);

  if (width >= bannerWidth + 2) {
    return (
      <Box flexDirection="column" marginBottom={1}>
        <Text color="cyan">{BANNER}</Text>
        <Box marginTop={1}>
          <Text color={THEME.muted}>{'━'.repeat(Math.min(width - 2, bannerWidth))}</Text>
        </Box>
        <Box>
          <Text color={THEME.muted}>Model: </Text>
          <Text color="cyan">{currentModel}</Text>
          <Text color="magenta"> • </Text>
          <Text color={THEME.muted}>Type </Text>
          <Text color="yellow">/model</Text>
          <Text color={THEME.muted}> to switch</Text>
          <Text color="magenta"> • </Text>
          <Text color={THEME.muted}>Press </Text>
          <Text color="yellow">ESC</Text>
          <Text color={THEME.muted}> to exit</Text>
        </Box>
      </Box>
    );
  }
  return (
    <Box flexDirection="column" marginBottom={1}>
      <Box><Text bold color="cyan">◆ MY AGENT</Text></Box>
      <Box>
        <Text color={THEME.muted}>Model: </Text>
        <Text color="cyan">{currentModel}</Text>
      </Box>
      <Text color={THEME.muted}>{'─'.repeat(Math.min(width - 2, 40))}</Text>
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
    <Box borderStyle="round" borderColor={disabled ? THEME.muted : 'cyan'} paddingX={1}>
      <Text wrap="wrap">
        <Text color={disabled ? THEME.muted : 'green'}>❯ </Text>
        <Text>{value}</Text>
        {!disabled && <Text color="cyan">█</Text>}
        {disabled && <Text color={THEME.muted}> thinking...</Text>}
      </Text>
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
          <Text color={THEME.muted}> {tc.name}</Text>
          {tc.args && <Text color={THEME.mutedDim}> {tc.args.slice(0, 40)}...</Text>}
        </Box>
      ))}
    </Box>
  );
}

function MessageDisplay({ msg }: { msg: DisplayMessage }) {
  const isUser = msg.role === 'user';
  const time = msg.timestamp.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
  const displayContent = isUser ? msg.content : formatMarkdown(msg.content);
  return (
    <Box flexDirection="column" marginBottom={1}>
      <Box>
        <Text bold color={isUser ? 'blue' : 'green'}>{isUser ? '● You' : '◆ MY AGENT'}</Text>
        <Text color={THEME.mutedDim}> {time}</Text>
      </Box>
      <Box paddingLeft={2}><Text wrap="wrap">{displayContent}</Text></Box>
    </Box>
  );
}

function StatusBar({ status }: { status: string }) {
  return <Box marginTop={1}><Text color={THEME.muted}>─ </Text><Text color="cyan">{status}</Text><Text color={THEME.muted}> ─</Text></Box>;
}

function ModelPicker({ models, currentModel, loading, onSelect, onCancel }: {
  models: ModelInfo[] | null;
  currentModel: string;
  loading: boolean;
  onSelect: (modelId: string) => void;
  onCancel: () => void;
}) {
  const [index, setIndex] = useState(0);
  const items: ModelInfo[] = models
    ? [{ id: DEFAULT_MODEL, name: 'Default (openrouter/auto)', contextLength: null, promptPricing: '0', completionPricing: '0' }, ...models]
    : [];

  useInput((_, key) => {
    if (loading) return;
    if (key.escape) { onCancel(); return; }
    if (key.return && items.length > 0) { onSelect(items[index].id); return; }
    if (key.upArrow) setIndex(i => Math.max(0, i - 1));
    if (key.downArrow) setIndex(i => Math.min(items.length - 1, i + 1));
  });

  if (loading) {
    return (
      <Box marginY={1} paddingLeft={2}>
        <Text color="cyan">Fetching tool-capable models...</Text>
      </Box>
    );
  }

  const WINDOW = 15;
  const start = Math.max(0, Math.min(index - Math.floor(WINDOW / 2), items.length - WINDOW));
  const visible = items.slice(start, start + WINDOW);

  return (
    <Box flexDirection="column" marginY={1} borderStyle="round" borderColor="cyan" paddingX={1}>
      <Text bold color="cyan">Select Model (↑↓ + Enter, ESC to cancel)</Text>
      <Box marginTop={1} flexDirection="column">
        {visible.map((m, i) => {
          const realIndex = start + i;
          const isHighlighted = realIndex === index;
          const isActive = m.id === currentModel;
          const ctx = m.contextLength ? ` (${(m.contextLength / 1000).toFixed(0)}k)` : '';
          return (
            <Box key={m.id}>
              <Text color={isHighlighted ? 'cyan' : THEME.muted}>
                {isHighlighted ? '❯ ' : '  '}{isActive ? '✓ ' : '  '}{m.name}{ctx}
              </Text>
            </Box>
          );
        })}
      </Box>
      {items.length > WINDOW && (
        <Text color={THEME.mutedDim}>  ({index + 1}/{items.length})</Text>
      )}
    </Box>
  );
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
  const [currentModel, setCurrentModel] = useState(agent.model);
  const [showModelPicker, setShowModelPicker] = useState(false);
  const [models, setModels] = useState<ModelInfo[] | null>(null);
  const [modelLoading, setModelLoading] = useState(false);

  useInput((_, key) => { if (key.escape && !showModelPicker) exit(); });

  useEffect(() => {
    const onStart = () => { setIsLoading(true); setToolCalls([]); setStreamingText(''); setError(null); setStatus('Processing...'); };
    const onDelta = (_d: string, acc: string) => { setStreamingText(acc); setStatus('Streaming...'); };
    const onToolCall = (name: string, args: unknown) => {
      setToolCalls(prev => [...prev, { name, status: 'running', args: JSON.stringify(args) }]);
      setStatus(`Running ${name}...`);
    };
    const onToolResult = () => setToolCalls(prev => prev.map(tc => tc.status === 'running' ? { ...tc, status: 'complete' as const } : tc));
    const onDone = () => {
      setMessages(agent.getMessages().map(m => ({ role: m.role, content: m.content, timestamp: new Date() })));
      setToolCalls([]); setStreamingText(''); setIsLoading(false); setStatus('Ready');
    };
    const onModelChanged = (modelId: string) => setCurrentModel(modelId);

    agent.on('thinking:start', onStart);
    agent.on('stream:delta', onDelta);
    agent.on('tool:call', onToolCall);
    agent.on('tool:result', onToolResult);
    agent.on('message:assistant', onDone);
    agent.on('model:changed', onModelChanged);
    return () => {
      agent.off('thinking:start', onStart);
      agent.off('stream:delta', onDelta);
      agent.off('tool:call', onToolCall);
      agent.off('tool:result', onToolResult);
      agent.off('message:assistant', onDone);
      agent.off('model:changed', onModelChanged);
    };
  }, []);

  const openModelPicker = useCallback(async () => {
    setShowModelPicker(true);
    if (!models) {
      setModelLoading(true);
      try {
        const list = await agent.listToolCapableModels();
        setModels(list);
      } catch (e) {
        setError(`Failed to fetch models: ${e instanceof Error ? e.message : String(e)}`);
        setShowModelPicker(false);
      } finally {
        setModelLoading(false);
      }
    }
  }, [models]);

  const send = useCallback(async () => {
    if (!input.trim() || isLoading) return;
    const text = input.trim();
    setInput('');
    if (text === '/model') {
      openModelPicker();
      return;
    }
    setMessages(prev => [...prev, { role: 'user', content: text, timestamp: new Date() }]);
    try {
      await agent.send(text);
    } catch (e) {
      setError(e instanceof Error ? e.message : String(e));
      setIsLoading(false);
    }
  }, [input, isLoading, openModelPicker]);

  return (
    <Box flexDirection="column" paddingX={1} paddingY={1}>
      <Header currentModel={currentModel} />
      <Box flexDirection="column" flexGrow={1}>
        {messages.map((msg, i) => <MessageDisplay key={i} msg={msg} />)}
        <ToolCallDisplay tools={toolCalls} />
        {streamingText && isLoading && (
          <Box flexDirection="column" marginBottom={1}>
            {/* SKILL INSTRUCTION: Replace "MY AGENT" with the user's agent name */}
            <Box><Text bold color="green">◆ MY AGENT</Text><Text color="yellow"> (streaming...)</Text></Box>
            <Box paddingLeft={2}><Text wrap="wrap">{streamingText}</Text></Box>
          </Box>
        )}
        {isLoading && !streamingText && !toolCalls.length && <Box marginY={1} paddingLeft={2}><Text color="cyan">◌ </Text><Text color={THEME.muted}>Thinking...</Text></Box>}
        {error && <Box marginY={1} paddingLeft={2}><Text color="red">✖ Error: {error}</Text></Box>}
      </Box>
      {showModelPicker ? (
        <ModelPicker
          models={models}
          currentModel={currentModel}
          loading={modelLoading}
          onSelect={(id) => { agent.setModel(id); setShowModelPicker(false); }}
          onCancel={() => setShowModelPicker(false)}
        />
      ) : (
        <>
          <StatusBar status={status} />
          <Box marginTop={1}><InputBox value={input} onChange={setInput} onSubmit={send} disabled={isLoading} /></Box>
        </>
      )}
    </Box>
  );
}

render(<App />);
````

---

### Step 9: Create src/headless.ts

Useful for CI/CD pipelines or API integration.

```typescript
import { DEFAULT_MODEL, type ModelInfo } from './agent.js';
import { agent } from './config.js';
import * as readline from 'readline';

let cachedModels: ModelInfo[] | null = null;

async function handleModelCommand(rl: readline.Interface): Promise<void> {
  console.log('\nFetching tool-capable models...');
  if (!cachedModels) {
    cachedModels = await agent.listToolCapableModels();
  }

  console.log(`\n  [0] Default (${DEFAULT_MODEL})`);
  cachedModels.forEach((m, i) => {
    const ctx = m.contextLength
      ? `${(m.contextLength / 1000).toFixed(0)}k ctx`
      : '';
    console.log(`  [${i + 1}] ${m.name} (${m.id})${ctx ? ` — ${ctx}` : ''}`);
  });
  console.log(`\nCurrent: ${agent.model}`);

  return new Promise<void>((resolve) => {
    rl.question('\nSelect model number (or Enter to cancel): ', (answer) => {
      const trimmed = answer.trim();
      if (trimmed === '') {
        console.log('Cancelled.\n');
        resolve();
        return;
      }
      const num = parseInt(trimmed, 10);
      if (isNaN(num) || num < 0 || !cachedModels || num > cachedModels.length) {
        console.log('Invalid selection.\n');
        resolve();
        return;
      }
      if (num === 0) {
        agent.setModel(DEFAULT_MODEL);
        console.log(`Switched to: Default (${DEFAULT_MODEL})\n`);
      } else {
        const selected = cachedModels[num - 1];
        agent.setModel(selected.id);
        console.log(`Switched to: ${selected.name} (${selected.id})\n`);
      }
      resolve();
    });
  });
}

async function main() {
  agent.on('thinking:start', () => console.log('\n🤔 Thinking...'));
  agent.on('tool:call', (name, args) =>
    console.log(`🔧 Using ${name}:`, JSON.stringify(args)),
  );
  agent.on('tool:result', (name) => console.log(`  ✅ Result from ${name}`));
  agent.on('stream:delta', (delta) => process.stdout.write(delta));
  agent.on('stream:end', () => console.log('\n'));

  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });
  console.log(
    // SKILL INSTRUCTION: Replace with the user's agent name
    '🤖 AI Agent (Headless Mode)\nType your message (Ctrl+C to exit):\nCommands: /model\n',
  );

  const prompt = () => {
    rl.question('You: ', async (input) => {
      if (!input.trim()) {
        prompt();
        return;
      }
      if (input.trim() === '/model') {
        try {
          await handleModelCommand(rl);
        } catch (e) {
          console.error(
            'Failed to list models:',
            e instanceof Error ? e.message : String(e),
          );
        }
        prompt();
        return;
      }
      try {
        await agent.send(input);
      } catch (e) {
        console.error('Error:', e instanceof Error ? e.message : String(e));
      }
      prompt();
    });
  };
  prompt();
}

main().catch(console.error);
```

---

## Running the Agent

```bash
npm start              # TUI mode
npm run start:headless # Headless mode
```

### Choosing a Model

By default the agent uses `openrouter/auto` (smart routing). Override via the `MODEL` environment variable:

```bash
MODEL=anthropic/claude-sonnet-4 npm start
```

Or set it in `.env`. Browse available models at [openrouter.ai/models](https://openrouter.ai/models).

Use the `/model` command at runtime to switch between tool-capable models.
