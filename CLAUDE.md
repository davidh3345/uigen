# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time: install deps + generate Prisma client + run migrations
npm run dev            # Start dev server at http://localhost:3000 (with turbopack)
npm run build          # Production build
npm run lint           # ESLint
npm test               # Run all tests (vitest)
npx vitest run <path>  # Run a single test file
npm run db:reset       # Reset SQLite database (destructive)
```

> All Next.js scripts require `NODE_OPTIONS='--require ./node-compat.cjs'` (already baked into package.json scripts) for bcrypt Node.js compatibility.

## Architecture

### Virtual File System
The core abstraction is `VirtualFileSystem` (`src/lib/file-system.ts`) — an in-memory file system that holds AI-generated React component files. Nothing is written to disk. It serializes to/from a plain JSON object (`Record<string, FileNode>`) for storage in Prisma.

### AI Code Generation Flow
1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The current virtual file system is serialized and sent in the request body
3. `streamText` (Vercel AI SDK) runs with two tools:
   - `str_replace_editor` — create/view/edit files via str-replace or insert
   - `file_manager` — rename or delete files
4. The AI calls these tools to write React components into the VirtualFileSystem
5. On finish, if authenticated with a `projectId`, the messages + file system state are persisted to SQLite via Prisma

### Live Preview
`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders generated code inside an `<iframe srcdoc>`. The pipeline:
1. All VFS files → `createImportMap()` in `src/lib/transform/jsx-transformer.ts`
2. Each JS/JSX/TS/TSX file is compiled in-browser using **Babel standalone**
3. Compiled modules become blob URLs; third-party packages resolve via `https://esm.sh/`
4. An import map + Tailwind CDN script are injected into an HTML shell
5. The entry point is resolved in order: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx`

### State Management
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — wraps the VirtualFileSystem, provides CRUD methods, handles tool call events (bridges AI tool results → VFS mutations), and triggers re-renders via a `refreshTrigger` counter
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — manages chat messages and streaming state

### Auth
JWT sessions stored in httpOnly cookies using `jose`. `src/lib/auth.ts` handles create/get/delete session (server-only). `src/middleware.ts` guards `/api/projects` and `/api/filesystem` routes. Users can use the app anonymously; only authenticated users can persist projects.

### Data Model (Prisma / SQLite)
- `User` — email + hashed password (bcrypt)
- `Project` — `messages` (JSON array) + `data` (JSON VFS snapshot), optionally linked to a user
- Prisma client is generated to `src/generated/prisma/` (non-default output path)

### Mock Provider
When `ANTHROPIC_API_KEY` is absent, `getLanguageModel()` returns `MockLanguageModel` (`src/lib/provider.ts`), which streams static component code to demonstrate the app without API access. Real model: `claude-haiku-4-5`.

## Key Conventions
- Path alias `@/` maps to `src/` (configured in `tsconfig.json` and supported by the VFS import map)
- Tests live alongside source in `__tests__/` subdirectories and use `vitest` + `@testing-library/react`
- Server actions are in `src/actions/` and use `"server-only"` or are Next.js server actions
