<p align="center">
  <img src="frontend/app/favicon.ico" alt="Dev-Blocks Logo" width="120" />
</p>

<h1 align="center">Dev-Blocks</h1>

## Description

Dev-Blocks lets developers write posts in a rich text editor, publish them, and build an audience through
likes, bookmarks and follows. The backend keeps the relational data (users, posts, follows, likes) exact and
consistent in PostgreSQL via Prisma, while Clerk handles authentication and is kept in sync with the database
through a signed webhook — the backend never issues its own passwords or sessions.

🔗 **Live:** [https://dev-blocks.vercel.app/](https://dev-blocks.vercel.app/)

🔗 **Demo Video:** [Watch on Google Drive](https://drive.google.com/file/d/1DebndTOLidLi4mo6EVBM2Xzm2sEtJXdy/view?usp=sharing)

## Tech stack

| Concern                                                   | Tool                                                                     |
| --------------------------------------------------------- | ------------------------------------------------------------------------ |
| Frontend                                                  | Next.js 16 (App Router), React 19, TypeScript                            |
| Styling                                                   | Tailwind CSS 4                                                           |
| Rich text editor                                          | TipTap (code blocks, images, links, highlights)                          |
| Frontend data fetching                                    | TanStack Query (React Query) + Axios                                     |
| Backend                                                   | Node.js 22, Express 5 (ESM, TypeScript), compiled with `tsc`             |
| Relational data (users, posts, follows, likes, bookmarks) | PostgreSQL + Prisma 7                                                    |
| Auth                                                      | Clerk (hosted sign-in/sign-up), kept in sync via a signed Svix webhook   |
| Validation                                                | Zod 4                                                                    |
| Content safety                                            | DOMPurify (server-side, via jsdom) sanitizes post HTML                   |
| Security headers / CORS                                   | Helmet, `cors` restricted to `FRONTEND_URL`                              |
| Rate limiting                                             | express-rate-limit (general, per-create, per-interaction tiers)          |
| Logging                                                   | Winston (console + file transports)                                      |
| Tests                                                     | Vitest + Supertest                                                       |
| Containerization                                          | Docker multi-stage builds + Docker Compose (Postgres, backend, frontend) |

## Architecture

```mermaid
flowchart TB
    User(["User"])

    subgraph FE["Frontend — Next.js (port 3000)"]
        Pages["App Router pages<br/>home, write, post, profile, bookmarks, drafts"]
        RQ["TanStack Query + Axios client"]
    end

    subgraph BE["Backend — Express (port 5000)"]
        MW["helmet, CORS, request logger,<br/>clerkMiddleware, routes, 404, errorHandler"]
        Routes["/api/user, /api/post, /health"]
        Webhook["/api/webhooks/clerk<br/>raw body, Svix-signed"]
    end

    User -->|HTTPS| Pages
    Pages --> RQ
    RQ -->|"Bearer <Clerk session token>"| MW
    MW --> Routes
    Routes -->|Prisma| DB[("PostgreSQL<br/>users, posts, tags, follows,<br/>likes, bookmarks")]
    Clerk["Clerk<br/>hosted auth"] -->|sign-in / sign-up| Pages
    Clerk -->|"user.created / updated / deleted<br/>(signed webhook)"| Webhook
    Webhook -->|upsert / update / delete| DB
    Pages -->|verifies session| Clerk
```

The frontend never talks to the database directly and never manages passwords — Clerk owns authentication
entirely, and the backend trusts its session token. The only thing the backend keeps in its own table is a
denormalized copy of the user (id, email, username, profile fields) so posts, follows and likes can reference
it with normal foreign keys; that copy is created and kept current by the `/api/webhooks/clerk` route, not by
any signup endpoint of its own.

### How a user row gets created

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant C as Clerk (hosted)
    participant W as POST /api/webhooks/clerk
    participant M as PostgreSQL

    U->>C: Signs up (email/password or social)
    C->>C: Creates the Clerk user
    C->>W: user.created webhook (Svix-signed, raw body)
    W->>W: verify svix-id / svix-timestamp / svix-signature
    alt signature invalid
        W-->>C: 400, nothing stored
    else signature valid
        W->>M: upsert User by clerkId (email, username, name, avatar)
        W-->>C: 200 received
    end
    Note over U,M: user.updated and user.deleted follow the same<br/>verify-then-write path, keeping the local User row in sync
```

### Publishing a post

A post starts as a `DRAFT` owned by its author and only becomes visible to everyone once explicitly published.

```mermaid
stateDiagram-v2
    [*] --> DRAFT: POST /api/post/create
    DRAFT --> DRAFT: PUT /api/post/update/:id<br/>(title, content, tags, cover image)
    DRAFT --> PUBLISHED: PATCH /api/post/publish/:id
    DRAFT --> [*]: DELETE /api/post/delete/:id (soft delete)
    PUBLISHED --> [*]: DELETE /api/post/delete/:id (soft delete)
```

Creating or updating a post derives a URL-friendly `slug` from the title (and appends a short random suffix on
a collision), estimates `readTime` from the word count, and the request body's HTML content is sanitized with
DOMPurify before anything is stored — only a fixed allow-list of tags and attributes survives. Deleting a post
sets `deletedAt` instead of removing the row, so it disappears from every query without losing the data.

### Likes, bookmarks and follows

Each is a simple join table (`Like`, `Bookmark`, `Follow`) with a unique constraint on the pair of ids, so the
same action is always a **toggle**: calling the endpoint again undoes it.

```mermaid
flowchart LR
    A["POST /api/post/like/:id"] --> B{"Like row for<br/>(userId, postId) exists?"}
    B -->|yes| C["delete it → { liked: false }"]
    B -->|no| D["create it → { liked: true }"]
```

The same shape applies to `POST /api/post/bookmark/:id` (`Bookmark`) and
`POST /api/user/:id/follow-toggle` (`Follow`), each guarded by its own unique index so a double-click can
never create a duplicate row.

### Request lifecycle

```mermaid
flowchart LR
    In(["Request"]) --> H["helmet"]
    H --> L["request logger"]
    L --> CORS["cors (FRONTEND_URL only)"]
    CORS --> WH{"/api/webhooks/*?"}
    WH -->|yes| RAW["raw body (Svix needs it unparsed)"]
    WH -->|no| JSON["express.json (10mb limit)"]
    RAW --> CK["clerkMiddleware<br/>attaches req.auth()"]
    JSON --> CK
    CK --> RT{"route matches?"}
    RT -->|no| NF["404: AppError.notFound"]
    RT -->|yes| RL["rate limiter<br/>apiLimiter / createLimiter / interactionLimiter"]
    RL --> AU["requireAuth()<br/>protected routes only"]
    AU --> VA["Zod validation middleware"]
    VA --> CT["controller → service → Prisma"]
    CT --> Out(["{ success, data, meta }"])

    AU -. "401" .-> EH
    VA -. "ZodError" .-> EH
    CT -. "throws" .-> EH
    NF --> EH["errorHandler<br/>AppError / ZodError / Prisma errors / unknown"]
    EH --> Err(["{ success: false, error: { message, statusCode, details? } }"])
```

### Data model

```mermaid
erDiagram
    USER ||--o{ POST : writes
    USER ||--o{ LIKE : likes
    USER ||--o{ BOOKMARK : bookmarks
    USER ||--o{ FOLLOW : follows
    USER ||--o{ COMMENT : writes
    USER ||--o{ NOTIFICATION : receives
    USER ||--o{ READING_HISTORY : has
    POST ||--o{ LIKE : "liked by"
    POST ||--o{ BOOKMARK : "bookmarked by"
    POST ||--o{ COMMENT : has
    POST ||--o{ POSTTAG : tagged
    TAG ||--o{ POSTTAG : "applied to"

    USER {
        string id PK
        string email UK
        string username UK
        string clerkId UK
        enum role "USER or ADMIN"
        string bio
        string avatar
    }
    POST {
        string id PK
        string authorId FK
        string title
        string slug UK
        string content "sanitized HTML"
        enum status "DRAFT, PUBLISHED, ARCHIVED"
        int viewCount
        int readTime
        datetime publishedAt
        datetime deletedAt "soft delete"
    }
    TAG {
        string id PK
        string name UK
        string slug UK
    }
    LIKE {
        string userId FK
        string postId FK
    }
    BOOKMARK {
        string userId FK
        string postId FK
    }
    FOLLOW {
        string followerId FK
        string followingId FK
    }
    COMMENT {
        string postId FK
        string userId FK
        string parentId FK "self-reference for replies"
    }
    NOTIFICATION {
        string userId FK
        enum type "NEW_FOLLOWER, POST_LIKED, ..."
        bool read
    }
    READING_HISTORY {
        string userId FK
        string postId FK
        int timeSpent
        int scrollDepth
    }
```

`Like`, `Bookmark` and `Follow` each carry a unique constraint on their pair of foreign keys, which is what
makes the toggle endpoints safe to call concurrently. `Comment`, `Notification` and `ReadingHistory` are
already modeled in the schema but not wired up to an API yet — see [Roadmap](#roadmap).

## Features

- **Authentication** — sign-up/sign-in via Clerk, with the local `User` row created and kept in sync by a
  signed webhook.
- **Rich text editor** — TipTap-based WYSIWYG editor for writing and formatting posts, with cover images.
- **Post lifecycle** — save as draft, edit, publish, and soft-delete; slug and read-time are generated
  automatically, content is sanitized on write.
- **Tags** — up to 5 per post, reused by name across posts.
- **Social interactions** — like, bookmark and follow, all idempotent toggles.
- **Profiles** — bio, avatar, website and social links, with follower/following counts.
- **Search, sort & pagination** — `GET /api/post` supports `search`, `sortBy` (`latest`, `oldest`, `popular`)
  and page/limit, capped at 50 per page.
- **Rate limiting** — tiered limits for general reads, post creation, and like/bookmark actions.
- **Health checks** — liveness (`/health`) and readiness (`/health/ready`, checks DB connectivity) endpoints.

## Getting started

### Prerequisites

- Node.js 22 or newer
- PostgreSQL (local, or via the included `docker-compose.yml`)
- A [Clerk](https://clerk.com) application (publishable key, secret key, and a webhook signing secret)

### 1. Clone the repository

```bash
git clone https://github.com/PraveenUppar/Dev-Blocks.git
cd Dev-Blocks
```

### 2. Environment setup

Create a `.env` file in **`backend/`** and **`frontend/`**.

**`backend/.env`**

```env
PORT=5000
DATABASE_URL=postgresql://user:password@localhost:5432/devblocks
FRONTEND_URL=http://localhost:3000
CLERK_SECRET_KEY=your_clerk_secret_key
CLERK_WEBHOOK_SECRET=your_clerk_webhook_secret
```

**`frontend/.env`**

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=your_clerk_publishable_key
CLERK_SECRET_KEY=your_clerk_secret_key
CLERK_WEBHOOK_SECRET=your_clerk_webhook_secret
NEXT_PUBLIC_BASE_URL=http://localhost:5000
```

In the Clerk dashboard, point a webhook endpoint at
`http://<your-backend-url>/api/webhooks/clerk` (use a tunnel such as ngrok for local development) subscribed
to `user.created`, `user.updated` and `user.deleted`, and copy its signing secret into
`CLERK_WEBHOOK_SECRET`.

### 3. Backend setup

```bash
cd backend
npm install
npx prisma generate
npx prisma migrate dev
npm run dev
```

### 4. Frontend setup

```bash
cd frontend
npm install
npm run dev
```

The backend listens on `http://localhost:5000` (check `GET /health`), the frontend on
`http://localhost:3000`.

### Running with Docker Compose

A `docker-compose.yml` at the project root runs PostgreSQL, the backend and the frontend together. Copy
`.env.example` to `.env`, fill in the Clerk keys, then:

```bash
docker compose up --build
```

## Database schema

![Database Schema](docs/schema.png)

## Screenshots

|       Home Page        |       Post View        |          Editor          |
| :--------------------: | :--------------------: | :----------------------: |
| ![Home](docs/home.png) | ![Post](docs/post.png) | ![Write](docs/write.png) |

|         User Profile         |            Bookmarks            |           Drafts           |
| :--------------------------: | :-----------------------------: | :------------------------: |
| ![Account](docs/account.png) | ![Bookmarks](docs/bookmark.png) | ![Drafts](docs/drafts.png) |

## API reference

All success responses share the shape `{ "success": true, "data": ... }` (list endpoints also include a
`pagination` object). Errors look like
`{ "success": false, "error": { "message", "statusCode", "details"? } }` (`details` on Zod validation
failures). Protected routes read the Clerk session from the `Authorization: Bearer <token>` header.

### Posts — `/api/post`

| Method | Path            | Access      | Notes                                                                               |
| ------ | --------------- | ----------- | ----------------------------------------------------------------------------------- |
| GET    | `/`             | public      | Query: `page`, `limit` (max 50), `search`, `sortBy` (`latest`, `oldest`, `popular`) |
| GET    | `/id/:id`       | public      | Full content by id; includes `isLiked`/`isBookmarked`/`isFollowing` when signed in  |
| GET    | `/slug/:slug`   | public      | Same as above, looked up by slug (SEO-friendly URLs)                                |
| POST   | `/create`       | auth        | `{ title, subtitle, content, tags? }` — creates a `DRAFT`                           |
| GET    | `/draft/:id`    | auth, owner | Fetch one of your own drafts                                                        |
| PUT    | `/update/:id`   | auth, owner | Any of title, subtitle, content, coverImage, tags                                   |
| DELETE | `/delete/:id`   | auth, owner | Soft delete (`deletedAt`)                                                           |
| PATCH  | `/publish/:id`  | auth, owner | `DRAFT` → `PUBLISHED`, sets `publishedAt`                                           |
| POST   | `/like/:id`     | auth        | Toggles a like                                                                      |
| POST   | `/bookmark/:id` | auth        | Toggles a bookmark                                                                  |

### Users — `/api/user`

| Method | Path                   | Access | Notes                                                        |
| ------ | ---------------------- | ------ | ------------------------------------------------------------ |
| GET    | `/profile`             | auth   | The current user (resolved from the Clerk session)           |
| PUT    | `/update`              | auth   | Any of name, bio, avatar, website, twitter, github, linkedin |
| GET    | `/drafts`              | auth   | Your own draft posts, paginated                              |
| GET    | `/bookmarks`           | auth   | Your bookmarked posts, paginated                             |
| POST   | `/:id/follow-toggle`   | auth   | Toggles following that user                                  |
| GET    | `/:username`           | public | Public profile + follower/following/post counts              |
| GET    | `/:username/posts`     | public | That user's published posts, paginated                       |
| GET    | `/:username/followers` | public | Paginated                                                    |
| GET    | `/:username/following` | public | Paginated                                                    |

### Webhooks & health

| Method | Path                  | Notes                                                                          |
| ------ | --------------------- | ------------------------------------------------------------------------------ |
| POST   | `/api/webhooks/clerk` | **Called by Clerk**, not by users. Verified via Svix signature headers, no JWT |
| GET    | `/health`             | Liveness: process uptime                                                       |
| GET    | `/health/ready`       | Readiness: checks the database with `SELECT 1`, reports latency and memory     |

## Project structure

```
backend/
  prisma/schema.prisma       User, Post, Tag, Like, Follow, Bookmark, Comment, Notification, ReadingHistory
  src/
    app.ts                   Express app: helmet, CORS, webhook route, json body, Clerk, routes, error handler
    server.ts                Starts the HTTP server
    config/                  env.ts, logger.ts (Winston), clerkwebhook.ts (Clerk → User sync)
    middleware/               auth via Clerk, validation.ts (Zod), rateLimiter.ts, errorHandler.ts, requestLogger.ts
    routes/ controllers/ services/   post and user features, one file per layer
    errors/                  AppError + HTTP status codes
    utils/                   slug/read-time helpers, DOMPurify content sanitization
    tests/                   Vitest + Supertest suites (health, post, user)
frontend/
  app/
    page.tsx                 Home feed
    write/                   Post editor (TipTap)
    posts/[id]/[slug]/       Post view
    users/[username]/        Public profile
    bookmarks/ drafts/       My bookmarks / my drafts
    notifications/           Placeholder — not implemented yet
    components/              Navbar, PostCard, Editor, FollowButton, AuthSetup, ToastProvider
  lib/                       Axios client, config
docker-compose.yml            Postgres + backend + frontend for local/production-like runs
```

## Testing

```bash
cd backend
npm test             # vitest run
npm run test:coverage
```

Tests cover health checks, and the post and user routes (creation, publishing, likes, bookmarks, follows,
pagination, validation errors) against the configured database.

## Roadmap

From `docs/improvements.txt`:

- Comments on posts (the `Comment` model exists in the schema; no API yet)
- Notification center for follows and likes (the `Notification` model exists; the frontend page is a
  placeholder)
- Reading history tracking
- Faster cold start after the backend has been idle

## Known limitations

- No refresh tokens or session management of its own — entirely delegated to Clerk.
- `Comment`, `Notification` and `ReadingHistory` are modeled in Prisma but have no routes/services yet.
- No Redis or caching layer — every list endpoint hits PostgreSQL directly.
- No background job queue — notification/email-style side effects would need one once comments/notifications
  land.
- Rate limiting is in-memory (per process), so it resets on restart and doesn't share state across multiple
  backend instances.
