# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Start dev server with Turbopack at http://localhost:3000
npm run build       # Production build
npm run lint        # ESLint
npm test            # Vitest (run all tests)
npx vitest run src/path/to/__tests__/file.test.tsx  # Run a single test file
npm run db:reset    # Reset SQLite database
```

Set `ANTHROPIC_API_KEY` in `.env` for real AI responses; omit it to use the mock provider.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface; Claude generates files in a virtual (in-memory) file system; the result is compiled with Babel and rendered inside a sandboxed iframe using esm.sh import maps.

### Key data flow

1. User sends message → `ChatContext` posts to `/api/chat`
2. `/api/chat` calls `streamText()` (Vercel AI SDK) with Claude + two tools
3. Claude invokes `str_replace_editor` / `file_manager` tool calls to write files
4. `FileSystemContext` applies those tool calls to the `VirtualFileSystem` instance
5. `PreviewFrame` picks up the updated VFS, transpiles files via `jsx-transformer.ts` (Babel in-browser), and injects them into the iframe via blob URLs + an import map
6. On stream completion, messages + VFS snapshot are saved to the project in SQLite (Prisma)

### Important modules

| Path | Purpose |
|------|---------|
| `src/lib/file-system.ts` | In-memory tree-based VFS — never writes to disk |
| `src/lib/provider.ts` | Dual-mode LLM: real Anthropic (`claude-haiku-4-5`) or mock streaming provider |
| `src/lib/transform/jsx-transformer.ts` | Babel JSX→ES5 transpiler + import map + blob URL generation for the iframe |
| `src/lib/tools/` | `str_replace_editor` and `file_manager` tool definitions given to Claude |
| `src/lib/prompts/generation.tsx` | System prompt: Claude must create `/App.jsx` as entry point, use Tailwind, use `@/` for custom imports |
| `src/lib/contexts/chat-context.tsx` | Wraps `useChat` (AI SDK); owns messages + streaming state |
| `src/lib/contexts/file-system-context.tsx` | Owns the VFS instance; dispatches AI tool calls to it |
| `src/app/api/chat/route.ts` | Streaming chat endpoint; saves to DB on finish |
| `src/lib/auth.ts` | JWT sessions via `jose` (7-day, HttpOnly cookies) |
| `src/actions/` | Server actions for auth and project CRUD |

### UI layout

Three-panel resizable layout in `src/app/main-content.tsx`:
- **Left (35%)**: Chat (`src/components/chat/`)
- **Right (65%)**: Preview iframe tab / Code tab (file tree + Monaco editor)

### Database

Schema is defined in `prisma/schema.prisma` — always reference it when working with DB models or queries. Prisma client is generated to `src/generated/prisma`. SQLite database at `prisma/dev.db`.

Two models:
- **User**: `id` (cuid), `email` (unique), `password` (bcrypt), timestamps, one-to-many projects
- **Project**: `id` (cuid), `name`, `userId` (nullable — anonymous projects allowed), `messages` (JSON string), `data` (JSON string — serialized VFS), timestamps

### Authentication

JWT in HttpOnly cookies. Anonymous users can generate components but cannot save projects. `src/middleware.ts` protects API routes; `src/actions/` handles sign-up/sign-in with bcrypt.

### Testing

Tests live in `__tests__/` subdirectories next to source. Vitest + jsdom + Testing Library. The mock provider in `src/lib/provider.ts` is what tests use to avoid real API calls.

### Path alias

`@/*` resolves to `src/*` — used throughout the codebase and required in generated component imports.
