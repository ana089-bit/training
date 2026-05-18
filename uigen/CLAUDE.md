# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies, generate Prisma client, and run migrations
npm run setup

# Start dev server (uses Turbopack + node-compat shim)
npm run dev

# Build for production
npm run build

# Run all tests
npm test

# Run a single test file
npx vitest src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Reset database (destructive)
npm run db:reset
```

**Environment**: Copy `.env.example` or set `ANTHROPIC_API_KEY` in `.env`. If absent, the app falls back to a `MockLanguageModel` that returns a hardcoded component — useful for development without an API key.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe UI components in a chat interface; Claude generates them in real-time and they render instantly in a sandboxed iframe.

### Core Data Flow

1. User message → `ChatInterface` → `useChat` (Vercel AI SDK) → `POST /api/chat`
2. `/api/chat` calls `streamText` with up to 40 agentic steps, using two tools:
   - `str_replace_editor` — view, create, and edit files via string replacement
   - `file_manager` — rename and delete files
3. Tool calls arrive on the client via streaming; `handleToolCall` in `ChatContext` updates the `VirtualFileSystem`
4. `PreviewFrame` detects `refreshTrigger` changes, runs `jsx-transformer` (Babel standalone) to compile JSX → JS with an import map, and injects the result into a sandboxed `<iframe>`

### Virtual File System

`src/lib/file-system.ts` — `VirtualFileSystem` class holds all project files **in memory only** (no disk I/O). It serializes to/from JSON for persistence in the `Project.data` column in SQLite. The AI tools operate entirely on this in-memory FS.

### AI & Prompting

- `src/lib/provider.ts` — `getLanguageModel()` returns either a real Anthropic Claude model or `MockLanguageModel`. Uses `claude-haiku-4-5-20251001` by default.
- `src/lib/prompts/generation.tsx` — System prompt for component generation, uses Anthropic cache control for ephemeral caching.
- `src/lib/tools/str-replace.ts` — Tool definition and logic for the text editor tool.
- `src/lib/tools/file-manager.ts` — Tool definition for file rename/delete.

### State Management

Two React contexts wrap the main UI:

- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — owns the `VirtualFileSystem` instance, selected file, and exposes mutation methods.
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`, intercepts tool calls to bridge AI output → file system mutations, and manages the refresh trigger for the preview.

### Authentication & Persistence

- JWT sessions via `jose` (`src/lib/auth.ts`); sessions stored in an HTTP-only cookie.
- `bcrypt` for password hashing.
- `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes.
- Server actions in `src/actions/` handle sign-up, sign-in, sign-out, and project CRUD.
- Anonymous users are tracked via `src/lib/anon-work-tracker.ts` (localStorage); their work merges into a real account on sign-up.

### Database

Prisma + SQLite. Always read `prisma/schema.prisma` before writing any DB-related code — it is the authoritative source for model fields, types, relations, and defaults. Schema has two models: `User` and `Project`. `Project.messages` and `Project.data` are stored as JSON strings.

```bash
npx prisma studio        # Browse database
npx prisma migrate dev   # Apply schema changes
```

### Preview Sandbox

`src/components/preview/PreviewFrame.tsx` renders an `<iframe sandbox>`. `src/lib/transform/jsx-transformer.ts` uses Babel standalone to transform JSX and builds an import map so React imports resolve correctly inside the sandbox.

### Path Alias

`@/*` maps to `./src/*` (configured in `tsconfig.json`).
