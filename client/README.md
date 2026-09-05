# DevPilot Client

The DevPilot client is a Next.js 16 (App Router) frontend for chatting with your GitHub repositories. It handles GitHub login, lets you pick and index a repo, and provides a streaming chat UI with inline code citations.

---

## What it does

1. **Landing page** (`/`) — marketing/entry page with a "Sign in" link.
2. **Login** (`/login`) — redirects the browser to the backend's GitHub OAuth flow (`getGithubLoginUrl()` → `{API}/oauth2/authorization/github`).
3. **OAuth callback** (`/auth/callback`) — lands here after GitHub login succeeds on the backend; marks the client as authenticated (sets a lightweight `devpilot_auth` cookie used only for route-guarding) and redirects into the app.
4. **Dashboard** (`/dashboard`, `/dashboard/overview`, `/dashboard/settings`) — lists the user's GitHub repositories (synced live from GitHub via the backend), shows indexing status (`PENDING/INDEXING/READY/FAILED`) with progress, and lets the user kick off indexing.
5. **Chat** (`/chat/[repoId]`) — the main experience: a sidebar of chat sessions for that repo, a message thread rendered as streaming markdown, and a composer. Sending a message opens a Server-Sent Events (SSE) connection to the backend and renders the assistant's reply token-by-token, along with citation chips linking back to the exact file/line ranges the model used.

A Next.js middleware-style `proxy.ts` guards `/dashboard/*` and `/chat/*`, redirecting unauthenticated visitors to `/login` (and vice versa), based on the `devpilot_auth` cookie set after a successful `/api/auth/me` check.

---

## Tech stack

- **Next.js 16** (App Router) + **React 19** + **TypeScript**
- **Tailwind CSS 4** + **shadcn/ui** component primitives (`components/ui/*`)
- **TanStack Query** for server-state (repos, chat sessions, messages)
- **Streamdown** for rendering streaming markdown in chat
- **lucide-react** / **react-icons** for icons, **recharts** for any charts, **embla-carousel** for carousels

---

## Repository structure

```
client/
├── app/
│   ├── page.tsx                    # Landing page
│   ├── login/page.tsx              # Login page → GitHub OAuth redirect
│   ├── auth/callback/page.tsx      # Post-OAuth landing, marks session as authed
│   ├── dashboard/
│   │   ├── page.tsx                # Repo list / dashboard shell
│   │   ├── overview/page.tsx       # Overview view
│   │   └── settings/page.tsx       # Settings view
│   ├── chat/[repoId]/page.tsx      # Chat UI for a specific repository
│   └── layout.tsx, globals.css
├── components/
│   ├── chat/         # chat-view, chat-messages, chat-composer, chat-sidebar,
│   │                 # chat-markdown (Streamdown wrapper), citation-chips, indexing-state
│   ├── dashboard/    # dashboard-header, repo-card, repo-status, repo-dashboard,
│   │                 # overview-dashboard, settings-dashboard, index-error-alert, language-badge
│   ├── layout/       # app-shell (nav/brand)
│   ├── providers/    # query-provider (TanStack Query), require-auth, theme-provider
│   ├── icons/        # devpilot-icon, github-icon, language-icon
│   └── ui/           # shadcn/ui primitives (button, dialog, sidebar, table, toast, etc.)
├── hooks/
│   ├── use-auth.ts    # useCurrentUser, useLogout, auth cookie helpers
│   ├── use-repos.ts   # list/get repos, start indexing, poll index status
│   ├── use-chat.ts    # chat sessions/messages queries + useStreamChat (SSE)
│   └── use-mobile.ts
├── lib/
│   ├── api.ts          # typed fetch wrapper + all REST calls to the backend
│   ├── stream-chat.ts   # parses the backend's SSE stream (user_message/token/assistant_message/done)
│   ├── query-keys.ts     # TanStack Query key factory
│   └── utils.ts
├── proxy.ts             # route guard: redirects based on the devpilot_auth cookie
├── next.config.ts, tsconfig.json, postcss.config.mjs, eslint.config.mjs
└── package.json
```

---

## Prerequisites

| Tool | Minimum version |
|---|---|
| Node.js | 18+ |
| A package manager | npm, pnpm, or bun (the repo ships a `bun.lock`, but any of the three works) |
| The DevPilot backend | running and reachable (see `../backend/README.md`) |

---

## Environment variables

Create `client/.env.local`:

| Variable | Description | Example |
|---|---|---|
| `NEXT_PUBLIC_API_BASE_URL` | Base URL of the DevPilot backend API | `http://localhost:8080` |

If unset, the client falls back to `http://localhost:8080` (see `getApiBaseUrl()` in `lib/api.ts`). Because this is a `NEXT_PUBLIC_*` variable, it's baked into the build — set it in your hosting provider's environment settings (e.g. Vercel) before building for production, pointing at your deployed backend's HTTPS URL.

---

## Running locally

```bash
cd client
npm install
npm run dev
```

The app is available at `http://localhost:3000`.

Other scripts:

```bash
npm run build   # production build
npm run start   # run the production build
npm run lint    # eslint
```

---

## How auth and API calls work

- The client never stores a token itself — the backend issues an HTTP-only **session cookie** after GitHub OAuth login. Every request from `lib/api.ts` is made with `credentials: "include"` so that cookie is sent along.
- `useCurrentUser()` calls `GET /api/auth/me`; on success it sets a non-HTTP-only `devpilot_auth=1` cookie purely so `proxy.ts` can make fast routing decisions without an extra network call.
- `useLogout()` calls `POST /api/auth/logout`, clears the auth cookie and cached user, and redirects to `/login`.
- CORS on the backend must allow the client's origin with credentials enabled (see `CORS_ALLOWED_ORIGIN` in the backend's environment variables) or cookies won't be sent/accepted cross-origin.

## How chat streaming works

`streamChatMessage()` in `lib/stream-chat.ts` POSTs the user's message to `/api/chat/sessions/{id}/messages` and manually reads the `fetch` response body as a stream, parsing SSE frames (`event: ...` / `data: ...` separated by blank lines) rather than using `EventSource` (which doesn't support POST bodies or credentials configuration as flexibly). It dispatches four event types back to `useStreamChat()`:

- `user_message` — the persisted user message (used to reconcile optimistic UI)
- `token` — appended to the in-progress assistant reply as it streams in
- `assistant_message` — the final persisted assistant message, including citations
- `done` — marks the stream complete

`components/chat/citation-chips.tsx` renders each citation's file path and line range so the user can see exactly which parts of the codebase the answer was grounded in.