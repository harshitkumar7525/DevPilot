# DevPilot

DevPilot is an AI assistant that lets you chat with your own GitHub repositories. Sign in with GitHub, pick a repo, index it, and ask questions in plain English — DevPilot retrieves the relevant code chunks with vector search and answers with inline file/line citations, streamed token-by-token.

---

## 🚀 Demo

| | |
|---|---|
| **Demo video** | [https://youtu.be/JMTr-Yd-FPI](https://youtu.be/JMTr-Yd-FPI) |

---

## How it works

1. **Sign in with GitHub** (OAuth2) — DevPilot stores an encrypted copy of your GitHub access token so it can read your repos on your behalf.
2. **Pick a repository** from your GitHub account on the dashboard.
3. **Index it** — the backend walks the repo's file tree via the GitHub API, filters out binaries/lockfiles/build output, splits the remaining source files into token-sized chunks, embeds them with OpenAI embeddings, and stores the vectors in Postgres (pgvector).
4. **Ask a question** in the chat view — the backend embeds your question, does a similarity search scoped to that repository, stuffs the retrieved chunks into a prompt, and streams the model's answer back over Server-Sent Events, with citations pointing at the exact files/lines used.

```
┌───────────────────────────┐         ┌─────────────────────────────────────────┐
│   client (Next.js 16)      │  HTTP   │   backend (Spring Boot 4 / Java 25)      │
│   React 19 + Tailwind 4    │◄───────►│   REST API + GitHub OAuth2 + SSE chat    │
│   shadcn/ui + Streamdown   │  SSE    │                                           │
└───────────────────────────┘         └───────────────┬───────────────────────────┘
                                                        │
                                        ┌───────────────┼───────────────┐
                                        │                               │
                                 ┌──────▼───────┐              ┌────────▼────────┐
                                 │  GitHub API   │              │  PostgreSQL 16   │
                                 │ (repo + files)│              │  + pgvector      │
                                 └───────────────┘              │  (Spring AI       │
                                                                 │  vector store +   │
                                                                 │  app tables)       │
                                                                 └────────────────────┘
                                                                          ▲
                                                                          │
                                                                 ┌────────┴────────┐
                                                                 │   OpenAI API     │
                                                                 │ (embeddings +    │
                                                                 │  chat model)     │
                                                                 └──────────────────┘
```

---

## Repository structure

```
DevPilot/
├── backend/                 # Spring Boot API — see backend/README.md
│   └── src/main/java/devPilot/backend/
│       ├── controllers/     # Auth, Repo, Chat REST endpoints
│       ├── entity/          # JPA entities (User, Repository, ChatSession, ChatMessage)
│       ├── security/        # GitHub OAuth2 login + current-user resolution
│       ├── services/
│       │   ├── ai/          # Prompt building, RAG retrieval, SSE streaming, citations
│       │   ├── indexing/    # Repo tree walking, file filtering, chunking, embedding
│       │   └── github/      # GitHub API client + rate limiting
│       └── config/          # CORS, Security, encryption config
│
├── client/                  # Next.js frontend — see client/README.md
│   ├── app/                 # Routes: /, /login, /auth/callback, /dashboard, /chat/[repoId]
│   ├── components/          # Chat UI, dashboard UI, shadcn/ui primitives
│   ├── hooks/                # use-auth, use-repos, use-chat (TanStack Query)
│   └── lib/                  # API client, SSE streaming helper, query keys
│
├── docker/postgres/          # Postgres init script (enables the pgvector extension)
└── docker-compose.yml         # One-command Postgres + pgvector for local dev
```

---

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4, shadcn/ui, TanStack Query, Streamdown (markdown streaming) |
| Backend | Java 25, Spring Boot 4, Spring Security (OAuth2 client), Spring AI, Spring Data JPA |
| AI / RAG | OpenAI (`gpt-4o-mini` chat, `text-embedding-3-small` embeddings) via Spring AI, pgvector similarity search |
| Database | PostgreSQL 16 with the `pgvector` extension (app data + vector store share one instance) |
| Auth | GitHub OAuth2 (login + repo access), token stored encrypted at rest |
| Infra | Docker Compose (Postgres), designed to run backend/frontend separately (e.g. Render + Vercel) |

---

## Prerequisites

| Tool | Minimum version |
|---|---|
| Java (JDK) | 25 |
| Node.js | 18+ |
| npm / pnpm / bun | any (client uses `bun.lock`, but npm/pnpm both work) |
| Docker | 24 (for the bundled Postgres + pgvector) |
| GitHub OAuth App | client ID + secret ([create one](https://github.com/settings/developers)) |
| OpenAI API key | for embeddings + chat completions |

---

## Quick start

### 1 — Clone

```bash
git clone <this-repo-url>
cd DevPilot
```

### 2 — Start Postgres (with pgvector)

```bash
docker compose up -d
```

This runs Postgres 16 with the `pgvector` extension pre-enabled, listening on `localhost:5432`, with database `devpilot` / user `postgres` / password `postgres`.

### 3 — Configure and run the backend

```bash
cd backend
cp .env.example .env
# fill in OPENAI_API_KEY, GITHUB_CLIENT_ID, GITHUB_CLIENT_SECRET, ENCRYPTION_PASSWORD
./mvnw spring-boot:run
```

The API listens on `http://localhost:8080`. See [`backend/README.md`](./backend/README.md) for full environment variable details.

### 4 — Configure and run the client

```bash
cd ../client
echo "NEXT_PUBLIC_API_BASE_URL=http://localhost:8080" > .env.local
npm install
npm run dev
```

The app is available at `http://localhost:3000`. See [`client/README.md`](./client/README.md) for details.

### 5 — Set up your GitHub OAuth App

In your GitHub OAuth App settings, set:
- **Homepage URL**: `http://localhost:3000`
- **Authorization callback URL**: `http://localhost:8080/login/oauth2/code/github`

---

## Learn more

- Backend details (API endpoints, indexing pipeline, RAG flow, environment variables): [`backend/README.md`](./backend/README.md)
- Frontend details (routes, hooks, chat streaming): [`client/README.md`](./client/README.md)