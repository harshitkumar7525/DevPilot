# DevPilot Backend

The DevPilot backend is a Spring Boot API that authenticates users with GitHub, lets them index their repositories into a vector database, and answers questions about that code using retrieval-augmented generation (RAG) over OpenAI, streaming the answer back over Server-Sent Events (SSE) with file/line citations.

---

## What it does

1. **GitHub OAuth2 login** — users sign in with GitHub; the backend receives their profile and access token, encrypts the token at rest, and starts a session (cookie-based, not JWT).
2. **Repository sync** — lists the user's GitHub repos and stores/refreshes them locally.
3. **Indexing** — for a chosen repo, walks its full file tree via the GitHub API, filters out anything irrelevant (binaries, lockfiles, `node_modules`, build output, etc.), splits eligible source files into token-sized chunks, embeds each chunk with OpenAI, and writes the vectors into Postgres (`pgvector`) tagged with the repository's ID. Indexing runs asynchronously with progress tracking (`files processed / total`, `chunk count`, status).
4. **Chat (RAG)** — for a question in a chat session: embeds the question, runs a similarity search scoped to that repository's vectors, builds a prompt with the retrieved code chunks as context, and streams the model's token-by-token reply to the client over SSE, alongside a list of citations (file path + line range) derived from the chunks actually used.
5. **Persistence** — chat sessions and messages (with their citations) are stored in Postgres so conversations survive a refresh.

---

## Architecture

```
Client (Next.js)
      │  REST (cookies) + SSE
      ▼
┌───────────────────────────────────────────────┐
│                Spring Boot API                 │
│                                                 │
│  controllers/   AuthController, RepoController,│
│                 ChatController                  │
│  security/      GitHub OAuth2 login,            │
│                 CurrentUser (session → User)    │
│  services/                                      │
│   ├─ RepoService       repo sync / ownership     │
│   ├─ UserService        token encrypt/decrypt    │
│   ├─ ChatService        session + message CRUD    │
│   ├─ ai/                RAG: retrieval, prompt     │
│   │                     building, SSE streaming,   │
│   │                     citation mapping           │
│   ├─ indexing/          file tree walk, filtering, │
│   │                     chunking, embedding         │
│   └─ github/            GitHub REST client +        │
│                         rate limiting                │
└──────────────┬─────────────────────┬────────────────┘
               │                     │
        ┌──────▼──────┐      ┌───────▼────────┐
        │ GitHub API   │      │  OpenAI API     │
        │ (repos, tree,│      │ (embeddings +   │
        │  file blobs) │      │  chat model)    │
        └──────────────┘      └────────────────┘
               │
        ┌──────▼───────────────────────┐
        │  PostgreSQL 16 + pgvector     │
        │  - users, repositories,       │
        │    chat_sessions, chat_messages│
        │  - vector_store (Spring AI)    │
        └────────────────────────────────┘
```

---

## Repository structure

```
backend/
├── src/main/java/devPilot/backend/
│   ├── BackendApplication.java
│   ├── config/
│   │   ├── SecurityConfig.java     # OAuth2 login, session/cookie auth, logout, route protection
│   │   ├── CorsConfig.java         # CORS allow-list (comma-separated origins)
│   │   ├── CryptoConfig.java       # TextEncryptor bean for encrypting GitHub tokens at rest
│   │   └── AppConfig.java
│   ├── controllers/
│   │   ├── AuthController.java     # /api/auth — login-url, current user
│   │   ├── RepoController.java     # /api/repos — list/sync, get, index, index status
│   │   └── ChatController.java     # /api/chat — sessions, messages, SSE streaming
│   ├── entity/                     # JPA entities: User, Repository, ChatSession, ChatMessage, enums
│   ├── dto/                        # Request/response payloads
│   ├── repository/                 # Spring Data JPA repositories
│   ├── security/
│   │   ├── GithubOAuth2UserService.java  # maps GitHub profile → local User, stores encrypted token
│   │   ├── AppUserPrincipal.java
│   │   └── CurrentUser.java              # resolves the authenticated User from the session
│   ├── services/
│   │   ├── UserService.java, RepoService.java, ChatService.java
│   │   ├── ai/            # ChatPromptBuilder, CodeContextRetriever, ChatStreamHandler,
│   │   │                  # CitationMapper, RagSettings, RetrievedContext
│   │   ├── indexing/      # IndexingService, CodeChunker, CodeFileFilter
│   │   └── github/        # GithubApiClient, GitHubRateLimiter
│   └── exceptions/        # BadRequestException, NotFoundException, UnauthorizedException, GlobalExceptionHandler
├── src/main/resources/application.properties
├── pom.xml
├── mvnw / mvnw.cmd
└── .env.example
```

---

## Prerequisites

| Tool | Minimum version |
|---|---|
| Java (JDK) | 25 |
| Maven | bundled via `./mvnw` |
| PostgreSQL | 16, with the `pgvector` extension enabled |
| GitHub OAuth App | client ID + secret |
| OpenAI API key | used for both embeddings and chat completion |

> The project root's `docker-compose.yml` starts a ready-to-use Postgres + pgvector instance — see the top-level README's Quick Start.

---

## Environment variables (`backend/.env`)

| Variable | Description | Example |
|---|---|---|
| `DB_URL` | JDBC URL for Postgres | `jdbc:postgresql://localhost:5432/devpilot` |
| `DB_USERNAME` | Postgres username | `postgres` |
| `DB_PASSWORD` | Postgres password | `postgres` |
| `OPENAI_API_KEY` | OpenAI API key (chat + embeddings) | `sk-...` |
| `GITHUB_CLIENT_ID` | GitHub OAuth App client ID | |
| `GITHUB_CLIENT_SECRET` | GitHub OAuth App client secret | |
| `ENCRYPTION_PASSWORD` | Password used to encrypt stored GitHub access tokens | long random string |
| `ENCRYPTION_SALT` | Hex salt used alongside `ENCRYPTION_PASSWORD` | hex string |
| `FRONTEND_URL` | Where to redirect after OAuth login/failure | `http://localhost:3000` |
| `CORS_ALLOWED_ORIGIN` | Comma-separated list of allowed CORS origins | `http://localhost:3000` |

All of these have local-dev fallbacks baked into `application.properties`, but a real `OPENAI_API_KEY`, `GITHUB_CLIENT_ID`/`GITHUB_CLIENT_SECRET`, and encryption values are required for login and chat to actually work.

**GitHub OAuth App callback URL:** `http://localhost:8080/login/oauth2/code/github` (in production, swap in your deployed backend domain).

The GitHub OAuth scope requested is `read:user,repo` (see `application.properties`), so indexing can read both public and private repos the user grants access to.

---

## Running locally

```bash
cd backend
cp .env.example .env
# fill in the values above

./mvnw spring-boot:run
```

The API listens on `http://localhost:8080` by default.

To build a runnable jar instead:

```bash
./mvnw clean package
java -jar target/backend-0.0.1-SNAPSHOT.jar
```

---

## API reference

Session-based auth: after GitHub OAuth login, the backend sets a session cookie. All `/api/**` routes (other than `/api/auth/login-url`) require an authenticated session; the client must send requests with `credentials: "include"`.

### Auth (`/api/auth`)

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/auth/login-url` | No | Returns `{ "url": "/oauth2/authorization/github" }` — where the client should redirect the browser to start GitHub login |
| GET | `/api/auth/me` | Yes | Returns the current user's profile (`id`, `githubId`, `githubUsername`, `displayName`, `avatarUrl`) |
| POST | `/api/auth/logout` | Yes | Ends the session and clears the session cookie (handled by Spring Security's logout filter) — returns `204` |

GitHub OAuth entry points (handled by Spring Security, not `AuthController`):
- `GET /oauth2/authorization/github` — kicks off the OAuth redirect to GitHub
- `GET /login/oauth2/code/github` — OAuth callback; on success redirects the browser to `{FRONTEND_URL}/auth/callback`, on failure to `{FRONTEND_URL}/login?error=oauth_failed`

### Repositories (`/api/repos`)

| Method | Path | Auth | Description |
|---|---|---|---|
| GET | `/api/repos?refresh=true` | Yes | Lists the user's repos. When `refresh=true` (default), re-syncs from GitHub first; `refresh=false` returns what's stored locally |
| GET | `/api/repos/{id}` | Yes | Get a single repo (must be owned by the caller) |
| POST | `/api/repos/{id}/index` | Yes | Kicks off (async) indexing of the repo; returns `202 Accepted` with the repo's updated status |
| GET | `/api/repos/{id}/status` | Yes | Returns indexing progress: `indexStatus`, `filesTotal`, `filesProcessed`, `chunkCount`, `indexedAt`, `errorMessage` |

`indexStatus` is one of `PENDING | INDEXING | READY | FAILED`.

### Chat (`/api/chat`)

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/chat/sessions` | Yes | Create a chat session for a repository (`{ repositoryId, title? }`) |
| GET | `/api/chat/sessions?repositoryId=...` | Yes | List chat sessions for a repository |
| GET | `/api/chat/sessions/{id}` | Yes | Get all messages in a session |
| POST | `/api/chat/sessions/{id}/messages` | Yes | Send a message; response is `text/event-stream` (SSE) |

**SSE event sequence for `POST /api/chat/sessions/{id}/messages`:**

| Event name | Payload | Meaning |
|---|---|---|
| `user_message` | the saved user `ChatMessage` | Confirms the user's message was persisted |
| `token` | a string | One streamed token of the assistant's reply |
| `assistant_message` | the saved assistant `ChatMessage` (with `citations`) | The complete, persisted assistant reply |
| `done` | `"[DONE]"` | Stream is finished |

Citations are `{ filePath, startLine, endLine, language }`, one per distinct code chunk that was retrieved and actually used as context for the answer.

---

## Data models

### User
```json
{ "id": "uuid", "githubId": 12345, "githubUsername": "string", "displayName": "string", "avatarUrl": "string|null" }
```
(The full entity also stores an encrypted `accessToken` and `tokenScopes`, never exposed via the API.)

### Repository
```json
{
  "id": "uuid",
  "githubRepoId": 12345,
  "owner": "string",
  "name": "string",
  "fullName": "owner/name",
  "isPrivate": false,
  "defaultBranch": "main",
  "language": "string|null",
  "htmlUrl": "string|null",
  "description": "string|null",
  "indexStatus": "PENDING|INDEXING|READY|FAILED",
  "indexedAt": "datetime|null",
  "chunkCount": 0,
  "filesTotal": 0,
  "filesProcessed": 0,
  "errorMessage": "string|null"
}
```

### ChatSession
```json
{ "id": "uuid", "repositoryId": "uuid", "title": "string", "createdAt": "datetime" }
```

### ChatMessage
```json
{
  "id": "uuid",
  "role": "USER|ASSISTANT",
  "content": "string",
  "citations": [ { "filePath": "string", "startLine": 0, "endLine": 0, "language": "string" } ],
  "createdAt": "datetime"
}
```

---

## Indexing pipeline details

- **File discovery**: `GithubApiClient` fetches the repo's git tree for its default branch; `CodeFileFilter` excludes build/dependency directories (`node_modules`, `.git`, `dist`, `build`, `target`, `.next`, `vendor`, `__pycache__`, `.idea`, `.vscode`, `coverage`, `out`), lockfiles, dotfiles, and anything over `app.indexing.max-file-bytes` (default 100 KB), keeping a broad allow-list of common source/config extensions (Java, Kotlin, TS/JS, Python, Go, Rust, Ruby, PHP, C/C++/C#, Swift, Markdown, YAML/JSON/TOML/XML, SQL, shell scripts, Dockerfiles, HTML/CSS/Vue/Svelte, etc).
- **Chunking**: `CodeChunker` prefixes each file with a `// File: <path>` header and splits it into ~`app.indexing.chunk-size` character chunks (default 800) using Spring AI's token splitter, tagging each chunk with `repoId`, `filePath`, `language`, and `chunkIndex` metadata.
- **Rate limiting**: `GitHubRateLimiter` pauses briefly (`app.github.api-delay-ms`) between file fetches to stay within GitHub's API limits.
- **Embedding + storage**: chunks are embedded in batches of 32 and written to the Spring AI `VectorStore` (pgvector, 1536 dimensions, HNSW index, cosine distance), scoped by `repoId` metadata so searches never leak across repositories or users.
- **Progress**: the `Repository` row is updated every 5 files (and on completion) with `filesProcessed`/`filesTotal`/`chunkCount`, so the frontend can poll `/api/repos/{id}/status` for a live progress bar.
- **Re-indexing**: starting a new index run first deletes existing vectors for that `repoId` before re-embedding, so it's safe to re-run after the repo changes.

## RAG / chat pipeline details

- `CodeContextRetriever` embeds the incoming question and runs a similarity search against the vector store, filtered to the current repository's `repoId`, returning the top 8 chunks (`RagSettings.TOP_K_CHUNKS`).
- `ChatPromptBuilder` assembles a system prompt instructing the model to answer only from the provided code context, cite file paths/line ranges, and say when it's unsure — plus a user prompt containing the retrieved context and the actual question.
- `ChatStreamHandler` calls the OpenAI chat model (`gpt-4o-mini` by default) via Spring AI's `ChatClient`, streaming tokens to the browser over an `SseEmitter` (180s timeout), and persists the completed assistant message with its citations once the stream completes.