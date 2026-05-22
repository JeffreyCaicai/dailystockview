# Daily Stock View MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the first online, mobile-first A-share morning brief system with saved briefs, theme files, watchlist items, reviews, and AI-assisted brief generation.

**Architecture:** Use a Next.js App Router application deployed on Vercel, with Supabase for authentication and Postgres persistence. Keep the MVP server-rendered where possible, use small client components only for forms and mobile interactions, and route all AI generation through a server-side API endpoint with structured JSON validation before saving.

**Tech Stack:** Next.js 15, React 19, TypeScript, Tailwind CSS, shadcn-style components, lucide-react, Supabase Auth/Postgres, OpenAI Responses API, Zod, Vitest, Playwright, Vercel.

---

## 1. Scope And Technical Decisions

This plan implements the product specification in `docs/superpowers/specs/2026-05-22-a-share-morning-brief-product-spec.md`.

### MVP Decisions

- Build as a web app, optimized first for mobile browser.
- Use Supabase Auth even if there is only one user at launch, so data ownership is clean from day one.
- Store generated briefs as structured JSON plus editable text fields.
- Store themes, symbols, reviews, and change logs as first-class tables.
- Keep data input manual in v0.1: the user pastes market data, news, screenshots converted to text, and notes.
- Use OpenAI only to draft structured content; the user must confirm before saving.
- Avoid direct trading language in system prompts, UI labels, and saved statuses.

### Non-Goals For This Plan

- No real-time quotes.
- No brokerage connection.
- No automatic trading.
- No payment or subscription.
- No multi-tenant admin console.
- No automatic web news crawler.

## 2. File Structure

Create this structure:

```text
.
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── globals.css
│   ├── (auth)/
│   │   └── login/
│   │       └── page.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx
│   │   ├── briefs/
│   │   │   ├── today/
│   │   │   │   └── page.tsx
│   │   │   └── [id]/
│   │   │       └── page.tsx
│   │   ├── history/
│   │   │   └── page.tsx
│   │   ├── themes/
│   │   │   └── page.tsx
│   │   ├── watchlist/
│   │   │   └── page.tsx
│   │   └── reviews/
│   │       └── page.tsx
│   └── api/
│       └── ai/
│           └── generate-brief/
│               └── route.ts
├── components/
│   ├── app-shell.tsx
│   ├── bottom-nav.tsx
│   ├── status-pill.tsx
│   ├── section-card.tsx
│   └── empty-state.tsx
├── features/
│   ├── briefs/
│   │   ├── actions.ts
│   │   ├── brief-editor.tsx
│   │   ├── brief-form.tsx
│   │   ├── brief-list.tsx
│   │   ├── brief-view.tsx
│   │   ├── schema.ts
│   │   └── types.ts
│   ├── themes/
│   │   ├── actions.ts
│   │   ├── schema.ts
│   │   ├── theme-card.tsx
│   │   └── theme-list.tsx
│   ├── watchlist/
│   │   ├── actions.ts
│   │   ├── schema.ts
│   │   ├── symbol-card.tsx
│   │   └── symbol-list.tsx
│   └── reviews/
│       ├── actions.ts
│       ├── schema.ts
│       ├── review-form.tsx
│       └── review-list.tsx
├── lib/
│   ├── ai/
│   │   ├── brief-generation.ts
│   │   ├── brief-generation.test.ts
│   │   └── prompt.ts
│   ├── db/
│   │   ├── queries.ts
│   │   └── types.ts
│   ├── supabase/
│   │   ├── client.ts
│   │   ├── middleware.ts
│   │   └── server.ts
│   └── utils.ts
├── supabase/
│   └── migrations/
│       └── 0001_initial_schema.sql
├── tests/
│   └── e2e/
│       └── smoke.spec.ts
├── middleware.ts
├── package.json
├── tsconfig.json
├── next.config.ts
├── vitest.config.ts
├── playwright.config.ts
├── tailwind.config.ts
├── postcss.config.mjs
├── .env.example
└── README.md
```

Responsibility boundaries:

- `app/`: routes and route-level composition only.
- `components/`: reusable presentational components with no domain data access.
- `features/*`: domain UI, forms, Zod validation, and server actions for one product area.
- `lib/ai`: AI prompt, structured schema validation, and model adapter.
- `lib/supabase`: Supabase browser/server clients and session middleware.
- `lib/db`: shared database types and read helpers.
- `supabase/migrations`: schema source of truth.
- `tests/e2e`: browser-level smoke tests for core flows.

## 3. Data Model

Create the following Supabase Postgres tables.

### Table: briefs

Stores one saved morning brief per user per trade date.

```sql
create table public.briefs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  trade_date date not null,
  market_tone text not null check (market_tone in ('进攻', '轮动', '防守', '高波动')),
  style_bias text not null check (style_bias in ('成长', '价值', '红利', '周期', '混合')),
  position_bias text not null check (position_bias in ('积极', '中性', '谨慎')),
  headline_view text not null,
  summary_points jsonb not null default '[]'::jsonb,
  market_review text not null default '',
  strategy_view text not null default '',
  industry_notes jsonb not null default '[]'::jsonb,
  watchlist_updates jsonb not null default '[]'::jsonb,
  risk_notes jsonb not null default '[]'::jsonb,
  verification_points jsonb not null default '[]'::jsonb,
  raw_input text not null default '',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (user_id, trade_date)
);
```

### Table: themes

Stores medium-term theme files.

```sql
create table public.themes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  name text not null,
  stage text not null check (stage in ('启动', '加速', '分歧', '兑现', '退潮')),
  priority text not null check (priority in ('高', '中', '低')),
  core_logic text not null default '',
  policy_logic text not null default '',
  industry_logic text not null default '',
  earnings_checks jsonb not null default '[]'::jsonb,
  capital_checks jsonb not null default '[]'::jsonb,
  representative_symbols jsonb not null default '[]'::jsonb,
  etf_tools jsonb not null default '[]'::jsonb,
  invalidation_conditions jsonb not null default '[]'::jsonb,
  latest_update text not null default '',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (user_id, name)
);
```

### Table: symbols

Stores stocks and ETFs in the observation pool.

```sql
create table public.symbols (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  ticker text not null,
  name text not null,
  market text not null check (market in ('A股', '港股', '美股', 'ETF')),
  theme_id uuid references public.themes(id) on delete set null,
  category text not null check (category in ('核心龙头', '景气验证', '低位修复', 'ETF工具', '暂停观察')),
  status text not null check (status in ('关注', '等待确认', '降低关注', '暂停观察', '逻辑证伪')),
  watch_reason text not null default '',
  company_logic text not null default '',
  verification_points jsonb not null default '[]'::jsonb,
  valuation_risks jsonb not null default '[]'::jsonb,
  technical_summary text not null default '',
  invalidation_conditions jsonb not null default '[]'::jsonb,
  latest_view_change text not null default '',
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  unique (user_id, ticker, market)
);
```

### Table: reviews

Stores daily and weekly review notes.

```sql
create table public.reviews (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  review_date date not null,
  review_type text not null check (review_type in ('日复盘', '周复盘')),
  previous_view_result text not null default '',
  validated_themes jsonb not null default '[]'::jsonb,
  invalidated_themes jsonb not null default '[]'::jsonb,
  new_risks jsonb not null default '[]'::jsonb,
  watchlist_changes jsonb not null default '[]'::jsonb,
  discipline_notes jsonb not null default '[]'::jsonb,
  next_verification_points jsonb not null default '[]'::jsonb,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);
```

### Table: change_logs

Stores state changes for themes and symbols.

```sql
create table public.change_logs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  entity_type text not null check (entity_type in ('brief', 'theme', 'symbol', 'review')),
  entity_id uuid not null,
  change_type text not null,
  before_value jsonb,
  after_value jsonb,
  reason text not null default '',
  created_at timestamptz not null default now()
);
```

### Row Level Security

Enable RLS on all domain tables and allow users to access only their own rows.

```sql
alter table public.briefs enable row level security;
alter table public.themes enable row level security;
alter table public.symbols enable row level security;
alter table public.reviews enable row level security;
alter table public.change_logs enable row level security;

create policy "Users can read own briefs" on public.briefs
  for select using (auth.uid() = user_id);
create policy "Users can insert own briefs" on public.briefs
  for insert with check (auth.uid() = user_id);
create policy "Users can update own briefs" on public.briefs
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "Users can delete own briefs" on public.briefs
  for delete using (auth.uid() = user_id);

create policy "Users can read own themes" on public.themes
  for select using (auth.uid() = user_id);
create policy "Users can insert own themes" on public.themes
  for insert with check (auth.uid() = user_id);
create policy "Users can update own themes" on public.themes
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "Users can delete own themes" on public.themes
  for delete using (auth.uid() = user_id);

create policy "Users can read own symbols" on public.symbols
  for select using (auth.uid() = user_id);
create policy "Users can insert own symbols" on public.symbols
  for insert with check (auth.uid() = user_id);
create policy "Users can update own symbols" on public.symbols
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "Users can delete own symbols" on public.symbols
  for delete using (auth.uid() = user_id);

create policy "Users can read own reviews" on public.reviews
  for select using (auth.uid() = user_id);
create policy "Users can insert own reviews" on public.reviews
  for insert with check (auth.uid() = user_id);
create policy "Users can update own reviews" on public.reviews
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "Users can delete own reviews" on public.reviews
  for delete using (auth.uid() = user_id);

create policy "Users can read own change logs" on public.change_logs
  for select using (auth.uid() = user_id);
create policy "Users can insert own change logs" on public.change_logs
  for insert with check (auth.uid() = user_id);
```

## 4. Environment Variables

Create `.env.example`:

```bash
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
OPENAI_API_KEY=
OPENAI_MODEL=gpt-4.1-mini
APP_URL=http://localhost:3000
```

Rules:

- `SUPABASE_SERVICE_ROLE_KEY` must never be exposed to browser code.
- AI calls must run only in server routes.
- The app should work without `OPENAI_API_KEY` for browsing saved data; only generation should fail with a clear message.

## 5. Implementation Tasks

### Task 1: Scaffold Next.js App

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `next.config.ts`
- Create: `postcss.config.mjs`
- Create: `tailwind.config.ts`
- Create: `app/layout.tsx`
- Create: `app/globals.css`
- Create: `app/page.tsx`
- Modify: `README.md`

- [ ] **Step 1: Add project dependencies**

Create `package.json`:

```json
{
  "name": "dailystockview",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "e2e": "playwright test"
  },
  "dependencies": {
    "@supabase/ssr": "^0.6.1",
    "@supabase/supabase-js": "^2.49.1",
    "clsx": "^2.1.1",
    "lucide-react": "^0.468.0",
    "next": "^15.1.0",
    "openai": "^4.77.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "tailwind-merge": "^2.5.5",
    "zod": "^3.24.1"
  },
  "devDependencies": {
    "@playwright/test": "^1.49.1",
    "@testing-library/react": "^16.1.0",
    "@types/node": "^22.10.2",
    "@types/react": "^19.0.2",
    "@types/react-dom": "^19.0.2",
    "autoprefixer": "^10.4.20",
    "eslint": "^9.17.0",
    "eslint-config-next": "^15.1.0",
    "jsdom": "^25.0.1",
    "postcss": "^8.4.49",
    "tailwindcss": "^3.4.17",
    "typescript": "^5.7.2",
    "vitest": "^2.1.8"
  }
}
```

- [ ] **Step 2: Add TypeScript config**

Create `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "es2022"],
    "allowJs": false,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "paths": {
      "@/*": ["./*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

- [ ] **Step 3: Add Next and Tailwind config**

Create `next.config.ts`:

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  experimental: {
    typedRoutes: true
  }
};

export default nextConfig;
```

Create `postcss.config.mjs`:

```js
const config = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {}
  }
};

export default config;
```

Create `tailwind.config.ts`:

```ts
import type { Config } from "tailwindcss";

const config: Config = {
  content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}", "./features/**/*.{ts,tsx}", "./lib/**/*.{ts,tsx}"],
  theme: {
    extend: {
      colors: {
        ink: "#17202A",
        paper: "#F7F8FA",
        line: "#E3E7ED",
        accent: "#1E5EFF",
        risk: "#B42318",
        good: "#067647"
      }
    }
  },
  plugins: []
};

export default config;
```

- [ ] **Step 4: Add root layout and styles**

Create `app/globals.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

:root {
  color-scheme: light;
}

html {
  background: #f7f8fa;
}

body {
  min-height: 100vh;
  background: #f7f8fa;
  color: #17202a;
}

button,
input,
textarea,
select {
  font: inherit;
}
```

Create `app/layout.tsx`:

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "Daily Stock View",
  description: "A personal A-share morning brief and research archive."
};

export default function RootLayout({ children }: Readonly<{ children: React.ReactNode }>) {
  return (
    <html lang="zh-CN">
      <body>{children}</body>
    </html>
  );
}
```

Create `app/page.tsx`:

```tsx
import { redirect } from "next/navigation";

export default function HomePage() {
  redirect("/briefs/today");
}
```

- [ ] **Step 5: Install dependencies**

Run:

```bash
npm install
```

Expected: `package-lock.json` is created and dependencies install successfully.

- [ ] **Step 6: Verify scaffold**

Run:

```bash
npm run typecheck
npm run build
```

Expected: both commands pass.

- [ ] **Step 7: Commit**

```bash
git add package.json package-lock.json tsconfig.json next.config.ts postcss.config.mjs tailwind.config.ts app README.md
git commit -m "feat: scaffold next app"
```

### Task 2: Add Supabase Schema And Clients

**Files:**
- Create: `.env.example`
- Create: `supabase/migrations/0001_initial_schema.sql`
- Create: `lib/supabase/client.ts`
- Create: `lib/supabase/server.ts`
- Create: `lib/supabase/middleware.ts`
- Create: `middleware.ts`
- Create: `lib/db/types.ts`

- [ ] **Step 1: Add env example**

Create `.env.example` with the values from section 4.

- [ ] **Step 2: Add migration**

Create `supabase/migrations/0001_initial_schema.sql` and include all SQL from section 3.

At the end of the migration add updated timestamp triggers:

```sql
create or replace function public.set_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger set_briefs_updated_at
  before update on public.briefs
  for each row execute function public.set_updated_at();

create trigger set_themes_updated_at
  before update on public.themes
  for each row execute function public.set_updated_at();

create trigger set_symbols_updated_at
  before update on public.symbols
  for each row execute function public.set_updated_at();

create trigger set_reviews_updated_at
  before update on public.reviews
  for each row execute function public.set_updated_at();
```

- [ ] **Step 3: Add Supabase browser client**

Create `lib/supabase/client.ts`:

```ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const anonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY;

  if (!url || !anonKey) {
    throw new Error("Missing Supabase browser environment variables.");
  }

  return createBrowserClient(url, anonKey);
}
```

- [ ] **Step 4: Add Supabase server client**

Create `lib/supabase/server.ts`:

```ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();
  const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const anonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY;

  if (!url || !anonKey) {
    throw new Error("Missing Supabase server environment variables.");
  }

  return createServerClient(url, anonKey, {
    cookies: {
      getAll() {
        return cookieStore.getAll();
      },
      setAll(cookiesToSet) {
        cookiesToSet.forEach(({ name, value, options }) => {
          cookieStore.set(name, value, options);
        });
      }
    }
  });
}
```

- [ ] **Step 5: Add middleware session refresh**

Create `lib/supabase/middleware.ts`:

```ts
import { createServerClient } from "@supabase/ssr";
import { type NextRequest, NextResponse } from "next/server";

export async function updateSession(request: NextRequest) {
  let response = NextResponse.next({ request });
  const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const anonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY;

  if (!url || !anonKey) {
    return response;
  }

  const supabase = createServerClient(url, anonKey, {
    cookies: {
      getAll() {
        return request.cookies.getAll();
      },
      setAll(cookiesToSet) {
        cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
        response = NextResponse.next({ request });
        cookiesToSet.forEach(({ name, value, options }) => response.cookies.set(name, value, options));
      }
    }
  });

  await supabase.auth.getUser();
  return response;
}
```

Create `middleware.ts`:

```ts
import type { NextRequest } from "next/server";
import { updateSession } from "@/lib/supabase/middleware";

export async function middleware(request: NextRequest) {
  return updateSession(request);
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"]
};
```

- [ ] **Step 6: Add database TypeScript types**

Create `lib/db/types.ts`:

```ts
export type MarketTone = "进攻" | "轮动" | "防守" | "高波动";
export type StyleBias = "成长" | "价值" | "红利" | "周期" | "混合";
export type PositionBias = "积极" | "中性" | "谨慎";
export type ThemeStage = "启动" | "加速" | "分歧" | "兑现" | "退潮";
export type Priority = "高" | "中" | "低";
export type Market = "A股" | "港股" | "美股" | "ETF";
export type SymbolCategory = "核心龙头" | "景气验证" | "低位修复" | "ETF工具" | "暂停观察";
export type SymbolStatus = "关注" | "等待确认" | "降低关注" | "暂停观察" | "逻辑证伪";
export type ReviewType = "日复盘" | "周复盘";

export interface IndustryNote {
  industry: string;
  priority: Priority;
  coreLogic: string;
  catalyst: string;
  verificationMetrics: string[];
  risks: string[];
  representativeSymbols: string[];
}

export interface WatchlistUpdate {
  ticker: string;
  name: string;
  action: "新增观察" | "继续观察" | "降低关注" | "暂停观察";
  reason: string;
}
```

- [ ] **Step 7: Run checks**

Run:

```bash
npm run typecheck
npm run build
```

Expected: both commands pass.

- [ ] **Step 8: Commit**

```bash
git add .env.example supabase lib/supabase lib/db middleware.ts
git commit -m "feat: add supabase schema and clients"
```

### Task 3: Add Shared UI Shell

**Files:**
- Create: `components/app-shell.tsx`
- Create: `components/bottom-nav.tsx`
- Create: `components/status-pill.tsx`
- Create: `components/section-card.tsx`
- Create: `components/empty-state.tsx`
- Create: `lib/utils.ts`
- Create: `app/(dashboard)/layout.tsx`

- [ ] **Step 1: Add class utility**

Create `lib/utils.ts`:

```ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

- [ ] **Step 2: Add bottom navigation**

Create `components/bottom-nav.tsx`:

```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { Archive, BookOpenText, ClipboardList, LineChart, Search } from "lucide-react";
import { cn } from "@/lib/utils";

const items = [
  { href: "/briefs/today", label: "今日", icon: BookOpenText },
  { href: "/history", label: "历史", icon: Archive },
  { href: "/themes", label: "主线", icon: LineChart },
  { href: "/watchlist", label: "观察", icon: Search },
  { href: "/reviews", label: "复盘", icon: ClipboardList }
];

export function BottomNav() {
  const pathname = usePathname();

  return (
    <nav className="fixed inset-x-0 bottom-0 z-30 border-t border-line bg-white/95 backdrop-blur md:hidden">
      <div className="mx-auto grid max-w-md grid-cols-5">
        {items.map((item) => {
          const active = pathname === item.href || pathname.startsWith(`${item.href}/`);
          const Icon = item.icon;

          return (
            <Link
              key={item.href}
              href={item.href}
              className={cn(
                "flex min-h-14 flex-col items-center justify-center gap-1 text-xs",
                active ? "text-accent" : "text-slate-500"
              )}
            >
              <Icon className="h-5 w-5" aria-hidden="true" />
              <span>{item.label}</span>
            </Link>
          );
        })}
      </div>
    </nav>
  );
}
```

- [ ] **Step 3: Add layout components**

Create `components/app-shell.tsx`:

```tsx
import Link from "next/link";
import { BottomNav } from "@/components/bottom-nav";

const links = [
  { href: "/briefs/today", label: "今日内参" },
  { href: "/history", label: "历史内参" },
  { href: "/themes", label: "主线档案" },
  { href: "/watchlist", label: "观察池" },
  { href: "/reviews", label: "复盘" }
];

export function AppShell({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-screen bg-paper">
      <aside className="fixed inset-y-0 left-0 hidden w-64 border-r border-line bg-white px-5 py-6 md:block">
        <Link href="/briefs/today" className="block text-lg font-semibold text-ink">
          Daily Stock View
        </Link>
        <nav className="mt-8 space-y-2">
          {links.map((link) => (
            <Link key={link.href} href={link.href} className="block rounded-md px-3 py-2 text-sm text-slate-700 hover:bg-slate-100">
              {link.label}
            </Link>
          ))}
        </nav>
      </aside>
      <main className="mx-auto min-h-screen max-w-4xl px-4 pb-24 pt-5 md:ml-64 md:px-8 md:pb-10">
        {children}
      </main>
      <BottomNav />
    </div>
  );
}
```

Create `components/status-pill.tsx`:

```tsx
import { cn } from "@/lib/utils";

const toneClass: Record<string, string> = {
  进攻: "bg-red-50 text-risk",
  轮动: "bg-blue-50 text-accent",
  防守: "bg-emerald-50 text-good",
  高波动: "bg-amber-50 text-amber-700"
};

export function StatusPill({ label, value }: { label: string; value: string }) {
  return (
    <span className={cn("inline-flex items-center gap-1 rounded-full px-3 py-1 text-xs font-medium", toneClass[value] ?? "bg-slate-100 text-slate-700")}>
      <span className="text-slate-500">{label}</span>
      {value}
    </span>
  );
}
```

Create `components/section-card.tsx`:

```tsx
export function SectionCard({ title, children }: { title: string; children: React.ReactNode }) {
  return (
    <section className="rounded-lg border border-line bg-white p-4 shadow-sm">
      <h2 className="text-base font-semibold text-ink">{title}</h2>
      <div className="mt-3 text-sm leading-6 text-slate-700">{children}</div>
    </section>
  );
}
```

Create `components/empty-state.tsx`:

```tsx
export function EmptyState({ title, description }: { title: string; description: string }) {
  return (
    <div className="rounded-lg border border-dashed border-line bg-white p-6 text-center">
      <h2 className="text-base font-semibold text-ink">{title}</h2>
      <p className="mt-2 text-sm leading-6 text-slate-600">{description}</p>
    </div>
  );
}
```

- [ ] **Step 4: Add dashboard layout**

Create `app/(dashboard)/layout.tsx`:

```tsx
import { AppShell } from "@/components/app-shell";

export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return <AppShell>{children}</AppShell>;
}
```

- [ ] **Step 5: Run checks**

Run:

```bash
npm run typecheck
npm run build
```

Expected: both commands pass.

- [ ] **Step 6: Commit**

```bash
git add components lib/utils.ts app/\(dashboard\)/layout.tsx
git commit -m "feat: add mobile dashboard shell"
```

### Task 4: Implement Brief Domain And AI Draft Schema

**Files:**
- Create: `features/briefs/schema.ts`
- Create: `features/briefs/types.ts`
- Create: `lib/ai/prompt.ts`
- Create: `lib/ai/brief-generation.ts`
- Create: `lib/ai/brief-generation.test.ts`
- Create: `vitest.config.ts`

- [ ] **Step 1: Add Vitest config**

Create `vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "jsdom",
    globals: true,
    include: ["**/*.test.ts", "**/*.test.tsx"]
  },
  resolve: {
    alias: {
      "@": new URL(".", import.meta.url).pathname
    }
  }
});
```

- [ ] **Step 2: Add brief schema**

Create `features/briefs/schema.ts`:

```ts
import { z } from "zod";

export const industryNoteSchema = z.object({
  industry: z.string().min(1),
  priority: z.enum(["高", "中", "低"]),
  coreLogic: z.string().min(1),
  catalyst: z.string().min(1),
  verificationMetrics: z.array(z.string().min(1)).min(1),
  risks: z.array(z.string().min(1)).min(1),
  representativeSymbols: z.array(z.string().min(1)).default([])
});

export const watchlistUpdateSchema = z.object({
  ticker: z.string().min(1),
  name: z.string().min(1),
  action: z.enum(["新增观察", "继续观察", "降低关注", "暂停观察"]),
  reason: z.string().min(1)
});

export const briefDraftSchema = z.object({
  tradeDate: z.string().regex(/^\\d{4}-\\d{2}-\\d{2}$/),
  marketTone: z.enum(["进攻", "轮动", "防守", "高波动"]),
  styleBias: z.enum(["成长", "价值", "红利", "周期", "混合"]),
  positionBias: z.enum(["积极", "中性", "谨慎"]),
  headlineView: z.string().min(12),
  summaryPoints: z.array(z.string().min(1)).min(3).max(7),
  marketReview: z.string().min(1),
  strategyView: z.string().min(1),
  industryNotes: z.array(industryNoteSchema).min(1),
  watchlistUpdates: z.array(watchlistUpdateSchema).default([]),
  riskNotes: z.array(z.string().min(1)).min(3),
  verificationPoints: z.array(z.string().min(1)).min(3),
  dataGaps: z.array(z.string().min(1)).default([])
});

export const manualBriefInputSchema = z.object({
  tradeDate: z.string().regex(/^\\d{4}-\\d{2}-\\d{2}$/),
  rawInput: z.string().min(100, "请至少输入 100 个字符的行情、新闻或笔记内容。")
});

export type BriefDraft = z.infer<typeof briefDraftSchema>;
```

- [ ] **Step 3: Add type re-exports**

Create `features/briefs/types.ts`:

```ts
import type { BriefDraft } from "@/features/briefs/schema";

export type BriefViewModel = BriefDraft & {
  id?: string;
  rawInput?: string;
  createdAt?: string;
  updatedAt?: string;
};
```

- [ ] **Step 4: Add AI prompt**

Create `lib/ai/prompt.ts`:

```ts
export const MORNING_BRIEF_SYSTEM_PROMPT = `
你是一名有十多年 A 股、港股、美股二级市场经验的买方策略研究员。
你的任务是把用户提供的行情、新闻、截图转写、研报摘要和个人笔记，整理成券商晨会纪要风格的结构化 A 股策略内参。

硬性规则：
1. 输出用于研究参考，不构成个性化投资建议。
2. 不使用“必买、梭哈、稳赚、明天必涨、无脑买入、目标翻倍”等表达。
3. 不输出直接交易指令，只能使用“关注、等待确认、降低关注、暂停观察、逻辑待验证、逻辑增强、逻辑证伪”等研究状态。
4. 必须写清楚市场基调、主线逻辑、风险提示和验证点。
5. 如果用户输入缺少关键数据，在 dataGaps 中列出，不要编造。
6. 中文输出，专业、克制、信息密度高，适合手机阅读。
`.trim();

export function buildMorningBriefUserPrompt(input: { tradeDate: string; rawInput: string }) {
  return `
交易日期：${input.tradeDate}

请基于以下资料生成晨会内参草稿：

${input.rawInput}
`.trim();
}
```

- [ ] **Step 5: Add AI generation adapter**

Create `lib/ai/brief-generation.ts`:

```ts
import OpenAI from "openai";
import { briefDraftSchema, type BriefDraft } from "@/features/briefs/schema";
import { buildMorningBriefUserPrompt, MORNING_BRIEF_SYSTEM_PROMPT } from "@/lib/ai/prompt";

export async function generateBriefDraft(input: { tradeDate: string; rawInput: string }): Promise<BriefDraft> {
  const apiKey = process.env.OPENAI_API_KEY;
  if (!apiKey) {
    throw new Error("OPENAI_API_KEY is not configured.");
  }

  const client = new OpenAI({ apiKey });
  const response = await client.responses.create({
    model: process.env.OPENAI_MODEL ?? "gpt-4.1-mini",
    input: [
      { role: "system", content: MORNING_BRIEF_SYSTEM_PROMPT },
      { role: "user", content: buildMorningBriefUserPrompt(input) }
    ],
    text: {
      format: {
        type: "json_schema",
        name: "morning_brief",
        schema: {
          type: "object",
          additionalProperties: false,
          required: [
            "tradeDate",
            "marketTone",
            "styleBias",
            "positionBias",
            "headlineView",
            "summaryPoints",
            "marketReview",
            "strategyView",
            "industryNotes",
            "watchlistUpdates",
            "riskNotes",
            "verificationPoints",
            "dataGaps"
          ],
          properties: {
            tradeDate: { type: "string" },
            marketTone: { type: "string", enum: ["进攻", "轮动", "防守", "高波动"] },
            styleBias: { type: "string", enum: ["成长", "价值", "红利", "周期", "混合"] },
            positionBias: { type: "string", enum: ["积极", "中性", "谨慎"] },
            headlineView: { type: "string" },
            summaryPoints: { type: "array", items: { type: "string" } },
            marketReview: { type: "string" },
            strategyView: { type: "string" },
            industryNotes: {
              type: "array",
              items: {
                type: "object",
                additionalProperties: false,
                required: ["industry", "priority", "coreLogic", "catalyst", "verificationMetrics", "risks", "representativeSymbols"],
                properties: {
                  industry: { type: "string" },
                  priority: { type: "string", enum: ["高", "中", "低"] },
                  coreLogic: { type: "string" },
                  catalyst: { type: "string" },
                  verificationMetrics: { type: "array", items: { type: "string" } },
                  risks: { type: "array", items: { type: "string" } },
                  representativeSymbols: { type: "array", items: { type: "string" } }
                }
              }
            },
            watchlistUpdates: {
              type: "array",
              items: {
                type: "object",
                additionalProperties: false,
                required: ["ticker", "name", "action", "reason"],
                properties: {
                  ticker: { type: "string" },
                  name: { type: "string" },
                  action: { type: "string", enum: ["新增观察", "继续观察", "降低关注", "暂停观察"] },
                  reason: { type: "string" }
                }
              }
            },
            riskNotes: { type: "array", items: { type: "string" } },
            verificationPoints: { type: "array", items: { type: "string" } },
            dataGaps: { type: "array", items: { type: "string" } }
          }
        },
        strict: true
      }
    }
  });

  const text = response.output_text;
  if (!text) {
    throw new Error("AI response did not include JSON text.");
  }

  return briefDraftSchema.parse(JSON.parse(text));
}
```

- [ ] **Step 6: Add tests**

Create `lib/ai/brief-generation.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { briefDraftSchema } from "@/features/briefs/schema";
import { buildMorningBriefUserPrompt, MORNING_BRIEF_SYSTEM_PROMPT } from "@/lib/ai/prompt";

describe("morning brief prompt", () => {
  it("forbids direct trading language", () => {
    expect(MORNING_BRIEF_SYSTEM_PROMPT).toContain("不构成个性化投资建议");
    expect(MORNING_BRIEF_SYSTEM_PROMPT).toContain("不输出直接交易指令");
  });

  it("includes user trade date and raw input", () => {
    const prompt = buildMorningBriefUserPrompt({
      tradeDate: "2026-05-22",
      rawInput: "上证指数震荡，半导体分歧，机器人活跃。"
    });
    expect(prompt).toContain("2026-05-22");
    expect(prompt).toContain("机器人活跃");
  });
});

describe("briefDraftSchema", () => {
  it("accepts a complete valid draft", () => {
    const result = briefDraftSchema.safeParse({
      tradeDate: "2026-05-22",
      marketTone: "轮动",
      styleBias: "成长",
      positionBias: "中性",
      headlineView: "市场处于高成交轮动阶段，科技主线分歧后需要观察龙头承接。",
      summaryPoints: ["成交额维持高位", "科技成长仍是中期主线", "高位拥挤度上升"],
      marketReview: "三大指数震荡，成交活跃。",
      strategyView: "关注主线回踩后的承接质量。",
      industryNotes: [
        {
          industry: "半导体",
          priority: "高",
          coreLogic: "国产替代和 AI 算力需求共振。",
          catalyst: "设备订单和存储周期修复。",
          verificationMetrics: ["成交额", "订单", "毛利率"],
          risks: ["估值较高", "高位分歧"],
          representativeSymbols: ["北方华创", "中微公司"]
        }
      ],
      watchlistUpdates: [],
      riskNotes: ["避免追高后排", "关注成交额回落", "留意外部扰动"],
      verificationPoints: ["半导体龙头是否止跌", "机器人能否持续", "成交额是否维持"],
      dataGaps: []
    });

    expect(result.success).toBe(true);
  });
});
```

- [ ] **Step 7: Run tests**

Run:

```bash
npm test
npm run typecheck
```

Expected: tests and typecheck pass.

- [ ] **Step 8: Commit**

```bash
git add features/briefs lib/ai vitest.config.ts
git commit -m "feat: add brief generation schema"
```

### Task 5: Add AI Generate API Route

**Files:**
- Create: `app/api/ai/generate-brief/route.ts`

- [ ] **Step 1: Add route handler**

Create `app/api/ai/generate-brief/route.ts`:

```ts
import { NextResponse } from "next/server";
import { manualBriefInputSchema } from "@/features/briefs/schema";
import { generateBriefDraft } from "@/lib/ai/brief-generation";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: Request) {
  const supabase = await createClient();
  const {
    data: { user }
  } = await supabase.auth.getUser();

  if (!user) {
    return NextResponse.json({ error: "请先登录后再生成内参。" }, { status: 401 });
  }

  const body = await request.json();
  const parsed = manualBriefInputSchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json({ error: parsed.error.issues[0]?.message ?? "输入内容格式不正确。" }, { status: 400 });
  }

  try {
    const draft = await generateBriefDraft(parsed.data);
    return NextResponse.json({ draft });
  } catch (error) {
    const message = error instanceof Error ? error.message : "生成内参失败。";
    return NextResponse.json({ error: message }, { status: 500 });
  }
}
```

- [ ] **Step 2: Run checks**

Run:

```bash
npm run typecheck
npm run build
```

Expected: both commands pass.

- [ ] **Step 3: Commit**

```bash
git add app/api/ai/generate-brief/route.ts
git commit -m "feat: add ai brief generation route"
```

### Task 6: Implement Brief Save And Display

**Files:**
- Create: `features/briefs/actions.ts`
- Create: `features/briefs/brief-form.tsx`
- Create: `features/briefs/brief-editor.tsx`
- Create: `features/briefs/brief-view.tsx`
- Create: `features/briefs/brief-list.tsx`
- Create: `app/(dashboard)/briefs/today/page.tsx`
- Create: `app/(dashboard)/briefs/[id]/page.tsx`
- Create: `app/(dashboard)/history/page.tsx`

- [ ] **Step 1: Add brief server actions**

Create `features/briefs/actions.ts`:

```ts
"use server";

import { revalidatePath } from "next/cache";
import { redirect } from "next/navigation";
import { briefDraftSchema } from "@/features/briefs/schema";
import { createClient } from "@/lib/supabase/server";

export async function saveBriefDraft(rawDraft: unknown, rawInput: string) {
  const draft = briefDraftSchema.parse(rawDraft);
  const supabase = await createClient();
  const {
    data: { user },
    error: userError
  } = await supabase.auth.getUser();

  if (userError || !user) {
    throw new Error("请先登录后再保存内参。");
  }

  const { data, error } = await supabase
    .from("briefs")
    .upsert(
      {
        user_id: user.id,
        trade_date: draft.tradeDate,
        market_tone: draft.marketTone,
        style_bias: draft.styleBias,
        position_bias: draft.positionBias,
        headline_view: draft.headlineView,
        summary_points: draft.summaryPoints,
        market_review: draft.marketReview,
        strategy_view: draft.strategyView,
        industry_notes: draft.industryNotes,
        watchlist_updates: draft.watchlistUpdates,
        risk_notes: draft.riskNotes,
        verification_points: draft.verificationPoints,
        raw_input: rawInput
      },
      { onConflict: "user_id,trade_date" }
    )
    .select("id")
    .single();

  if (error) {
    throw new Error(error.message);
  }

  revalidatePath("/briefs/today");
  revalidatePath("/history");
  redirect(`/briefs/${data.id}`);
}
```

- [ ] **Step 2: Add brief view component**

Create `features/briefs/brief-view.tsx`:

```tsx
import { SectionCard } from "@/components/section-card";
import { StatusPill } from "@/components/status-pill";
import type { BriefDraft } from "@/features/briefs/schema";

export function BriefView({ brief }: { brief: BriefDraft }) {
  return (
    <article className="space-y-4">
      <header className="space-y-3">
        <p className="text-sm text-slate-500">{brief.tradeDate}</p>
        <h1 className="text-2xl font-semibold leading-tight text-ink">{brief.headlineView}</h1>
        <div className="flex flex-wrap gap-2">
          <StatusPill label="市场" value={brief.marketTone} />
          <StatusPill label="风格" value={brief.styleBias} />
          <StatusPill label="仓位" value={brief.positionBias} />
        </div>
      </header>

      <SectionCard title="今日摘要">
        <ul className="list-disc space-y-2 pl-5">
          {brief.summaryPoints.map((point) => (
            <li key={point}>{point}</li>
          ))}
        </ul>
      </SectionCard>

      <SectionCard title="市场回顾">
        <p>{brief.marketReview}</p>
      </SectionCard>

      <SectionCard title="策略观点">
        <p>{brief.strategyView}</p>
      </SectionCard>

      <SectionCard title="行业晨会纪要">
        <div className="space-y-4">
          {brief.industryNotes.map((note) => (
            <div key={note.industry} className="border-b border-line pb-4 last:border-0 last:pb-0">
              <div className="flex items-center justify-between gap-3">
                <h3 className="font-semibold text-ink">{note.industry}</h3>
                <StatusPill label="优先级" value={note.priority} />
              </div>
              <p className="mt-2">{note.coreLogic}</p>
              <p className="mt-2 text-slate-600">催化：{note.catalyst}</p>
              <p className="mt-2 text-slate-600">代表标的：{note.representativeSymbols.join("、") || "未列出"}</p>
            </div>
          ))}
        </div>
      </SectionCard>

      <SectionCard title="风险提示">
        <ul className="list-disc space-y-2 pl-5">
          {brief.riskNotes.map((risk) => (
            <li key={risk}>{risk}</li>
          ))}
        </ul>
      </SectionCard>

      <SectionCard title="本周验证点">
        <ul className="list-disc space-y-2 pl-5">
          {brief.verificationPoints.map((point) => (
            <li key={point}>{point}</li>
          ))}
        </ul>
      </SectionCard>
    </article>
  );
}
```

- [ ] **Step 3: Add AI generation form**

Create `features/briefs/brief-form.tsx`:

```tsx
"use client";

import { useState, useTransition } from "react";
import { briefDraftSchema, type BriefDraft } from "@/features/briefs/schema";
import { BriefEditor } from "@/features/briefs/brief-editor";

export function BriefForm() {
  const today = new Date().toISOString().slice(0, 10);
  const [tradeDate, setTradeDate] = useState(today);
  const [rawInput, setRawInput] = useState("");
  const [draft, setDraft] = useState<BriefDraft | null>(null);
  const [error, setError] = useState("");
  const [isPending, startTransition] = useTransition();

  function generateDraft() {
    setError("");
    startTransition(async () => {
      const response = await fetch("/api/ai/generate-brief", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ tradeDate, rawInput })
      });
      const result = await response.json();

      if (!response.ok) {
        setError(result.error ?? "生成失败。");
        return;
      }

      const parsed = briefDraftSchema.safeParse(result.draft);
      if (!parsed.success) {
        setError("AI 返回内容未通过结构校验。");
        return;
      }

      setDraft(parsed.data);
    });
  }

  if (draft) {
    return <BriefEditor draft={draft} rawInput={rawInput} />;
  }

  return (
    <div className="space-y-4 rounded-lg border border-line bg-white p-4 shadow-sm">
      <div>
        <label className="text-sm font-medium text-ink" htmlFor="trade-date">
          交易日期
        </label>
        <input
          id="trade-date"
          type="date"
          value={tradeDate}
          onChange={(event) => setTradeDate(event.target.value)}
          className="mt-2 w-full rounded-md border border-line px-3 py-2"
        />
      </div>
      <div>
        <label className="text-sm font-medium text-ink" htmlFor="raw-input">
          粘贴行情、新闻、截图文字或笔记
        </label>
        <textarea
          id="raw-input"
          value={rawInput}
          onChange={(event) => setRawInput(event.target.value)}
          rows={12}
          className="mt-2 w-full rounded-md border border-line px-3 py-2"
          placeholder="例如：指数表现、成交额、强弱板块、政策新闻、重点公司公告、个人观察..."
        />
      </div>
      {error ? <p className="text-sm text-risk">{error}</p> : null}
      <button
        type="button"
        onClick={generateDraft}
        disabled={isPending || rawInput.length < 100}
        className="w-full rounded-md bg-accent px-4 py-3 text-sm font-semibold text-white disabled:cursor-not-allowed disabled:opacity-50"
      >
        {isPending ? "正在生成..." : "生成晨会内参草稿"}
      </button>
    </div>
  );
}
```

- [ ] **Step 4: Add draft editor**

Create `features/briefs/brief-editor.tsx`:

```tsx
"use client";

import { useState, useTransition } from "react";
import { saveBriefDraft } from "@/features/briefs/actions";
import type { BriefDraft } from "@/features/briefs/schema";
import { BriefView } from "@/features/briefs/brief-view";

export function BriefEditor({ draft, rawInput }: { draft: BriefDraft; rawInput: string }) {
  const [editableDraft] = useState(draft);
  const [error, setError] = useState("");
  const [isPending, startTransition] = useTransition();

  function save() {
    setError("");
    startTransition(async () => {
      try {
        await saveBriefDraft(editableDraft, rawInput);
      } catch (saveError) {
        setError(saveError instanceof Error ? saveError.message : "保存失败。");
      }
    });
  }

  return (
    <div className="space-y-4">
      <BriefView brief={editableDraft} />
      {error ? <p className="text-sm text-risk">{error}</p> : null}
      <button
        type="button"
        onClick={save}
        disabled={isPending}
        className="w-full rounded-md bg-accent px-4 py-3 text-sm font-semibold text-white disabled:opacity-50"
      >
        {isPending ? "正在保存..." : "确认保存今日内参"}
      </button>
    </div>
  );
}
```

- [ ] **Step 5: Add brief list**

Create `features/briefs/brief-list.tsx`:

```tsx
import Link from "next/link";
import { StatusPill } from "@/components/status-pill";

interface BriefListItem {
  id: string;
  trade_date: string;
  market_tone: string;
  style_bias: string;
  position_bias: string;
  headline_view: string;
}

export function BriefList({ briefs }: { briefs: BriefListItem[] }) {
  return (
    <div className="space-y-3">
      {briefs.map((brief) => (
        <Link key={brief.id} href={`/briefs/${brief.id}`} className="block rounded-lg border border-line bg-white p-4 shadow-sm">
          <p className="text-xs text-slate-500">{brief.trade_date}</p>
          <h2 className="mt-2 text-base font-semibold leading-snug text-ink">{brief.headline_view}</h2>
          <div className="mt-3 flex flex-wrap gap-2">
            <StatusPill label="市场" value={brief.market_tone} />
            <StatusPill label="风格" value={brief.style_bias} />
            <StatusPill label="仓位" value={brief.position_bias} />
          </div>
        </Link>
      ))}
    </div>
  );
}
```

- [ ] **Step 6: Add today page**

Create `app/(dashboard)/briefs/today/page.tsx`:

```tsx
import { EmptyState } from "@/components/empty-state";
import { BriefForm } from "@/features/briefs/brief-form";
import { BriefView } from "@/features/briefs/brief-view";
import { briefDraftSchema } from "@/features/briefs/schema";
import { createClient } from "@/lib/supabase/server";

export default async function TodayBriefPage() {
  const supabase = await createClient();
  const today = new Date().toISOString().slice(0, 10);
  const { data } = await supabase.from("briefs").select("*").eq("trade_date", today).maybeSingle();

  if (!data) {
    return (
      <div className="space-y-4">
        <EmptyState title="今天还没有晨会内参" description="粘贴行情、新闻和你的观察笔记，生成一份结构化策略内参草稿。" />
        <BriefForm />
      </div>
    );
  }

  const brief = briefDraftSchema.parse({
    tradeDate: data.trade_date,
    marketTone: data.market_tone,
    styleBias: data.style_bias,
    positionBias: data.position_bias,
    headlineView: data.headline_view,
    summaryPoints: data.summary_points,
    marketReview: data.market_review,
    strategyView: data.strategy_view,
    industryNotes: data.industry_notes,
    watchlistUpdates: data.watchlist_updates,
    riskNotes: data.risk_notes,
    verificationPoints: data.verification_points,
    dataGaps: []
  });

  return <BriefView brief={brief} />;
}
```

- [ ] **Step 7: Add detail and history pages**

Create `app/(dashboard)/briefs/[id]/page.tsx`:

```tsx
import { notFound } from "next/navigation";
import { BriefView } from "@/features/briefs/brief-view";
import { briefDraftSchema } from "@/features/briefs/schema";
import { createClient } from "@/lib/supabase/server";

export default async function BriefDetailPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const supabase = await createClient();
  const { data } = await supabase.from("briefs").select("*").eq("id", id).single();

  if (!data) {
    notFound();
  }

  const brief = briefDraftSchema.parse({
    tradeDate: data.trade_date,
    marketTone: data.market_tone,
    styleBias: data.style_bias,
    positionBias: data.position_bias,
    headlineView: data.headline_view,
    summaryPoints: data.summary_points,
    marketReview: data.market_review,
    strategyView: data.strategy_view,
    industryNotes: data.industry_notes,
    watchlistUpdates: data.watchlist_updates,
    riskNotes: data.risk_notes,
    verificationPoints: data.verification_points,
    dataGaps: []
  });

  return <BriefView brief={brief} />;
}
```

Create `app/(dashboard)/history/page.tsx`:

```tsx
import { EmptyState } from "@/components/empty-state";
import { BriefList } from "@/features/briefs/brief-list";
import { createClient } from "@/lib/supabase/server";

export default async function HistoryPage() {
  const supabase = await createClient();
  const { data } = await supabase
    .from("briefs")
    .select("id, trade_date, market_tone, style_bias, position_bias, headline_view")
    .order("trade_date", { ascending: false });

  if (!data || data.length === 0) {
    return <EmptyState title="暂无历史内参" description="保存第一份晨会内参后，这里会形成你的投研时间线。" />;
  }

  return (
    <div className="space-y-4">
      <h1 className="text-2xl font-semibold text-ink">历史内参</h1>
      <BriefList briefs={data} />
    </div>
  );
}
```

- [ ] **Step 8: Run checks**

Run:

```bash
npm run typecheck
npm run build
```

Expected: both commands pass.

- [ ] **Step 9: Commit**

```bash
git add features/briefs app/\(dashboard\)/briefs app/\(dashboard\)/history
git commit -m "feat: add brief creation and history"
```

### Task 7: Implement Themes, Watchlist, And Reviews Read Views

**Files:**
- Create: `features/themes/theme-card.tsx`
- Create: `features/themes/theme-list.tsx`
- Create: `app/(dashboard)/themes/page.tsx`
- Create: `features/watchlist/symbol-card.tsx`
- Create: `features/watchlist/symbol-list.tsx`
- Create: `app/(dashboard)/watchlist/page.tsx`
- Create: `features/reviews/review-list.tsx`
- Create: `app/(dashboard)/reviews/page.tsx`

- [ ] **Step 1: Add theme list**

Create `features/themes/theme-card.tsx`:

```tsx
import { StatusPill } from "@/components/status-pill";

interface ThemeCardProps {
  name: string;
  stage: string;
  priority: string;
  coreLogic: string;
  latestUpdate: string;
}

export function ThemeCard({ name, stage, priority, coreLogic, latestUpdate }: ThemeCardProps) {
  return (
    <article className="rounded-lg border border-line bg-white p-4 shadow-sm">
      <div className="flex items-start justify-between gap-3">
        <h2 className="text-lg font-semibold text-ink">{name}</h2>
        <div className="flex flex-wrap justify-end gap-2">
          <StatusPill label="阶段" value={stage} />
          <StatusPill label="优先级" value={priority} />
        </div>
      </div>
      <p className="mt-3 text-sm leading-6 text-slate-700">{coreLogic || "尚未记录核心逻辑。"}</p>
      {latestUpdate ? <p className="mt-3 text-xs text-slate-500">最新更新：{latestUpdate}</p> : null}
    </article>
  );
}
```

Create `features/themes/theme-list.tsx`:

```tsx
import { ThemeCard } from "@/features/themes/theme-card";

interface ThemeRecord {
  id: string;
  name: string;
  stage: string;
  priority: string;
  core_logic: string;
  latest_update: string;
}

export function ThemeList({ themes }: { themes: ThemeRecord[] }) {
  return (
    <div className="space-y-3">
      {themes.map((theme) => (
        <ThemeCard
          key={theme.id}
          name={theme.name}
          stage={theme.stage}
          priority={theme.priority}
          coreLogic={theme.core_logic}
          latestUpdate={theme.latest_update}
        />
      ))}
    </div>
  );
}
```

Create `app/(dashboard)/themes/page.tsx`:

```tsx
import { EmptyState } from "@/components/empty-state";
import { ThemeList } from "@/features/themes/theme-list";
import { createClient } from "@/lib/supabase/server";

export default async function ThemesPage() {
  const supabase = await createClient();
  const { data } = await supabase.from("themes").select("id, name, stage, priority, core_logic, latest_update").order("updated_at", { ascending: false });

  if (!data || data.length === 0) {
    return <EmptyState title="暂无主线档案" description="保存晨会内参后，可以逐步沉淀半导体、AI 算力、机器人等中期主线。" />;
  }

  return (
    <div className="space-y-4">
      <h1 className="text-2xl font-semibold text-ink">主线档案</h1>
      <ThemeList themes={data} />
    </div>
  );
}
```

- [ ] **Step 2: Add watchlist list**

Create `features/watchlist/symbol-card.tsx`:

```tsx
import { StatusPill } from "@/components/status-pill";

interface SymbolCardProps {
  ticker: string;
  name: string;
  market: string;
  category: string;
  status: string;
  watchReason: string;
}

export function SymbolCard({ ticker, name, market, category, status, watchReason }: SymbolCardProps) {
  return (
    <article className="rounded-lg border border-line bg-white p-4 shadow-sm">
      <div className="flex items-start justify-between gap-3">
        <div>
          <h2 className="text-lg font-semibold text-ink">{name}</h2>
          <p className="mt-1 text-xs text-slate-500">{ticker} · {market}</p>
        </div>
        <StatusPill label={category} value={status} />
      </div>
      <p className="mt-3 text-sm leading-6 text-slate-700">{watchReason || "尚未记录观察原因。"}</p>
    </article>
  );
}
```

Create `features/watchlist/symbol-list.tsx`:

```tsx
import { SymbolCard } from "@/features/watchlist/symbol-card";

interface SymbolRecord {
  id: string;
  ticker: string;
  name: string;
  market: string;
  category: string;
  status: string;
  watch_reason: string;
}

export function SymbolList({ symbols }: { symbols: SymbolRecord[] }) {
  return (
    <div className="space-y-3">
      {symbols.map((symbol) => (
        <SymbolCard
          key={symbol.id}
          ticker={symbol.ticker}
          name={symbol.name}
          market={symbol.market}
          category={symbol.category}
          status={symbol.status}
          watchReason={symbol.watch_reason}
        />
      ))}
    </div>
  );
}
```

Create `app/(dashboard)/watchlist/page.tsx`:

```tsx
import { EmptyState } from "@/components/empty-state";
import { SymbolList } from "@/features/watchlist/symbol-list";
import { createClient } from "@/lib/supabase/server";

export default async function WatchlistPage() {
  const supabase = await createClient();
  const { data } = await supabase
    .from("symbols")
    .select("id, ticker, name, market, category, status, watch_reason")
    .order("updated_at", { ascending: false });

  if (!data || data.length === 0) {
    return <EmptyState title="暂无观察池标的" description="这里用于保存股票和 ETF 的研究状态，不作为直接交易指令。" />;
  }

  return (
    <div className="space-y-4">
      <h1 className="text-2xl font-semibold text-ink">观察池</h1>
      <SymbolList symbols={data} />
    </div>
  );
}
```

- [ ] **Step 3: Add review list**

Create `features/reviews/review-list.tsx`:

```tsx
interface ReviewRecord {
  id: string;
  review_date: string;
  review_type: string;
  previous_view_result: string;
}

export function ReviewList({ reviews }: { reviews: ReviewRecord[] }) {
  return (
    <div className="space-y-3">
      {reviews.map((review) => (
        <article key={review.id} className="rounded-lg border border-line bg-white p-4 shadow-sm">
          <p className="text-xs text-slate-500">{review.review_date} · {review.review_type}</p>
          <p className="mt-3 text-sm leading-6 text-slate-700">{review.previous_view_result || "尚未记录复盘结论。"}</p>
        </article>
      ))}
    </div>
  );
}
```

Create `app/(dashboard)/reviews/page.tsx`:

```tsx
import { EmptyState } from "@/components/empty-state";
import { ReviewList } from "@/features/reviews/review-list";
import { createClient } from "@/lib/supabase/server";

export default async function ReviewsPage() {
  const supabase = await createClient();
  const { data } = await supabase
    .from("reviews")
    .select("id, review_date, review_type, previous_view_result")
    .order("review_date", { ascending: false });

  if (!data || data.length === 0) {
    return <EmptyState title="暂无复盘记录" description="每周记录一次判断验证情况，系统会逐步形成你的投研记忆。" />;
  }

  return (
    <div className="space-y-4">
      <h1 className="text-2xl font-semibold text-ink">复盘</h1>
      <ReviewList reviews={data} />
    </div>
  );
}
```

- [ ] **Step 4: Run checks**

Run:

```bash
npm run typecheck
npm run build
```

Expected: both commands pass.

- [ ] **Step 5: Commit**

```bash
git add features/themes features/watchlist features/reviews app/\(dashboard\)/themes app/\(dashboard\)/watchlist app/\(dashboard\)/reviews
git commit -m "feat: add research archive views"
```

### Task 8: Add Login Page And Route Protection

**Files:**
- Create: `app/(auth)/login/page.tsx`
- Modify: `app/(dashboard)/layout.tsx`

- [ ] **Step 1: Add login page**

Create `app/(auth)/login/page.tsx`:

```tsx
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";

export default async function LoginPage({
  searchParams
}: {
  searchParams: Promise<{ email?: string; sent?: string }>;
}) {
  const params = await searchParams;
  const supabase = await createClient();
  const {
    data: { user }
  } = await supabase.auth.getUser();

  if (user) {
    redirect("/briefs/today");
  }

  async function signIn(formData: FormData) {
    "use server";

    const email = String(formData.get("email") ?? "");
    const supabaseServer = await createClient();
    const origin = process.env.APP_URL ?? "http://localhost:3000";

    await supabaseServer.auth.signInWithOtp({
      email,
      options: {
        emailRedirectTo: `${origin}/briefs/today`
      }
    });

    redirect(`/login?sent=1&email=${encodeURIComponent(email)}`);
  }

  return (
    <main className="mx-auto flex min-h-screen max-w-md flex-col justify-center px-4">
      <div className="rounded-lg border border-line bg-white p-6 shadow-sm">
        <h1 className="text-2xl font-semibold text-ink">登录 Daily Stock View</h1>
        <p className="mt-2 text-sm leading-6 text-slate-600">输入邮箱获取登录链接。第一版使用 Supabase Magic Link，减少密码管理成本。</p>
        {params.sent ? <p className="mt-4 rounded-md bg-emerald-50 p-3 text-sm text-good">登录链接已发送至 {params.email}。</p> : null}
        <form action={signIn} className="mt-5 space-y-4">
          <input name="email" type="email" required placeholder="you@example.com" className="w-full rounded-md border border-line px-3 py-3" />
          <button className="w-full rounded-md bg-accent px-4 py-3 text-sm font-semibold text-white">发送登录链接</button>
        </form>
      </div>
    </main>
  );
}
```

- [ ] **Step 2: Protect dashboard layout**

Modify `app/(dashboard)/layout.tsx`:

```tsx
import { redirect } from "next/navigation";
import { AppShell } from "@/components/app-shell";
import { createClient } from "@/lib/supabase/server";

export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const supabase = await createClient();
  const {
    data: { user }
  } = await supabase.auth.getUser();

  if (!user) {
    redirect("/login");
  }

  return <AppShell>{children}</AppShell>;
}
```

- [ ] **Step 3: Run checks**

Run:

```bash
npm run typecheck
npm run build
```

Expected: both commands pass.

- [ ] **Step 4: Commit**

```bash
git add app/\(auth\)/login app/\(dashboard\)/layout.tsx
git commit -m "feat: add passwordless login"
```

### Task 9: Add E2E Smoke Test

**Files:**
- Create: `playwright.config.ts`
- Create: `tests/e2e/smoke.spec.ts`

- [ ] **Step 1: Add Playwright config**

Create `playwright.config.ts`:

```ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  timeout: 30_000,
  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry"
  },
  projects: [
    {
      name: "Mobile Safari",
      use: { ...devices["iPhone 13"] }
    },
    {
      name: "Desktop Chrome",
      use: { ...devices["Desktop Chrome"] }
    }
  ],
  webServer: {
    command: "npm run dev",
    url: "http://localhost:3000",
    reuseExistingServer: true,
    timeout: 120_000
  }
});
```

- [ ] **Step 2: Add smoke test**

Create `tests/e2e/smoke.spec.ts`:

```ts
import { expect, test } from "@playwright/test";

test("login page renders", async ({ page }) => {
  await page.goto("/login");
  await expect(page.getByRole("heading", { name: "登录 Daily Stock View" })).toBeVisible();
  await expect(page.getByPlaceholder("you@example.com")).toBeVisible();
});
```

- [ ] **Step 3: Install browsers**

Run:

```bash
npx playwright install
```

Expected: Playwright browser binaries install successfully.

- [ ] **Step 4: Run E2E**

Run:

```bash
npm run e2e
```

Expected: the login page renders in mobile and desktop projects.

- [ ] **Step 5: Commit**

```bash
git add playwright.config.ts tests/e2e
git commit -m "test: add smoke coverage"
```

### Task 10: Prepare Deployment

**Files:**
- Modify: `README.md`
- Create: `docs/deployment.md`

- [ ] **Step 1: Add deployment doc**

Create `docs/deployment.md`:

```md
# Deployment

## Services

- App hosting: Vercel
- Database and auth: Supabase
- AI: OpenAI Responses API

## Required Environment Variables

- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`
- `OPENAI_API_KEY`
- `OPENAI_MODEL`
- `APP_URL`

## Supabase Setup

1. Create a Supabase project.
2. Run `supabase/migrations/0001_initial_schema.sql` in the SQL editor or through Supabase CLI.
3. Enable email Magic Link auth.
4. Add the deployed Vercel URL to Supabase Auth redirect URLs.

## Vercel Setup

1. Import the GitHub repository.
2. Set the environment variables.
3. Deploy the `main` branch.
4. Confirm `/login` renders.
5. Confirm Magic Link redirects to `/briefs/today`.

## Local Development

1. Copy `.env.example` to `.env.local`.
2. Fill Supabase and OpenAI keys.
3. Run `npm install`.
4. Run `npm run dev`.
```

- [ ] **Step 2: Update README**

Modify `README.md` by adding:

````md
## Technical Direction

- Next.js App Router
- Supabase Auth and Postgres
- OpenAI structured draft generation
- Vercel deployment

## Local Development

```bash
npm install
npm run dev
```

See `docs/deployment.md` for deployment setup.
````

- [ ] **Step 3: Run final checks**

Run:

```bash
npm test
npm run typecheck
npm run build
```

Expected: all checks pass.

- [ ] **Step 4: Commit**

```bash
git add README.md docs/deployment.md
git commit -m "docs: add deployment guide"
```

### Task 11: Push And Verify Remote

**Files:**
- No code files.

- [ ] **Step 1: Review commit history**

Run:

```bash
git log --oneline --max-count=12
```

Expected: commits from Tasks 1-10 appear in order.

- [ ] **Step 2: Push**

Run:

```bash
git push origin main
```

Expected: GitHub remote receives all commits.

- [ ] **Step 3: Verify clean working tree**

Run:

```bash
git status --short
```

Expected: no output.

## 6. Testing Strategy

Run these before merging or deploying:

```bash
npm test
npm run typecheck
npm run build
npm run e2e
```

Coverage priorities:

- Zod schemas reject malformed AI output.
- Prompt contains compliance and risk-control language.
- Login page renders on mobile.
- Dashboard routes redirect unauthenticated users.
- Brief generation form can show an AI draft.
- Saved briefs render in detail and history views.

## 7. Deployment Strategy

Use this order:

1. Create Supabase project.
2. Run migration.
3. Configure Magic Link redirect URLs.
4. Create Vercel project from GitHub.
5. Add environment variables.
6. Deploy `main`.
7. Open `/login` on mobile.
8. Create first user through Magic Link.
9. Generate and save a test brief.
10. Confirm data appears in Supabase tables.

## 8. Spec Coverage Review

Covered:

- Daily morning brief: Tasks 4-6.
- Historical strategy archive: Task 6.
- Theme files: Task 7 read view and schema in Task 2.
- Watchlist: Task 7 read view and schema in Task 2.
- Reviews: Task 7 read view and schema in Task 2.
- AI-assisted generation: Tasks 4-5.
- Human confirmation before save: Task 6.
- Mobile-first navigation: Task 3.
- Long-term saved data: Task 2.
- Login and ownership: Task 8.
- Deployment path: Task 10.

Deferred by spec:

- Real-time行情.
- 自动交易.
- 盘中短线提醒.
- 订阅支付.
- 自动新闻抓取.

## 9. Risk Notes

- Supabase generated TypeScript types are not included in this plan. Add them after Supabase CLI setup if the project grows; the MVP can use explicit domain types first.
- The AI route depends on current OpenAI structured output behavior. If the SDK changes, keep `briefDraftSchema` as the final validation gate.
- The draft editor in Task 6 previews and saves the generated draft but does not yet provide field-by-field editing. Add editable fields after the first usable version is deployed.
- Theme, watchlist, and review pages are read-only in this plan. Create forms after validating the morning brief workflow.
- Server actions require a working Supabase session; test login before debugging save behavior.
