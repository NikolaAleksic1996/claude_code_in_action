# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (http://localhost:3000)
npm run dev

# Build for production
npm run build

# Run tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Database reset
npm run db:reset
```

## Environment

`.env` requires `ANTHROPIC_API_KEY`. If missing, the app falls back to a `MockLanguageModel` that returns static component examples — useful for development without an API key.

## Architecture

UIGen is a Next.js 15 app where users describe React components in chat and Claude generates them live.

### Request Flow

1. User sends a chat message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. API streams a response from Claude (claude-haiku-4-5) via Vercel AI SDK
3. Claude calls tools (`str_replace_editor`, `file_manager`) to create/edit virtual files
4. Tool calls are relayed back to the client through the stream
5. Client applies file changes via `FileSystemContext` → `VirtualFileSystem`
6. `PreviewFrame` re-renders the iframe by transforming JSX to HTML via `@babel/standalone`

### Virtual File System (`src/lib/file-system.ts`)

An in-memory tree — no files are written to disk. The `VirtualFileSystem` class is serialized to JSON and stored in the `Project.data` column (SQLite). Tools (`src/lib/tools/`) wrap it for Claude to call.

### State Management

Two React contexts manage all runtime state:
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — owns VirtualFileSystem state; exposes create/update/delete operations
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`; handles tool call results and routes file changes to FileSystemContext

### AI Provider (`src/lib/provider.ts`)

Returns an Anthropic model (claude-haiku-4-5-20251001) or `MockLanguageModel` based on whether `ANTHROPIC_API_KEY` is set. The mock is used in tests and local dev without an API key.

### JSX Preview (`src/lib/transform/jsx-transformer.ts`)

Transforms virtual `.tsx`/`.jsx` files into a self-contained HTML document injected into a sandboxed `<iframe>`. Uses `@babel/standalone` at runtime, builds an import map for module resolution.

### Authentication

JWT sessions via `jose` + bcrypt passwords. Sessions stored in HTTP-only cookies (7-day expiry). `src/middleware.ts` enforces auth on protected API routes. Anonymous users can generate components without signing up; their work is tracked in `localStorage`.

### Database

Prisma ORM with SQLite (`prisma/dev.db`). Schema is defined in `prisma/schema.prisma`. Two models: `User` and `Project`. `Project.messages` and `Project.data` are JSON strings (chat history and serialized VirtualFileSystem).

## Key File Locations

| Concern | Path |
|---|---|
| AI chat endpoint | `src/app/api/chat/route.ts` |
| Claude system prompt | `src/lib/prompts/generation.tsx` |
| AI tools (str_replace, file_manager) | `src/lib/tools/` |
| Virtual file system | `src/lib/file-system.ts` |
| JSX → HTML transform | `src/lib/transform/jsx-transformer.ts` |
| AI provider / mock | `src/lib/provider.ts` |
| Auth (JWT) | `src/lib/auth.ts` |
| Server actions | `src/actions/` |
| Main layout with panels | `src/app/main-content.tsx` |

## Code Style

Use comments sparingly. Only comment complex code.

## Testing

Tests use Vitest + jsdom + `@testing-library/react`. Test files live in `__tests__/` directories adjacent to the code they test. The vitest config is at `vitest.config.mts`.
