# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps + prisma generate + migrate
npm run dev          # Start dev server (Turbopack)
npm run dev:daemon   # Dev server in background, logs to logs.txt
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (all tests)
npm run db:reset     # Reset SQLite database
```

Tests live in `**/__tests__/*.test.ts(x)` and run with jsdom. To run a single test file:
```bash
npx vitest src/components/editor/__tests__/file-tree.test.tsx
```

## Environment Variables

```
ANTHROPIC_API_KEY=""   # Optional – falls back to MockLanguageModel if absent
JWT_SECRET=""          # Required – generate with: openssl rand -base64 32
```

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe a component in chat, Claude generates it via tool calls, and the result renders instantly in an iframe.

### Data Flow

1. User sends a message in `ChatInterface` → posted to `POST /api/chat` along with the serialized virtual file system and `projectId`
2. `getLanguageModel()` (`lib/provider.ts`) returns either the real Claude model (Haiku 4.5) or a `MockLanguageModel` (when no API key)
3. Claude uses two tools to create/modify files:
   - `str_replace_editor` – view, create, replace, or insert content in files
   - `file_manager` – file operations (move, delete, etc.)
4. Tool calls are handled by `handleToolCall()` in `FileSystemContext`, which mutates a `VirtualFileSystem` instance (in-memory; no disk writes)
5. On stream finish, the serialized file system + messages are persisted to SQLite via Prisma (`Project.data`, `Project.messages`)
6. `PreviewFrame` picks up the updated file system, transforms JSX with Babel standalone (`lib/transform/jsx-transformer.ts`), and renders the component inside an iframe with an import map

### Key Modules

| Module | Purpose |
|--------|---------|
| `src/app/api/chat/route.ts` | Streaming AI endpoint; wires tools, system prompt, and model |
| `src/lib/file-system.ts` | `VirtualFileSystem` – in-memory FS with serialize/deserialize |
| `src/lib/contexts/file-system-context.tsx` | React context wrapping VirtualFileSystem; exposes `handleToolCall()` |
| `src/lib/contexts/chat-context.tsx` | Wraps Vercel AI SDK `useChat()`; sends FS state with each message |
| `src/lib/provider.ts` | Selects real vs. mock AI model |
| `src/lib/transform/jsx-transformer.ts` | Babel-based JSX→HTML transformer for iframe preview |
| `src/lib/prompts/generation.tsx` | System prompt sent to Claude |
| `src/lib/auth.ts` | JWT session management (server-only, HttpOnly cookies) |
| `src/actions/` | Next.js server actions for auth and project CRUD |

### UI Layout

Three-panel layout using `react-resizable-panels` (in `src/app/main-content.tsx`):
- **Left panel** (25–50%, default 35%): Chat interface
- **Right panel**: Tabs switching between Preview and Code views
  - **Preview**: `PreviewFrame` iframe
  - **Code**: File tree (30%) + Monaco editor (70%)

### Database

SQLite via Prisma. Schema at `prisma/schema.prisma`; generated client at `src/generated/prisma`.

- **User**: email + bcrypt password
- **Project**: optional `userId` (supports anonymous use), `messages` (JSON), `data` (serialized VirtualFileSystem JSON)

Projects work without authentication; signing in enables persistence across sessions.
