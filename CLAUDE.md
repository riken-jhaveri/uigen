# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack at http://localhost:3000
npm run dev:daemon   # Start dev server in background, logs to logs.txt
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all Vitest tests
npx vitest run src/path/to/test.ts  # Run a single test file
npm run db:reset     # Reset SQLite database (destructive)
```

> **Do not run `npm audit fix`** — dependencies are pinned to specific compatible versions.

## Environment

Copy `.env.example` to `.env` and set `ANTHROPIC_API_KEY`. Without a valid key (or if the placeholder `your-api-key-here` is left), the app falls back to `MockLanguageModel` in `src/lib/provider.ts`, which returns canned components instead of calling Claude.

The model used is `claude-haiku-4-5` (set in `src/lib/provider.ts`).

## Architecture

### Virtual File System

The core abstraction is `VirtualFileSystem` (`src/lib/file-system.ts`) — an in-memory tree of files and directories. Generated React components live entirely in this VFS; nothing is written to disk. It supports create/read/update/delete/rename and serializes to/from plain JSON for persistence and network transport.

The VFS instance is managed in `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) and shared across the app. `handleToolCall` in that context applies AI tool call results (file creates, string replacements, renames, deletes) to the VFS and triggers a re-render.

### AI Integration

`POST /api/chat` (`src/app/api/chat/route.ts`) is the sole AI endpoint. It:
1. Receives the full message history plus the serialized VFS state (`files`)
2. Prepends the system prompt from `src/lib/prompts/generation.tsx` with Anthropic prompt caching
3. Streams responses via Vercel AI SDK (`streamText`) with up to 40 steps
4. Provides two tools to the model:
   - `str_replace_editor` — view/create/str_replace/insert on VFS files
   - `file_manager` — rename/delete VFS files
5. On finish, saves messages + VFS state to the DB if the user is authenticated

The system prompt requires the AI to always create `/App.jsx` as the entry point and use `@/` import aliases for local files.

### Live Preview

`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders an iframe whose `srcdoc` is rebuilt whenever `refreshTrigger` increments. The pipeline (`src/lib/transform/jsx-transformer.ts`):
1. Transforms each `.jsx`/`.tsx` file in the VFS with Babel standalone
2. Creates blob URLs for each transformed module
3. Builds an ES import map wiring `react`, `react-dom`, `@/` aliases, and all local files to their blob URLs
4. Third-party npm imports are resolved via `https://esm.sh/`
5. CSS files are collected and injected as `<style>` tags
6. The iframe runs the app with an error boundary; syntax errors are displayed inline

### Authentication & Sessions

JWT sessions stored in `auth-token` httpOnly cookies (7-day expiry). Passwords hashed with bcrypt. `src/lib/auth.ts` provides `createSession`/`getSession`/`deleteSession` (server-only) and `verifySession` (for middleware). The middleware at `src/middleware.ts` protects `/api/projects` and `/api/filesystem`.

### Data Persistence

SQLite via Prisma (`prisma/schema.prisma`). Two models:
- `User` — email + bcrypt password
- `Project` — stores `messages` (JSON array) and `data` (serialized VFS JSON), linked to a User

Prisma client output is at `src/generated/prisma`.

Anonymous users' work is tracked in `localStorage` via `src/lib/anon-work-tracker.ts` and offered for save on sign-up/sign-in.

### Routing

- `/` — anonymous entry: shows the editor without a project; authenticated users are immediately redirected to their most recent project (or a newly created one)
- `/[projectId]` — loads a specific project's messages and VFS state from the DB, requires auth

### Contexts

Both contexts must wrap the component tree in order:
1. `FileSystemContext` — owns the VFS instance and exposes file operations + `handleToolCall`
2. `ChatContext` — wraps `useChat` from `@ai-sdk/react`, wires `onToolCall` to `FileSystemContext.handleToolCall`, and serializes the VFS into every chat request body

### Testing

Tests use Vitest + jsdom + React Testing Library. Test files live alongside their subjects in `__tests__/` directories. The vitest config (`vitest.config.mts`) uses `vite-tsconfig-paths` so `@/` path aliases work in tests.
