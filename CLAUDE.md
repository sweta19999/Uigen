# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**On Windows, npm scripts that set `NODE_OPTIONS='...'` fail in CMD. Run the dev server directly via bash:**
```bash
NODE_OPTIONS='--require ./node-compat.cjs' npx next dev --turbopack
```

```bash
npm run setup        # First-time: install deps + prisma generate + migrate
npm run dev          # Start dev server (broken on Windows, use bash command above)
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run all tests via vitest
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx  # Single test file
npm run db:reset     # Reset SQLite database (destructive)
```

## Environment Variables

Create a `.env` file:
```
ANTHROPIC_API_KEY=your_key_here   # If empty/missing, falls back to MockLanguageModel
JWT_SECRET=your_secret            # Optional; has a hardcoded dev default
```

## Architecture

### AI Generation Pipeline

User message → `useAIChat` hook → `POST /api/chat` → `streamText()` (Claude Haiku via `@ai-sdk/anthropic`) → tool calls streamed back via SSE → `ChatContext` applies tools to `VirtualFileSystem` → preview re-renders.

The model has two tools:
- **`str_replace_editor`** (`src/lib/tools/str-replace.ts`): create/view/edit files (`create`, `str_replace`, `insert`, `view` commands)
- **`file_manager`** (`src/lib/tools/file-manager.ts`): rename/delete files and directories

System prompt lives in `src/lib/prompts/generation.tsx`. Max 40 tool steps (4 for mock), 10k output tokens.

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory `Map<path, FileNode>`. Nothing is written to disk during generation. It serializes to/from JSON for persistence. The AI tools operate on this object; the Monaco editor also writes to it directly.

### Live Preview

`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) finds the entry point (`/App.jsx`, `/App.tsx`, `/index.jsx`, `/src/App.jsx`), passes all files through the JSX transformer, and sets `iframe.srcdoc`.

`src/lib/transform/jsx-transformer.ts` uses `@babel/standalone` to transpile JSX/TS to JS in the browser, builds an import map (local files → blob URLs, unknown packages → `esm.sh`), injects Tailwind CDN, and wraps everything in an error boundary.

### Authentication

JWT-based via `jose`. Sessions stored in an httpOnly `auth-token` cookie (7-day expiry). `src/lib/auth.ts` handles create/verify/delete. Passwords bcrypt-hashed (10 rounds). `middleware.ts` protects `/api/projects` and `/api/filesystem` routes.

Server actions in `src/actions/`: `signUp`, `signIn`, `signOut`, `getUser`, `createProject`, `getProject`, `getProjects`.

### Project Persistence

Projects saved to SQLite via Prisma. `messages` and `data` (file system) are JSON strings. Saving happens in the `onFinish` callback of `streamText` in `POST /api/chat`. Anonymous users get sessionStorage tracking (`src/lib/anon-work-tracker.ts`) but no DB persistence.

**Prisma schema** (`prisma/schema.prisma`): `User` (id, email, password) → `Project` (id, name, userId?, messages, data). Client generated to `src/generated/prisma`.

### State Management

Two React contexts:
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): wraps `useChat` from `ai/react`, handles incoming tool calls, drives file system updates
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): exposes `VirtualFileSystem` instance and CRUD operations to all components

### `node-compat.cjs`

Patches Node 25+ which exposes non-functional `localStorage`/`sessionStorage` globals, causing SSR crashes. Required via `NODE_OPTIONS` before starting Next.js.

## Key File Locations

| Concern | Path |
|---|---|
| Chat API route | `src/app/api/chat/route.ts` |
| AI tools | `src/lib/tools/` |
| System prompt | `src/lib/prompts/generation.tsx` |
| JSX→HTML transform | `src/lib/transform/jsx-transformer.ts` |
| Auth utilities | `src/lib/auth.ts` |
| Virtual file system | `src/lib/file-system.ts` |
| AI model factory | `src/lib/provider.ts` |
| Server actions | `src/actions/` |
| DB schema | `prisma/schema.prisma` |
| Test config | `vitest.config.mts` (jsdom environment, `@/*` path alias) |
| Main UI layout | `src/app/main-content.tsx` |
