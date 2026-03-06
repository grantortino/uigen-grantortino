# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (uses Turbopack + node-compat shim)
npm run dev

# Build for production
npm run build

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Lint
npm run lint

# Reset database (destructive)
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Run database migrations
npx prisma migrate dev
```

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface, Claude generates them using tool calls, and the result renders live in a sandboxed iframe.

### Data flow

1. **Chat** (`src/app/api/chat/route.ts`) — POST handler receives `messages`, `files` (serialized VFS), and optional `projectId`. It reconstructs a `VirtualFileSystem`, calls `streamText` with two tools, and on finish saves to DB if authenticated.

2. **AI Tools** — Two tools exposed to the model:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — view/create/str_replace/insert file operations on the VFS
   - `file_manager` (`src/lib/tools/file-manager.ts`) — rename/delete operations on the VFS

3. **Virtual File System** (`src/lib/file-system.ts`) — In-memory tree of `FileNode` objects. Never writes to disk. Serializes to/from plain objects for DB persistence and API transport.

4. **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`) — React context wrapping the VFS. `handleToolCall` processes incoming AI tool calls and updates the VFS + triggers re-render via `refreshTrigger`.

5. **Preview** (`src/components/preview/PreviewFrame.tsx`) — Watches `refreshTrigger`. Calls `createImportMap` to Babel-transform all VFS files into blob URLs, builds an HTML document with an import map, and sets it as `srcdoc` of a sandboxed iframe. Third-party npm packages are resolved via `https://esm.sh/`.

6. **Provider** (`src/lib/provider.ts`) — Returns a real Anthropic model (`claude-haiku-4-5`) if `ANTHROPIC_API_KEY` is set, otherwise falls back to `MockLanguageModel` which returns static hardcoded components.

### Auth

Custom JWT auth (no NextAuth). `src/lib/auth.ts` issues httpOnly cookies via `jose`. `src/middleware.ts` protects routes. Anonymous users can use the app; projects are only persisted for authenticated users.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models: `User` and `Project`. Project stores `messages` and `data` (serialized VFS) as JSON strings.

### Key paths

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | AI streaming endpoint |
| `src/lib/file-system.ts` | VirtualFileSystem class |
| `src/lib/transform/jsx-transformer.ts` | Babel transform + import map + preview HTML generation |
| `src/lib/provider.ts` | Model selection (real vs mock) |
| `src/lib/prompts/generation.tsx` | System prompt for component generation |
| `src/lib/contexts/file-system-context.tsx` | VFS React context + tool call handler |
| `src/components/preview/PreviewFrame.tsx` | Live preview iframe |
| `src/components/chat/` | Chat UI components |
| `src/components/editor/` | Monaco code editor + file tree |
| `prisma/schema.prisma` | DB schema |

### Notes

- `node-compat.cjs` is required at startup via `NODE_OPTIONS` to polyfill Node.js APIs used by dependencies.
- The preview iframe uses `allow-scripts allow-same-origin` sandbox to support blob URL import maps.
- The `@/` import alias in generated user code is resolved at preview time by `createImportMap` (maps `@/foo` → `/foo` blob URL); it is not a Next.js path alias inside the preview iframe.
- Tests use Vitest + jsdom + React Testing Library.
