# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack at localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run all tests with Vitest
npx vitest run src/path/to/file.test.ts  # Run a single test file
npm run db:reset     # Reset SQLite database (destructive)
```

`ANTHROPIC_API_KEY` in `.env` is optional — without it, a mock provider returns static code instead of calling Claude.

## Architecture

### AI Component Generation Pipeline

1. **Chat UI** (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK `useChat`, serializing the current VFS state and `projectId` in every request body.
2. **API route** (`src/app/api/chat/route.ts`) — reconstructs the VFS from the request, calls `streamText` with two tools, and on finish persists messages + VFS to Prisma if the user is authenticated.
3. **AI tools** exposed to Claude:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — `view`, `create`, `str_replace`, `insert` on the VFS
   - `file_manager` (`src/lib/tools/file-manager.ts`) — `rename`, `delete` on the VFS
4. **System prompt** (`src/lib/prompts/generation.tsx`) — instructs the model: always create `/App.jsx` as entry, use Tailwind CSS, use `@/` alias for local imports, no HTML files.

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory tree of `FileNode` objects. It never writes to disk. It serializes to/from plain `Record<string, FileNode>` for API transport and database storage.

`FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) wraps an instance and dispatches `handleToolCall` when the AI uses a tool, keeping the React state in sync via a `refreshTrigger` counter.

### Live Preview

`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) re-renders whenever `refreshTrigger` changes. It calls `createImportMap` which:
- Transforms JSX/TSX files with Babel standalone into blob URLs
- Resolves third-party imports to `https://esm.sh/<package>`
- Creates placeholder modules for missing local imports
- Returns an HTML document injected into an `<iframe srcdoc>` with `allow-scripts allow-same-origin`

The preview always looks for `/App.jsx` (or `/App.tsx`, `/index.jsx`, etc.) as the entry point.

### Authentication & Persistence

- JWT sessions stored in httpOnly cookies (`src/lib/auth.ts`), 7-day expiry
- Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem`
- Database schema: `prisma/schema.prisma` — the canonical reference for all models and relations
- Prisma/SQLite: `User` → `Project[]`. Projects store `messages` (JSON array) and `data` (serialized VFS JSON) as text columns.
- Prisma client is generated to `src/generated/prisma/` (non-standard output path set in `prisma/schema.prisma`)
- Anonymous users can work locally; their work is tracked in `src/lib/anon-work-tracker.ts` for migration on sign-up.

### Key Conventions

- `@/` path alias maps to `src/` (configured in `tsconfig.json`)
- UI primitives live in `src/components/ui/` (shadcn/ui components)
- Server-only modules use the `server-only` package (e.g., `src/lib/auth.ts`, `src/lib/prisma.ts`)
- Tests use Vitest + jsdom + React Testing Library; test files sit alongside source in `__tests__/` subdirectories
