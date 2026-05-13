# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

AInforma — React Native + Expo mobile app aggregating AI news, tools, papers, tutorials, and media. Two target audiences: curious/enthusiasts (beginner) and technical developers. Backend: Supabase (PostgreSQL + Edge Functions + Auth).

## Commands

```bash
# Dev
npx expo start
npx expo start --ios
npx expo start --android

# Tests
npm test
npm test -- --testPathPattern=<file>
npm test -- --coverage

# Supabase (local)
npx supabase start
npx supabase db push          # apply migrations
npx supabase functions deploy fetch-articles
```

## Architecture

**Navigation**: Expo Router v6 (file-based). All routes live in `app/`. Tab screens in `app/(tabs)/`. Dynamic routes follow `app/article/[id].tsx` pattern. No React Navigation manually — Expo Router only.

**Data fetching**: TanStack Query via custom hooks in `hooks/`. Never fetch directly in components. One hook per Supabase entity (`useArticles`, `useTools`, `usePapers`, `useBookmarks`, `useProfile`, `useSearch`).

**State**: Zustand in `stores/` — only for UI state (active feed filters, authenticated user). Server/remote data always stays in TanStack Query cache.

**Styles**: NativeWind (Tailwind classes on RN components). Avoid `StyleSheet` where possible.

**Backend**: Supabase Edge Functions (Deno/TypeScript) in `supabase/functions/`. Four scheduled functions: `fetch-articles` (daily RSS), `fetch-papers` (weekly arXiv), `fetch-tools` (weekly), `generate-digest` (weekly/monthly). Always verify the corresponding SQL migration exists before writing an Edge Function.

**AI calls**: Claude API (claude-haiku) invoked only from Edge Functions — for `simplified_summary` on papers and "Spiega in semplice" on articles. Results cached in Supabase to avoid repeat calls.

**Supabase client**: `lib/supabase.ts`. TanStack Query config: `lib/queryClient.ts`.

## Key conventions

- TypeScript strict mode (already set in `tsconfig.json`)
- Named exports for every component, one component per file
- Path alias `@/*` maps to repo root
- `EXPO_PUBLIC_` prefix required for env vars exposed to client
- `SUPABASE_SERVICE_ROLE_KEY` and `ANTHROPIC_API_KEY` — Edge Functions only, never in client bundle
- Test hooks with React Native Testing Library before wiring into components

## Env vars

| Variable | Where used |
|---|---|
| `EXPO_PUBLIC_SUPABASE_URL` | Client |
| `EXPO_PUBLIC_SUPABASE_ANON_KEY` | Client |
| `SUPABASE_SERVICE_ROLE_KEY` | Edge Functions only |
| `ANTHROPIC_API_KEY` | Edge Functions only |
| `YOUTUBE_API_KEY` | Edge Functions only |

## Database schema summary

Core tables: `articles`, `tools`, `papers`, `tutorials`, `media_items`, `profiles`, `bookmarks`, `reading_history`. RLS enabled: public content is readable by all; `bookmarks` and `reading_history` are user-scoped (`auth.uid() = user_id`). Full-text search via `to_tsvector`/`plainto_tsquery`. SQL migrations live in `supabase/migrations/`.
