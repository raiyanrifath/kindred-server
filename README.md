# Kindred Server

REST API for **Kindred** (*Small Acts. Stronger Communities.*), a community platform for discovering, organizing and joining local volunteering and social-service events in Bangladesh.

The server stores events and event sign-ups in MongoDB. It trusts only **JSON Web Tokens issued by Better Auth** (which runs in the [Kindred client](../kindred-client)), and it checks each token's signature with the public keys (JWKS) that Better Auth publishes.

- **Live API:** _not deployed yet. Add your Render URL here after deployment (see [Deployment](#deployment-render))._
- **Frontend repository:** `kindred-client`

## Technologies

| Purpose | Technology |
| --- | --- |
| Runtime / language | Node.js 20.9+ and TypeScript |
| Web framework | Express 5 |
| Database | MongoDB (official `mongodb` driver), MongoDB Atlas in production |
| Token verification | `jose`: `createRemoteJWKSet` + `jwtVerify` (EdDSA) |
| Validation | `zod` |
| Security headers / CORS | `helmet`, `cors` |
| Tests | Vitest, Supertest, `mongodb-memory-server` (temporary in-memory database) |
| Hosting | Render (see `render.yaml`) |

## How authentication works

```
Browser ── login ──▶ Next.js + Better Auth ──▶ MongoDB (user, session, account, jwks)
   │                         │
   │  authClient.token()     │  issues a 15-minute JWT signed with a private EdDSA key
   ◀─────────────────────────┘
   │
   │  Authorization: Bearer <JWT>
   ▼
Express API ── downloads public keys from BETTER_AUTH_URL/api/auth/jwks (cached)
   │           verifies signature, issuer, audience and expiry with jose
   │           reads the user id / email / name only from the verified claims
   ▼
MongoDB (events, joinedEvents)
```

* No token, or an invalid, tampered or expired token: **401 Unauthorized**.
* A valid token, but the event belongs to someone else: **403 Forbidden**.
* The JWKS cannot be downloaded (frontend down or wrong URL): **503 AUTH_UNAVAILABLE**.
* `creatorId`, `creatorEmail`, `participantCount` and other server-owned fields sent by a browser are ignored.

## API overview

Every response uses the same JSON shape:

```jsonc
// success
{ "success": true, "message": "optional", "data": { /* ... */ }, "pagination": { /* lists only */ } }
// error
{ "success": false, "error": { "code": "VALIDATION_ERROR", "message": "...", "details": { "title": "..." } } }
```

### Public endpoints

| Method | Path | Description |
| --- | --- | --- |
| GET | `/health` | Server and database status |
| GET | `/api/events/recent?limit=6` | Next upcoming events, soonest first (MongoDB `.limit(6)`) |
| GET | `/api/events?search=&category=&location=&page=1&limit=9` | All **upcoming** events with search, filters and pagination, sorted by date |
| GET | `/api/events/:id` | Details of one event, including organizer and participant count |
| GET | `/api/stats` | Live numbers for the home page |

### Protected endpoints (`Authorization: Bearer <JWT>`)

| Method | Path | Description | Errors |
| --- | --- | --- | --- |
| POST | `/api/events` | Create an event (creator taken from the token) | 400, 401 |
| GET | `/api/events/mine` | Events created by me | 401 |
| PATCH | `/api/events/:id` | Update my event | 400, 401, **403**, 404 |
| DELETE | `/api/events/:id` | Delete my event and its sign-ups | 401, **403**, 404 |
| POST | `/api/events/:id/join` | Join an upcoming event | 400 (past or own event), 401, 404, **409** (already joined) |
| GET | `/api/events/:id/participation` | Have I joined / am I the organizer? | 401, 404 |
| GET | `/api/joined-events` | Events I joined, sorted by event date | 401 |
| GET | `/api/users/me` | My profile and activity counts | 401 |
| PATCH | `/api/users/me` | Update my name and profile picture | 400, 401 |

Categories: `Cleanup`, `Plantation`, `Donation`, `Food Distribution`, `Blood Donation`, `Education Support`, `Winter Clothing Distribution`, `Other`.

### Database

| Collection | Owner | Important fields / indexes |
| --- | --- | --- |
| `events` | this API | `_id` (Event ID), `title`, `description`, `category`, `thumbnailUrl`, `location`, `eventDate`, `creatorId`, `creatorEmail`, `creatorName`, `participantCount`, `createdAt`, `updatedAt`. Indexes: `eventDate`, `creatorId + eventDate`, `category + eventDate` |
| `joinedEvents` | this API | `eventId`, `userId`, `userEmail`, `userName`, `joinedAt`. **Unique index on `eventId + userId`** (makes duplicate joins impossible); index on `userId + joinedAt` |
| `user`, `session`, `account`, `verification`, `jwks` | Better Auth | Managed by Better Auth. The API only reads users and updates `name` / `image` |

## Project structure

```
src/
  app.ts              Express app (middleware, routes, error handling)
  server.ts           Startup: env check, MongoDB connection, JWKS, listen
  config/env.ts       Validates environment variables at startup (zod)
  config/db.ts        MongoDB connection, collections and indexes
  middleware/auth.ts  JWT verification with jose (401 / 503)
  middleware/error.ts 404 handler and centralized error handler
  middleware/logger.ts Request logging
  controllers/        Event, joined-event, user and stats logic
  routes/             URL to controller mapping
  validators/         zod schemas for request bodies and queries
  scripts/seed.ts     Optional sample events (npm run seed)
tests/                Automated API tests (in-memory MongoDB, test keys)
scripts/e2e-check.mjs End-to-end check of the whole running stack
```

## Environment setup

1. Copy the example file (PowerShell, inside `kindred-server`):

   ```powershell
   Copy-Item ".env.example" ".env"
   ```

2. Open `.env` and fill in your own values:

| Variable | Example | Meaning |
| --- | --- | --- |
| `PORT` | `5000` | Port the API listens on |
| `NODE_ENV` | `development` | `production` on Render |
| `MONGODB_URI` | `mongodb+srv://user:pass@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority` | Your MongoDB Atlas connection string, **the same as in the client** |
| `MONGODB_DB_NAME` | `kindred` | Database name, **the same as in the client** |
| `CLIENT_URL` | `http://localhost:3000` | Frontend address(es) allowed by CORS (comma separated) |
| `BETTER_AUTH_URL` | `http://localhost:3000` | Where Better Auth runs; JWKS is read from `BETTER_AUTH_URL/api/auth/jwks`. **The same as in the client** |

The server will not start if a value is missing or malformed. It prints which variable is wrong. A complete beginner-friendly walkthrough is in `SETUP_GUIDE.md` in the main project folder.

## Installation and running

```powershell
cd kindred-server
npm install
npm run dev        # development with auto-reload  ->  http://localhost:5000
```

Other commands:

| Command | What it does |
| --- | --- |
| `npm run build` | Compile TypeScript to `dist/` |
| `npm start` | Run the compiled server (production) |
| `npm run typecheck` | TypeScript type check |
| `npm run lint` | ESLint code check |
| `npm test` | Automated tests (55 tests, temporary in-memory MongoDB, never touches your data) |
| `npm run seed` | Add 7 sample events to an empty database (safe to run again) |
| `npm run e2e` | With the client **and** server running: full end-to-end check (creates 2 test accounts) |

Check that it works: open <http://localhost:5000/health>. You should see `"database":"connected"`.

> The first `npm test` downloads a MongoDB test binary (about 80 MB), which can take a minute.

## Deployment (Render)

1. Push this folder to its own GitHub repository.
2. In Render, choose **New > Blueprint** and select the repository (`render.yaml` is detected), or create a **Web Service** with:
   * Build command: `npm ci --include=dev && npm run build`
   * Start command: `npm start`
   * Health check path: `/health`
3. Set the environment variables `MONGODB_URI`, `MONGODB_DB_NAME`, `CLIENT_URL` and `BETTER_AUTH_URL` (both URLs are your Vercel address, without a trailing slash).
4. In MongoDB Atlas > Network Access, allow `0.0.0.0/0` (Render has no fixed IP on the free plan).
5. Put the Render URL into the client's `NEXT_PUBLIC_API_URL` on Vercel and redeploy the client.

The free Render plan sleeps after inactivity, so the first request can take around 30 to 60 seconds.
