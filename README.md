# Stash

A developer-first snippet manager for storing, tagging, organizing, and sharing
code snippets. Built with [Supabase](https://supabase.com/) (PostgreSQL, Auth,
Edge Functions) and [Next.js](https://nextjs.org/) (TypeScript).

V1 targets individual developers and small teams. V2 extends into multi-tenant
organizations with federation, relationship-based access control, auditing, and
trust — enabling Stash to become the **system of record for reusable code**
across teams and organizations. V3 adds AI intelligence (semantic search,
auto-tagging, explanations), a full ecosystem of integrations (VS Code,
JetBrains, Neovim, browser extension, Slack, webhooks), and advanced DX
features (live playgrounds, snippet templates, forking, analytics, offline
mode) — making Stash **proactively useful** wherever developers write code.

## Why Stash Exists

Developers accumulate snippets everywhere — in Gist, Notion, bookmarks, random
text files, chat histories. None of these are purpose-built for the workflow:
save a snippet, tag it, group it into a project, find it again in seconds, share
it with a colleague via a single link.

Stash fills that gap: **GitHub Gist meets Notion meets Bookmarks — without
the overhead.** It is intentionally kept lean: one managed PostgreSQL database,
one frontend. No custom API server, no Elasticsearch, no Redis, no S3 —
PostgreSQL handles full-text search, JSON label indexing, and relational data in
a single engine.

**Positioning:** "Your personal snippet library — searchable in milliseconds,
shareable in one click."

**Core differentiators:**

- **Faster than Gist** — instant search, keyboard-first, no GitHub UI overhead
- **More structured than Notion** — projects, labels, language detection
- **More shareable than local files** — one-click links, embeds, raw endpoints
- **Developer-native** — CLI, API, editor integration, `curl`-friendly

### User Personas

| Persona | Description | Key Need |
|---------|-------------|----------|
| **Solo Dev** | Freelancer or hobbyist, 50-500 snippets | Fast capture and retrieval |
| **Team Lead** | Manages 3-10 engineers, shares patterns | Team snippet library with permissions |
| **DevOps Engineer** | Heavy scripting, many one-liners | CLI integration, raw endpoints, `curl` |
| **Learner** | Bootcamp/university student | Organize notes and examples by topic |

---

## Overview

Stash lets users store code snippets with syntax highlighting, organize
them into projects, tag them with labels, search across title/description/content
with ranked full-text search, and share them via unlisted or public links.

### Capabilities

- Store and manage code snippets with syntax highlighting and Markdown
  descriptions
- **Two snippet types**: single-file (one code block) and repository
  (multi-file directory tree, like a small Git repo)
- Organize snippets into projects (optional grouping)
- **Collections**: curated groups of snippets with their own title, description,
  labels, and sharing — like playlists for code
- **Snippet links**: lightweight, directional references between snippets
  (related, depends_on, supersedes, etc.)
- Tag snippets with labels (stored as JSON arrays, GIN-indexed)
- Full-text search across title, description, and content with weighted ranking
  (title > description > content)
- **Snippet versioning**: every save creates a version with commit message,
  diff view, and restore capability
- Three visibility levels: private, unlisted (share-link), public
- Share snippets via UUID-based tokens without changing visibility
- GitHub OAuth2 authentication (via Supabase Auth)
- Raw content endpoint for `curl` / scripting integration
- **Git-like CLI workflow**: clone, pull, push, diff, status, log — works for
  both single-file and repository snippets with conflict detection
- Label autocomplete with usage frequency
- Multi-instance deployable from a single codebase

### Tech Stack

| Component  | Technology                                            |
|------------|-------------------------------------------------------|
| Frontend   | Next.js 15+ (App Router), Tailwind CSS, CodeMirror 6 |
| Backend    | Supabase (PostgREST, Auth, Edge Functions)            |
| Database   | PostgreSQL 15+ (managed by Supabase) with GIN indexes |
| Auth       | Supabase Auth with GitHub OAuth2 provider             |
| Search     | PostgreSQL Full-Text Search (`tsvector`, weighted)    |
| IaC        | Supabase CLI, SQL migrations, GitHub Actions          |
| Deployment | Supabase (hosted) + Vercel/Cloudflare (frontend)      |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Supabase (managed)                                  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                        Next.js Frontend                                │  │
│  │                                                                        │  │
│  │  ┌──────────────┐  ┌───────────────┐  ┌────────────────────────────┐   │  │
│  │  │  Dashboard   │  │ Snippet View  │  │  Snippet Editor            │   │  │
│  │  │  /           │  │ /snippets/:id │  │  /snippets/new             │   │  │
│  │  │              │  │               │  │  /snippets/:id/edit        │   │  │
│  │  │  - Search    │  │ - Syntax HL   │  │                            │   │  │
│  │  │  - Filters   │  │ - Markdown    │  │  - CodeMirror 6            │   │  │
│  │  │  - Quick Add │  │ - Copy / Raw  │  │  - Label Autocomplete      │   │  │
│  │  └──────────────┘  └───────────────┘  │  - Language Detect         │   │  │
│  │                                       └────────────────────────────┘   │  │
│  │  ┌──────────────────┐  ┌───────────────────┐                           │  │
│  │  │ Project View     │  │ Public Share      │                           │  │
│  │  │ /projects/:id    │  │ /shared/:token    │                           │  │
│  │  └──────────────────┘  └───────────────────┘                           │  │
│  └──────────────────────────┬─────────────────────────────────────────────┘  │
│                             │ @supabase/ssr                                  │
│                             ▼                                                │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                     Supabase Platform                                  │  │
│  │                                                                        │  │
│  │  ┌──────────────┐  ┌───────────────┐  ┌────────────────────────────┐   │  │
│  │  │  Auth        │  │  PostgREST    │  │  Edge Functions            │   │  │
│  │  │  (GoTrue)    │  │  (auto API)   │  │                            │   │  │
│  │  │              │  │               │  │  raw-snippet               │   │  │
│  │  │  GitHub OAuth│  │  CRUD for     │  │  → text/plain export       │   │  │
│  │  │  JWT sessions│  │  all tables   │  │  → curl/scripting          │   │  │
│  │  └──────────────┘  └───────────────┘  └────────────────────────────┘   │  │
│  │                              │                                         │  │
│  │                     RPC Functions (SQL)                                │  │
│  │          ┌───────────────────┼───────────────────────┐                 │  │
│  │          │                   │                       │                 │  │
│  │  ┌───────────────┐  ┌───────────────┐  ┌────────────────────┐          │  │
│  │  │ search_       │  │ get_shared_   │  │ get_user_labels    │          │  │
│  │  │ snippets      │  │ snippet       │  │                    │          │  │
│  │  │               │  │               │  │ generate_share_    │          │  │
│  │  │ FTS + rank +  │  │ SECURITY      │  │ token              │          │  │
│  │  │ filters       │  │ DEFINER       │  │                    │          │  │
│  │  └───────────────┘  └───────────────┘  │ revoke_share_token │          │  │
│  │                                        └────────────────────┘          │  │
│  │                              │                                         │  │
│  │                              ▼                                         │  │
│  │  ┌────────────────────────────────────────────────────────────────┐    │  │
│  │  │                    PostgreSQL 15+                              │    │  │
│  │  │                                                                │    │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │    │  │
│  │  │  │ profiles │  │ projects │  │ snippets │                      │    │  │
│  │  │  └──────────┘  └──────────┘  └────┬─────┘                      │    │  │
│  │  │                                   │                            │    │  │
│  │  │                    ┌──────────────┼──────────────┐             │    │  │
│  │  │                    │              │              │             │    │  │
│  │  │              ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴──────┐      │    │  │
│  │  │              │ GIN index │  │ tsvector  │  │ share      │      │    │  │
│  │  │              │ (labels)  │  │ (FTS)     │  │ tokens     │      │    │  │
│  │  │              └───────────┘  └───────────┘  └────────────┘      │    │  │
│  │  │                                                                │    │  │
│  │  │        RLS Policies (authorization in the database)            │    │  │
│  │  └────────────────────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Design Principles

- **No custom backend** — Supabase provides Auth, REST API (PostgREST), and Edge
  Functions. No FastAPI, no SQLAlchemy, no custom JWT logic.
- **PostgreSQL-first** — Full-text search, JSON indexing, RLS authorization, and
  business logic (RPC functions) all live in PostgreSQL.
- **Infrastructure as Code** — Zero configuration via UI. Everything is SQL
  migrations, `config.toml`, and shell scripts. Deployable entirely via CI.
- **Multi-instance** — Same codebase deploys N independent Stash instances via
  parameterized environment files and GitHub Actions matrix.
- **Progressive complexity** — Snippets can exist without projects, labels are
  optional, sharing is opt-in.

---

## UX Concept

### Three-Second Rule

Every primary action must complete in under three seconds. This is the core UX
principle that makes Stash feel like a native tool rather than a web app.

### Command Palette (Cmd+K)

The **single most important UX element**. A universal command palette that
combines search, navigation, and actions — modeled after Raycast/Linear/Notion.

```
┌─────────────────────────────────────────────────────────┐
│  > search snippets, projects, labels, commands...       │
├─────────────────────────────────────────────────────────┤
│  Recent                                                 │
│  ┌─────────────────────────────────────────────────┐    │
│  │  deploy-k8s.sh          bash  k8s,deploy  2h ago│    │
│  │  useAuth hook           tsx   react,auth  1d ago│    │
│  │  nginx reverse proxy    nginx  devops     3d ago│    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  Actions                                                │
│  ┌─────────────────────────────────────────────────┐    │
│  │  + New Snippet                          Cmd+N   │    │
│  │  + New Project                      Cmd+Shift+N │    │
│  │  > Import from Gist                             │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Behavior:**

- Opens instantly on `Cmd+K` (or `Ctrl+K`)
- Typing filters snippets via full-text search in real-time
- Arrow keys navigate, Enter opens, `Cmd+C` copies content directly
- Prefix with `>` for commands, `#` for labels, `@` for projects, `&` for
  collections
- Results show inline preview (first 3 lines of code)

### Dashboard

The dashboard is **not** a landing page. It is a **workspace**.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Stash                                    Cmd+K Search    [avatar]       │
├────────────────┬─────────────────────────────────────────────────────────┤
│                │                                                         │
│  PINNED (3)    │  -- All Snippets ──────────────────── 147 snippets ──   │
│  * deploy.sh   │                                                         │
│  * useAuth     │     List | Grid      Sort: Updated                      │
│  * .gitignore  │     [+] New Snippet                         Cmd+N       │
│                │                                                         │
│  PROJECTS      │  ┌──────────────────────────────────────────────────┐   │
│  > infra (23)  │  │ deploy-k8s.sh                    2 hours ago     │   │
│  > react (41)  │  │ #!/bin/bash                                      │   │
│  > scripts (8) │  │ kubectl apply -f deployment.yaml                 │   │
│                │  │ kubectl rollout status...                        │   │
│  COLLECTIONS   │  │                                                  │   │
│  > K8s Recipes │  │                                                  │   │
│  > React Pat.. │  │                                                  │   │
│  > Onboarding  │  │                                                  │   │
│                │  │                                                  │   │
│  LABELS        │  │                                                  │   │
│  bash (34)     │  │ bash  k8s  deploy  infra       *  Share  ...     │   │
│  python (28)   │  └──────────────────────────────────────────────────┘   │
│  react (22)    │                                                         │
│  docker (15)   │  ┌──────────────────────────────────────────────────┐   │
│  k8s (12)      │  │ useAuth.tsx                       yesterday      │   │
│                │  │ export function useAuth() {                      │   │
│  QUICK FILTERS │  │   const [user, setUser] = useState(null)         │   │
│  [ ] Has share │  │   ...                                            │   │
│  [ ] Public    │  │ tsx  react  auth  hooks        *  Share  ...     │   │
│  [ ] AI-sourced│  └──────────────────────────────────────────────────┘   │
│                │                                                         │
└────────────────┴─────────────────────────────────────────────────────────┘
```

**Key UX decisions:**

1. **Snippet cards show code preview** — Users recognize snippets by their
   content, not just titles. Show 3-4 lines of syntax-highlighted code
   directly on the card.

2. **Left sidebar is navigational** — Projects and labels are clickable
   filters. Clicking a label instantly filters the list. Multiple labels can
   be combined (AND logic).

3. **Pinned snippets** — Up to 10 pinned snippets appear at the top of the
   sidebar. These are the user's "hot" snippets — the ones they reach for
   daily. Pin/unpin via right-click or star icon.

4. **Inline actions** — Every snippet card has hover actions:
   - Star (pin/unpin)
   - Share (generate/copy link)
   - More menu (edit, move to project, duplicate, delete)
   - Copy content (one click, no navigation needed)

5. **Empty state is an onboarding** — First visit shows:
   - "Create your first snippet" with a pre-filled example
   - "Import from GitHub Gist" button
   - "Install the CLI" link

6. **Grid vs. List toggle** — List view for scanning, grid view for visual
   browsing. User preference is persisted.

### Snippet Editor

The editor must feel like a miniature IDE, not a textarea.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  < Back                    Edit Snippet                    Save  Cmd+S   │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Title    ┌─────────────────────────────────────────────────────────┐    │
│           │ Kubernetes deployment script                            │    │
│           └─────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Language  bash (auto-detected)                                          │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  1  #!/bin/bash                                                  │    │
│  │  2  set -euo pipefail                                            │    │
│  │  3                                                               │    │
│  │  4  NAMESPACE="${1:-default}"                                    │    │
│  │  5  kubectl apply -f deployment.yaml -n "$NAMESPACE"             │    │
│  │  6  kubectl rollout status deployment/app -n "$NAMESPACE"        │    │
│  │                                                      CodeMirror 6│    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Description (Markdown)                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │ Deploys the app to a Kubernetes namespace. Pass namespace as     │    │
│  │ first argument, defaults to `default`.                           │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Labels   ┌──────────────────────────────────────────┐                   │
│           │ bash x | k8s x | deploy x | type to add  │                   │
│           └──────────────────────────────────────────┘                   │
│           Suggestions: kubernetes, devops, scripting                     │
│                                                                          │
│  Project  [ infra            ]  Visibility  [ Private    ]               │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Key UX decisions:**

1. **Auto-save** — Draft is saved to `localStorage` every 5 seconds. Explicit
   "Save" creates/updates the snippet in Supabase. No work is ever lost.

2. **Language auto-detection** — On paste or after typing, detect the
   language via heuristics. User can override.

3. **Label autocomplete** — Type in the label field, see suggestions ranked
   by usage frequency (from `get_user_labels` RPC). New labels are created
   inline without a separate form.

4. **AI label suggestions** — After content is entered, suggest 2-3 labels
   based on content analysis. Displayed as clickable chips below the label
   input. Non-intrusive — user ignores or clicks to add.

5. **Keyboard-first** — `Tab` moves between fields. `Cmd+S` saves.
   `Cmd+Enter` saves and closes. `Esc` goes back (with unsaved changes
   warning).

### Snippet View

Read-only view with maximum focus on the code.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  < Back                                        Edit  Share  ...  Delete  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Kubernetes deployment script                                            │
│  infra / bash                                        Updated 2h ago      │
│  [ bash ] [ k8s ] [ deploy ]                                             │
│                                                                          │
│  ┌────────────────────────────────────────────────── Copy  Raw  Embed ┐  │
│  │  1  #!/bin/bash                                                    │  │
│  │  2  set -euo pipefail                                              │  │
│  │  3                                                                 │  │
│  │  4  NAMESPACE="${1:-default}"                                      │  │
│  │  5  kubectl apply -f deployment.yaml -n "$NAMESPACE"               │  │
│  │  6  kubectl rollout status deployment/app -n "$NAMESPACE"          │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  Deploys the app to a Kubernetes namespace. Pass namespace as first      │
│  argument, defaults to `default`.                                        │
│                                                                          │
│  -- Share ──────────────────────────────────────────────────────────     │
│  https://stash.dev/s/a1b2c3d4          Copy Link    Revoke               │
│  Shared 3 times - Last accessed 1h ago                                   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Key UX decisions:**

1. **One-click copy** — The "Copy" button copies the entire snippet content
   to clipboard. Visual confirmation (checkmark) for 2 seconds.

2. **Raw endpoint visible** — Show the `curl` command to fetch the snippet
   as plain text. Developers love this.

3. **Embed code** — Generate an embeddable `<script>` or `<iframe>` tag for
   blog posts, documentation, READMEs. Like GitHub Gist embeds but faster.

4. **Share stats** — When a snippet is shared, show access count and last
   access time.

### Shared Snippet View (Public)

The public view for shared/public snippets works without authentication
and is optimized for first impressions — this is how new users discover Stash.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Stash                                            Sign up - it's free    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Kubernetes deployment script                                            │
│  by @alice - bash - 2 hours ago                                          │
│                                                                          │
│  ┌────────────────────────────────────────────────────────── Copy ──┐    │
│  │  1  #!/bin/bash                                                  │    │
│  │  2  set -euo pipefail                                            │    │
│  │  3                                                               │    │
│  │  4  NAMESPACE="${1:-default}"                                    │    │
│  │  5  kubectl apply -f deployment.yaml -n "$NAMESPACE"             │    │
│  │  6  kubectl rollout status deployment/app -n "$NAMESPACE"        │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  Deploys the app to a Kubernetes namespace. Pass namespace as first      │
│  argument, defaults to `default`.                                        │
│                                                                          │
│  [ bash ] [ k8s ] [ deploy ]                                             │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  Save snippets like this one to your own Stash.                  │    │
│  │  Sign up with GitHub ->                                          │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Key UX decisions:**

1. **No login wall** — The snippet is fully visible and copyable without
   authentication. Every shared snippet is a marketing surface.

2. **Subtle CTA** — A non-intrusive call-to-action at the bottom. Not a
   modal, not a banner. Just a clean card.

3. **Author attribution** — Shows author's GitHub username and avatar.

4. **SEO-optimized** — Server-rendered, proper meta tags, Open Graph preview
   with code snippet image. Shared links look great in Slack/Twitter/Discord.

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Cmd+K` | Open command palette / search |
| `Cmd+N` | New snippet |
| `Cmd+Shift+N` | New project |
| `Cmd+S` | Save current snippet |
| `Cmd+Enter` | Save and close |
| `Cmd+C` (in view) | Copy snippet content |
| `Cmd+Shift+C` | Copy share link |
| `Cmd+E` | Edit current snippet |
| `Cmd+D` | Duplicate snippet |
| `Cmd+Backspace` | Delete snippet (with confirmation) |
| `Esc` | Go back / close modal |
| `j/k` | Navigate list (vim-style) |
| `Enter` | Open selected snippet |
| `/` | Focus search (when not in editor) |
| `?` | Show keyboard shortcuts |
| `Cmd+Shift+L` | Toggle sidebar |
| `1-9` | Quick-switch to pinned snippets |

### Onboarding Flow

First-time user experience determines conversion. The onboarding must be
**fast, useful, and leave the user with real content** in their Stash.

```
Step 1: Sign in with GitHub (one click)
         |
Step 2: "Welcome to Stash" — 5-second product tour (skippable)
         |
Step 3: Choose one:
         a) "Create your first snippet" (pre-filled editor with example)
         b) "Import from GitHub Gist" (OAuth scope already granted)
         c) "Start empty" (power users who want to explore)
         |
Step 4: Dashboard with content (either created or imported)
         Floating tooltip: "Press Cmd+K to search anytime"
```

**Anti-patterns to avoid:**

- No empty dashboard (import or create something during onboarding)
- No multi-page wizard (max 2 clicks to the dashboard)
- No email verification step (GitHub OAuth is sufficient)
- No feature announcements overlay on first load

### Embeddable Snippets

Like GitHub Gist embeds but faster and more customizable.

```html
<!-- Embed a snippet -->
<script src="https://stash.dev/embed/a1b2c3d4.js"></script>

<!-- Or as iframe -->
<iframe src="https://stash.dev/embed/a1b2c3d4"
        width="100%" height="300"
        style="border: 1px solid #e5e7eb; border-radius: 8px;">
</iframe>
```

The embed endpoint renders a self-contained HTML snippet with syntax
highlighting (server-rendered, no JS dependency), a copy button, a "View on
Stash" link, responsive width, and dark/light theme (respects
`prefers-color-scheme`).

Every embedded snippet is a growth mechanism — a Stash advertisement.

### Design System

**Color palette (dark mode default):**

```
Primary:     #6366F1 (Indigo-500) — actions, links, active states
Background:  #0F172A (Slate-900) — dark mode default
Surface:     #1E293B (Slate-800) — cards, panels
Text:        #F8FAFC (Slate-50)  — primary text
Muted:       #94A3B8 (Slate-400) — secondary text, metadata
Border:      #334155 (Slate-700) — subtle borders
Success:     #10B981 (Emerald-500) — save confirmation, copy feedback
Danger:      #EF4444 (Red-500) — delete, destructive actions
Warning:     #F59E0B (Amber-500) — warnings, expiring shares
```

**Typography:**

- Headings/Body: Inter (system font stack fallback)
- Code: JetBrains Mono (with ligatures)
- Monospace fallback: `ui-monospace, SFMono-Regular, Menlo, Monaco`

**Motion:**

- All transitions: 150ms ease-out
- No page transitions or loading spinners for < 200ms operations
- Skeleton loaders for content that takes > 200ms
- Copy confirmation: checkmark icon, 2-second display, green flash

---

## Data Model

Core entities with extensions for collections, links, and versioning. Supabase
Auth owns `auth.users`; app-specific user data lives in `public.profiles`.

### Profiles

Synced from `auth.users` via a database trigger on signup/login. Replaces the
custom `users` table.

| Column       | Type                       | Notes                          |
|--------------|----------------------------|--------------------------------|
| `id`         | UUID (PK, FK → auth.users) | Same ID as Supabase Auth user  |
| `github_id`  | BIGINT                     | Unique, from GitHub            |
| `username`   | VARCHAR                    | GitHub username                |
| `avatar_url` | VARCHAR                    | GitHub avatar                  |
| `created_at` | TIMESTAMPTZ                | Account creation               |
| `updated_at` | TIMESTAMPTZ                | Last login sync                |

A trigger function `handle_new_user()` fires `AFTER INSERT ON auth.users` and
extracts `github_id`, `username`, and `avatar_url` from
`raw_user_meta_data`. A companion `handle_user_update()` trigger syncs
changes on subsequent logins.

### Projects

Optional grouping of snippets by theme.

| Column        | Type                     | Notes                           |
|---------------|--------------------------|---------------------------------|
| `id`          | UUID (PK)                | `gen_random_uuid()`             |
| `owner_id`    | UUID (FK → auth.users)   | Project owner                   |
| `name`        | VARCHAR                  | Project name                    |
| `description` | TEXT                     | Optional description            |
| `visibility`  | ENUM                     | `private`, `public`             |
| `created_at`  | TIMESTAMPTZ              | Creation timestamp              |
| `updated_at`  | TIMESTAMPTZ              | Last modification               |

### Snippets

The core entity. Comes in two types: `single` (one code block) and `repository`
(multi-file directory tree).

| Column          | Type                  | Notes                                          |
|-----------------|-----------------------|------------------------------------------------|
| `id`            | UUID (PK)             | `gen_random_uuid()`                            |
| `owner_id`      | UUID (FK → auth.users)| Snippet owner                                  |
| `project_id`    | UUID (FK → projects)  | Optional, nullable                             |
| `type`          | ENUM                  | `single`, `repository` — default `single`      |
| `title`         | VARCHAR               | Snippet title                                  |
| `description`   | TEXT                  | Markdown-rendered description                  |
| `language`      | VARCHAR               | Programming language (primary, for repo type)  |
| `content`       | TEXT                  | The actual code (NULL for `repository` type)   |
| `labels`        | JSONB                 | Array of strings, GIN-indexed                  |
| `visibility`    | ENUM                  | `private`, `public`, `unlisted`                |
| `source`        | VARCHAR               | Origin: `manual`, `claude`, `imported`         |
| `version`       | INT                   | Current version number, starts at 1            |
| `share_token`   | UUID                  | Nullable, generated on share                   |
| `search_vector` | TSVECTOR (generated)  | Weighted FTS: title(A), description(B), content(C) |
| `created_at`    | TIMESTAMPTZ           | Creation timestamp                             |
| `updated_at`    | TIMESTAMPTZ           | Last modification                              |

### Snippet Files

Files belonging to a repository snippet. Only used when `snippets.type = 'repository'`.

| Column        | Type                     | Notes                                      |
|---------------|--------------------------|--------------------------------------------|
| `id`          | UUID (PK)                | `gen_random_uuid()`                        |
| `snippet_id`  | UUID (FK → snippets)     | Parent repository snippet                  |
| `file_path`   | VARCHAR                  | Relative path (e.g. `src/index.ts`)        |
| `content`     | TEXT                     | File content                               |
| `language`    | VARCHAR                  | Auto-detected per file                     |
| `size_bytes`  | INT                      | File size for limit enforcement            |
| `ordinal`     | INT                      | Display order in file tree                 |
| `created_at`  | TIMESTAMPTZ              | Creation timestamp                         |
| `updated_at`  | TIMESTAMPTZ              | Last modification                          |

**Constraints:** `UNIQUE(snippet_id, file_path)`. Max 50 files per snippet, max 1 MB
total content (configurable per tier). `ON DELETE CASCADE` from parent snippet.

For repository snippets, the `search_vector` on the parent snippet is maintained
via a trigger that concatenates all file contents from `snippet_files` plus the
title and description (see [Indexes](#indexes)).

### Indexes

```sql
-- Full-text search (weighted: title > description > content)
-- Not GENERATED ALWAYS AS because repository snippets need content from
-- snippet_files (cross-table), which generated columns cannot reference.
ALTER TABLE snippets ADD COLUMN search_vector tsvector;

CREATE INDEX idx_snippets_search ON snippets USING GIN(search_vector);

-- Trigger: maintain search_vector for both snippet types
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  IF NEW.type = 'single' THEN
    NEW.search_vector :=
      setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
      setweight(to_tsvector('english', coalesce(NEW.description, '')), 'B') ||
      setweight(to_tsvector('english', coalesce(NEW.content, '')), 'C');
  ELSE
    -- Repository: concatenate all file contents for weight C
    NEW.search_vector :=
      setweight(to_tsvector('english', coalesce(NEW.title, '')), 'A') ||
      setweight(to_tsvector('english', coalesce(NEW.description, '')), 'B') ||
      setweight(to_tsvector('english', coalesce(
        (SELECT string_agg(content, ' ') FROM snippet_files
         WHERE snippet_id = NEW.id), ''
      )), 'C');
  END IF;
  RETURN NEW;
END;
$$;

CREATE TRIGGER trg_snippets_search_vector
  BEFORE INSERT OR UPDATE ON snippets
  FOR EACH ROW EXECUTE FUNCTION update_search_vector();

-- Re-index parent snippet when snippet_files change
CREATE OR REPLACE FUNCTION reindex_repo_search_vector()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
  UPDATE snippets SET updated_at = now()
    WHERE id = coalesce(NEW.snippet_id, OLD.snippet_id);
  RETURN coalesce(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_snippet_files_reindex
  AFTER INSERT OR UPDATE OR DELETE ON snippet_files
  FOR EACH ROW EXECUTE FUNCTION reindex_repo_search_vector();

-- Label filtering (supports @> containment queries)
CREATE INDEX idx_snippets_labels ON snippets USING GIN(labels);

-- Lookup indexes
CREATE INDEX idx_snippets_owner ON snippets(owner_id);
CREATE INDEX idx_snippets_project ON snippets(project_id);
CREATE INDEX idx_snippets_share_token ON snippets(share_token)
  WHERE share_token IS NOT NULL;
CREATE INDEX idx_snippets_type ON snippets(type);
CREATE INDEX idx_projects_owner ON projects(owner_id);

-- Snippet files (repository snippets)
CREATE INDEX idx_snippet_files_snippet ON snippet_files(snippet_id);

-- Collections
CREATE INDEX idx_collections_owner ON collections(owner_id);
CREATE INDEX idx_collections_project ON collections(project_id);
CREATE INDEX idx_collections_labels ON collections USING GIN(labels);
CREATE INDEX idx_collection_items_collection ON collection_items(collection_id);
CREATE INDEX idx_collection_items_snippet ON collection_items(snippet_id);

-- Snippet links
CREATE INDEX idx_snippet_links_source ON snippet_links(source_id);
CREATE INDEX idx_snippet_links_target ON snippet_links(target_id);

-- Snippet versions
CREATE INDEX idx_snippet_versions_snippet ON snippet_versions(snippet_id, version_number DESC);
```

### Entity Relationships

```
auth.users 1──────1 profiles
    |
    |--1────────N projects
    |                |
    |                +--1────────N collections
    |
    +--1────────N snippets N──────1 projects (optional)
                    |
                    |--1────────N snippet_files (if type = 'repository')
                    |
                    |--1────────N snippet_versions
                    |                |
                    |                +--1────N snippet_file_versions (if repo)
                    |
                    |--N────────N collections (via collection_items)
                    |
                    +--N────────N snippets (via snippet_links, bidirectional)
```

A snippet can exist without a project. Deleting a project detaches its snippets
(sets `project_id = NULL`, via `ON DELETE SET NULL`) but does not delete them.

### Collections

Curated groups of snippets — like playlists for code.

| Column        | Type                     | Notes                                      |
|---------------|--------------------------|--------------------------------------------|
| `id`          | UUID (PK)                | `gen_random_uuid()`                        |
| `owner_id`    | UUID (FK → auth.users)   | Collection creator                         |
| `project_id`  | UUID (FK → projects)     | Optional, nullable — scoped to a project   |
| `title`       | VARCHAR                  | Collection name                            |
| `description` | TEXT                     | Markdown description                       |
| `labels`      | JSONB                    | Array of strings, GIN-indexed              |
| `visibility`  | ENUM                     | `private`, `public`, `unlisted`            |
| `share_token` | UUID                     | Nullable, for sharing the whole collection |
| `created_at`  | TIMESTAMPTZ              | Creation timestamp                         |
| `updated_at`  | TIMESTAMPTZ              | Last modification                          |

### Collection Items

Junction table linking snippets to collections (many-to-many).

| Column          | Type                     | Notes                                    |
|-----------------|--------------------------|------------------------------------------|
| `id`            | UUID (PK)                | `gen_random_uuid()`                      |
| `collection_id` | UUID (FK → collections) | Parent collection                         |
| `snippet_id`    | UUID (FK → snippets)    | Included snippet                          |
| `ordinal`       | INT                      | Display order within the collection      |
| `note`          | TEXT                     | Optional note explaining why included    |
| `added_by`      | UUID (FK → auth.users)  | Who added this item                       |
| `added_at`      | TIMESTAMPTZ              | When added                               |

**Constraints:** `UNIQUE(collection_id, snippet_id)`. Deleting a snippet removes
it from all collections (`ON DELETE CASCADE`). Deleting a collection removes all
items but not the snippets themselves.

### Snippet Links

Directional references between snippets.

| Column          | Type                     | Notes                                    |
|-----------------|--------------------------|------------------------------------------|
| `id`            | UUID (PK)                | `gen_random_uuid()`                      |
| `source_id`     | UUID (FK → snippets)    | Snippet that links                        |
| `target_id`     | UUID (FK → snippets, nullable) | Snippet being linked to (NULL = deleted target) |
| `link_type`     | ENUM                     | `related`, `depends_on`, `supersedes`, `derived_from`, `see_also` |
| `note`          | VARCHAR(280)             | Optional explanation                     |
| `created_by`    | UUID (FK → auth.users)  | Who created the link                      |
| `created_at`    | TIMESTAMPTZ              | Creation timestamp                       |

**Constraints:** `UNIQUE(source_id, target_id, link_type)`. A snippet cannot link
to itself. Links are directional but the UI displays both directions
(outgoing: "links to", incoming: "referenced by"). When a target snippet is
deleted, the link becomes a dead reference (`ON DELETE SET NULL` on `target_id`
with nullable FK).

### Snippet Versions

Full version history for every snippet. Each save creates a new version.

| Column           | Type                     | Notes                                   |
|------------------|--------------------------|-----------------------------------------|
| `id`             | UUID (PK)                | `gen_random_uuid()`                     |
| `snippet_id`     | UUID (FK → snippets)     | Parent snippet                          |
| `version_number` | INT                      | Auto-incremented per snippet            |
| `content`        | TEXT                     | Full snapshot (NULL for repository type) |
| `title`          | VARCHAR                  | Snapshot of title at this version       |
| `description`    | TEXT                     | Snapshot of description                 |
| `language`       | VARCHAR                  | Snapshot of language                    |
| `labels`         | JSONB                    | Snapshot of labels                      |
| `commit_message` | TEXT                     | User-provided or auto-generated         |
| `content_hash`   | VARCHAR                  | SHA-256 of content (integrity)          |
| `changed_by`     | UUID (FK → auth.users)   | Who made this change                    |
| `created_at`     | TIMESTAMPTZ              | Version creation timestamp              |

**Constraints:** `UNIQUE(snippet_id, version_number)`. `ON DELETE CASCADE` from
parent snippet. Version 1 is the initial creation; subsequent saves increment.

### Snippet File Versions

Per-file state at each version — only for repository snippets.

| Column           | Type                      | Notes                                  |
|------------------|---------------------------|----------------------------------------|
| `id`             | UUID (PK)                 | `gen_random_uuid()`                    |
| `version_id`     | UUID (FK → snippet_versions) | Parent version                      |
| `file_path`      | VARCHAR                   | Relative path in the repo              |
| `content`        | TEXT                      | File content (NULL if deleted)         |
| `language`       | VARCHAR                   | Language at this version               |
| `content_hash`   | VARCHAR                   | SHA-256 of file content                |
| `change_type`    | ENUM                      | `added`, `modified`, `deleted`         |

**Constraints:** `UNIQUE(version_id, file_path)`. Only files that **changed** in
this version are stored. To reconstruct the full file tree at version N, walk
back through versions collecting the latest state of each file path. This avoids
duplicating unchanged files across versions.

---

## API Layer

There is no custom API server. The frontend communicates with Supabase via three
mechanisms:

1. **PostgREST (auto-generated REST API)** — CRUD operations on all tables,
   automatically secured by RLS policies.
2. **RPC Functions (PostgreSQL functions)** — Complex queries (full-text search
   with ranking, label aggregation, share-token operations) that PostgREST cannot
   express.
3. **Edge Functions (Deno)** — The single case where a non-JSON HTTP response is
   needed (raw snippet export as `text/plain`).

### PostgREST Endpoints (auto-generated)

All CRUD operations are handled by PostgREST via `@supabase/supabase-js`. No
custom routes needed.

| Operation                | supabase-js Call                                                   |
|--------------------------|--------------------------------------------------------------------|
| Create snippet           | `supabase.from('snippets').insert({...})`                          |
| Get snippet              | `supabase.from('snippets').select().eq('id', id).single()`         |
| Update snippet           | `supabase.from('snippets').update({...}).eq('id', id)`             |
| Delete snippet           | `supabase.from('snippets').delete().eq('id', id)`                  |
| Create project           | `supabase.from('projects').insert({...})`                          |
| List projects            | `supabase.from('projects').select()`                               |
| Get project + snippets   | `supabase.from('projects').select('*, snippets(*)').eq('id', id)`  |
| Update project           | `supabase.from('projects').update({...}).eq('id', id)`             |
| Delete project           | `supabase.from('projects').delete().eq('id', id)`                  |
| Get current user profile | `supabase.from('profiles').select().eq('id', user.id).single()`    |

RLS policies ensure every query is automatically scoped to the authenticated
user's data. No manual `WHERE owner_id = ...` needed in application code.

### RPC Functions

Complex queries that require ranking, aggregation, or elevated privileges.

#### `search_snippets`

Replaces `GET /snippets` with full-text search, filters, sorting, and pagination.

```sql
search_snippets(
  search_query   TEXT    DEFAULT NULL,
  filter_labels  JSONB   DEFAULT NULL,   -- e.g. '["bash","k8s"]'
  filter_project UUID    DEFAULT NULL,
  filter_lang    TEXT    DEFAULT NULL,
  filter_type    TEXT    DEFAULT NULL,    -- 'single', 'repository'
  sort_by        TEXT    DEFAULT 'updated',  -- 'updated', 'created', 'title'
  page_limit     INT     DEFAULT 20,
  page_offset    INT     DEFAULT 0
) RETURNS TABLE (...)
```

- `SECURITY INVOKER` — respects RLS, returns only the caller's snippets.
- When `search_query` is provided, orders by `ts_rank(search_vector, query)`.
- Applies `labels @> filter_labels` when labels are provided.
- Applies `type = filter_type` when type is provided (`single` or `repository`).
- Called via `supabase.rpc('search_snippets', { ... })`.

#### `get_shared_snippet`

Replaces `GET /shared/{token}`. Allows unauthenticated access to shared snippets.

```sql
get_shared_snippet(token UUID) RETURNS SETOF snippets
```

- `SECURITY DEFINER` — bypasses RLS since the caller may not be authenticated.
- Looks up `WHERE share_token = token`.
- Returns zero or one row, joined with profile info (username, avatar_url).

#### `generate_share_token`

Replaces `POST /snippets/{id}/share`.

```sql
generate_share_token(snippet_id UUID) RETURNS UUID
```

- `SECURITY DEFINER` — internally verifies `owner_id = auth.uid()`.
- Generates `gen_random_uuid()`, writes to `snippets.share_token`.
- Idempotent: returns existing token if one exists.

#### `revoke_share_token`

```sql
revoke_share_token(snippet_id UUID) RETURNS VOID
```

- `SECURITY DEFINER` — internally verifies ownership.
- Sets `share_token = NULL`.

#### `get_user_labels`

Replaces `GET /labels`. Returns labels with usage frequency for autocomplete.

```sql
get_user_labels() RETURNS TABLE (label TEXT, count BIGINT)
```

- `SECURITY INVOKER` — RLS scopes to the caller's snippets.
- Aggregates via `jsonb_array_elements_text(labels)`, groups by label, orders by
  count descending.

### Edge Function: `raw-snippet`

Replaces `GET /snippets/{id}/raw`. PostgREST only returns JSON; this function
returns `Content-Type: text/plain` for `curl`/scripting use.

- Accepts `GET` with query params: `id` (required), `token` (optional share token).
- If `Authorization` header is present: validates JWT, checks snippet ownership.
- If `token` param is present: validates against `share_token` column.
- Returns snippet `content` as `text/plain; charset=utf-8`.
- Deployed with `verify_jwt = false` (handles auth internally to support both
  authenticated and unauthenticated access).
- **Rate limiting:** 60 requests/minute per IP for unauthenticated access,
  300 requests/minute per user for authenticated access. Enforced via
  Supabase Edge Function rate limiting headers (`X-RateLimit-Limit`,
  `X-RateLimit-Remaining`, `X-RateLimit-Reset`). Returns `429 Too Many
  Requests` when exceeded. Prevents abuse of the unauthenticated endpoint.

### Auth Flow

Authentication is handled entirely by Supabase Auth. No custom auth endpoints.

```
Browser → supabase.auth.signInWithOAuth({ provider: 'github' })
        → 302 Redirect to GitHub OAuth
GitHub  → Callback to Supabase Auth (GoTrue)
GoTrue  → Exchanges code for GitHub access token
        → Fetches user info from GitHub API
        → Creates/updates auth.users
        → Fires handle_new_user() trigger → inserts/updates profiles
        → Returns session (JWT in cookies)
Browser → Next.js /auth/callback route handler
        → supabase.auth.exchangeCodeForSession(code)
        → Redirects to /
```

Next.js middleware refreshes the session on every request via
`supabase.auth.getUser()` and redirects unauthenticated users to `/auth/login`.

---

## Row Level Security (RLS)

Authorization lives in PostgreSQL, not in application code. All tables have
`ENABLE ROW LEVEL SECURITY` and `FORCE ROW LEVEL SECURITY`.

### profiles

| Policy                  | Operation | Check                                    |
|-------------------------|-----------|------------------------------------------|
| `profiles_select_all`   | SELECT    | `true` (public — needed for author info on shared snippets) |
| `profiles_update_own`   | UPDATE    | `auth.uid() = id`                        |
| `profiles_insert_trigger`| INSERT   | `false` (only the auth trigger inserts)  |

### projects

| Policy               | Operation | Check                    |
|----------------------|-----------|--------------------------|
| `projects_select_own`| SELECT    | `owner_id = auth.uid()`  |
| `projects_insert_own`| INSERT    | `auth.uid() = owner_id`  |
| `projects_update_own`| UPDATE    | `owner_id = auth.uid()`  |
| `projects_delete_own`| DELETE    | `owner_id = auth.uid()`  |

### snippets

| Policy                 | Operation | Check                    |
|------------------------|-----------|--------------------------|
| `snippets_select_own`  | SELECT    | `owner_id = auth.uid()`  |
| `snippets_select_public`| SELECT   | `visibility = 'public'`  |
| `snippets_insert_own`  | INSERT    | `auth.uid() = owner_id`  |
| `snippets_update_own`  | UPDATE    | `owner_id = auth.uid()`  |
| `snippets_delete_own`  | DELETE    | `owner_id = auth.uid()`  |

### collections

| Policy                    | Operation | Check                    |
|---------------------------|-----------|--------------------------|
| `collections_select_own`  | SELECT    | `owner_id = auth.uid()`  |
| `collections_select_public`| SELECT  | `visibility = 'public'`  |
| `collections_insert_own`  | INSERT    | `auth.uid() = owner_id`  |
| `collections_update_own`  | UPDATE    | `owner_id = auth.uid()`  |
| `collections_delete_own`  | DELETE    | `owner_id = auth.uid()`  |

### collection_items

| Policy                      | Operation | Check                                        |
|-----------------------------|-----------|----------------------------------------------|
| `coll_items_select`         | SELECT    | Collection owner or public collection        |
| `coll_items_insert_own`     | INSERT    | `auth.uid()` owns the parent collection      |
| `coll_items_delete_own`     | DELETE    | `auth.uid()` owns the parent collection      |

### snippet_links

| Policy                    | Operation | Check                                          |
|---------------------------|-----------|------------------------------------------------|
| `links_select`            | SELECT    | Source or target snippet is viewable by user   |
| `links_insert_own`        | INSERT    | `auth.uid()` owns the source snippet           |
| `links_delete_own`        | DELETE    | `auth.uid() = created_by`                      |

### snippet_versions

| Policy                     | Operation | Check                    |
|----------------------------|-----------|--------------------------|
| `versions_select_own`      | SELECT    | Parent snippet's `owner_id = auth.uid()` |
| `versions_insert_trigger`  | INSERT    | System only (via trigger on snippet update) |

Share-token access is handled via `get_shared_snippet` / `get_shared_collection`
(`SECURITY DEFINER`), not via RLS. This is the standard Supabase pattern for
unauthenticated access to specific rows.

---

## Frontend

The frontend uses Next.js 15+ with App Router, Tailwind CSS for styling, and
CodeMirror 6 for the code editor. It communicates directly with Supabase via
`@supabase/ssr`.

### Supabase Client Setup

Four client utilities in `src/lib/supabase/`:

| Utility          | Used In                      | Purpose                                |
|------------------|------------------------------|----------------------------------------|
| `browser.ts`     | Client Components            | Singleton, manages cookies via `document.cookie` |
| `server.ts`      | Server Components, Actions   | Read-only cookie access (`getAll`)     |
| `middleware.ts`   | Next.js middleware           | Read/write cookies, token refresh      |
| `admin.ts`       | Server-side only             | Service role key, bypasses RLS (shared snippet rendering) |

### Page Structure

| Route                         | Component            | Data Source                                      |
|-------------------------------|----------------------|--------------------------------------------------|
| `/`                           | Dashboard            | `supabase.rpc('search_snippets', {...})`         |
| `/snippets/[id]`             | Snippet View         | `supabase.from('snippets').select().eq('id', id)` |
| `/snippets/new`              | Snippet Editor       | `supabase.from('snippets').insert({...})`        |
| `/snippets/[id]/edit`        | Snippet Editor       | `supabase.from('snippets').update({...})`        |
| `/snippets/[id]/versions`    | Version History      | `supabase.from('snippet_versions').select()`     |
| `/snippets/[id]/diff/[v1]/[v2]` | Diff View        | RPC: compare two versions                        |
| `/projects/[id]`             | Project View         | `supabase.from('projects').select('*, snippets(*)')` |
| `/collections/[id]`          | Collection View      | `supabase.from('collections').select('*, collection_items(*, snippets(*))')` |
| `/collections/new`           | Collection Editor    | `supabase.from('collections').insert({...})`     |
| `/shared/[token]`            | Public Share         | RPC: `get_shared_snippet` or `get_shared_collection` |
| `/auth/login`                | Login                | `supabase.auth.signInWithOAuth({ provider: 'github' })` |
| `/auth/callback`             | Route Handler        | `supabase.auth.exchangeCodeForSession(code)`     |

### Environment Variables

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=http://localhost:54321
NEXT_PUBLIC_SUPABASE_ANON_KEY=<from supabase status>
SUPABASE_SERVICE_ROLE_KEY=<from supabase status>  # server-side only
```

---

## Search Strategy

Full-text search uses PostgreSQL's built-in `tsvector` with weighted ranking:

- **Weight A** (highest): `title` — title matches rank highest
- **Weight B**: `description` — description matches rank medium
- **Weight C** (lowest): `content` — code content matches rank lowest

The search vector is a generated stored column, automatically updated on
insert/update. A GIN index on the vector enables fast full-text queries.

Label filtering uses JSONB containment (`labels @> '["bash"]'::jsonb`) with a
separate GIN index. Both filters are combinable in a single query.

The `search_snippets` RPC function combines FTS ranking, label filtering,
project/language filters, and pagination in a single PostgreSQL function.
PostgREST cannot express `ts_rank` ordering, which is why this is an RPC
function rather than a direct table query.

---

## Sharing Model

Three visibility levels:

- **Private** — only the owner can see the snippet (RLS: `owner_id = auth.uid()`)
- **Unlisted** — anyone with the share link can view it; does not appear in
  public listings
- **Public** — visible to anyone (RLS: `visibility = 'public'`)

Share links are UUID-based and independent of visibility. A private snippet can
be temporarily shared via a share token without changing its visibility setting.
The token can be revoked at any time via `revoke_share_token`.

Shared snippet access uses `get_shared_snippet` (`SECURITY DEFINER`) which
bypasses RLS entirely, ensuring unauthenticated users can view shared snippets
without needing a Supabase session.

---

## Snippet Types

Stash supports two snippet types, selectable at creation time.

### Single-File Snippets

The default. A single code block with title, description, language, and content.
This is what most snippets are — a bash one-liner, a React hook, a SQL query.
The `content` column on the `snippets` table holds the code.

### Repository Snippets

A repository snippet is a **collection of files and directories** — like a small
Git repository. Use cases:

- A Docker setup (`Dockerfile` + `docker-compose.yml` + `.env.example`)
- A React component with its test, styles, and story file
- A Terraform module (`main.tf` + `variables.tf` + `outputs.tf`)
- A project boilerplate/template
- A multi-file configuration (nginx + systemd unit + logrotate)

Repository snippets have:

- A file tree (files and nested directories)
- A root-level title, description, labels, and visibility (same as single-file)
- Per-file language detection
- A combined `search_vector` spanning all file contents

**Storage:** Repository snippets use the `snippet_files` table. Each file is a
row with its relative path, content, and language. The snippet's `content` column
is `NULL` for repository type — all content lives in `snippet_files`.

```
snippets (type = 'repository')
  └── snippet_files
      ├── Dockerfile              (language: dockerfile)
      ├── docker-compose.yml      (language: yaml)
      ├── src/
      │   ├── index.ts            (language: typescript)
      │   └── config.ts           (language: typescript)
      └── .env.example            (language: dotenv)
```

**Limits:** Repository snippets are limited to 50 files and 1 MB total content
(configurable per tier). Directories are implicit — derived from file paths
(like Git, no empty directories).

### Editor UX for Repository Snippets

The editor switches to a file-tree + tabbed layout when the type is `repository`:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  < Back                 Edit Repository Snippet              Save  Cmd+S │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Title    ┌─────────────────────────────────────────────────────────┐    │
│           │ Docker + Node.js Starter                                │    │
│           └─────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────┬─────────────────────────────────────────────────┐    │
│  │ FILES          │  docker-compose.yml                   yaml      │    │
│  │                │                                                 │    │
│  │ Dockerfile     │  1  version: "3.8"                              │    │
│  │ docker-compose │  2  services:                                   │    │
│  │ .env.example   │  3    app:                                      │    │
│  │ ▸ src/         │  4      build: .                                │    │
│  │   index.ts     │  5      ports:                                  │    │
│  │   config.ts    │  6        - "3000:3000"                         │    │
│  │                │  7      volumes:                                │    │
│  │ [+ Add File]   │  8        - ./src:/app/src                      │    │
│  │ [+ Add Folder] │                                     CodeMirror 6│    │
│  └────────────────┴─────────────────────────────────────────────────┘    │
│                                                                          │
│  Labels   ┌──────────────────────────────────────────┐                   │
│           │ docker x | nodejs x | starter x          │                   │
│           └──────────────────────────────────────────┘                   │
│                                                                          │
│  Project  [ templates        ]  Visibility  [ Private    ]               │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**View** shows a file tree on the left, file content on the right. Users can
browse files, copy individual files or download the entire repository snippet
as a ZIP.

---

## Collections

Collections are **curated groups of snippets** — like playlists for code. They
provide a layer of organization above and across projects.

### Concept

- A collection belongs to a **project** (optional) or stands alone
- A collection has its own **title**, **description**, and **labels**
- Snippets can belong to **multiple collections** (many-to-many)
- Collections can contain both single-file and repository snippets
- Each item in a collection has an **ordinal position** and an optional **note**
  explaining why it's included
- Collections are **shareable as a whole** — one share link for the entire
  collection
- Individual snippets within a shared collection respect their own visibility
  settings

### Use Cases

- "Kubernetes Recipes" — a curated set of k8s deployment, service, and ingress
  snippets
- "React Patterns" — common hooks, HOCs, and context patterns
- "Onboarding" — the 10 snippets every new team member needs
- "Interview Prep" — coding problems with solutions
- "Terraform Modules" — reusable infrastructure building blocks

### Collection Visibility & Sharing

Collections have the same three visibility levels as snippets (`private`,
`unlisted`, `public`). When sharing a collection:

- **Whole collection**: A share link gives access to the collection view with all
  its snippets listed
- **Partial sharing**: The collection can be filtered — only snippets matching
  certain labels or languages are included in the shared view
- **Inherited visibility**: Private snippets in a shared collection are only
  visible to authenticated users with access. Public/unlisted snippets are
  visible to anyone with the collection link.

### Collection UX

```
┌──────────────────────────────────────────────────────────────────────────┐
│  < Back                                     Edit  Share  ...  Delete     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Kubernetes Recipes                                                      │
│  infra / 12 snippets                              Updated 3h ago         │
│  [ k8s ] [ devops ] [ production ]                                       │
│                                                                          │
│  A curated set of production-ready Kubernetes manifests and scripts.     │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  1. deploy-k8s.sh                                     bash       │    │
│  │     Standard deployment script with rollback support             │    │
│  │     #!/bin/bash                                                  │    │
│  │     kubectl apply -f deployment.yaml...                          │    │
│  ├──────────────────────────────────────────────────────────────────┤    │
│  │  2. service.yaml                                      yaml       │    │
│  │     ClusterIP service template                                   │    │
│  │     apiVersion: v1                                               │    │
│  │     kind: Service...                                             │    │
│  ├──────────────────────────────────────────────────────────────────┤    │
│  │  3. hpa.yaml                                          yaml       │    │
│  │     Horizontal Pod Autoscaler with CPU/memory targets            │    │
│  │     apiVersion: autoscaling/v2...                                │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  -- Share ────────────────────────────────────────────────────────────   │
│  https://stash.dev/c/e5f6g7h8          Copy Link    Revoke               │
│  Shared as: Full collection   Filter: [ All labels ] [ All languages ]   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Drag-and-drop reordering** of items within a collection. Items show the
snippet's code preview, language badge, and the curator's note (if any).

### CLI for Collections

```bash
# List collections
stash collections                         # all collections
stash collections --project infra         # scoped to project

# Create a collection
stash collection create "K8s Recipes" --project infra --labels k8s,devops

# Add/remove snippets
stash collection add <collection-id> <snippet-id> --note "Standard deploy"
stash collection remove <collection-id> <snippet-id>

# View collection
stash collection show <collection-id>     # lists all snippets

# Share
stash collection share <collection-id>    # generates share link

# Get all snippets from a collection as files
stash collection export <collection-id> --out ./k8s-recipes/
```

---

## Snippet Links & References

Snippet links are **lightweight, directional connections** between snippets.
Unlike collections (which group snippets into a curated set), links express a
**relationship between exactly two snippets**.

### Link Types

| Type | Meaning | Example |
|------|---------|---------|
| `related` | General relation | A bash script and its Python equivalent |
| `depends_on` | Functional dependency | A deploy script that requires a config snippet |
| `supersedes` | Newer version / replacement | A v2 hook replacing a v1 hook |
| `derived_from` | Fork or adaptation | A snippet adapted from a public one |
| `see_also` | Informational cross-reference | A snippet mentioning a related pattern |

### Behavior

- Links are **directional** but displayed bidirectionally in the UI
  (A links to B → B shows "referenced by A")
- Any snippet owner can create a link **from** their snippet to any snippet they
  can view
- Links include an optional **note** (free text, max 280 chars) to explain the
  relationship
- Links are **not** broken when a snippet is deleted — the link becomes a
  dead reference shown with strikethrough in the UI
- Links appear in the snippet view under a collapsible "Related Snippets" section

### UX

In the snippet view, below the code block:

```
  -- Related Snippets (3) ─────────────────────────────────────────────
  → depends_on   config.env.example        "Required env vars for deploy"
  → related      deploy-docker.sh          "Docker-based alternative"
  ← derived_from useAuth-v2.tsx  @bob       "Adapted from this hook"
```

`→` = outgoing link (this snippet links to another),
`←` = incoming link (another snippet links to this one).

### CLI for Links

```bash
# Create a link
stash link <source-id> <target-id> --type depends_on --note "Needs this config"

# List links for a snippet
stash links <snippet-id>

# Remove a link
stash unlink <source-id> <target-id> --type depends_on
```

---

## Snippet Versioning

Every snippet (single-file and repository) maintains a **full version history**.
Versioning is the backbone for collaborative workflows, the git-like CLI, and
content integrity.

### Concept

- Every save creates a new **version** (auto-incremented version number)
- The current snippet state is always the latest version
- Users can **view any previous version**, **compare versions** (diff), and
  **restore** a previous version (creates a new version with the old content)
- For repository snippets, a version captures the state of **all files** at that
  point in time
- Each version stores a **commit message** (optional, user-provided or
  auto-generated like "Updated content") and the **author** who made the change

### Version Storage

Versions store **full snapshots** of metadata (title, description, labels,
language). For single-file snippets, the `content` column holds the full code.
For repository snippets, only **changed files** are stored in
`snippet_file_versions` — the full tree at version N is reconstructed by walking
back through versions.

This approach keeps storage efficient (no duplication of unchanged files) while
remaining simple to query for any single version.

### Diff Display

Diffs are computed on-demand (not stored). The frontend renders a standard
unified diff format:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Version History — deploy-k8s.sh                                         │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Comparing: v3 (current) ← v2                         Restore v2         │
│  Changed by @alice · 2 hours ago · "Add rollback support"                │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  @@ -4,3 +4,7 @@                                                 │    │
│  │   NAMESPACE="${1:-default}"                                      │    │
│  │   kubectl apply -f deployment.yaml -n "$NAMESPACE"               │    │
│  │  -kubectl rollout status deployment/app -n "$NAMESPACE"          │    │
│  │  +kubectl rollout status deployment/app -n "$NAMESPACE" \        │    │
│  │  +  --timeout=300s                                               │    │
│  │  +                                                               │    │
│  │  +# Rollback on failure                                          │    │
│  │  +if [ $? -ne 0 ]; then                                          │    │
│  │  +  kubectl rollout undo deployment/app -n "$NAMESPACE"          │    │
│  │  +fi                                                             │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  -- All Versions ────────────────────────────────────────────────        │
│  v3  @alice  2h ago   "Add rollback support"            ● current        │
│  v2  @alice  1d ago   "Add namespace parameter"                          │
│  v1  @alice  3d ago   "Initial version"                                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

For repository snippets, the diff view additionally shows a per-file change
summary with a file navigator:

```
  Changed Files (3)
  ├── M  Dockerfile              +2 -1
  ├── M  src/index.ts            +15 -3
  └── A  src/health.ts           +22

  (click a file to see its diff)
```

### Restore

Restoring a previous version creates a **new version** (not a revert). For
example, restoring v2 when the current version is v4 creates v5 with the
content of v2. The commit message is auto-generated:
`"Restored from v2"`. The full history is preserved.

### Version Pruning

To prevent unbounded growth, old versions can be pruned:

- Free tier: keep last 10 versions per snippet
- Pro tier: keep last 50 versions
- Team/Enterprise: configurable retention policy

Pruning deletes intermediate versions but always keeps v1 (initial) and the
current version.

---

## Git-Like CLI Workflow

The CLI enables a **git-like workflow** for working with snippets locally. This
works for both single-file and repository snippets, with full conflict detection
and version tracking.

### Local Tracking (`.stash/`)

When a snippet is checked out locally, Stash creates a `.stash/` metadata
directory (analogous to `.git/`) to track state:

```
.stash/
├── config.json        # snippet ID, remote URL, type, instance
├── HEAD               # current local version number
├── tracking.json      # per-file content hashes for change detection
└── remote.json        # cached remote version for conflict detection
```

For single-file snippets, `.stash/` sits alongside the file. For repository
snippets, `.stash/` sits at the root of the directory tree.

### Workflow: Single-File Snippets

```bash
# Check out a snippet to work on locally
stash checkout <id>                   # saves file + creates .stash/
stash checkout <id> -o deploy.sh      # explicit filename

# Edit locally with any editor...
vim deploy.sh

# See what changed
stash diff                            # local changes vs. last known version

# Push changes back (creates a new version)
stash push -m "Added error handling"
# → fetches current remote version
# → if remote unchanged since checkout: pushes new version
# → if remote changed: CONFLICT — see below

# Update local copy with remote changes
stash pull
# → if no local changes: updates file and .stash/HEAD
# → if local changes exist: attempts merge, shows conflicts if any
```

### Workflow: Repository Snippets

```bash
# Clone a repository snippet
stash clone <id>                       # creates directory with .stash/
stash clone <id> ./my-docker-setup     # explicit directory name

# Work as usual...
vim Dockerfile
vim src/index.ts
touch src/health.ts                    # new file

# Check status (like git status)
stash status
#   modified:  Dockerfile
#   modified:  src/index.ts
#   new file:  src/health.ts

# See diffs
stash diff                             # all changes
stash diff Dockerfile                  # single file

# Push all changes
stash push -m "Add health check endpoint"
# → detects added/modified/deleted files via content hashes
# → checks remote for concurrent changes
# → creates new version with file-level change tracking

# Pull remote changes
stash pull
# → fetches latest version
# → merges file changes (no conflict if different files changed)
# → shows conflicts for same-file changes
```

### Conflict Detection & Resolution

Conflicts occur when the remote snippet was modified after the local checkout.
Stash uses a **three-way comparison**: local state, common ancestor (checkout
version), and current remote state.

```
stash push -m "Updated config"

⚠  Remote has changed since your checkout (v3 → v5)

  Conflicting files:
    Dockerfile       — modified both locally and remotely
    src/index.ts     — modified remotely, deleted locally

  Non-conflicting:
    src/health.ts    — added locally (no conflict)
    .env.example     — modified remotely only (auto-merged)

  Options:
    stash pull --merge     # fetch remote changes, attempt auto-merge
    stash diff --remote    # view remote changes before deciding
    stash push --force     # overwrite remote (creates new version, old preserved)
```

For single-file snippets, conflicts show an inline diff:

```
stash pull --merge

<<<<<<< LOCAL (your changes)
NAMESPACE="${1:-default}"
TIMEOUT=300
=======
NAMESPACE="${1:?namespace required}"
>>>>>>> REMOTE (v5 by @bob, "Make namespace required")

Resolve the conflict in deploy.sh, then run:
  stash push -m "Merged: error handling + required namespace"
```

### Version History via CLI

```bash
# View version log
stash log
#  v5  @bob    1h ago   "Make namespace required"
#  v4  @alice  3h ago   "Add timeout parameter"
#  v3  @alice  1d ago   "Add rollback support"
#  v2  @alice  2d ago   "Add namespace parameter"
#  v1  @alice  5d ago   "Initial version"

# View diff between versions
stash diff v3 v5                       # compare any two versions
stash diff v4                          # compare v4 with current

# Restore a previous version
stash restore v3 -m "Reverting to pre-timeout version"
```

---

## Project Structure

```
stash/
├── supabase/
│   ├── config.toml                     # Local dev config (auth, ports, etc.)
│   ├── migrations/
│   │   ├── 00001_create_enums.sql
│   │   ├── 00002_create_profiles.sql
│   │   ├── 00003_create_projects.sql
│   │   ├── 00004_create_snippets.sql
│   │   ├── 00005_create_snippet_files.sql
│   │   ├── 00006_create_collections.sql
│   │   ├── 00007_create_snippet_links.sql
│   │   ├── 00008_create_snippet_versions.sql
│   │   ├── 00009_create_indexes.sql
│   │   ├── 00010_enable_rls.sql
│   │   ├── 00011_create_rpc_functions.sql
│   │   └── 00012_create_auth_trigger.sql
│   ├── seed.sql                        # Test data for local dev
│   ├── functions/
│   │   └── raw-snippet/
│   │       └── index.ts                # Deno edge function
│   └── tests/
│       ├── rls_policies_test.sql       # pgTAP tests
│       └── rpc_functions_test.sql
├── frontend/
│   ├── package.json
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   ├── .env.local.example
│   └── src/
│       ├── lib/supabase/
│       │   ├── browser.ts
│       │   ├── server.ts
│       │   ├── middleware.ts
│       │   └── admin.ts
│       ├── types/
│       │   └── database.ts            # Generated via supabase gen types
│       ├── app/
│       │   ├── layout.tsx
│       │   ├── page.tsx               # Dashboard
│       │   ├── auth/
│       │   │   ├── callback/route.ts
│       │   │   └── login/page.tsx
│       │   ├── snippets/
│       │   │   ├── [id]/page.tsx
│       │   │   ├── [id]/edit/page.tsx
│       │   │   ├── [id]/versions/page.tsx
│       │   │   ├── [id]/diff/[v1]/[v2]/page.tsx
│       │   │   └── new/page.tsx
│       │   ├── collections/
│       │   │   ├── [id]/page.tsx
│       │   │   └── new/page.tsx
│       │   ├── projects/
│       │   │   └── [id]/page.tsx
│       │   └── shared/
│       │       └── [token]/page.tsx
│       ├── components/
│       └── middleware.ts              # Session refresh + auth guard
├── scripts/
│   ├── configure-auth.sh             # Supabase Management API call
│   └── create-instance.sh            # Provision new Supabase project
├── instances/
│   ├── personal.env                   # Non-secret instance config
│   └── team-acme.env
├── .github/
│   └── workflows/
│       ├── ci.yml                     # Lint + typecheck + DB tests
│       └── deploy.yml                 # Deploy per instance via matrix
├── README.md
└── .gitignore
```

---

## Multi-Instance Strategy

The codebase is instance-agnostic. All instance-specific configuration is
injected at deploy time via environment variables. Each instance is a separate
Supabase project with its own database, auth, and edge functions.

### Instance Configuration

Each instance has a non-secret env file in `instances/`:

```bash
# instances/personal.env
INSTANCE_NAME=personal
SUPABASE_PROJECT_REF=abcdefghijklmnop
FRONTEND_URL=https://stash-personal.vercel.app
GITHUB_ENVIRONMENT=production-personal
```

### Secrets

Stored in GitHub Actions **Environments** (one per instance). Each environment
holds:

| Secret                         | Description                          |
|--------------------------------|--------------------------------------|
| `SUPABASE_ACCESS_TOKEN`        | Supabase personal access token       |
| `SUPABASE_DB_PASSWORD`         | Database password                    |
| `SUPABASE_SERVICE_ROLE_KEY`    | For admin client (server-side only)  |
| `SUPABASE_ANON_KEY`            | For browser/server clients           |
| `NEXT_PUBLIC_SUPABASE_URL`     | Supabase project URL                 |
| `GITHUB_OAUTH_CLIENT_ID`       | GitHub OAuth App client ID           |
| `GITHUB_OAUTH_CLIENT_SECRET`   | GitHub OAuth App client secret       |

Each instance requires its own GitHub OAuth App with the callback URL pointing
to `https://<project-ref>.supabase.co/auth/v1/callback`.

### Provisioning a New Instance

`scripts/create-instance.sh` automates:

1. Create Supabase project via `supabase projects create`
2. Write `instances/<name>.env` with the returned project ref
3. Prompt to create a GitHub OAuth App (or automate via GitHub API)
4. Store secrets in the GitHub Actions Environment
5. Trigger the deploy workflow

---

## Infrastructure as Code

Everything is deployable via CI. Zero clicks in the Supabase Dashboard.

### Auth Configuration

GitHub OAuth is configured via the Supabase Management API in
`scripts/configure-auth.sh`:

```bash
curl -X PATCH \
  "https://api.supabase.com/v1/projects/${PROJECT_REF}/config/auth" \
  -H "Authorization: Bearer ${SUPABASE_ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "site_url": "'${FRONTEND_URL}'",
    "external_github_enabled": true,
    "external_github_client_id": "'${GITHUB_OAUTH_CLIENT_ID}'",
    "external_github_secret": "'${GITHUB_OAUTH_CLIENT_SECRET}'"
  }'
```

### Local Development Config

The `supabase/config.toml` configures GitHub OAuth for `supabase start`:

```toml
[project]
id = "stash"

[auth]
site_url = "http://localhost:3000"
additional_redirect_urls = ["http://localhost:3000/auth/callback"]

[auth.external.github]
enabled = true
client_id = "env(GITHUB_CLIENT_ID)"
secret = "env(GITHUB_CLIENT_SECRET)"

[functions.raw-snippet]
verify_jwt = false
```

---

## CI/CD

### `ci.yml` — Pull Request Checks

Triggers on `pull_request` against `main`.

```
Jobs:
  lint-and-typecheck:
    - npm ci (frontend/)
    - ESLint
    - tsc --noEmit

  db-test:
    - supabase start
    - supabase db reset (migrations + seed)
    - supabase test db (pgTAP)
    - supabase stop

  types-check:
    - supabase start
    - supabase gen types --local > /tmp/database.ts
    - diff against committed frontend/src/types/database.ts
    - Fail if diverged (ensures types match migrations)
```

### `deploy.yml` — Production Deploy

Triggers on `push` to `main` or `workflow_dispatch` with instance name.

```
Jobs:
  deploy:
    strategy:
      matrix:
        instance: [personal]   # Add instances here

    environment: production-${{ matrix.instance }}

    steps:
      # Backend
      - supabase link --project-ref $SUPABASE_PROJECT_REF
      - supabase db push
      - supabase functions deploy
      - ./scripts/configure-auth.sh

      # Types
      - supabase gen types --linked > frontend/src/types/database.ts

      # Frontend
      - npm ci && npm run build
      - Deploy to Vercel (with instance-specific env vars)
```

Adding a new instance = add entry to `instances/`, create GitHub Environment,
add to the matrix array.

---

## Local Development

```bash
# Prerequisites: Supabase CLI, Node.js 20+, Docker

# 1. Start Supabase (Postgres, Auth, PostgREST, Edge Functions)
supabase start

# 2. Apply migrations and seed data
supabase db reset

# 3. Start frontend
cd frontend
cp .env.local.example .env.local   # Fill with values from `supabase status`
npm install && npm run dev

# 4. Serve Edge Functions (separate terminal)
supabase functions serve

# 5. Create a new migration
supabase migration new add_some_column
# Edit supabase/migrations/<timestamp>_add_some_column.sql
supabase db reset

# 6. Regenerate TypeScript types after schema changes
supabase gen types --local > frontend/src/types/database.ts

# 7. Run database tests
supabase test db
```

---

## CLI Tool (`stash`)

The CLI is critical for developer adoption. It must be installable in one
command and usable in under 30 seconds.

```bash
# Install
brew install b42labs/tap/stash
# or
npm install -g @b42labs/stash-cli

# Authenticate
stash login   # Opens browser, OAuth flow, stores token locally
```

### Basic Operations

```bash
# Push a new snippet (single-file)
stash push script.sh --title "Deploy script" --labels bash,k8s

# Push from stdin
cat script.sh | stash push --title "Deploy script" --lang bash

# Push a directory as a repository snippet
stash push ./docker-setup/ --title "Docker Starter" --labels docker,nodejs

# Search
stash search "kubernetes deploy"
stash search --label k8s --lang bash

# Get a snippet
stash get <id>           # prints content to stdout (single-file)
stash get <id> --meta    # prints metadata as JSON
stash get <id> > out.sh  # pipe to file

# List
stash list               # recent snippets
stash list --project infra
stash list --type repository   # only repo snippets

# Copy to clipboard
stash copy <id>          # copies content to clipboard

# Share
stash share <id>         # generates share link, prints it

# Open in browser
stash open <id>          # opens snippet in default browser
stash open               # opens dashboard
```

### Git-Like Workflow

The CLI supports a full checkout/edit/push cycle with version tracking and
conflict detection. See [Git-Like CLI Workflow](#git-like-cli-workflow) for
the complete concept.

```bash
# --- Single-file snippets ---
stash checkout <id>                   # saves file locally + .stash/ tracking
stash checkout <id> -o deploy.sh      # explicit filename
# ... edit file ...
stash diff                            # show local changes
stash push -m "Added error handling"  # push new version (with conflict check)
stash pull                            # fetch remote changes

# --- Repository snippets ---
stash clone <id>                      # download directory + .stash/ tracking
stash clone <id> ./my-setup           # explicit directory name
# ... edit files, add/remove files ...
stash status                          # show modified/added/deleted files
stash diff                            # unified diff of all changes
stash diff Dockerfile                 # diff a single file
stash push -m "Add health endpoint"   # push all changes as new version
stash pull                            # fetch & merge remote changes

# --- Version history ---
stash log                             # version history
stash log --diff                      # with inline diffs
stash diff v2 v4                      # compare specific versions
stash restore v3 -m "Revert to v3"   # restore a previous version
```

### Collections & Links

```bash
# Collections
stash collections                                    # list collections
stash collection create "K8s Recipes" --labels k8s   # create
stash collection add <coll-id> <snippet-id>          # add snippet
stash collection show <coll-id>                      # view contents
stash collection share <coll-id>                     # share link
stash collection export <coll-id> --out ./recipes/   # download all

# Links
stash link <from-id> <to-id> --type depends_on       # create link
stash links <snippet-id>                              # list links
stash unlink <from-id> <to-id>                        # remove link
```

**Authentication:** The CLI uses Personal Access Tokens (PATs) stored in the
system keychain (`~/.stash/config.toml` for non-secret config, keychain for
token). OAuth flow opens a browser, the callback writes the token.

---

## Import & Export

**Import sources (V1):**

- GitHub Gist (bulk import via API, preserving labels from filenames)
- File upload (drag-and-drop onto dashboard)
- Paste detection (paste code anywhere -> "Save as snippet?" toast)

**Import sources (V2):**

- VS Code snippets (JSON format)
- JetBrains live templates
- Alfred/Raycast snippets
- Notion code blocks

**Export:**

- Single snippet as file download
- Bulk export as ZIP (grouped by project)
- JSON export (full metadata, for backup/migration)
- GitHub Gist export (push to Gist)

---

## Monetization

| Tier | Price | Target | Features |
|------|-------|--------|----------|
| **Free** | $0 | Individual devs | 100 snippets, 5 projects, 3 share links active, community support |
| **Pro** | $8/mo | Power users | Unlimited snippets/projects/shares, CLI, API keys, priority support, custom domain |
| **Team** | $12/user/mo | Small teams (3-20) | Pro + shared projects, team labels, basic audit log, SSO (V2) |
| **Enterprise** | Custom | Organizations | Team + federation, ReBAC, full audit, compliance exports, SLA |

**Free tier philosophy:** Generous enough to be genuinely useful (100 snippets
covers most individual needs). The limit is on quantity, never on quality — all
features work the same, just with caps.

**Conversion triggers:**

- Hitting the 100-snippet limit (natural growth)
- Wanting to share with a team (collaboration features)
- Needing CLI/API access (developer workflow integration)
- Needing audit/compliance (enterprise requirements)

---

## Scope Boundaries

### What is in V1

- Snippet CRUD with syntax highlighting and Markdown descriptions
- **Two snippet types**: single-file and repository (multi-file)
- Project-based grouping (optional)
- **Collections** for curated snippet groups (with description, labels, sharing)
- **Snippet links** (related, depends_on, supersedes, derived_from, see_also)
- Label tagging with JSONB + GIN indexes
- Full-text search with weighted ranking (via RPC function)
- **Snippet versioning** — version history, diff view, restore
- GitHub OAuth2 authentication (via Supabase Auth)
- Share links with UUID tokens (for snippets and collections)
- Raw content endpoint for scripting (via Edge Function)
- Label autocomplete (via RPC function)
- Multi-instance deployment from single codebase
- Full Infrastructure as Code (CI-deployable, no dashboard clicks)
- RLS-based authorization (no custom middleware)
- Command palette (Cmd+K) with real-time search
- Keyboard shortcuts (vim-style navigation)
- Onboarding flow with Gist import
- Embeddable snippets
- **Git-like CLI** (`stash checkout/clone/push/pull/diff/status/log`)
- Personal Access Tokens for programmatic access

### What is explicitly NOT in V1

- Collaboration / multi-user editing → V2 (ReBAC) + V3 (comments)
- Comments on snippets → V3
- Forking / cloning snippets between users → V3
- Organizations / teams → V2
- AI features (auto-tagging, description generation) → V3
- Editor/IDE integrations → V3
- Live execution / playgrounds → V3

---

## V2: Enterprise & Federation

V2 transforms Stash from a personal tool into an organizational platform.
The four pillars are: **Federation**, **ReBAC**, **Auditing**, and **Trust**.

### V2.1 Federation

#### Problem

Organizations often have multiple teams, each with their own Stash instance
(leveraging V1's multi-instance architecture). But snippets don't exist in
isolation — a DevOps team's Terraform modules are useful to the platform team,
a shared React component library benefits all frontend teams.

Federation allows Stash instances to **discover, reference, and selectively
share snippets** across organizational boundaries without centralizing all data
in a single instance.

#### Architecture

```
┌─────────────────────────┐     Federation      ┌─────────────────────────┐
│   Instance A            │      Protocol       │   Instance B            │
│   (Team Platform)       │<------------------->│   (Team Frontend)       │
│                         │                     │                         │
│  ┌───────────────────┐  │  ┌──────────────┐   │  ┌───────────────────┐  │
│  │ Local Snippets    │  │  │  Federation  │   │  │ Local Snippets    │  │
│  │ (owned data)      │  │  │  Registry    │   │  │ (owned data)      │  │
│  └───────────────────┘  │  │              │   │  └───────────────────┘  │
│  ┌───────────────────┐  │  │  - Instance  │   │  ┌───────────────────┐  │
│  │ Federated Index   │  │  │    directory │   │  │ Federated Index   │  │
│  │ (remote metadata) │  │  │  - Trust     │   │  │ (remote metadata) │  │
│  └───────────────────┘  │  │    anchors   │   │  └───────────────────┘  │
│                         │  │  - Policy    │   │                         │
│                         │  │    sync      │   │                         │
└─────────────────────────┘  └──────────────┘   └─────────────────────────┘
```

#### Federation Protocol

A lightweight REST-based protocol (not ActivityPub — too complex for this use
case). Each instance exposes a federation API:

```
GET  /.well-known/stash-instance    # Instance metadata & public key
POST /federation/connect            # Request federation peering
GET  /federation/search             # Federated search query
GET  /federation/snippets/:id       # Fetch a federated snippet
POST /federation/webhooks           # Event notifications
```

**Instance Discovery:**

```json
// GET /.well-known/stash-instance
{
  "instance_id": "platform-team",
  "display_name": "Platform Team Stash",
  "version": "2.1.0",
  "federation_version": "1.0",
  "public_key": "-----BEGIN PUBLIC KEY-----...",
  "federation_endpoint": "https://stash-platform.acme.com/federation",
  "capabilities": ["search", "fork", "subscribe"],
  "admin_contact": "platform@acme.com"
}
```

**Peering flow:**

1. Instance A sends a peering request to Instance B
2. Instance B's admin reviews and approves/denies
3. On approval, instances exchange signing keys
4. All federation requests are signed with the instance's private key
5. Each instance maintains a list of trusted peers

**Federated Search:**

1. User searches on Instance A
2. Instance A queries its local database
3. Instance A fans out the search to all federated peers
4. Peers return matching snippet metadata (not full content)
5. Instance A merges and ranks results (local results weighted higher)
6. User clicks a federated result -> fetches full content on demand

**Data Sovereignty:**

- Snippet content is never replicated to peer instances
- Only metadata (title, labels, language, author, snippet ID) is indexed
- Full content is fetched on-demand and not cached permanently
- Instance admins control which snippets/projects are federatable
- Federation can be unilateral (A sees B's public snippets, B doesn't
  necessarily see A's)

#### Data Model Extension

```sql
-- Federated instances registry
CREATE TABLE federation_peers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  instance_id VARCHAR NOT NULL UNIQUE,
  display_name VARCHAR NOT NULL,
  endpoint_url VARCHAR NOT NULL,
  public_key TEXT NOT NULL,
  status VARCHAR NOT NULL DEFAULT 'pending',   -- pending, active, suspended, revoked
  trust_level VARCHAR NOT NULL DEFAULT 'basic', -- basic, verified, trusted
  capabilities JSONB DEFAULT '[]',
  last_seen_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  approved_by UUID REFERENCES auth.users(id),
  approved_at TIMESTAMPTZ
);

-- Federated snippet index (metadata only, no content)
CREATE TABLE federated_snippets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  peer_id UUID NOT NULL REFERENCES federation_peers(id) ON DELETE CASCADE,
  remote_snippet_id UUID NOT NULL,
  title VARCHAR NOT NULL,
  language VARCHAR,
  labels JSONB DEFAULT '[]',
  author_name VARCHAR,
  visibility VARCHAR NOT NULL,
  remote_updated_at TIMESTAMPTZ,
  indexed_at TIMESTAMPTZ DEFAULT now(),
  search_vector TSVECTOR GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(labels::text, '')), 'B')
  ) STORED,
  UNIQUE(peer_id, remote_snippet_id)
);

CREATE INDEX idx_federated_search ON federated_snippets USING GIN(search_vector);

-- Federation event log
CREATE TABLE federation_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  peer_id UUID REFERENCES federation_peers(id),
  event_type VARCHAR NOT NULL,  -- peer_connected, search_query, snippet_fetched, sync_completed
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

### V2.2 ReBAC (Relationship-Based Access Control)

#### Problem

V1's authorization model is simple: you own your snippets, period. This breaks
down when teams need:

- Shared project libraries (team members can view/edit, others can't)
- Organization-wide snippet collections (everyone can read, editors can modify)
- Granular sharing (Alice can edit, Bob can only view)
- Hierarchical permissions (org admin -> team admin -> team member)

#### Authorization Model

Inspired by Google Zanzibar, permissions are expressed as **relationships**
between subjects and objects.

**Entities:**

```
User          — an authenticated person
Team          — a group of users
Organization  — a group of teams
Snippet       — a code snippet
Project       — a group of snippets
Collection    — a curated set of snippets (cross-project)
```

**Relations:**

```
Organization:
  member    — belongs to the org
  admin     — can manage the org, teams, and all resources

Team:
  member    — belongs to the team
  admin     — can manage team membership and team resources
  parent    — the organization this team belongs to

Project:
  owner     — full control (creator)
  admin     — can manage project settings and membership
  editor    — can create/edit/delete snippets in the project
  viewer    — can view all snippets in the project
  parent    — the team or org this project belongs to

Snippet:
  owner     — full control (creator)
  editor    — can edit the snippet
  viewer    — can view the snippet
  parent    — the project this snippet belongs to

Collection:
  owner     — full control (creator)
  curator   — can add/remove snippets
  viewer    — can view the collection
```

**Permission Resolution:**

Permissions are resolved by walking the relationship graph. A user has
`view` permission on a snippet if ANY of these are true:

1. User is the snippet's `owner`
2. User is a `viewer`, `editor`, or `admin` of the snippet
3. User is a `viewer`, `editor`, or `admin` of the snippet's parent project
4. User is a `member` or `admin` of a team that has access to the project
5. User is a `member` or `admin` of the organization that owns the project
6. The snippet is `public`
7. The user has a valid share token

```
                        ┌──────────────┐
                        │ Organization │
                        │   (acme)     │
                        └──────┬───────┘
                               │ member/admin
                    ┌──────────┼──────────┐
                    v                     v
             ┌───────────┐         ┌───────────┐
             │  Team     │         │  Team     │
             │ (platform)│         │ (frontend)│
             └────┬──────┘         └────┬──────┘
                  │ viewer/editor       │ editor
                  v                     v
             ┌──────────┐         ┌──────────┐
             │ Project  │         │ Project  │
             │ (infra)  │         │ (ui-lib) │
             └────┬─────┘         └────┬─────┘
                  │ parent             │ parent
           ┌──────┼──────┐         ┌───┴────┐
           v      v      v         v        v
        ┌─────┐┌─────┐┌─────┐  ┌─────┐  ┌─────┐
        │snip1││snip2││snip3│  │snip4│  │snip5│
        └─────┘└─────┘└─────┘  └─────┘  └─────┘
```

#### Data Model

```sql
-- Organizations
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR NOT NULL,
  slug VARCHAR NOT NULL UNIQUE,
  avatar_url VARCHAR,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Teams
CREATE TABLE teams (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name VARCHAR NOT NULL,
  slug VARCHAR NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE(org_id, slug)
);

-- Central relationship table (Zanzibar-inspired)
CREATE TABLE relationships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  -- Subject: who has the relationship
  subject_type VARCHAR NOT NULL CHECK (subject_type IN ('user', 'team', 'org')),
  subject_id UUID NOT NULL,
  -- Relation: what kind of relationship
  relation VARCHAR NOT NULL CHECK (relation IN (
    'owner', 'admin', 'editor', 'viewer', 'member', 'curator', 'parent'
  )),
  -- Object: what the relationship is with
  object_type VARCHAR NOT NULL CHECK (object_type IN (
    'org', 'team', 'project', 'snippet', 'collection'
  )),
  object_id UUID NOT NULL,
  -- Metadata
  created_at TIMESTAMPTZ DEFAULT now(),
  created_by UUID REFERENCES auth.users(id),
  expires_at TIMESTAMPTZ,  -- for time-limited access
  -- Uniqueness
  UNIQUE(subject_type, subject_id, relation, object_type, object_id)
);

CREATE INDEX idx_rel_subject ON relationships(subject_type, subject_id);
CREATE INDEX idx_rel_object ON relationships(object_type, object_id);
CREATE INDEX idx_rel_lookup ON relationships(object_type, object_id, relation);
```

#### Permission Check Function

```sql
CREATE OR REPLACE FUNCTION check_permission(
  p_user_id UUID,
  p_permission VARCHAR,  -- 'view', 'edit', 'delete', 'share', 'admin'
  p_object_type VARCHAR,
  p_object_id UUID
) RETURNS BOOLEAN
LANGUAGE sql STABLE SECURITY DEFINER
AS $$
  WITH RECURSIVE access_chain AS (
    -- Direct user relationships
    SELECT r.relation, r.object_type, r.object_id
    FROM relationships r
    WHERE r.subject_type = 'user'
      AND r.subject_id = p_user_id
      AND r.object_type = p_object_type
      AND r.object_id = p_object_id
      AND (r.expires_at IS NULL OR r.expires_at > now())

    UNION

    -- Team memberships -> team relationships
    SELECT tr.relation, tr.object_type, tr.object_id
    FROM relationships tm
    JOIN relationships tr ON tr.subject_type = 'team'
      AND tr.subject_id = tm.object_id
    WHERE tm.subject_type = 'user'
      AND tm.subject_id = p_user_id
      AND tm.relation IN ('member', 'admin')
      AND tm.object_type = 'team'
      AND tr.object_type = p_object_type
      AND tr.object_id = p_object_id

    UNION

    -- Parent chain (snippet -> project -> team -> org)
    SELECT ac.relation, pr.object_type, pr.object_id
    FROM access_chain ac
    JOIN relationships pr ON pr.subject_type = ac.object_type
      AND pr.subject_id = ac.object_id
      AND pr.relation = 'parent'
  )
  SELECT EXISTS (
    SELECT 1 FROM access_chain
    WHERE CASE p_permission
      WHEN 'view' THEN relation IN ('owner', 'admin', 'editor', 'viewer', 'member', 'curator')
      WHEN 'edit' THEN relation IN ('owner', 'admin', 'editor')
      WHEN 'delete' THEN relation IN ('owner', 'admin')
      WHEN 'share' THEN relation IN ('owner', 'admin', 'editor')
      WHEN 'admin' THEN relation IN ('owner', 'admin')
    END
  );
$$;
```

#### RLS Integration

The existing RLS policies are extended to use `check_permission`:

```sql
-- Replace simple owner check with ReBAC check
CREATE POLICY snippets_select_rebac ON snippets FOR SELECT
  USING (
    owner_id = auth.uid()
    OR visibility = 'public'
    OR check_permission(auth.uid(), 'view', 'snippet', id)
  );

CREATE POLICY snippets_update_rebac ON snippets FOR UPDATE
  USING (
    owner_id = auth.uid()
    OR check_permission(auth.uid(), 'edit', 'snippet', id)
  );
```

#### UX for Permissions

Permissions must be **simple to manage** despite the powerful model underneath.
The share dialog is modeled after Google Docs — familiar, intuitive, no
learning curve. The complex ReBAC model is hidden behind a simple UI.

```
┌─────────────────────────────────────────────────────────┐
│  Share "deploy-k8s.sh"                                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  People with access                                     │
│  ┌─────────────────────────────────────────────────┐    │
│  │  alice (you)                       Owner        │    │
│  │  bob@acme.com                      Can edit     │    │
│  │  Team Platform                     Can view     │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  Add people or teams                                    │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Search by name or email...          Can view    │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  -- Or share via link ──────────────────────────────    │
│  Anyone with the link              Can view             │
│  https://stash.dev/s/a1b2c3d4      Copy                 │
│                                                         │
│                                    Done                 │
└─────────────────────────────────────────────────────────┘
```

---

### V2.3 Auditing

#### Problem

Teams and organizations need to know:

- Who accessed which snippets and when (compliance)
- What changed and who changed it (accountability)
- How shared snippets are being used (analytics)
- Whether access patterns are anomalous (security)

#### Audit Log Architecture

```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  -- Actor
  actor_id UUID,                     -- null for anonymous/system
  actor_type VARCHAR NOT NULL        -- 'user', 'service', 'system', 'federation'
    CHECK (actor_type IN ('user', 'service', 'system', 'federation')),
  actor_name VARCHAR,                -- denormalized for log readability
  -- Action
  action VARCHAR NOT NULL            -- structured action name
    CHECK (action IN (
      'snippet.create', 'snippet.read', 'snippet.update', 'snippet.delete',
      'snippet.copy', 'snippet.share', 'snippet.unshare', 'snippet.fork',
      'snippet.export', 'snippet.embed_view',
      'project.create', 'project.update', 'project.delete',
      'collection.create', 'collection.update', 'collection.delete',
      'share.generate_token', 'share.revoke_token', 'share.access',
      'auth.login', 'auth.logout', 'auth.token_create', 'auth.token_revoke',
      'permission.grant', 'permission.revoke', 'permission.change',
      'team.create', 'team.update', 'team.add_member', 'team.remove_member',
      'org.create', 'org.update',
      'federation.peer_request', 'federation.peer_approve',
      'federation.peer_revoke', 'federation.search', 'federation.fetch',
      'export.bulk', 'import.gist', 'import.file'
    )),
  -- Resource
  resource_type VARCHAR,             -- 'snippet', 'project', 'collection', etc.
  resource_id UUID,
  resource_name VARCHAR,             -- denormalized
  -- Context
  metadata JSONB DEFAULT '{}',       -- action-specific details
  ip_address INET,
  user_agent TEXT,
  session_id UUID,
  -- Federation context
  source_instance VARCHAR,           -- originating instance (for federated actions)
  -- Timestamp
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Indexes for common queries
CREATE INDEX idx_audit_actor ON audit_log(actor_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id, created_at DESC);
CREATE INDEX idx_audit_action ON audit_log(action, created_at DESC);
CREATE INDEX idx_audit_time ON audit_log(created_at DESC);
```

#### Audit Capture

Audit events are captured at the database level via triggers, not in
application code. This ensures every access path (UI, CLI, API, federation)
is audited consistently.

```sql
CREATE OR REPLACE FUNCTION audit_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql SECURITY DEFINER
AS $$
BEGIN
  INSERT INTO audit_log (
    actor_id, actor_type, actor_name,
    action, resource_type, resource_id, resource_name,
    metadata
  ) VALUES (
    auth.uid(),
    'user',
    (SELECT username FROM profiles WHERE id = auth.uid()),
    TG_ARGV[0] || '.' || lower(TG_OP),
    TG_ARGV[0],
    CASE TG_OP WHEN 'DELETE' THEN OLD.id ELSE NEW.id END,
    CASE TG_OP WHEN 'DELETE' THEN OLD.title ELSE NEW.title END,
    CASE TG_OP
      WHEN 'UPDATE' THEN jsonb_build_object(
        'changed_fields', (
          SELECT jsonb_object_agg(key, value)
          FROM jsonb_each(to_jsonb(NEW))
          WHERE to_jsonb(NEW) ->> key IS DISTINCT FROM to_jsonb(OLD) ->> key
            AND key NOT IN ('updated_at', 'search_vector')
        )
      )
      ELSE '{}'
    END
  );
  RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER audit_snippets
  AFTER INSERT OR UPDATE OR DELETE ON snippets
  FOR EACH ROW EXECUTE FUNCTION audit_trigger('snippet');

CREATE TRIGGER audit_projects
  AFTER INSERT OR UPDATE OR DELETE ON projects
  FOR EACH ROW EXECUTE FUNCTION audit_trigger('project');
```

#### Audit Dashboard

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Audit Log                           Export CSV   Export JSON   Filter   │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Filter: All actions   All users   All resources   Last 7 days           │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │ 14:23  alice   edited    deploy-k8s.sh       Changed: content      │  │
│  │ 14:21  alice   viewed    deploy-k8s.sh                             │  │
│  │ 13:45  bob     copied    useAuth.tsx          Via share link       │  │
│  │ 13:30  system  shared    deploy-k8s.sh       Token generated       │  │
│  │ 12:00  carol   searched  "kubernetes deploy"  3 results            │  │
│  │ 11:45  alice   created   nginx-config         In project: infra    │  │
│  │ 11:30  ext     fetched   deploy-k8s.sh       From: team-frontend   │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  -- Activity Summary (7 days) ──────────────────────────────────────     │
│                                                                          │
│    Creates  ########..........  12                                       │
│    Views    ################..  89                                       │
│    Edits    ####..............   8                                       │
│    Shares   ##................   4                                       │
│    Copies   ##########........  34                                       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Compliance Features

- **Retention policies** — Auto-delete audit logs after N days (configurable)
- **Immutability** — Audit log table is append-only (no UPDATE/DELETE policies)
- **Export** — CSV/JSON export for compliance tools (Splunk, Datadog, etc.)
- **Webhooks** — Stream audit events to external SIEM systems
- **Data residency** — Audit logs respect instance data residency requirements

---

### V2.4 Trust

#### Problem

As Stash grows beyond personal use — into teams, organizations, and federated
instances — users need to answer: "Can I trust this snippet?"

Trust operates at multiple levels:

- **Author trust** — Is this person who they say they are?
- **Content trust** — Has this snippet been tampered with?
- **Instance trust** — Is this federated instance legitimate?
- **Quality trust** — Is this snippet any good?

#### Trust Model

```
Trust Layers:

  Layer 1: Identity
  |-- GitHub-verified identity (OAuth)
  |-- Organization membership verification
  |-- Email domain verification (corp email -> org trust)
  +-- Verified badge on profiles

  Layer 2: Content Integrity
  |-- SHA-256 content hash per snippet version
  |-- Signed snippets (optional GPG/SSH signature)
  |-- Tamper-evident modification history
  +-- Provenance tracking (source chain)

  Layer 3: Instance Trust (Federation)
  |-- mTLS between federated instances
  |-- Instance verification (admin-approved peering)
  |-- Trust levels: basic -> verified -> trusted
  +-- Trust degradation on policy violations

  Layer 4: Quality & Safety
  |-- Usage metrics (copies, views, forks)
  |-- Community signals (upvote/flag on public snippets)
  |-- Automated safety scan (detect secrets, eval, etc.)
  +-- License metadata for shared/public snippets
```

#### Content Integrity

```sql
-- Extend the V1 snippet_versions table with integrity and trust columns
ALTER TABLE snippet_versions
  ADD COLUMN metadata_hash VARCHAR,  -- SHA-256 of (title + labels + language)
  ADD COLUMN signature TEXT;         -- optional GPG/SSH signature

-- Integrity verification function
CREATE OR REPLACE FUNCTION verify_snippet_integrity(p_snippet_id UUID)
RETURNS TABLE (
  version_number INT,
  content_valid BOOLEAN,
  metadata_valid BOOLEAN,
  signature_valid BOOLEAN
)
LANGUAGE plpgsql SECURITY INVOKER
AS $$
BEGIN
  RETURN QUERY
  SELECT
    sv.version_number,
    sv.content_hash = encode(digest(sv.content, 'sha256'), 'hex'),
    sv.metadata_hash = encode(digest(
      s.title || array_to_string(
        ARRAY(SELECT jsonb_array_elements_text(s.labels)), ','
      ) || s.language,
      'sha256'
    ), 'hex'),
    sv.signature IS NOT NULL
  FROM snippet_versions sv
  JOIN snippets s ON s.id = sv.snippet_id
  WHERE sv.snippet_id = p_snippet_id
  ORDER BY sv.version_number;
END;
$$;
```

#### Safety Scanning

Automated scanning of snippet content for security concerns:

| Check | Description | Action |
|-------|-------------|--------|
| Secret detection | API keys, passwords, tokens in content | Warning badge, block public sharing |
| Dangerous patterns | `eval()`, `exec()`, `rm -rf /`, SQL injection | Warning badge |
| License compliance | Detect license headers, check compatibility | Info badge |
| Dependency risk | Detect `import`/`require` of known-vulnerable packages | Warning badge |

Scanning runs in two layers:

1. **Synchronous (trigger):** Fast regex-based checks on insert/update for
   high-confidence patterns (AWS keys, private keys, dangerous commands).
   Uses patterns derived from [gitleaks](https://github.com/gitleaks/gitleaks)
   and [trufflehog](https://github.com/trufflesecurity/trufflehog).
2. **Asynchronous (Edge Function):** Deeper analysis (AI-based, entropy
   detection, license scanning) runs via a database webhook after insert/update.

```sql
CREATE OR REPLACE FUNCTION scan_snippet_content()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
  warnings JSONB := '[]'::JSONB;
BEGIN
  -- AWS Access Key ID (high confidence, fixed format)
  IF NEW.content ~* 'AKIA[0-9A-Z]{16}' THEN
    warnings := warnings || '["aws_access_key_detected"]'::JSONB;
  END IF;

  -- Private keys (PEM headers)
  IF NEW.content ~* '-----BEGIN (RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----' THEN
    warnings := warnings || '["private_key_detected"]'::JSONB;
  END IF;

  -- GitHub tokens (ghp_, gho_, ghu_, ghs_, ghr_)
  IF NEW.content ~* 'gh[pousr]_[A-Za-z0-9_]{36,}' THEN
    warnings := warnings || '["github_token_detected"]'::JSONB;
  END IF;

  -- Generic high-entropy secret assignment (key/secret/password/token = "...")
  IF NEW.content ~* '(api[_-]?key|secret[_-]?key|password|token)\s*[:=]\s*[''"][A-Za-z0-9+/=]{20,}' THEN
    warnings := warnings || '["potential_secret_detected"]'::JSONB;
  END IF;

  -- Dangerous shell commands
  IF NEW.content ~* 'rm\s+-rf\s+/' THEN
    warnings := warnings || '["dangerous_command_detected"]'::JSONB;
  END IF;

  NEW.safety_warnings := warnings;
  RETURN NEW;
END;
$$;
```

#### Trust Indicators in UX

Trust information is visible but not obtrusive. Users understand trust at a
glance:

```
┌────────────────────────────────────────────────────────┐
│ deploy-k8s.sh                              2 hours ago │
│ #!/bin/bash                                            │
│ kubectl apply -f deployment.yaml                       │
│                                                        │
│ [v] @alice (verified)  [!] Contains secrets  154 uses  │
│ bash  k8s  deploy      Signed                          │
└────────────────────────────────────────────────────────┘
```

| Indicator | Meaning |
|-----------|---------|
| `[v] verified` | Author identity confirmed via GitHub + org membership |
| `Signed` | Content has a valid cryptographic signature |
| `[!] warning` | Safety scan detected a potential issue |
| `154 uses` | Usage signal — frequently copied = likely useful |
| Instance badge | For federated snippets: shows originating instance |

---

## V2 Candidates (Additional)

1. **Auto-Tagging** — Language detection + LLM-based label suggestions on create
2. **Claude Integration** — Extract snippets from Claude conversations
3. **Snippet Forking** — Fork public/shared snippets into your own library
4. **Comments** — Inline comments on snippets and collections
5. **Collection Templates** — Pre-built collection templates for common patterns
6. **Auto-Linking** — AI-powered suggestion of links between related snippets

---

## Competitive Analysis

| Feature | Stash | GitHub Gist | SnippetsLab | Cacher | Snipit |
|---------|-------|-------------|-------------|--------|--------|
| Full-text search | Weighted FTS | Basic | Local | Basic | Basic |
| Keyboard-first UX | Cmd+K, vim keys | No | macOS native | Partial | No |
| CLI tool | Yes | `gh gist` | No | Yes | No |
| Team sharing | ReBAC (V2) | Public only | No | Teams | Teams |
| Federation | Yes (V2) | No | No | No | No |
| Self-hostable | Yes | No | No | No | No |
| Embeddable | Yes | Yes | No | No | No |
| Audit log | Yes (V2) | No | No | No | No |
| API/scripting | curl, CLI, API | API | No | API | API |
| Pricing | Freemium | Free | $9.99 once | $6/mo | $4/mo |

**Key differentiators:**

1. **PostgreSQL-first search** — faster, more relevant results than
   Elasticsearch-based alternatives
2. **Federation** — no competitor offers cross-instance snippet discovery
3. **ReBAC** — Google Zanzibar-level permissions in a snippet manager
4. **Audit trail** — compliance-ready, no competitor offers this
5. **Self-hostable + SaaS** — deploy anywhere or use hosted version
6. **Developer-native** — CLI, API, curl, embeds — built for how developers
   actually work

---

## Architectural Decisions

### Why PostgreSQL-Only (No SpiceDB/OpenFGA for ReBAC)

SpiceDB and OpenFGA are purpose-built for Zanzibar-style authorization. However,
adding them introduces another service to operate, network latency for every
permission check, data consistency challenges, and increased deployment
complexity. For Stash's scale (< 1M relationships for most instances),
PostgreSQL handles ReBAC with a recursive CTE and proper indexing. The
`check_permission` function runs in < 5ms for typical relationship graphs.
If a single instance grows beyond this, SpiceDB can be introduced as a caching
layer without changing the data model.

### Why REST Federation (Not ActivityPub)

ActivityPub is designed for social networking (posts, follows, likes). Its
vocabulary and assumptions don't map cleanly to code snippets. A custom REST
protocol is simpler to implement and debug, tailored to Stash's specific needs
(search, fetch, verify), easier to version and evolve, and not burdened by
ActivityPub's complexity (JSON-LD, inbox/outbox, etc.).

### Why Database-Level Auditing (Not Application-Level)

Application-level audit logging misses direct database access (admin queries,
migrations), edge function operations, federation queries that bypass the
frontend, and any new access path added in the future. Database triggers catch
everything, regardless of how the data is accessed. The trade-off is slightly
higher write latency (< 1ms per audited operation), which is acceptable.

### Why PostgreSQL for Repository Snippets (Not Git/Object Storage)

Repository snippets could be backed by actual Git repositories (e.g., via
`libgit2` or a Git hosting layer) or stored as blobs in S3. However, this
introduces external dependencies that break the "PostgreSQL-only" principle.
Snippet-sized repositories (max 50 files, max 1 MB total) fit comfortably in
PostgreSQL rows with TOAST compression. This keeps the architecture simple:
one database, one query language, one backup strategy. Full-text search spans
all files in a repository snippet via the same `tsvector` mechanism. If storage
needs grow beyond this, an object storage layer can be added later without
changing the data model (just the storage backend for `content` columns).

### Why Full Snapshots for Versioning (Not Deltas)

Delta-based versioning (storing only the diff between versions) saves storage
but adds complexity: reconstructing any version requires replaying all deltas
from a base. For snippet-sized content (< 1 MB), full snapshots are simpler,
allow O(1) access to any version, and simplify diff computation (compare any
two snapshots directly). PostgreSQL's TOAST compression naturally deduplicates
similar text content at the storage level. The trade-off is acceptable for the
expected content sizes.

### Content Integrity Without Blockchain

Blockchain is unnecessary for snippet integrity. A simpler approach: SHA-256
hash per version (tamper-evident), optional GPG/SSH signatures
(non-repudiation), append-only audit log (accountability), PostgreSQL WAL
(point-in-time recovery). This provides equivalent guarantees without the
operational overhead.

---

## Implementation Roadmap

### Phase 1: V1 Core

| Step | Milestone |
|------|-----------|
| 1.1 | Database schema (incl. snippet_files, collections, links, versions), migrations, RLS, RPC functions |
| 1.2 | Next.js app: auth flow, dashboard, single-file snippet CRUD |
| 1.3 | Search, labels, projects, sharing, command palette |
| 1.4 | Repository snippets (file tree editor, multi-file storage) |
| 1.5 | Collections (CRUD, sharing, collection view) |
| 1.6 | Snippet links (create, display, bidirectional view) |
| 1.7 | Snippet versioning (version history, diff view, restore) |
| 1.8 | Edge function (raw endpoint), embeds, polish |
| 1.9 | Multi-instance CI/CD, staging deploy, testing |

### Phase 2: V1 Growth

| Step | Milestone |
|------|-----------|
| 2.1 | CLI: basic operations (`stash push/search/get`) |
| 2.2 | CLI: git-like workflow (`stash checkout/clone/pull/push/diff/status/log`) |
| 2.3 | CLI: collections & links commands |
| 2.4 | Gist import, file upload, paste detection |
| 2.5 | Embeddable snippets, OG previews |
| 2.6 | Free/Pro tier implementation, Stripe integration |
| 2.7 | Public launch, marketing site |

### Phase 3: V2 Foundation

| Step | Milestone |
|------|-----------|
| 3.1 | Organizations, teams, ReBAC data model |
| 3.2 | Permission UI (share dialog, team management) |
| 3.3 | Audit log (triggers, dashboard, export) |
| 3.4 | Content integrity (versioning, hashing, safety scan) |

### Phase 4: V2 Federation

| Step | Milestone |
|------|-----------|
| 4.1 | Federation protocol, instance discovery, peering |
| 4.2 | Federated search, snippet fetching |
| 4.3 | Trust model, instance verification, trust indicators |
| 4.4 | Enterprise tier, compliance features, SLA |

---

## V3: Intelligence, Ecosystem & Developer Experience

V3 transforms Stash from a **storage and sharing tool** into an **intelligent
developer knowledge platform**. The three pillars are: **AI Intelligence**,
**Ecosystem & Integrations**, and **Advanced Developer Experience**.

V1 gave developers a place to store and find snippets. V2 gave organizations
control and trust. V3 makes Stash **proactively useful** — it understands your
code, learns your patterns, and meets you where you work.

---

### V3.1 AI Intelligence

#### Problem

Developers don't just want to store snippets — they want to **find the right
code at the right time**, understand unfamiliar snippets instantly, and avoid
duplicating work that already exists in their library. Manual tagging is
inconsistent, search requires knowing the right keywords, and there's no way
to discover that someone on your team already solved the same problem.

#### Semantic Search (Vector Embeddings)

PostgreSQL full-text search (V1) is excellent for keyword matching but fails
when the user describes intent rather than syntax. "function that retries HTTP
requests with exponential backoff" won't match a snippet titled
`resilient_fetch.py` tagged with `python, http`.

Semantic search uses vector embeddings to understand **meaning**, not just words.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Search Architecture (V3)                           │
│                                                                              │
│  User Query: "retry logic with backoff"                                      │
│                                                                              │
│  ┌────────────────┐     ┌────────────────┐      ┌────────────────────────┐   │
│  │ FTS (V1)       │     │ Semantic (V3)  │      │ Result Fusion          │   │
│  │                │     │                │      │                        │   │
│  │ tsvector match │     │ pgvector       │      │ RRF (Reciprocal Rank   │   │
│  │ "retry" "back" │     │ cosine sim     │      │ Fusion) combining      │   │
│  │ → 2 results    │     │ → 8 results    │      │ both result sets       │   │
│  └───────┬────────┘     └───────┬────────┘      │                        │   │
│          │                      │               │ FTS weight:  0.4       │   │
│          └──────────────────────┘               │ Semantic:    0.6       │   │
│                      │                          │                        │   │
│                      ▼                          │ → ranked results       │   │
│              ┌──────────────┐                   └────────────────────────┘   │
│              │ Re-ranker    │                                                │
│              │ (optional)   │                                                │
│              └──────────────┘                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

**Implementation:**

- **Embedding model:** Voyage Code 3 (preferred — optimized for code retrieval,
  1024 dimensions) with fallback to OpenAI `text-embedding-3-small` (1536
  dimensions). Voyage Code 3 outperforms general-purpose models on code search
  benchmarks. Dimension count is configurable via environment variable.
- **Storage:** `pgvector` extension in Supabase (native PostgreSQL, no external
  service)
- **Indexing:** Embeddings are computed on snippet create/update via Edge
  Function and stored in an `embedding` column
- **Hybrid search:** Reciprocal Rank Fusion (RRF) combines FTS results
  (keyword relevance) with vector similarity (semantic relevance)

```sql
-- Add embedding column to snippets
ALTER TABLE snippets ADD COLUMN embedding vector(1024);
CREATE INDEX idx_snippets_embedding ON snippets
  USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- Hybrid search function
CREATE OR REPLACE FUNCTION search_snippets_v3(
  search_query   TEXT    DEFAULT NULL,
  filter_labels  JSONB   DEFAULT NULL,
  filter_project UUID    DEFAULT NULL,
  filter_lang    TEXT    DEFAULT NULL,
  query_embedding vector(1024) DEFAULT NULL,
  semantic_weight FLOAT  DEFAULT 0.6,
  sort_by        TEXT    DEFAULT 'relevance',
  page_limit     INT     DEFAULT 20,
  page_offset    INT     DEFAULT 0
) RETURNS TABLE (
  id UUID, title VARCHAR, description TEXT, language VARCHAR,
  labels JSONB, visibility VARCHAR, updated_at TIMESTAMPTZ,
  fts_rank FLOAT, semantic_score FLOAT, combined_score FLOAT
)
LANGUAGE plpgsql SECURITY INVOKER
AS $$
BEGIN
  RETURN QUERY
  WITH fts_results AS (
    SELECT s.id, ts_rank(s.search_vector, websearch_to_tsquery('english', search_query)) AS rank
    FROM snippets s
    WHERE search_query IS NOT NULL
      AND s.search_vector @@ websearch_to_tsquery('english', search_query)
  ),
  semantic_results AS (
    SELECT s.id, 1 - (s.embedding <=> query_embedding) AS similarity
    FROM snippets s
    WHERE query_embedding IS NOT NULL
      AND s.embedding IS NOT NULL
    ORDER BY s.embedding <=> query_embedding
    LIMIT 100
  ),
  fused AS (
    SELECT
      COALESCE(f.id, se.id) AS id,
      (1 - semantic_weight) * COALESCE(1.0 / (60 + ROW_NUMBER() OVER (ORDER BY f.rank DESC NULLS LAST)), 0)
      + semantic_weight * COALESCE(1.0 / (60 + ROW_NUMBER() OVER (ORDER BY se.similarity DESC NULLS LAST)), 0)
      AS score
    FROM fts_results f
    FULL OUTER JOIN semantic_results se ON f.id = se.id
  )
  SELECT s.id, s.title, s.description, s.language, s.labels, s.visibility,
         s.updated_at,
         COALESCE(f.rank, 0), COALESCE(se.similarity, 0), fu.score
  FROM fused fu
  JOIN snippets s ON s.id = fu.id
  LEFT JOIN fts_results f ON f.id = fu.id
  LEFT JOIN semantic_results se ON se.id = fu.id
  WHERE (filter_labels IS NULL OR s.labels @> filter_labels)
    AND (filter_project IS NULL OR s.project_id = filter_project)
    AND (filter_lang IS NULL OR s.language = filter_lang)
  ORDER BY fu.score DESC
  LIMIT page_limit OFFSET page_offset;
END;
$$;
```

#### AI Auto-Tagging & Categorization

Manual tagging is inconsistent — one developer tags `javascript`, another tags
`js`, a third forgets to tag entirely. AI auto-tagging solves this at creation
time.

**Flow:**

```
Snippet created/updated
    │
    ▼
Edge Function: generate-metadata
    │
    ├── Detect language (heuristic + fallback to LLM)
    ├── Generate 3-5 label suggestions (from existing user labels + new)
    ├── Generate one-line description (if empty)
    ├── Classify category (algorithm, config, boilerplate, utility, etc.)
    └── Compute embedding vector
    │
    ▼
Store suggestions in snippet.ai_metadata (JSONB)
    │
    ▼
UI shows suggestions as clickable chips (non-intrusive)
```

**Key design decisions:**

- Labels are **suggested, never auto-applied** — the user stays in control
- Suggestions prefer existing labels from the user's library (consistency)
- Language detection is a **fast heuristic first** (file extension, shebang,
  syntax patterns), LLM fallback only when ambiguous
- Description generation only triggers when the field is empty
- All AI metadata is stored separately in `ai_metadata JSONB` — never mixed
  with user data

#### Snippet Explanation & Documentation

"What does this snippet do?" — answered instantly by AI.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  deploy-k8s.sh                                              2 hours ago  │
│  ┌────────────────────────────────────────────────────── Copy  Raw ──┐   │
│  │  1  #!/bin/bash                                                   │   │
│  │  2  set -euo pipefail                                             │   │
│  │  3  NAMESPACE="${1:-default}"                                     │   │
│  │  4  kubectl apply -f deployment.yaml -n "$NAMESPACE"              │   │
│  │  5  kubectl rollout status deployment/app -n "$NAMESPACE" \       │   │
│  │  6    --timeout=300s                                              │   │
│  │  7  if [ $? -ne 0 ]; then                                         │   │
│  │  8    kubectl rollout undo deployment/app -n "$NAMESPACE"         │   │
│  │  9  fi                                                            │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  [Explain]  [Generate Docs]  [Find Similar]                              │
│                                                                          │
│  ┌── AI Explanation ─────────────────────────────────────────────────┐   │
│  │ Deploys a Kubernetes application with automatic rollback.         │   │
│  │                                                                   │   │
│  │ 1. Sets strict error handling (set -euo pipefail)                 │   │
│  │ 2. Takes namespace as first argument (defaults to "default")      │   │
│  │ 3. Applies deployment manifest to the target namespace            │   │
│  │ 4. Waits up to 300s for rollout to complete                       │   │
│  │ 5. If rollout fails, automatically rolls back to previous version │   │
│  │                                                                   │   │
│  │ Usage: ./deploy-k8s.sh production                                 │   │
│  └───────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

- **Explain**: One-click AI explanation — especially valuable for snippets
  shared by others, inherited from team members, or imported from Gists
- **Generate Docs**: Creates Markdown documentation (parameters, usage
  examples, caveats) — insertable as the snippet's description
- **Find Similar**: Discovers semantically similar snippets in the user's
  library (and federated instances, if enabled)

#### Duplicate & Near-Duplicate Detection

Before creating a new snippet, Stash checks for existing duplicates:

```
stash push deploy.sh --title "K8s deploy"

⚠  Similar snippets found in your library:

  93% match  deploy-k8s.sh       (updated 2d ago)
             "Kubernetes deployment script"
  71% match  deploy-docker.sh    (updated 1w ago)
             "Docker deployment with rollback"

  [c] Continue and create new snippet
  [v] View matching snippet
  [u] Update existing snippet instead
  [q] Cancel
```

Detection uses a combination of:

- **Content hash** — exact match (SHA-256)
- **Embedding similarity** — semantic near-duplicate (cosine > 0.92)
- **Structural similarity** — AST-level comparison for supported languages

#### Smart Suggestions (Context-Aware)

Stash learns what you need based on what you're working on:

- **Clipboard integration**: Detect copied code in clipboard, suggest saving
  as snippet (macOS/Linux via CLI daemon)
- **Git context**: The CLI reads your current Git branch/diff and suggests
  relevant snippets ("You're working on auth — here are your auth snippets")
- **Time-based**: Surface snippets you use at certain times (deploy scripts
  on Fridays, standup queries on Monday mornings)
- **Project context**: When working in a directory with a `package.json`,
  suggest relevant Node.js snippets

#### Data Model Extension (AI)

```sql
-- AI metadata stored alongside snippets
ALTER TABLE snippets ADD COLUMN ai_metadata JSONB DEFAULT '{}';
-- Structure: {
--   "suggested_labels": ["k8s", "deploy", "bash"],
--   "suggested_description": "...",
--   "category": "devops",
--   "complexity": "intermediate",
--   "similar_snippets": ["uuid1", "uuid2"],
--   "explanation_cache": "...",
--   "last_analyzed_at": "2026-03-14T..."
-- }

-- Embedding for semantic search (via pgvector)
ALTER TABLE snippets ADD COLUMN embedding vector(1024);
CREATE INDEX idx_snippets_embedding ON snippets
  USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

-- AI usage tracking (for model improvement)
CREATE TABLE ai_interactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id),
  snippet_id UUID REFERENCES snippets(id),
  interaction_type VARCHAR NOT NULL, -- 'explain', 'generate_docs', 'find_similar',
                                     -- 'accept_label', 'reject_label', 'accept_description'
  input JSONB,
  output JSONB,
  accepted BOOLEAN,      -- did the user accept the suggestion?
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

### V3.2 Ecosystem & Integrations

#### Problem

Stash's CLI is powerful, but developers live in their editors. Context-
switching to a browser or terminal to save/find snippets is friction that
reduces adoption. Stash needs to be **available wherever code is written,
read, or discussed**.

#### VS Code Extension

The highest-impact integration. VS Code has ~75% market share among
developers.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  VS Code                                                                 │
│  ┌───────────────────────────────────────────────────────────────────┐   │
│  │  editor.tsx                                                       │   │
│  │                                                                   │   │
│  │  export function useDebounce<T>(value: T, delay: number): T {     │   │
│  │    const [debounced, setDebounced] = useState(value);             │   │
│  │    useEffect(() => {                                              │   │
│  │      const timer = setTimeout(() => setDebounced(value), delay);  │   │
│  │      return () => clearTimeout(timer);                            │   │
│  │    }, [value, delay]);                                            │   │
│  │    return debounced;  ┌─────────────────────────────────┐         │   │
│  │  }                    │  Stash: Save Selection          │         │   │
│  │                       │  Stash: Search Snippets         │         │   │
│  │                       │  Stash: Insert Snippet Here     │         │   │
│  │                       │  Stash: Explain Selection (AI)  │         │   │
│  │                       └─────────────────────────────────┘         │   │
│  └───────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

**Features:**

| Command | Action |
|---------|--------|
| `Stash: Save Selection` | Save highlighted code as snippet (pre-fills language, suggests labels) |
| `Stash: Save File` | Save entire file as a snippet |
| `Stash: Search` | Cmd+Shift+S — search snippets inline, insert at cursor |
| `Stash: Insert Snippet` | Browse and insert a snippet at the current cursor position |
| `Stash: Explain` | AI explanation of selected code (powered by V3.1) |
| `Stash: Find Similar` | Find similar snippets to the current selection |
| `Stash: Sidebar` | Tree view of projects, collections, and pinned snippets |
| `Stash: Diff` | Compare local file against its Stash version |

**Sidebar panel:**

```
STASH
├── PINNED
│   ├── deploy-k8s.sh
│   └── useAuth.tsx
├── PROJECTS
│   ├── infra (23)
│   └── react (41)
├── COLLECTIONS
│   ├── K8s Recipes (12)
│   └── React Patterns (8)
└── RECENT
    ├── nginx-config     2h ago
    └── .dockerignore    1d ago
```

**Implementation:** TypeScript extension using the Stash REST API. Auth via
Personal Access Token (PAT) stored in VS Code settings. Snippets are fetched
on-demand, not synced locally.

#### JetBrains Plugin

Same feature set as VS Code, adapted for the IntelliJ platform (WebStorm,
PyCharm, GoLand, etc.). Built with Kotlin, distributed via JetBrains
Marketplace.

Key differences from VS Code:
- Uses JetBrains tool window instead of sidebar
- Integrates with JetBrains' built-in Live Templates for snippet insertion
- Supports JetBrains' built-in diff viewer for version comparison

#### Neovim Plugin

Lua-based plugin for Neovim. Minimal, keyboard-driven, respects the Neovim
philosophy.

```lua
-- ~/.config/nvim/lua/plugins/stash.lua
require('stash').setup({
  keymaps = {
    save_selection = '<leader>ss',   -- save visual selection
    search         = '<leader>sf',   -- telescope picker for snippets
    insert         = '<leader>si',   -- insert snippet at cursor
    explain        = '<leader>se',   -- AI explain selection
  },
  telescope = true,                  -- use telescope.nvim for search UI
})
```

**Features:**

- `<leader>ss` — save visual selection as snippet
- `<leader>sf` — Telescope picker with live search (FTS + semantic)
- `<leader>si` — insert snippet at cursor
- `<leader>se` — AI explanation in a floating window
- `:Stash push %` — push current file as snippet
- `:Stash pull <id>` — pull snippet into current buffer

#### Browser Extension (Chrome/Firefox/Safari)

Capture code from anywhere on the web.

```
┌──────────────────────────────────────────────────────────┐
│  Stack Overflow - How to retry HTTP requests             │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  async function fetchWithRetry(url, retries = 3) { │  │
│  │    for (let i = 0; i < retries; i++) {             │  │
│  │      try {                                         │  │
│  │        return await fetch(url);   ┌─────────────┐  │  │
│  │      } catch (e) {                │ Save to     │  │  │
│  │        if (i === retries - 1)     │ Stash       │  │  │
│  │          throw e;                 │             │  │  │
│  │        await sleep(2 ** i * 1000) │ Source: SO  │  │  │
│  │      }                            │ Lang: JS    │  │  │
│  │    }                              │ Labels: ... │  │  │
│  │  }                                └─────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

**Features:**

- Right-click on any code block → "Save to Stash"
- Auto-detects language from the page context (`<code>` class, syntax
  highlighting CSS classes)
- Captures source URL as metadata (provenance tracking)
- One-click popup for quick save with title, labels, project
- "Stash this page" — extracts all code blocks from a page into a collection
- Keyboard shortcut: `Alt+Shift+S` to save selected text

#### Raycast Extension

For macOS Raycast users — quick snippet access without opening any app.

```
┌─────────────────────────────────────────────────────────┐
│  > stash deploy                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  deploy-k8s.sh                    bash  ⌘+Enter to copy │
│  #!/bin/bash                                            │
│  kubectl apply -f deployment.yaml                       │
│                                                         │
│  deploy-docker.sh                 bash                  │
│  docker compose up -d --build                           │
│                                                         │
│  deploy-vercel.sh                 bash                  │
│  vercel --prod --yes                                    │
│                                                         │
│  Actions:                                               │
│  ⌘+Enter  Copy to clipboard                             │
│  ⌘+E      Edit in Stash                                 │
│  ⌘+S      Share                                         │
│  ⌘+N      New Snippet (from clipboard)                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### GitHub Integration

Stash integrates with GitHub to close the loop between snippets and real
code.

**GitHub App:**

- **Auto-import**: Watch repos for commonly reusable patterns (utility
  functions, config templates) and suggest saving them as snippets
- **PR Snippets**: Save code from PR reviews directly to Stash
- **Gist Sync**: Bidirectional sync between GitHub Gists and Stash snippets
- **README Embeds**: Embed Stash snippets in GitHub READMEs with live
  updates (via Stash embed endpoint)

**GitHub Action:**

```yaml
# .github/workflows/snippet-sync.yml
- uses: b42labs/stash-action@v1
  with:
    action: sync
    source: ./snippets/      # directory of snippets to sync
    project: shared-utils
    labels: team,utilities
    stash-token: ${{ secrets.STASH_TOKEN }}
```

#### Slack Integration

Share and discover snippets without leaving Slack.

- `/stash search <query>` — search snippets, results as rich message
- `/stash share <id>` — post a formatted snippet to the channel
- Unfurling: paste a Stash URL → Slack shows a rich preview with code
- Bot: "Save to Stash" message action on any code block in Slack

#### Webhook System

Event-driven integrations for automation:

```json
{
  "event": "snippet.created",
  "webhook_url": "https://hooks.example.com/stash",
  "secret": "whsec_...",
  "filters": {
    "labels": ["production"],
    "projects": ["infra"]
  }
}
```

**Supported events:**

| Event | Trigger |
|-------|---------|
| `snippet.created` | New snippet created |
| `snippet.updated` | Snippet content or metadata changed |
| `snippet.shared` | Share token generated |
| `snippet.accessed` | Shared snippet viewed (rate-limited) |
| `collection.updated` | Collection items added/removed |
| `version.created` | New version pushed |

**Use cases:**

- Post to Slack when a production snippet is modified
- Trigger CI validation when infrastructure snippets change
- Sync snippets to a documentation site on update
- Alert on-call when deployment scripts are edited

---

### V3.3 Advanced Developer Experience

#### Snippet Playground (Live Execution)

Run snippets directly in Stash — no copy-paste to a terminal needed.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  deploy-k8s.sh                                          Run  Run in...   │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  1  #!/bin/bash                                                  │    │
│  │  2  echo "Hello from $NAMESPACE"                                 │    │
│  │  3  curl -s https://httpbin.org/ip | jq .origin                  │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌── Output ─────────────────────────────────────────────────────────┐   │
│  │  $ NAMESPACE=production bash snippet.sh                           │   │
│  │  Hello from production                                            │   │
│  │  "203.0.113.42"                                                   │   │
│  │                                                                   │   │
│  │  Exit code: 0  |  Runtime: 1.2s  |  Sandbox: wasm                 │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Environment Variables                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  NAMESPACE = production                                          │    │
│  │  + Add variable                                                  │    │
│  └──────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────┘
```

**Supported runtimes:**

| Language | Runtime | Sandbox |
|----------|---------|---------|
| JavaScript/TypeScript | V8 isolate | Cloudflare Workers / Deno Deploy |
| Python | Pyodide (WASM) | Browser-side, no server needed |
| Bash | WebAssembly shell | Restricted, no filesystem access |
| SQL | In-browser SQLite (WASM) | sql.js |
| Go | Go Playground API | Remote (optional, self-hosted) |
| Rust | Rust Playground API | Remote (optional) |

**Security model:**

- All execution happens in sandboxed environments (WASM or V8 isolates)
- No access to the host filesystem, network restricted to allowlist
- Execution timeout: 30 seconds (configurable)
- Output truncated at 10 KB
- Users can self-host execution backends for full control

#### Snippet Templates & Generators

Parameterized snippets that generate code from templates:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Template: React Component                                              │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Parameters                                                      │   │
│  │                                                                  │   │
│  │  Component Name:  [ UserProfile        ]                         │   │
│  │  Props:           [ name, email, avatar ]                        │   │
│  │  Style:           (•) Tailwind  ( ) CSS Modules  ( ) Styled      │   │
│  │  Tests:           [x] Include test file                          │   │
│  │  Storybook:       [x] Include story                              │   │
│  │                                                                  │   │
│  │                    [Generate]                                    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌── Generated ──────────────────────────────────────────────────────┐  │
│  │  FILES (3)                                                        │  │
│  │  ├── UserProfile.tsx           ✓ generated                        │  │
│  │  ├── UserProfile.test.tsx      ✓ generated                        │  │
│  │  └── UserProfile.stories.tsx   ✓ generated                        │  │
│  │                                                                   │  │
│  │  [Save as Repository Snippet]  [Download ZIP]  [Copy All]         │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

Templates use **Handlebars-style** variables (`{{componentName}}`,
`{{#if includeTests}}`) inside snippet content. Template parameters are
defined in snippet metadata:

```json
{
  "template": true,
  "parameters": [
    { "name": "componentName", "type": "string", "required": true },
    { "name": "props", "type": "csv", "default": "" },
    { "name": "style", "type": "enum", "options": ["tailwind", "css-modules", "styled"], "default": "tailwind" },
    { "name": "includeTests", "type": "boolean", "default": true }
  ]
}
```

**CLI support:**

```bash
# Generate from template
stash generate <template-id> \
  --set componentName=UserProfile \
  --set props=name,email,avatar \
  --set style=tailwind \
  --out ./src/components/

# Interactive mode
stash generate <template-id> --interactive
```

#### Snippet Analytics & Insights

A personal and team-level analytics dashboard that helps developers
understand their knowledge base.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Insights                                      Last 30 days    Export    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  -- Your Library ────────────────────────────────────────────────────    │
│                                                                          │
│  Total Snippets: 247      Languages: 12      Projects: 8                 │
│  Total Versions: 1,203    Avg versions/snippet: 4.9                      │
│                                                                          │
│  -- Most Used (copies + views) ─────────────────────────────────────     │
│                                                                          │
│  1. deploy-k8s.sh           ██████████████████████  89 uses              │
│  2. useAuth.tsx             █████████████████       67 uses              │
│  3. docker-compose.yml      ████████████            48 uses              │
│  4. .prettierrc             ██████████              41 uses              │
│  5. retry-fetch.ts          █████████               37 uses              │
│                                                                          │
│  -- Language Distribution ──────────────────────────────────────────     │
│                                                                          │
│  TypeScript  ████████████████████              34%                       │
│  Bash        █████████████                     22%                       │
│  Python      ████████████                      19%                       │
│  YAML        ███████                           12%                       │
│  SQL         ████                               8%                       │
│  Other       ███                                5%                       │
│                                                                          │
│  -- Activity Heatmap ───────────────────────────────────────────────     │
│     Mon  ░░▓▓░░▓                                                         │
│     Tue  ░▓▓▓░░░                                                         │
│     Wed  ░░▓▓▓░░                                                         │
│     Thu  ░▓▓░░▓░                                                         │
│     Fri  ▓▓▓▓░░░                                                         │
│                                                                          │
│  -- Stale Snippets (no use in 90+ days) ────────────────────────────     │
│                                                                          │
│  12 snippets haven't been used in 90+ days.  [Review]                    │
│                                                                          │
│  -- Knowledge Gaps ─────────────────────────────────────────────────     │
│                                                                          │
│  Your team frequently searches for "terraform" and "aws" but you have    │
│  no snippets with these labels. [Browse Team Library]                    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Team insights (Team/Enterprise tier):**

- **Contribution leaderboard** — who creates/shares the most snippets
- **Knowledge coverage** — which topics are well-covered vs. gaps
- **Adoption metrics** — snippet reuse rate across the team
- **Search miss rate** — queries that return zero results (content gaps)
- **Onboarding acceleration** — time-to-first-snippet for new team members

#### Snippet Comments & Discussions

Inline comments on snippets and collections — enabling asynchronous code
knowledge sharing.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  deploy-k8s.sh                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐   │
│  │  4  NAMESPACE="${1:-default}"                                     │   │
│  │  5  kubectl apply -f deployment.yaml -n "$NAMESPACE"              │   │
│  │     💬 @bob: Should we add --server-side here for large           │   │
│  │         manifests? Server-side apply handles conflicts better.    │   │
│  │         ↳ @alice: Good call. Created a follow-up snippet          │   │
│  │           with server-side apply → deploy-k8s-ssa.sh              │   │
│  │  6  kubectl rollout status deployment/app -n "$NAMESPACE" \       │   │
│  │  7    --timeout=300s                                              │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  -- Discussion (2) ──────────────────────────────────────────────────    │
│  @carol · 3h ago                                                         │
│  We should add a `--prune` flag for cleaning up removed resources.       │
│  Anyone want to add that?                                                │
│                                                                          │
│  @alice · 1h ago                                                         │
│  Done — see v4 of this snippet. Added `--prune` as an optional flag.     │
│                                                                          │
│  [Add comment...]                                                        │
└──────────────────────────────────────────────────────────────────────────┘
```

**Comment types:**

- **Inline comments** — attached to a specific line (like GitHub PR reviews)
- **General comments** — discussion below the snippet
- **Version comments** — comments on a specific version (useful for code
  reviews of snippet changes)

Comments are lightweight — no threading beyond one level of replies. Comments
on deleted lines are preserved but marked as "on removed code".

#### Snippet Forking

Fork public or shared snippets into your own library — like GitHub fork but
for snippets.

```bash
stash fork <snippet-id>                    # fork to your library
stash fork <snippet-id> --project infra    # fork into a project

# In the UI:
# [Fork] button on any viewable snippet
# Creates a copy with a `derived_from` link to the original
```

**Behavior:**

- Forking creates a **new snippet** in your library with full ownership
- A `derived_from` link is automatically created to the original
- The original author sees "forked N times" on their snippet
- Fork stays independent — no auto-sync with the original (but the link
  enables manual comparison)
- Federated forking works: fork snippets from other instances

#### Offline Mode (CLI + PWA)

Work with snippets without an internet connection.

**CLI offline mode:**

```bash
stash sync                            # download all snippets locally
stash sync --project infra            # sync only a project
stash sync --labels k8s,deploy        # sync by labels

# While offline:
stash search "deploy"                 # searches local cache
stash get <id>                        # reads from local cache
stash push script.sh --title "New"    # queued, pushed on reconnect

stash status --sync                   # show sync status
#  Local:  247 snippets (synced 10 min ago)
#  Queued: 2 changes pending upload
```

**PWA (Progressive Web App):**

- Service worker caches the application shell and recent snippets
- Offline search works against locally cached snippets
- New snippets and edits are queued and synced on reconnect
- Conflict resolution on reconnect (same as git-like CLI workflow)

#### Data Model Extension (V3)

```sql
-- Comments
CREATE TABLE comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  snippet_id UUID NOT NULL REFERENCES snippets(id) ON DELETE CASCADE,
  version_number INT,             -- NULL for general comments, version for inline
  line_number INT,                -- NULL for general comments
  parent_id UUID REFERENCES comments(id) ON DELETE CASCADE,  -- one level threading
  author_id UUID NOT NULL REFERENCES auth.users(id),
  body TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_comments_snippet ON comments(snippet_id, created_at);

-- Snippet templates
ALTER TABLE snippets ADD COLUMN is_template BOOLEAN DEFAULT false;
ALTER TABLE snippets ADD COLUMN template_params JSONB;
-- template_params: [{ "name": "...", "type": "...", "default": "..." }]

-- Snippet analytics (aggregated, not per-event)
CREATE TABLE snippet_stats (
  snippet_id UUID PRIMARY KEY REFERENCES snippets(id) ON DELETE CASCADE,
  view_count INT DEFAULT 0,
  copy_count INT DEFAULT 0,
  fork_count INT DEFAULT 0,
  share_access_count INT DEFAULT 0,
  last_used_at TIMESTAMPTZ,
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Webhooks
CREATE TABLE webhooks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  owner_id UUID NOT NULL REFERENCES auth.users(id),
  url VARCHAR NOT NULL,
  secret VARCHAR NOT NULL,
  events JSONB NOT NULL,           -- ["snippet.created", "snippet.updated"]
  filters JSONB DEFAULT '{}',     -- { "labels": ["production"], "projects": ["uuid"] }
  active BOOLEAN DEFAULT true,
  last_triggered_at TIMESTAMPTZ,
  failure_count INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Offline sync tracking
CREATE TABLE sync_cursors (
  user_id UUID NOT NULL REFERENCES auth.users(id),
  device_id VARCHAR NOT NULL,
  last_synced_at TIMESTAMPTZ NOT NULL,
  sync_scope JSONB DEFAULT '{}',  -- { "projects": [...], "labels": [...] }
  PRIMARY KEY (user_id, device_id)
);
```

---

### V3 Candidates (Additional)

1. **MCP Server** — Expose Stash as an MCP (Model Context Protocol) tool
   server, so AI coding assistants (Claude, Cursor, Copilot) can search and
   insert snippets directly during conversations
2. **Snippet Dependencies** — Declare that snippet A requires snippet B to
   run; `stash get --resolve` downloads the full dependency tree
3. **Multi-Language Transpilation** — AI-powered: "Show me this Python
   snippet in Go" with one click
4. **Scheduled Snippets** — Cron-like execution of bash/Python snippets
   directly from Stash (e.g., health checks, cleanup scripts)
5. **Snippet Marketplace** — Public registry of verified, high-quality
   snippet collections (like npm but for reusable code fragments)
6. **AI Code Review** — Automated review of snippet quality (error handling,
   edge cases, security) with inline suggestions
7. **Interactive Tutorials** — Chain snippets into step-by-step tutorials
   with explanatory text between each snippet (like Jupyter notebooks for
   any language)

---

### V3 Implementation Roadmap

### Phase 5: V3 Intelligence

| Step | Milestone |
|------|-----------|
| 5.1 | pgvector extension, embedding pipeline (Edge Function), semantic search |
| 5.2 | Hybrid search (RRF fusion of FTS + vector), search UI update |
| 5.3 | AI auto-tagging, description generation, duplicate detection |
| 5.4 | Snippet explanation, documentation generation, "Find Similar" |
| 5.5 | Smart suggestions engine (clipboard, git context, usage patterns) |

### Phase 6: V3 Ecosystem

| Step | Milestone |
|------|-----------|
| 6.1 | VS Code extension (save, search, insert, sidebar) |
| 6.2 | Browser extension (Chrome/Firefox — save from any page) |
| 6.3 | JetBrains plugin, Neovim plugin |
| 6.4 | Raycast extension, Alfred workflow |
| 6.5 | Webhook system, GitHub App, Slack integration |
| 6.6 | MCP server for AI coding assistants |

### Phase 7: V3 Advanced DX

| Step | Milestone |
|------|-----------|
| 7.1 | Snippet templates & generators (parameterized snippets) |
| 7.2 | Snippet playground (WASM-based execution for JS/Python/Bash/SQL) |
| 7.3 | Comments & discussions (inline + general) |
| 7.4 | Snippet forking with provenance tracking |
| 7.5 | Analytics & insights dashboard (personal + team) |
| 7.6 | Offline mode (CLI sync + PWA) |
