# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

```bash
npm run setup        # Install deps, generate Prisma client, run migrations
```

Add `ANTHROPIC_API_KEY` to `.env` (optional — falls back to mock provider).

## Common Commands

```bash
npm run dev          # Dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest
npm run db:reset     # Force reset SQLite database
```

Run a single test file:
```bash
npx vitest src/lib/__tests__/file-system.test.ts
```

## Architecture

UIGen is a Next.js 15 (App Router) AI-powered React component generator.

### Core Data Flow

1. User sends a message via `ChatInterface`
2. `ChatContext` POSTs to `/api/chat/route.ts`
3. The API deserializes the `VirtualFileSystem` from the client, calls Claude (or mock) with streaming + tool use
4. Tools (`str_replace_editor`, `file_manager`) are executed client-side inside `FileSystemContext`, modifying in-memory files
5. File changes trigger `PreviewFrame` to re-render the iframe
6. On stream completion, messages and file data are saved to SQLite via Prisma

### Key Abstractions

**`VirtualFileSystem`** (`src/lib/file-system.ts`) — all file operations (create, read, update, delete, rename) operate on an in-memory tree. Nothing is written to disk. Supports serialization for persistence and tool call processing.

**`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — React context wrapping `VirtualFileSystem`. Owns tool call processing from AI stream. Triggers preview reloads.

**`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — manages message history, streams responses from `/api/chat`, dispatches tool calls to `FileSystemContext`, and auto-saves project state to the database.

**`PreviewFrame`** (`src/components/preview/PreviewFrame.tsx`) — renders the virtual file system's `/App.jsx` inside an iframe, using Babel standalone (in-browser JSX compilation) via the transform module at `src/lib/transform/`.

**AI Provider** (`src/lib/provider.ts`) — uses Claude Haiku 4.5 via the Vercel AI SDK when `ANTHROPIC_API_KEY` is present; otherwise falls back to `MockLanguageModel` which returns static component examples. The generation system prompt lives in `src/lib/prompts/generation.tsx`.

### Routing

- `/` — checks auth, redirects to latest project or creates one
- `/[projectId]` — main editor page; loads project from DB, renders `MainContent`
- `/api/chat` — streaming POST endpoint for AI interaction

### Auth

JWT sessions via `jose`, stored in a secure httpOnly cookie (`session`). `src/middleware.ts` protects routes. Anonymous users can use the app without signing up; their project has `userId: null`.

### Database

Prisma with SQLite (`prisma/dev.db`). Two models: `User` (email + bcrypt password) and `Project` (stores `messages` and `data` as JSON strings). Path alias `@/*` maps to `src/*`.

### Testing

Vitest with jsdom and React Testing Library. Tests are colocated under `__tests__/` directories next to the code they test.
