# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, and Claude generates them in a virtual file system with real-time preview using client-side JSX transformation.

## Commands

### Development
```bash
npm run dev          # Start Next.js dev server with Turbopack
npm run build        # Build for production
npm start            # Start production server
npm test             # Run Vitest tests
npm run lint         # Run ESLint
```

### Database
```bash
npm run setup        # Install deps + Prisma generate + migrate
npm run db:reset     # Reset database (force)
```

The Prisma client is generated to `src/generated/prisma` (not the default location).

### Running Single Tests
```bash
npm test -- <test-file-pattern>
npm test -- --watch  # Watch mode
```

## Architecture

### Virtual File System
- **Core**: `VirtualFileSystem` class in `src/lib/file-system.ts`
- Files are stored in-memory as a tree structure (`FileNode`)
- Nothing is written to disk - all component code exists only in the VFS
- The VFS is serialized to JSON and stored in the database for authenticated users
- On the frontend, files are transformed from JSX/TSX to JS using Babel (`src/lib/transform/jsx-transformer.ts`)

### AI Component Generation
- **Endpoint**: `src/app/api/chat/route.ts`
- **Model**: Claude via AI SDK (falls back to mock provider if no `ANTHROPIC_API_KEY`)
- **System Prompt**: `src/lib/prompts/generation.tsx` - instructs Claude on React/Tailwind conventions
- **Tools**:
  - `str_replace_editor` (`src/lib/tools/str-replace.ts`): Create/edit files via string replacement
  - `file_manager` (`src/lib/tools/file-manager.ts`): Rename/delete files and folders
- Claude uses multi-step tool calling to create and refine components in the VFS
- The mock provider (when no API key) returns hardcoded component templates based on prompt keywords

### Preview System
- **Component**: `PreviewFrame.tsx`
- Transforms JSX/TSX files to executable JS using Babel (`@babel/standalone`)
- Creates blob URLs for transformed modules
- Uses dynamic imports to load components in an iframe
- CSS imports are detected and handled separately
- Import alias `@/` resolves to the VFS root

### Project Persistence
- Anonymous users: Work tracked in-memory only (`src/lib/anon-work-tracker.ts`)
- Authenticated users: Projects saved to SQLite via Prisma
  - `messages`: Serialized conversation history
  - `data`: Serialized VFS state
- Projects are auto-saved on chat completion (`onFinish` callback)

### Authentication
- JWT-based sessions using `jose` library (`src/lib/auth.ts`)
- Passwords hashed with bcrypt
- Middleware protects API routes (`src/middleware.ts`)
- Users can continue as anonymous or sign up to save work

### Code Organization
```
src/
├── actions/         # Server actions (create/get projects)
├── app/             # Next.js App Router pages and API routes
│   ├── api/chat/    # AI chat endpoint
│   └── [projectId]/ # Dynamic project page
├── components/
│   ├── auth/        # Sign in/up forms
│   ├── chat/        # Chat interface and message rendering
│   ├── editor/      # Monaco code editor and file tree
│   └── preview/     # Live component preview frame
├── lib/
│   ├── contexts/    # React contexts
│   ├── prompts/     # AI system prompts
│   ├── tools/       # AI SDK tools (file operations)
│   └── transform/   # JSX to JS transformation (Babel)
└── hooks/           # React hooks
```

## Key Constraints

1. **All components must export from /App.jsx**: The preview system expects a root `App.jsx` file as the entry point
2. **Use `@/` import alias**: All local imports should use `@/` which resolves to the VFS root (e.g., `import Foo from '@/components/Foo'`)
3. **Tailwind for styling**: Components should use Tailwind classes, not inline styles
4. **No HTML files**: The preview system works with JSX/TSX components only
5. **VFS operates on root `/`**: This is a virtual filesystem, not a real OS filesystem

## Testing

- Framework: Vitest with React Testing Library
- Test files: `**/__tests__/*.test.tsx`
- JSDOM environment configured for React component tests
- Path aliases configured via `vite-tsconfig-paths`
