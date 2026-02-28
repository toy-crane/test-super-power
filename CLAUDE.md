# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

```bash
bun dev          # Start dev server at http://localhost:3000
bun run build    # Production build
bun run lint     # Run ESLint
bun start        # Start production server
```

Package manager is **Bun** (use `bun add` for dependencies).

## Architecture

Next.js 16 app using the **App Router** with React 19 and TypeScript (strict mode).

### Key Structure

- `app/` — Next.js App Router pages and layouts
- `components/ui/` — shadcn/ui component library (radix-nova style, managed via `shadcn` CLI)
- `components/` — Application-level components built on top of ui primitives
- `lib/utils.ts` — `cn()` utility combining clsx + tailwind-merge for class merging

### Path Alias

`@/*` maps to the project root (e.g., `@/components/ui/button`).

## Styling

- **Tailwind CSS 4** via PostCSS plugin (`@tailwindcss/postcss`) — no `tailwind.config` file
- Theme defined as CSS custom properties in `app/globals.css` using `@theme inline` block
- Colors use **oklch()** format with light/dark mode support (`.dark` class)
- Component variants managed with **class-variance-authority (CVA)**
- Always use `cn()` from `@/lib/utils` when combining Tailwind classes

## shadcn/ui

Configuration in `components.json`:
- Style: `radix-nova`
- Built on **Radix UI** primitives via `radix-ui` package
- Icons: **Lucide React**
- RSC-compatible (`"rsc": true`)
- Add components: `bunx shadcn@latest add <component-name>`

## Conventions

- Interactive components require `"use client"` directive
- Components use `data-slot` attributes for styling hooks
- Fonts: Inter (primary sans), Geist Mono (monospace) — configured in `app/layout.tsx`
