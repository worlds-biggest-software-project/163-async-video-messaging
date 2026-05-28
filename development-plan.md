# Async Video Messaging — Phased Development Plan

> Project: 163-async-video-messaging · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (full-stack) | Video capture relies on browser APIs (MediaRecorder, getDisplayMedia) — TypeScript gives type safety across client capture code, API server, and shared types. Most async video competitors (Loom, Tella) are TypeScript-first. |
| API framework | Next.js 15 (App Router) + tRPC | Next.js provides SSR for the video library/player pages (critical for oEmbed and social previews), API routes for webhooks, and React Server Components for performance. tRPC provides end-to-end type-safe RPC for internal client-server calls without OpenAPI ceremony. A separate REST layer handles public API and webhooks. |
| Database | PostgreSQL 16 | The data model requires relational integrity (FK constraints across organisations, videos, comments, analytics), JSONB for flexible metadata (provider-specific media configs, AI outputs), full-text search (transcript search), and array types (tags). Matches Data Model Suggestion 3 (Hybrid Relational + JSONB). |
| ORM / query builder | Drizzle ORM | Type-safe SQL with first-class JSONB and PostgreSQL array support. Generates migrations from schema definitions. Lighter than Prisma; closer to SQL. |
| Video infrastructure | Mux (primary), Cloudflare Stream (alternative) | Mux provides upload, transcoding, HLS/DASH delivery, thumbnail generation, and analytics via a well-documented API with React components (@mux/mux-player-react, @mux/mux-uploader-react). Cloudflare Stream is the self-hosted-friendly alternative. The JSONB `media.source.provider` field supports swapping providers without schema changes. |
| Transcription / ASR | OpenAI Whisper (large-v3) via API | Best-in-class accuracy for async video transcription. WebVTT output with word-level timestamps. Self-hostable via whisper.cpp for privacy-first deployments. |
| AI summarisation | Claude API (claude-sonnet-4-20250514) | Transcript → summary, show notes, key points, action items, chapter detection. Claude's long context window handles full video transcripts without chunking. |
| Task queue | BullMQ (Redis-backed) | Video processing is inherently async — upload → transcode → transcribe → summarise → chapter-detect is a multi-step pipeline. BullMQ provides job chaining, retry with backoff, progress tracking, and priority queues. Redis also serves as the cache and real-time pub/sub layer. |
| Real-time | WebSocket via Socket.io | Real-time notifications for video-ready events, comment activity, and viewer presence. RFC 6455 compliant. Socket.io provides fallback transports and room-based subscriptions per video. |
| Authentication | NextAuth.js v5 (Auth.js) | Email/password + Google + SAML SSO. JWT-based sessions with signed playback tokens (RFC 7519) for private video access. OAuth 2.0 (RFC 6749) for CRM integrations. |
| Object storage | S3-compatible (AWS S3 / MinIO) | Raw uploads stored in S3 before handoff to Mux/Cloudflare for transcoding. MinIO provides the self-hosted equivalent. |
| Frontend UI | React 19 + Tailwind CSS + shadcn/ui | shadcn/ui provides accessible, unstyled components. Tailwind handles brand customisation. React 19 for streaming SSR and concurrent rendering. |
| Video player | @mux/mux-player-react | HLS/DASH adaptive bitrate playback, WebVTT caption rendering, chapter markers, accessibility features (WCAG 2.2 SC 1.2.2). Falls back to hls.js for self-hosted deployments. |
| Screen capture | MediaRecorder API + getDisplayMedia | W3C MediaStream Recording API for browser-native screen + webcam capture. No browser extensions required. |
| Testing | Vitest (unit/integration) + Playwright (E2E) | Vitest for fast TypeScript unit and integration tests with built-in mocking. Playwright for browser-based recording and playback E2E tests. |
| Code quality | ESLint + Prettier + TypeScript strict mode | ESLint flat config with typescript-eslint. Prettier for formatting. `strict: true` in tsconfig. |
| Containerisation | Docker + docker-compose | Multi-service deployment: Next.js app, PostgreSQL, Redis, MinIO (for self-hosted). Single `docker compose up` for local development. |
| Package manager | pnpm | Monorepo-friendly with workspace support. Faster and more disk-efficient than npm/yarn. |
| Monorepo structure | pnpm workspaces + Turborepo | Shared types, UI components, and utilities across packages. Turborepo for incremental builds and task caching. |

### Project Structure

```
async-video-messaging/
├── package.json                        # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json                          # Turborepo pipeline config
├── docker-compose.yml                  # PostgreSQL, Redis, MinIO, app
├── Dockerfile                          # Production multi-stage build
├── .env.example
├── apps/
│   └── web/                            # Next.js 15 application
│       ├── package.json
│       ├── next.config.ts
│       ├── tailwind.config.ts
│       ├── src/
│       │   ├── app/                    # App Router pages
│       │   │   ├── layout.tsx
│       │   │   ├── page.tsx            # Landing / dashboard
│       │   │   ├── (auth)/
│       │   │   │   ├── login/page.tsx
│       │   │   │   ├── register/page.tsx
│       │   │   │   └── sso/page.tsx
│       │   │   ├── (dashboard)/
│       │   │   │   ├── library/page.tsx
│       │   │   │   ├── video/[id]/page.tsx
│       │   │   │   ├── analytics/page.tsx
│       │   │   │   ├── settings/page.tsx
│       │   │   │   └── workspace/[slug]/page.tsx
│       │   │   ├── (public)/
│       │   │   │   └── v/[shareToken]/page.tsx   # Public video player
│       │   │   └── api/
│       │   │       ├── trpc/[trpc]/route.ts
│       │   │       ├── webhooks/
│       │   │       │   ├── mux/route.ts
│       │   │       │   └── stripe/route.ts
│       │   │       ├── oembed/route.ts
│       │   │       └── v1/                       # Public REST API
│       │   │           ├── videos/route.ts
│       │   │           └── organizations/route.ts
│       │   ├── components/
│       │   │   ├── ui/                 # shadcn/ui components
│       │   │   ├── recorder/           # Screen/webcam capture UI
│       │   │   │   ├── RecordingControls.tsx
│       │   │   │   ├── CameraPreview.tsx
│       │   │   │   ├── ScreenSelector.tsx
│       │   │   │   └── RecordingOverlay.tsx
│       │   │   ├── player/             # Video playback components
│       │   │   │   ├── VideoPlayer.tsx
│       │   │   │   ├── ChapterMarkers.tsx
│       │   │   │   ├── TranscriptPanel.tsx
│       │   │   │   └── CTAOverlay.tsx
│       │   │   ├── comments/           # Threaded comments
│       │   │   ├── library/            # Video library views
│       │   │   └── analytics/          # Charts and heatmaps
│       │   ├── hooks/                  # React hooks
│       │   │   ├── useRecorder.ts
│       │   │   ├── useUpload.ts
│       │   │   └── usePlayer.ts
│       │   ├── lib/                    # Client utilities
│       │   │   ├── trpc.ts
│       │   │   └── auth.ts
│       │   └── styles/
│       │       └── globals.css
│       └── public/
│           └── embed.js                # Embeddable player script
├── packages/
│   ├── db/                             # Database schema + migrations
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── schema/
│   │   │   │   ├── organisations.ts
│   │   │   │   ├── users.ts
│   │   │   │   ├── memberships.ts
│   │   │   │   ├── workspaces.ts
│   │   │   │   ├── videos.ts
│   │   │   │   ├── comments.ts
│   │   │   │   ├── share-links.ts
│   │   │   │   ├── video-views.ts
│   │   │   │   ├── integrations.ts
│   │   │   │   ├── processing-jobs.ts
│   │   │   │   ├── notifications.ts
│   │   │   │   ├── collections.ts
│   │   │   │   └── audit-log.ts
│   │   │   ├── client.ts               # Drizzle client factory
│   │   │   └── migrate.ts              # Migration runner
│   │   └── drizzle/                    # Generated migrations
│   ├── api/                            # tRPC routers + business logic
│   │   ├── package.json
│   │   └── src/
│   │       ├── router.ts               # Root tRPC router
│   │       ├── routers/
│   │       │   ├── video.ts
│   │       │   ├── comment.ts
│   │       │   ├── organisation.ts
│   │       │   ├── workspace.ts
│   │       │   ├── analytics.ts
│   │       │   ├── share.ts
│   │       │   ├── integration.ts
│   │       │   └── notification.ts
│   │       ├── services/
│   │       │   ├── video-service.ts
│   │       │   ├── transcription-service.ts
│   │       │   ├── ai-service.ts
│   │       │   ├── upload-service.ts
│   │       │   ├── analytics-service.ts
│   │       │   └── notification-service.ts
│   │       └── middleware/
│   │           ├── auth.ts
│   │           ├── rate-limit.ts
│   │           └── tenant.ts
│   ├── jobs/                           # BullMQ job processors
│   │   ├── package.json
│   │   └── src/
│   │       ├── worker.ts               # Job worker entry point
│   │       ├── queues.ts               # Queue definitions
│   │       └── processors/
│   │           ├── transcode.ts
│   │           ├── transcribe.ts
│   │           ├── summarise.ts
│   │           ├── chapter-detect.ts
│   │           ├── filler-removal.ts
│   │           ├── thumbnail-generate.ts
│   │           ├── crm-sync.ts
│   │           └── notification-send.ts
│   ├── shared/                         # Shared types + utilities
│   │   ├── package.json
│   │   └── src/
│   │       ├── types/
│   │       │   ├── video.ts
│   │       │   ├── organisation.ts
│   │       │   ├── user.ts
│   │       │   └── analytics.ts
│   │       ├── constants.ts
│   │       ├── errors.ts
│   │       └── utils/
│   │           ├── share-token.ts
│   │           ├── duration.ts
│   │           └── validators.ts
│   └── config/                         # ESLint, TypeScript, Tailwind shared configs
│       ├── eslint/
│       ├── typescript/
│       └── tailwind/
└── tests/
    ├── e2e/                            # Playwright E2E tests
    │   ├── recording.spec.ts
    │   ├── playback.spec.ts
    │   ├── sharing.spec.ts
    │   └── comments.spec.ts
    └── fixtures/                       # Test data
        ├── sample-video.webm
        ├── sample-transcript.vtt
        └── sample-config.json
```

---

## Phase 1: Foundation — Project Scaffolding, Database, and Authentication

### Purpose

Establish the monorepo structure, database schema, authentication system, and core multi-tenancy model. After this phase, a developer can register, log in, create an organisation, and navigate an empty dashboard. Every subsequent phase builds on this foundation without restructuring.

### Tasks

#### 1.1 — Monorepo Scaffolding and Build Pipeline

**What**: Create the pnpm workspace with Turborepo, shared configs, and Docker Compose development environment.

**Design**:

```typescript
// pnpm-workspace.yaml
packages:
  - "apps/*"
  - "packages/*"

// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [".env"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {},
    "typecheck": {},
    "test": {
      "dependsOn": ["^build"]
    },
    "db:migrate": {
      "cache": false
    }
  }
}
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    ports: ["5432:5432"]
    environment:
      POSTGRES_DB: async_video
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev_password
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    ports: ["9000:9000", "9001:9001"]
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - miniodata:/data

volumes:
  pgdata:
  miniodata:
```

```typescript
// packages/shared/src/constants.ts
export const VIDEO_STATUS = {
  UPLOADING: "uploading",
  PROCESSING: "processing",
  READY: "ready",
  FAILED: "failed",
  DELETED: "deleted",
} as const;

export type VideoStatus = (typeof VIDEO_STATUS)[keyof typeof VIDEO_STATUS];

export const RECORDING_TYPE = {
  SCREEN_ONLY: "screen_only",
  CAMERA_ONLY: "camera_only",
  SCREEN_CAMERA: "screen_camera",
} as const;

export type RecordingType =
  (typeof RECORDING_TYPE)[keyof typeof RECORDING_TYPE];

export const VISIBILITY = {
  PRIVATE: "private",
  WORKSPACE: "workspace",
  ORGANISATION: "organisation",
  PUBLIC: "public",
} as const;

export type Visibility = (typeof VISIBILITY)[keyof typeof VISIBILITY];

export const ORG_ROLE = {
  OWNER: "owner",
  ADMIN: "admin",
  MEMBER: "member",
} as const;

export type OrgRole = (typeof ORG_ROLE)[keyof typeof ORG_ROLE];

export const PLAN = {
  FREE: "free",
  BUSINESS: "business",
  ENTERPRISE: "enterprise",
} as const;

export type Plan = (typeof PLAN)[keyof typeof PLAN];
```

**Testing**:
- `Unit: shared constants are exported with correct values and types`
- `Integration: pnpm install succeeds without peer dependency conflicts`
- `Integration: turbo build compiles all packages without errors`
- `Integration: docker compose up starts PostgreSQL, Redis, and MinIO — all reachable on expected ports`
- `Integration: turbo typecheck passes across all packages`
- `Integration: turbo lint passes with zero warnings`

---

#### 1.2 — Database Schema and Migrations

**What**: Implement the Hybrid Relational + JSONB data model (Data Model Suggestion 3) using Drizzle ORM, covering identity, videos, collaboration, analytics, integrations, processing, and audit tables.

**Design**:

```typescript
// packages/db/src/schema/organisations.ts
import { pgTable, uuid, varchar, jsonb, timestamp, boolean } from "drizzle-orm/pg-core";

export const organisations = pgTable("organisations", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 100 }).notNull().unique(),
  plan: varchar("plan", { length: 50 }).notNull().default("free"),
  settings: jsonb("settings").notNull().default({}),
  // settings shape: {
  //   storage_limit_bytes: number;
  //   custom_domain?: string;
  //   branding?: { logo_url: string; accent_color: string; player_theme: string };
  //   retention?: { video_days: number; analytics_days: number; audit_days: number };
  //   features?: { ai_summaries: boolean; crm_sync: boolean; personalisation: boolean };
  //   security?: { sso_provider?: string; enforce_2fa: boolean; ip_allowlist?: string[] };
  // }
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/users.ts
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  name: varchar("name", { length: 255 }).notNull(),
  avatarUrl: text("avatar_url"),
  auth: jsonb("auth").notNull().default({}),
  // auth shape: {
  //   provider: "email" | "google" | "saml";
  //   provider_id?: string;
  //   password_hash?: string;
  //   email_verified: boolean;
  //   mfa_enabled: boolean;
  // }
  preferences: jsonb("preferences").notNull().default({}),
  lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/memberships.ts
export const memberships = pgTable("memberships", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  workspaceIds: uuid("workspace_ids").array().notNull().default([]),
  orgRole: varchar("org_role", { length: 50 }).notNull().default("member"),
  workspaceRoles: jsonb("workspace_roles").notNull().default({}),
  invitedBy: uuid("invited_by").references(() => users.id),
  joinedAt: timestamp("joined_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/videos.ts
export const videos = pgTable("videos", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  workspaceId: uuid("workspace_id").references(() => workspaces.id, { onDelete: "set null" }),
  creatorId: uuid("creator_id").notNull().references(() => users.id),
  title: varchar("title", { length: 500 }),
  status: varchar("status", { length: 50 }).notNull().default("uploading"),
  recordingType: varchar("recording_type", { length: 50 }).notNull().default("screen_camera"),
  visibility: varchar("visibility", { length: 50 }).notNull().default("workspace"),
  durationMs: integer("duration_ms"),
  shareToken: varchar("share_token", { length: 64 }).unique(),
  viewCount: integer("view_count").notNull().default(0),
  commentCount: integer("comment_count").notNull().default(0),
  media: jsonb("media").notNull().default({}),
  ai: jsonb("ai").notNull().default({}),
  sharing: jsonb("sharing").notNull().default({}),
  tags: text("tags").array().notNull().default([]),
  deletedAt: timestamp("deleted_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

Additional tables follow the same pattern for: `workspaces`, `comments`, `share_links`, `video_views`, `integrations`, `crm_sync_log`, `processing_jobs`, `notifications`, `collections`, and `audit_log` — all matching Data Model Suggestion 3's DDL.

Indexes to create:
- `idx_videos_org` on `videos(organisation_id)`
- `idx_videos_workspace` on `videos(workspace_id)`
- `idx_videos_creator` on `videos(creator_id)`
- `idx_videos_status` on `videos(status)` WHERE `deleted_at IS NULL`
- `idx_videos_created` on `videos(created_at DESC)`
- `idx_videos_share_token` on `videos(share_token)` WHERE `share_token IS NOT NULL`
- `idx_videos_tags` GIN index on `videos(tags)`
- `idx_videos_search` GIN on `to_tsvector('english', coalesce(title, '') || ' ' || coalesce(ai->>'summary', ''))`
- All indexes from Data Model Suggestion 3

```typescript
// packages/db/src/client.ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

export function createDb(connectionString: string) {
  const pool = new Pool({ connectionString });
  return drizzle(pool, { schema });
}

export type Database = ReturnType<typeof createDb>;
```

**Testing**:
- `Unit: all Drizzle schema files export valid table definitions — no TypeScript errors`
- `Integration: drizzle-kit generate produces migration SQL without errors`
- `Integration: drizzle-kit migrate runs against empty PostgreSQL — all tables created`
- `Integration: drizzle-kit migrate is idempotent — running twice produces no errors`
- `Unit: createDb returns a valid Drizzle client with all schema tables accessible`
- `Integration: insert + select round-trip for organisations, users, memberships, videos, comments`
- `Integration: FK constraint prevents inserting a video with a non-existent creator_id`
- `Integration: CASCADE delete — deleting an organisation removes its videos`
- `Integration: JSONB fields accept and return structured data (media, ai, sharing)`
- `Integration: GIN index on tags supports @> containment query`

---

#### 1.3 — Authentication and Session Management

**What**: Implement email/password and Google OAuth authentication with NextAuth.js v5, JWT session tokens, and signed video playback tokens.

**Design**:

```typescript
// apps/web/src/lib/auth.ts
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";
import Google from "next-auth/providers/google";
import { DrizzleAdapter } from "@auth/drizzle-adapter";
import { createDb } from "@async-video/db";
import bcrypt from "bcryptjs";
import { z } from "zod";

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: DrizzleAdapter(createDb(process.env.DATABASE_URL!)),
  providers: [
    Credentials({
      credentials: { email: { type: "email" }, password: { type: "password" } },
      async authorize(credentials) {
        const { email, password } = loginSchema.parse(credentials);
        // lookup user, verify bcrypt hash, return user or null
      },
    }),
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  session: { strategy: "jwt", maxAge: 30 * 24 * 60 * 60 }, // 30 days
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.userId = user.id;
        // attach organisationId, orgRole from memberships table
      }
      return token;
    },
    async session({ session, token }) {
      session.user.id = token.userId as string;
      session.user.organisationId = token.organisationId as string;
      session.user.orgRole = token.orgRole as string;
      return session;
    },
  },
});

// Signed playback token for private videos (RFC 7519)
import { SignJWT, jwtVerify } from "jose";

export async function createPlaybackToken(
  videoId: string,
  viewerId: string | null,
  expiresInSeconds = 3600
): Promise<string> {
  const secret = new TextEncoder().encode(process.env.PLAYBACK_TOKEN_SECRET!);
  return new SignJWT({ videoId, viewerId })
    .setProtectedHeader({ alg: "HS256" })
    .setExpirationTime(`${expiresInSeconds}s`)
    .setIssuedAt()
    .sign(secret);
}

export async function verifyPlaybackToken(
  token: string
): Promise<{ videoId: string; viewerId: string | null }> {
  const secret = new TextEncoder().encode(process.env.PLAYBACK_TOKEN_SECRET!);
  const { payload } = await jwtVerify(token, secret);
  return { videoId: payload.videoId as string, viewerId: payload.viewerId as string | null };
}
```

```typescript
// packages/api/src/middleware/auth.ts
import { TRPCError } from "@trpc/server";
import { auth } from "@async-video/web/lib/auth";

export async function requireAuth(ctx: Context) {
  const session = await auth();
  if (!session?.user?.id) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return session.user;
}

// packages/api/src/middleware/tenant.ts
export async function requireOrgMembership(
  ctx: Context,
  organisationId: string
) {
  const user = await requireAuth(ctx);
  const membership = await ctx.db.query.memberships.findFirst({
    where: and(
      eq(memberships.userId, user.id),
      eq(memberships.organisationId, organisationId)
    ),
  });
  if (!membership) {
    throw new TRPCError({ code: "FORBIDDEN", message: "Not a member of this organisation" });
  }
  return { user, membership };
}
```

Environment variables:
```
DATABASE_URL=postgresql://app:dev_password@localhost:5432/async_video
REDIS_URL=redis://localhost:6379
NEXTAUTH_SECRET=<random-32-bytes>
NEXTAUTH_URL=http://localhost:3000
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
PLAYBACK_TOKEN_SECRET=<random-32-bytes>
```

**Testing**:
- `Unit: loginSchema rejects empty email — ZodError with path "email"`
- `Unit: loginSchema rejects password under 8 chars — ZodError with path "password"`
- `Unit: createPlaybackToken generates a valid JWT with videoId and viewerId claims`
- `Unit: verifyPlaybackToken decodes a valid token — returns correct videoId and viewerId`
- `Unit: verifyPlaybackToken rejects an expired token — throws JWTExpired error`
- `Unit: verifyPlaybackToken rejects a token signed with wrong secret — throws JWSSignatureVerificationFailed`
- `Integration (mocked DB): Credentials authorize with valid email+password → returns user object`
- `Integration (mocked DB): Credentials authorize with wrong password → returns null`
- `Integration (mocked DB): requireAuth with no session → throws UNAUTHORIZED`
- `Integration (mocked DB): requireOrgMembership with non-member → throws FORBIDDEN`
- `E2E: register new account with email/password → redirected to dashboard`
- `E2E: login with valid credentials → session cookie set, dashboard loads`
- `E2E: login with invalid password → error message displayed, no session`

---

#### 1.4 — Organisation and Workspace Management

**What**: Implement CRUD operations for organisations, workspaces, and memberships with role-based access control.

**Design**:

```typescript
// packages/api/src/routers/organisation.ts
import { router, protectedProcedure } from "../trpc";
import { z } from "zod";

export const organisationRouter = router({
  create: protectedProcedure
    .input(z.object({
      name: z.string().min(1).max(255),
      slug: z.string().min(3).max(100).regex(/^[a-z0-9-]+$/),
    }))
    .mutation(async ({ ctx, input }) => {
      // 1. Create organisation with plan = 'free'
      // 2. Create default workspace "General"
      // 3. Create membership with role = 'owner'
      // 4. Insert audit_log entry: org.created
      // Returns: { organisation, workspace, membership }
    }),

  get: protectedProcedure
    .input(z.object({ organisationId: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      // Verify membership, return org with member count and storage used
    }),

  update: protectedProcedure
    .input(z.object({
      organisationId: z.string().uuid(),
      name: z.string().min(1).max(255).optional(),
      settings: z.record(z.unknown()).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Require admin or owner role
      // Update fields, insert audit_log entry: org.updated
    }),

  inviteMember: protectedProcedure
    .input(z.object({
      organisationId: z.string().uuid(),
      email: z.string().email(),
      role: z.enum(["admin", "member"]),
    }))
    .mutation(async ({ ctx, input }) => {
      // Require admin role
      // Find or create user by email
      // Create membership, send invitation email
      // Insert audit_log entry: member.invited
    }),

  listMembers: protectedProcedure
    .input(z.object({ organisationId: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      // Return all members with user details and roles
    }),
});

// packages/api/src/routers/workspace.ts
export const workspaceRouter = router({
  create: protectedProcedure
    .input(z.object({
      organisationId: z.string().uuid(),
      name: z.string().min(1).max(255),
      slug: z.string().min(3).max(100).regex(/^[a-z0-9-]+$/),
      description: z.string().max(1000).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Require admin role on organisation
      // Create workspace, add creator to workspace_ids in membership
    }),

  list: protectedProcedure
    .input(z.object({ organisationId: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      // Return workspaces the current user belongs to
    }),
});
```

**Testing**:
- `Unit: org.create with valid name and slug → organisation created with plan "free"`
- `Unit: org.create generates default workspace named "General"`
- `Unit: org.create assigns creator as owner`
- `Unit: org.create with duplicate slug → unique constraint error`
- `Unit: org.inviteMember as member (not admin) → FORBIDDEN`
- `Unit: org.inviteMember as admin → membership created with correct role`
- `Unit: org.inviteMember with email of existing member → error "already a member"`
- `Unit: workspace.create as non-admin → FORBIDDEN`
- `Unit: workspace.create with duplicate slug within org → error`
- `Integration: full flow — create org → create workspace → invite member → list members → verify`
- `E2E: create organisation from dashboard → org appears in navigation`

---

#### 1.5 — Dashboard Shell and Navigation

**What**: Build the authenticated dashboard layout with sidebar navigation, organisation switcher, and empty-state library page.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/layout.tsx
export default async function DashboardLayout({ children }: { children: React.ReactNode }) {
  const session = await auth();
  if (!session) redirect("/login");

  return (
    <div className="flex h-screen">
      <Sidebar session={session} />
      <main className="flex-1 overflow-y-auto">
        <TopBar session={session} />
        {children}
      </main>
    </div>
  );
}

// Sidebar navigation items:
// - Library (video list)
// - Record (new recording)
// - Collections
// - Analytics
// - Settings
//   - Organisation
//   - Workspace
//   - Integrations
//   - Billing
```

Navigation state is driven by URL segments. Organisation context comes from the session JWT claim.

**Testing**:
- `E2E: unauthenticated user visiting /library → redirected to /login`
- `E2E: authenticated user sees sidebar with Library, Record, Collections, Analytics, Settings`
- `E2E: organisation switcher shows all organisations the user belongs to`
- `E2E: library page shows empty state with "Record your first video" prompt`

---

## Phase 2: Video Recording and Upload

### Purpose

Enable browser-based screen and webcam recording using the MediaRecorder API (W3C MediaStream Recording) and upload recordings to S3-compatible storage. After this phase, users can record a video from their browser and see it appear in their library with an "uploading" status.

### Tasks

#### 2.1 — Browser Media Capture

**What**: Implement screen and webcam capture using getDisplayMedia and getUserMedia APIs, compositing them into a single MediaRecorder stream.

**Design**:

```typescript
// apps/web/src/hooks/useRecorder.ts
import { useState, useRef, useCallback } from "react";

interface RecorderState {
  status: "idle" | "requesting" | "recording" | "paused" | "stopped" | "error";
  duration: number;        // current recording duration in ms
  recordingType: RecordingType;
  error: string | null;
}

interface RecorderOptions {
  recordingType: RecordingType;      // screen_only | camera_only | screen_camera
  videoBitsPerSecond?: number;        // default: 2_500_000 (2.5 Mbps)
  audioBitsPerSecond?: number;        // default: 128_000 (128 kbps)
  mimeType?: string;                  // default: "video/webm;codecs=vp9,opus"
  maxDurationMs?: number;             // default: 30 * 60 * 1000 (30 min)
}

interface UseRecorderReturn {
  state: RecorderState;
  startRecording: (options: RecorderOptions) => Promise<void>;
  stopRecording: () => Promise<Blob>;
  pauseRecording: () => void;
  resumeRecording: () => void;
  cancelRecording: () => void;
}

export function useRecorder(): UseRecorderReturn {
  // Implementation flow:
  // 1. Request screen capture: navigator.mediaDevices.getDisplayMedia({ video: true, audio: true })
  // 2. Request camera/mic: navigator.mediaDevices.getUserMedia({ video: true, audio: true })
  // 3. Composite streams using a <canvas> element:
  //    - Draw screen capture full-size
  //    - Draw camera as picture-in-picture circle (bottom-right, 200x200)
  //    - captureStream(30) on canvas → combined video track
  //    - Mix audio tracks using AudioContext.createMediaStreamDestination()
  // 4. Create MediaRecorder on composited stream
  // 5. Collect chunks via ondataavailable → Blob array
  // 6. On stop → concatenate chunks → return single Blob
}

// Compositing helper
function compositeStreams(
  screenStream: MediaStream,
  cameraStream: MediaStream | null,
  canvas: HTMLCanvasElement
): MediaStream {
  const ctx = canvas.getContext("2d")!;
  const screenTrack = screenStream.getVideoTracks()[0];
  const settings = screenTrack.getSettings();
  canvas.width = settings.width ?? 1920;
  canvas.height = settings.height ?? 1080;

  const screenVideo = document.createElement("video");
  screenVideo.srcObject = screenStream;
  screenVideo.play();

  let cameraVideo: HTMLVideoElement | null = null;
  if (cameraStream) {
    cameraVideo = document.createElement("video");
    cameraVideo.srcObject = cameraStream;
    cameraVideo.play();
  }

  // Draw loop at 30fps
  function draw() {
    ctx.drawImage(screenVideo, 0, 0, canvas.width, canvas.height);
    if (cameraVideo) {
      const pipSize = 200;
      const pipX = canvas.width - pipSize - 20;
      const pipY = canvas.height - pipSize - 20;
      // Circular mask for PiP
      ctx.save();
      ctx.beginPath();
      ctx.arc(pipX + pipSize / 2, pipY + pipSize / 2, pipSize / 2, 0, Math.PI * 2);
      ctx.clip();
      ctx.drawImage(cameraVideo, pipX, pipY, pipSize, pipSize);
      ctx.restore();
    }
    requestAnimationFrame(draw);
  }
  draw();

  const videoStream = canvas.captureStream(30);

  // Mix audio from both streams
  const audioCtx = new AudioContext();
  const dest = audioCtx.createMediaStreamDestination();
  [screenStream, cameraStream].filter(Boolean).forEach((s) => {
    s!.getAudioTracks().forEach((track) => {
      const source = audioCtx.createMediaStreamSource(new MediaStream([track]));
      source.connect(dest);
    });
  });

  return new MediaStream([
    ...videoStream.getVideoTracks(),
    ...dest.stream.getAudioTracks(),
  ]);
}
```

**Testing**:
- `Unit: useRecorder initial state is { status: "idle", duration: 0, error: null }`
- `Unit: compositeStreams with screen-only → output stream has 1 video + audio tracks`
- `Unit: compositeStreams with screen+camera → PiP drawn at bottom-right of canvas`
- `Integration (mocked mediaDevices): startRecording("screen_camera") → calls getDisplayMedia + getUserMedia`
- `Integration (mocked mediaDevices): startRecording when getDisplayMedia rejects → state.error set, status "error"`
- `Integration (mocked mediaDevices): stopRecording returns a Blob with correct MIME type`
- `Integration (mocked mediaDevices): recording exceeds maxDurationMs → auto-stops`
- `E2E: user clicks Record → browser permission dialog appears (manual verification)`

---

#### 2.2 — Upload Pipeline (Client → S3)

**What**: Implement multipart upload of recorded video blobs to S3-compatible storage with progress tracking and resume support.

**Design**:

```typescript
// apps/web/src/hooks/useUpload.ts
interface UploadState {
  status: "idle" | "requesting" | "uploading" | "completed" | "error";
  progress: number;       // 0-100
  videoId: string | null;
  error: string | null;
}

interface UseUploadReturn {
  state: UploadState;
  upload: (blob: Blob, metadata: UploadMetadata) => Promise<string>;  // returns videoId
  cancel: () => void;
}

interface UploadMetadata {
  title?: string;
  recordingType: RecordingType;
  workspaceId?: string;
  visibility?: Visibility;
}

export function useUpload(): UseUploadReturn {
  // Flow:
  // 1. POST /api/v1/videos/initiate-upload → { videoId, uploadUrl, uploadId, parts }
  // 2. Split blob into 5MB chunks
  // 3. Upload each chunk to presigned part URL (PUT)
  // 4. Track progress: (completedParts / totalParts) * 100
  // 5. POST /api/v1/videos/complete-upload → { videoId }
  // 6. Server enqueues transcode job
}

// Server-side upload initiation
// packages/api/src/services/upload-service.ts
import { S3Client, CreateMultipartUploadCommand, UploadPartCommand, CompleteMultipartUploadCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

export class UploadService {
  constructor(private s3: S3Client, private bucket: string) {}

  async initiateUpload(
    videoId: string,
    fileSize: number,
    contentType: string
  ): Promise<{ uploadId: string; parts: { partNumber: number; url: string }[] }> {
    const key = `uploads/${videoId}/source.webm`;
    const { UploadId } = await this.s3.send(
      new CreateMultipartUploadCommand({ Bucket: this.bucket, Key: key, ContentType: contentType })
    );

    const partSize = 5 * 1024 * 1024; // 5MB
    const partCount = Math.ceil(fileSize / partSize);
    const parts = await Promise.all(
      Array.from({ length: partCount }, async (_, i) => ({
        partNumber: i + 1,
        url: await getSignedUrl(this.s3,
          new UploadPartCommand({
            Bucket: this.bucket, Key: key, UploadId, PartNumber: i + 1
          }),
          { expiresIn: 3600 }
        ),
      }))
    );

    return { uploadId: UploadId!, parts };
  }

  async completeUpload(
    videoId: string,
    uploadId: string,
    parts: { partNumber: number; eTag: string }[]
  ): Promise<void> {
    const key = `uploads/${videoId}/source.webm`;
    await this.s3.send(
      new CompleteMultipartUploadCommand({
        Bucket: this.bucket, Key: key, UploadId: uploadId,
        MultipartUpload: { Parts: parts.map(p => ({ PartNumber: p.partNumber, ETag: p.eTag })) },
      })
    );
  }
}
```

**Testing**:
- `Unit: upload splits a 12MB blob into 3 chunks (5MB + 5MB + 2MB)`
- `Unit: upload progress updates correctly — 0%, 33%, 66%, 100%`
- `Unit: upload cancel aborts in-flight XHR requests`
- `Integration (mocked S3): initiateUpload → returns uploadId and correct number of presigned URLs`
- `Integration (mocked S3): completeUpload with valid parts → success`
- `Integration (mocked S3): completeUpload with missing part → S3 error propagated`
- `Integration (MinIO): full upload cycle — initiate → upload parts → complete → object exists in bucket`
- `Unit: video record created with status "uploading" after initiate`
- `Unit: video status transitions to "processing" after complete`

---

#### 2.3 — Recording UI

**What**: Build the recording interface with camera preview, screen selector, recording controls (start, pause, stop, cancel), and timer display.

**Design**:

Components:
- `RecordingControls` — Start/pause/stop/cancel buttons, timer, recording type selector
- `CameraPreview` — Live camera feed in a circular frame
- `ScreenSelector` — Browser-native screen picker via getDisplayMedia
- `RecordingOverlay` — Floating widget during recording showing timer and controls

State machine for recording flow:
```
idle → requesting_permissions → previewing → recording → paused → recording → stopped → uploading → done
                                                                                    ↘ cancelled
```

```typescript
// apps/web/src/components/recorder/RecordingControls.tsx
interface RecordingControlsProps {
  onRecordingComplete: (blob: Blob, metadata: UploadMetadata) => void;
}

// Post-recording: title input, workspace selector, visibility toggle, then upload
interface PostRecordingFormData {
  title: string;
  workspaceId: string;
  visibility: Visibility;
}
```

**Testing**:
- `E2E: click "Record" → recording type selector appears (Screen, Camera, Screen+Camera)`
- `E2E: select "Screen + Camera" → browser permission prompt, then camera preview appears`
- `E2E: click Start → timer counts up, controls show Pause and Stop`
- `E2E: click Pause → timer pauses, controls show Resume and Stop`
- `E2E: click Stop → post-recording form appears with title input`
- `E2E: click Cancel during recording → recording discarded, returns to library`
- `E2E: complete recording flow → video appears in library with "uploading" status`

---

## Phase 3: Video Processing Pipeline

### Purpose

Build the asynchronous job pipeline that transforms raw uploads into playable videos with transcriptions, chapters, and summaries. After this phase, recorded videos are automatically transcoded, transcribed, and enriched with AI-generated content. The video status progresses through uploading → processing → ready.

### Tasks

#### 3.1 — BullMQ Job Infrastructure

**What**: Set up BullMQ queues, workers, and job chaining for the video processing pipeline.

**Design**:

```typescript
// packages/jobs/src/queues.ts
import { Queue, FlowProducer } from "bullmq";
import { Redis } from "ioredis";

const connection = new Redis(process.env.REDIS_URL!);

export const videoQueue = new Queue("video-processing", { connection });
export const transcriptionQueue = new Queue("transcription", { connection });
export const aiQueue = new Queue("ai-enrichment", { connection });
export const notificationQueue = new Queue("notifications", { connection });

export const flowProducer = new FlowProducer({ connection });

// Job chaining: transcode → transcribe → [summarise, chapter-detect, filler-removal] → notify
export async function enqueueVideoProcessing(videoId: string) {
  return flowProducer.add({
    name: "video-ready-notify",
    queueName: "notifications",
    data: { videoId, type: "video_ready" },
    children: [
      {
        name: "summarise",
        queueName: "ai-enrichment",
        data: { videoId, task: "summarise" },
        children: [{
          name: "transcribe",
          queueName: "transcription",
          data: { videoId },
          children: [{
            name: "transcode",
            queueName: "video-processing",
            data: { videoId },
          }],
        }],
      },
      {
        name: "chapter-detect",
        queueName: "ai-enrichment",
        data: { videoId, task: "chapter_detect" },
        // depends on transcription completing (shared parent)
      },
    ],
  });
}

// packages/jobs/src/worker.ts
import { Worker } from "bullmq";

// Each queue gets its own worker with concurrency limits
const videoWorker = new Worker("video-processing", processTranscode, {
  connection,
  concurrency: 2, // limit concurrent transcodes
});

const transcriptionWorker = new Worker("transcription", processTranscription, {
  connection,
  concurrency: 4,
});

const aiWorker = new Worker("ai-enrichment", processAiEnrichment, {
  connection,
  concurrency: 4,
});
```

```typescript
// packages/db/src/schema/processing-jobs.ts — job state tracking
export const processingJobs = pgTable("processing_jobs", {
  id: uuid("id").primaryKey().defaultRandom(),
  videoId: uuid("video_id").notNull().references(() => videos.id, { onDelete: "cascade" }),
  jobType: varchar("job_type", { length: 50 }).notNull(),
  // job_type: transcode, transcribe, summarise, chapter_detect, filler_removal, thumbnail_generate
  status: varchar("status", { length: 50 }).notNull().default("queued"),
  // status: queued, processing, completed, failed, cancelled
  provider: varchar("provider", { length: 50 }),
  config: jsonb("config").notNull().default({}),
  result: jsonb("result"),
  progress: integer("progress").default(0),
  error: text("error"),
  startedAt: timestamp("started_at", { withTimezone: true }),
  completedAt: timestamp("completed_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- `Unit: enqueueVideoProcessing creates a flow with correct parent-child dependencies`
- `Unit: flow structure — notify depends on summarise, which depends on transcribe, which depends on transcode`
- `Integration (Redis): enqueue a job → job appears in queue with correct data`
- `Integration (Redis): worker processes a job → job moves from active to completed`
- `Integration (Redis): job failure → moves to failed with error message, retry attempted`
- `Unit: processing_jobs table updated when job starts (status=processing, startedAt set)`
- `Unit: processing_jobs table updated when job completes (status=completed, result set, completedAt set)`

---

#### 3.2 — Video Transcoding via Mux

**What**: Implement the transcode processor that uploads source video to Mux, receives webhook callbacks for completion, and stores HLS/DASH playback URLs.

**Design**:

```typescript
// packages/jobs/src/processors/transcode.ts
import Mux from "@mux/mux-node";

const mux = new Mux({
  tokenId: process.env.MUX_TOKEN_ID!,
  tokenSecret: process.env.MUX_TOKEN_SECRET!,
});

export async function processTranscode(job: Job<{ videoId: string }>) {
  const { videoId } = job.data;

  // 1. Get source file URL from S3
  const video = await db.query.videos.findFirst({ where: eq(videos.id, videoId) });
  const sourceUrl = getPresignedUrl(video!.media.source.storage_key);

  // 2. Create Mux asset from URL
  const asset = await mux.video.assets.create({
    input: [{ url: sourceUrl }],
    playback_policy: ["signed"],
    encoding_tier: "smart",
    mp4_support: "standard",
  });

  // 3. Update video with Mux asset ID (full update comes via webhook)
  await db.update(videos).set({
    media: sql`jsonb_set(media, '{source,asset_id}', '"${asset.id}"')`,
    status: "processing",
  }).where(eq(videos.id, videoId));

  // 4. Store processing job reference
  await db.update(processingJobs).set({
    status: "processing",
    provider: "mux",
    config: { asset_id: asset.id },
    startedAt: new Date(),
  }).where(and(
    eq(processingJobs.videoId, videoId),
    eq(processingJobs.jobType, "transcode"),
  ));

  return { assetId: asset.id };
}

// apps/web/src/app/api/webhooks/mux/route.ts
// Mux webhook handler for asset.ready and asset.errored events
export async function POST(req: Request) {
  const body = await req.text();
  const signature = req.headers.get("mux-signature")!;

  // Verify webhook signature (HMAC-SHA256)
  if (!verifyMuxWebhookSignature(body, signature, process.env.MUX_WEBHOOK_SECRET!)) {
    return new Response("Invalid signature", { status: 401 });
  }

  const event = JSON.parse(body);

  if (event.type === "video.asset.ready") {
    const asset = event.data;
    const playbackId = asset.playback_ids[0].id;
    const hlsUrl = `https://stream.mux.com/${playbackId}.m3u8`;
    const thumbnailUrl = `https://image.mux.com/${playbackId}/thumbnail.jpg`;

    await db.update(videos).set({
      status: "ready",  // or still "processing" if transcription pending
      durationMs: Math.round(asset.duration * 1000),
      media: {
        source: { provider: "mux", asset_id: asset.id },
        playback: {
          hls_url: hlsUrl,
          thumbnail_url: thumbnailUrl,
          playback_id: playbackId,
          mp4_downloads: asset.static_renditions?.files?.reduce(...) ?? {},
        },
        encoding: {
          codec: "h264",
          width: asset.max_stored_resolution === "UHD" ? 3840 : 1920,
          height: asset.max_stored_resolution === "UHD" ? 2160 : 1080,
        },
      },
    }).where(sql`media->'source'->>'asset_id' = ${asset.id}`);
  }

  return new Response("OK", { status: 200 });
}
```

**Testing**:
- `Unit: processTranscode calls Mux API with correct source URL and playback policy`
- `Unit: processTranscode updates video status to "processing"`
- `Unit: processTranscode creates processing_jobs record with Mux asset ID`
- `Integration (mocked Mux API): successful asset creation → returns asset ID`
- `Integration (mocked Mux API): Mux API failure → job fails with error message`
- `Unit: Mux webhook with valid signature → processes event`
- `Unit: Mux webhook with invalid signature → returns 401, no DB changes`
- `Unit: asset.ready webhook → video status updated to "ready", HLS URL stored in media JSONB`
- `Unit: asset.errored webhook → video status updated to "failed", error stored`
- `Integration: full flow — upload completes → transcode job enqueued → Mux asset created → webhook received → video ready`

---

#### 3.3 — AI Transcription (Whisper)

**What**: Implement automatic speech-to-text transcription using OpenAI Whisper API, producing WebVTT captions stored in the video's `ai` JSONB column.

**Design**:

```typescript
// packages/jobs/src/processors/transcribe.ts
import OpenAI from "openai";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function processTranscription(job: Job<{ videoId: string }>) {
  const { videoId } = job.data;
  const video = await db.query.videos.findFirst({ where: eq(videos.id, videoId) });

  // 1. Download audio from source (or extract from Mux asset)
  const audioBuffer = await downloadAudioTrack(video!);

  // 2. Transcribe with Whisper
  const transcription = await openai.audio.transcriptions.create({
    model: "whisper-1",
    file: new File([audioBuffer], "audio.webm", { type: "audio/webm" }),
    response_format: "verbose_json",
    timestamp_granularities: ["word", "segment"],
    language: "en",
  });

  // 3. Convert to WebVTT format
  const webvttContent = segmentsToWebVTT(transcription.segments!);

  // 4. Store in video's ai JSONB
  await db.update(videos).set({
    ai: sql`jsonb_set(ai, '{transcript}', ${JSON.stringify({
      content: webvttContent,
      language: transcription.language,
      word_count: transcription.text.split(/\s+/).length,
      model: "whisper-large-v3",
      confidence: averageConfidence(transcription.segments!),
      generated_at: new Date().toISOString(),
    })}::jsonb)`,
  }).where(eq(videos.id, videoId));

  // 5. Update full-text search index (via trigger or manual update)
  return { wordCount: transcription.text.split(/\s+/).length };
}

function segmentsToWebVTT(
  segments: Array<{ start: number; end: number; text: string }>
): string {
  let vtt = "WEBVTT\n\n";
  for (const seg of segments) {
    const startTime = formatVTTTimestamp(seg.start);
    const endTime = formatVTTTimestamp(seg.end);
    vtt += `${startTime} --> ${endTime}\n${seg.text.trim()}\n\n`;
  }
  return vtt;
}

function formatVTTTimestamp(seconds: number): string {
  const h = Math.floor(seconds / 3600).toString().padStart(2, "0");
  const m = Math.floor((seconds % 3600) / 60).toString().padStart(2, "0");
  const s = Math.floor(seconds % 60).toString().padStart(2, "0");
  const ms = Math.round((seconds % 1) * 1000).toString().padStart(3, "0");
  return `${h}:${m}:${s}.${ms}`;
}
```

**Testing**:
- `Unit: segmentsToWebVTT converts segments array → valid WebVTT string with WEBVTT header`
- `Unit: formatVTTTimestamp(65.5) → "00:01:05.500"`
- `Unit: formatVTTTimestamp(3723.123) → "01:02:03.123"`
- `Integration (mocked OpenAI): processTranscription → calls Whisper API with correct audio file`
- `Integration (mocked OpenAI): transcript stored in video.ai.transcript with correct fields`
- `Integration (mocked OpenAI): word_count calculated correctly from transcript text`
- `Unit: empty audio → graceful handling, transcript content is empty WEBVTT`
- `Unit: OpenAI API rate limit → job retries with exponential backoff`
- `Fixture-based: transcribe sample-video.webm → output matches expected WebVTT structure`

---

#### 3.4 — AI Summarisation and Chapter Detection

**What**: Use Claude API to generate summaries, show notes, key points, action items, and chapter markers from transcripts.

**Design**:

```typescript
// packages/jobs/src/processors/summarise.ts
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

interface AiEnrichment {
  summary: string;
  showNotes: string;
  keyPoints: string[];
  actionItems: string[];
  chapters: Array<{ title: string; startMs: number; endMs: number }>;
}

export async function processAiEnrichment(
  job: Job<{ videoId: string; task: string }>
) {
  const { videoId, task } = job.data;
  const video = await db.query.videos.findFirst({ where: eq(videos.id, videoId) });
  const transcript = video!.ai?.transcript?.content;

  if (!transcript) {
    throw new Error("No transcript available for AI enrichment");
  }

  if (task === "summarise") {
    const result = await generateSummary(transcript, video!.durationMs!);
    await db.update(videos).set({
      ai: sql`ai || ${JSON.stringify({
        summary: result.summary,
        show_notes: result.showNotes,
        key_points: result.keyPoints,
        action_items: result.actionItems,
      })}::jsonb`,
    }).where(eq(videos.id, videoId));
  }

  if (task === "chapter_detect") {
    const chapters = await detectChapters(transcript, video!.durationMs!);
    await db.update(videos).set({
      ai: sql`jsonb_set(ai, '{chapters}', ${JSON.stringify(chapters)}::jsonb)`,
    }).where(eq(videos.id, videoId));
  }
}

async function generateSummary(
  transcript: string,
  durationMs: number
): Promise<AiEnrichment> {
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 2048,
    system: `You are an assistant that analyses video transcripts. The video is ${Math.round(durationMs / 1000)} seconds long. Return JSON only.`,
    messages: [{
      role: "user",
      content: `Analyse this video transcript and return a JSON object with:
- "summary": 2-3 sentence summary of the video
- "showNotes": markdown-formatted show notes with sections
- "keyPoints": array of 3-7 key takeaways
- "actionItems": array of action items mentioned (empty array if none)

Transcript:
${transcript}`,
    }],
  });
  return JSON.parse(response.content[0].text);
}

async function detectChapters(
  transcript: string,
  durationMs: number
): Promise<Array<{ title: string; startMs: number; endMs: number }>> {
  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 1024,
    system: `You are an assistant that identifies logical chapters in video transcripts. The video is ${Math.round(durationMs / 1000)} seconds long. Use the WebVTT timestamps to determine chapter boundaries. Return JSON only.`,
    messages: [{
      role: "user",
      content: `Identify 2-8 logical chapters in this video transcript. Return a JSON array of objects with "title" (string), "startMs" (number), "endMs" (number).

Transcript:
${transcript}`,
    }],
  });
  return JSON.parse(response.content[0].text);
}
```

**Testing**:
- `Unit: generateSummary returns object with summary, showNotes, keyPoints, actionItems`
- `Unit: detectChapters returns array of chapters with title, startMs, endMs`
- `Unit: chapters are non-overlapping and cover the full video duration`
- `Integration (mocked Claude API): summarise task stores result in video.ai JSONB`
- `Integration (mocked Claude API): chapter_detect task stores chapters array in video.ai.chapters`
- `Unit: missing transcript → throws error, job fails gracefully`
- `Unit: Claude API returns malformed JSON → error handled, job retried`
- `Fixture-based: summarise sample transcript → output has expected structure and reasonable content`

---

## Phase 4: Video Playback and Sharing

### Purpose

Build the video player experience with HLS adaptive bitrate playback, WebVTT captions, chapter navigation, transcript panel, and link-based sharing. After this phase, users can watch videos in a rich player and share them with anyone via a link.

### Tasks

#### 4.1 — Video Player Component

**What**: Build a video player with Mux Player React, WebVTT caption rendering, chapter markers, and keyboard controls.

**Design**:

```typescript
// apps/web/src/components/player/VideoPlayer.tsx
import MuxPlayer from "@mux/mux-player-react";

interface VideoPlayerProps {
  video: {
    id: string;
    title: string;
    playbackId: string;
    durationMs: number;
    captions?: { language: string; url: string }[];
    chapters?: { title: string; startMs: number; endMs: number }[];
  };
  playbackToken: string;          // JWT signed token for private videos
  onTimeUpdate?: (timeMs: number) => void;
  onViewEvent?: (event: ViewerEvent) => void;
}

type ViewerEvent =
  | { type: "play"; timestampMs: number }
  | { type: "pause"; timestampMs: number }
  | { type: "seek"; timestampMs: number; fromMs: number }
  | { type: "complete" }
  | { type: "cta_click"; ctaLabel: string; timestampMs: number };

export function VideoPlayer({
  video, playbackToken, onTimeUpdate, onViewEvent
}: VideoPlayerProps) {
  return (
    <MuxPlayer
      playbackId={video.playbackId}
      tokens={{ playback: playbackToken }}
      metadata={{
        video_id: video.id,
        video_title: video.title,
      }}
      streamType="on-demand"
      primaryColor="#3B82F6"
      accentColor="#1E40AF"
      // Captions
      defaultHiddenCaptions={false}
      // Keyboard: space=play/pause, ←→=seek, f=fullscreen, m=mute, c=captions
    >
      {video.captions?.map((cap) => (
        <track
          key={cap.language}
          kind="subtitles"
          srcLang={cap.language}
          src={cap.url}
          default={cap.language === "en"}
        />
      ))}
    </MuxPlayer>
  );
}

// apps/web/src/components/player/ChapterMarkers.tsx
interface ChapterMarkersProps {
  chapters: { title: string; startMs: number; endMs: number }[];
  currentTimeMs: number;
  durationMs: number;
  onSeek: (timeMs: number) => void;
}

// Renders clickable chapter markers on the progress bar + chapter list sidebar

// apps/web/src/components/player/TranscriptPanel.tsx
interface TranscriptPanelProps {
  webvttContent: string;
  currentTimeMs: number;
  onSeek: (timeMs: number) => void;
}

// Parses WebVTT, highlights current cue, scrolls to active segment, click-to-seek
```

**Testing**:
- `Unit: VideoPlayer renders MuxPlayer with correct playbackId and token`
- `Unit: VideoPlayer renders caption tracks for each language`
- `Unit: ChapterMarkers highlights the active chapter based on currentTimeMs`
- `Unit: ChapterMarkers click → calls onSeek with chapter startMs`
- `Unit: TranscriptPanel parses WebVTT content into cue objects`
- `Unit: TranscriptPanel highlights cue matching currentTimeMs`
- `Unit: TranscriptPanel click on cue → calls onSeek with cue start time`
- `E2E: video page loads → player renders with thumbnail`
- `E2E: click play → video streams via HLS, progress bar updates`
- `E2E: toggle captions → subtitles appear/disappear`

---

#### 4.2 — Video Detail Page

**What**: Build the full video viewing experience page with player, transcript, chapters, AI summary, comments, and metadata.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/video/[id]/page.tsx
interface VideoPageParams {
  params: { id: string };
}

export default async function VideoPage({ params }: VideoPageParams) {
  // 1. Fetch video with all AI enrichment from tRPC
  // 2. Generate signed playback token
  // 3. Record view event
  // Layout:
  //   ┌─────────────────────────────────────────┐
  //   │          Video Player                    │
  //   ├────────────────────┬────────────────────┤
  //   │  Tabs:             │  Transcript Panel  │
  //   │  - Summary         │  (scrolling,       │
  //   │  - Key Points      │   click-to-seek)   │
  //   │  - Action Items    │                    │
  //   │  - Show Notes      │                    │
  //   ├────────────────────┴────────────────────┤
  //   │  Comments (threaded, timestamp-anchored) │
  //   └─────────────────────────────────────────┘
}
```

**Testing**:
- `E2E: navigate to video page → player, transcript, and summary tabs render`
- `E2E: summary tab shows AI-generated summary text`
- `E2E: key points tab shows bulleted list of key takeaways`
- `E2E: action items tab shows action items (or "No action items" if empty)`
- `E2E: clicking a transcript cue seeks the video to that timestamp`
- `E2E: non-existent video ID → 404 page`
- `E2E: video with status "processing" → shows processing spinner, no player`

---

#### 4.3 — Link-Based Sharing

**What**: Implement share link creation, public video page, email-gating, password protection, and link expiry.

**Design**:

```typescript
// packages/api/src/routers/share.ts
export const shareRouter = router({
  createLink: protectedProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      requiresEmail: z.boolean().default(false),
      password: z.string().optional(),
      expiresAt: z.string().datetime().optional(),
      maxViews: z.number().positive().optional(),
      allowedDomains: z.array(z.string()).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      // 1. Verify user owns or has edit access to video
      // 2. Generate share token (crypto.randomBytes(32).toString("base64url"))
      // 3. Hash password if provided (bcrypt)
      // 4. Insert share_links row
      // 5. Return { token, url: `${BASE_URL}/v/${token}` }
    }),

  revokeLink: protectedProcedure
    .input(z.object({ linkId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => {
      // Set is_active = false
    }),

  listLinks: protectedProcedure
    .input(z.object({ videoId: z.string().uuid() }))
    .query(async ({ ctx, input }) => {
      // Return all share links for a video with view counts
    }),
});

// apps/web/src/app/(public)/v/[shareToken]/page.tsx
// Public video player page — no authentication required
export default async function PublicVideoPage({ params }: { params: { shareToken: string } }) {
  // 1. Look up share_links by token
  // 2. Check: is_active, not expired, max_views not exceeded
  // 3. If requires_email → show email capture form first
  // 4. If password-protected → show password form first
  // 5. Generate signed playback token (short-lived, viewer-specific)
  // 6. Render lightweight player page (no dashboard chrome)
  // 7. Increment share_links.view_count
}
```

oEmbed support (per oEmbed standard):
```typescript
// apps/web/src/app/api/oembed/route.ts
export async function GET(req: Request) {
  const { searchParams } = new URL(req.url);
  const url = searchParams.get("url");
  const format = searchParams.get("format") ?? "json";

  // Parse video share token from URL
  // Return oEmbed response:
  // {
  //   "type": "video",
  //   "version": "1.0",
  //   "title": "Q2 Product Update",
  //   "author_name": "Jane Smith",
  //   "provider_name": "Async Video",
  //   "provider_url": "https://asyncvideo.com",
  //   "thumbnail_url": "...",
  //   "html": "<iframe src='...' width='640' height='360' ...></iframe>",
  //   "width": 640,
  //   "height": 360
  // }
}
```

**Testing**:
- `Unit: createLink generates a unique 32-byte base64url token`
- `Unit: createLink with password → password_hash stored, plaintext not stored`
- `Unit: revokeLink sets is_active = false`
- `Unit: public page with expired link → 410 Gone`
- `Unit: public page with max_views exceeded → 410 Gone`
- `Unit: public page with requires_email → renders email form`
- `Unit: public page with password → renders password form`
- `Unit: correct password submission → video player renders`
- `Unit: incorrect password → error message, player not rendered`
- `Integration: oEmbed endpoint returns valid response matching oEmbed spec`
- `E2E: create share link → copy URL → open in incognito → video plays`
- `E2E: revoke share link → URL returns 410`

---

## Phase 5: Video Library and Search

### Purpose

Build the video library with grid/list views, search (by title and transcript content), filtering, tagging, collections, and bulk operations. After this phase, users can organise, find, and manage their video content efficiently.

### Tasks

#### 5.1 — Video Library Page

**What**: Implement the main library view with grid and list modes, sorting, pagination, and empty states.

**Design**:

```typescript
// packages/api/src/routers/video.ts
export const videoRouter = router({
  list: protectedProcedure
    .input(z.object({
      workspaceId: z.string().uuid().optional(),
      status: z.enum(["uploading", "processing", "ready", "failed"]).optional(),
      visibility: z.enum(["private", "workspace", "organisation", "public"]).optional(),
      tags: z.array(z.string()).optional(),
      search: z.string().max(200).optional(),
      sortBy: z.enum(["created_at", "view_count", "title", "duration_ms"]).default("created_at"),
      sortOrder: z.enum(["asc", "desc"]).default("desc"),
      cursor: z.string().uuid().optional(),      // cursor-based pagination
      limit: z.number().min(1).max(100).default(24),
    }))
    .query(async ({ ctx, input }) => {
      // Build query with filters
      // If search: use full-text search on title + ai.summary
      //   WHERE to_tsvector('english', ...) @@ plainto_tsquery('english', input.search)
      // If tags: WHERE tags @> ARRAY[input.tags]
      // Cursor pagination: WHERE created_at < cursor_timestamp
      // Returns: { videos: Video[], nextCursor: string | null }
    }),

  delete: protectedProcedure
    .input(z.object({ videoId: z.string().uuid() }))
    .mutation(async ({ ctx, input }) => {
      // Soft delete: set deleted_at = now()
      // Enqueue cleanup job (delete from Mux, S3, CDN after 30 days)
    }),

  bulkDelete: protectedProcedure
    .input(z.object({ videoIds: z.array(z.string().uuid()).min(1).max(50) }))
    .mutation(async ({ ctx, input }) => {
      // Soft delete all, enqueue cleanup jobs
    }),

  updateTags: protectedProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      tags: z.array(z.string().max(50)).max(20),
    }))
    .mutation(async ({ ctx, input }) => {
      // Replace tags array on video
    }),
});

// Library view types
interface VideoCard {
  id: string;
  title: string | null;
  thumbnailUrl: string | null;
  durationMs: number | null;
  status: VideoStatus;
  viewCount: number;
  commentCount: number;
  creatorName: string;
  creatorAvatar: string | null;
  tags: string[];
  createdAt: string;
  hasTranscript: boolean;
  hasSummary: boolean;
}
```

**Testing**:
- `Unit: video.list returns videos filtered by workspaceId`
- `Unit: video.list with search query uses full-text search index`
- `Unit: video.list with tags filter returns only matching videos`
- `Unit: video.list cursor pagination — second page starts after cursor`
- `Unit: video.list does not return soft-deleted videos`
- `Unit: video.delete sets deleted_at, does not hard-delete`
- `Unit: video.bulkDelete handles up to 50 videos`
- `Unit: video.updateTags replaces tags array`
- `E2E: library page shows grid of video thumbnails with metadata`
- `E2E: toggle list/grid view → layout changes`
- `E2E: search "roadmap" → videos with matching title or transcript appear`
- `E2E: filter by tag → only tagged videos shown`

---

#### 5.2 — Collections Management

**What**: Implement collections (folders) for organizing videos with drag-and-drop reordering and nested hierarchy.

**Design**:

```typescript
// packages/api/src/routers/collection.ts (new router)
export const collectionRouter = router({
  create: protectedProcedure
    .input(z.object({
      organisationId: z.string().uuid(),
      workspaceId: z.string().uuid().optional(),
      name: z.string().min(1).max(255),
      parentId: z.string().uuid().optional(),     // for nested collections
    }))
    .mutation(/* ... */),

  addVideos: protectedProcedure
    .input(z.object({
      collectionId: z.string().uuid(),
      videoIds: z.array(z.string().uuid()),
    }))
    .mutation(async ({ ctx, input }) => {
      // Append videoIds to collection.video_ids array
      // UPDATE collections SET video_ids = video_ids || $1
    }),

  reorderVideos: protectedProcedure
    .input(z.object({
      collectionId: z.string().uuid(),
      videoIds: z.array(z.string().uuid()),   // new order
    }))
    .mutation(async ({ ctx, input }) => {
      // Replace video_ids array with new order
    }),

  list: protectedProcedure
    .input(z.object({ organisationId: z.string().uuid() }))
    .query(/* ... */),

  getWithVideos: protectedProcedure
    .input(z.object({ collectionId: z.string().uuid() }))
    .query(/* ... */),
});
```

**Testing**:
- `Unit: collection.create creates collection with empty video_ids`
- `Unit: collection.create with parentId → sets parent reference`
- `Unit: collection.addVideos appends video IDs to array`
- `Unit: collection.reorderVideos replaces video_ids with new order`
- `Unit: collection.list returns collections scoped to organisation`
- `Unit: collection.getWithVideos returns collection with resolved video objects`
- `E2E: create collection → appears in sidebar`
- `E2E: drag video to collection → video appears in collection view`

---

## Phase 6: Comments and Collaboration

### Purpose

Build threaded, timestamp-anchored comments with reactions, mentions, and resolution tracking. After this phase, viewers can leave feedback at specific moments in a video and have conversations.

### Tasks

#### 6.1 — Comment System

**What**: Implement threaded comments anchored to video timestamps with creation, editing, deletion, and resolution.

**Design**:

```typescript
// packages/api/src/routers/comment.ts
export const commentRouter = router({
  create: protectedProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      body: z.string().min(1).max(5000),
      parentId: z.string().uuid().optional(),         // for replies
      timestampMs: z.number().int().nonnegative().optional(),  // anchor in video
    }))
    .mutation(async ({ ctx, input }) => {
      // 1. Verify access to video
      // 2. Insert comment
      // 3. Increment videos.comment_count
      // 4. Send notification to video creator (and parent comment author if reply)
      // 5. Insert audit_log entry
      // Returns: comment with user info
    }),

  list: protectedProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      sortBy: z.enum(["timestamp", "created_at"]).default("timestamp"),
    }))
    .query(async ({ ctx, input }) => {
      // Fetch all comments for video, grouped by thread (parent_id)
      // If sortBy=timestamp: order by timestamp_ms ASC (nulls last)
      // If sortBy=created_at: order by created_at DESC
      // Returns flat list — client groups by parent_id
    }),

  resolve: protectedProcedure
    .input(z.object({ commentId: z.string().uuid() }))
    .mutation(/* set is_resolved = true */),

  addReaction: protectedProcedure
    .input(z.object({
      commentId: z.string().uuid(),
      emoji: z.string().max(20),     // thumbsup, heart, fire, etc.
    }))
    .mutation(async ({ ctx, input }) => {
      // Update comment.reactions JSONB
      // { "thumbsup": ["user-id-1", "user-id-2"], "heart": ["user-id-3"] }
      // Toggle: add user if not present, remove if present
    }),

  delete: protectedProcedure
    .input(z.object({ commentId: z.string().uuid() }))
    .mutation(/* soft delete — set body to "[deleted]", keep for thread integrity */),
});
```

**Testing**:
- `Unit: comment.create with timestampMs → comment anchored to video position`
- `Unit: comment.create with parentId → reply linked to parent`
- `Unit: comment.create increments video.comment_count`
- `Unit: comment.create sends notification to video creator`
- `Unit: comment.list sorted by timestamp → comments ordered by video position`
- `Unit: comment.resolve sets is_resolved = true`
- `Unit: comment.addReaction toggles user in emoji array`
- `Unit: comment.addReaction — second call with same emoji removes user`
- `Unit: comment.delete replaces body with "[deleted]", preserves thread`
- `Integration: create parent comment → create reply → list shows threaded structure`
- `E2E: click on video timeline → comment input appears with timestamp pre-filled`
- `E2E: submit comment → comment appears in comment list at correct timestamp`
- `E2E: click comment timestamp → video seeks to that position`

---

#### 6.2 — Real-Time Notifications

**What**: Implement WebSocket-based real-time notifications for comments, reactions, video-ready events, and view milestones.

**Design**:

```typescript
// packages/api/src/services/notification-service.ts
import { Server as SocketServer } from "socket.io";

export class NotificationService {
  constructor(private io: SocketServer, private db: Database) {}

  // Notification types:
  // - video_ready: "Your video 'X' is ready to share"
  // - comment_added: "Jane commented on your video 'X'"
  // - comment_reply: "Jane replied to your comment on 'X'"
  // - reaction_added: "Jane reacted to your comment"
  // - view_milestone: "Your video 'X' reached 100 views"
  // - mention: "@you was mentioned in a comment on 'X'"

  async notify(userId: string, notification: {
    type: string;
    title: string;
    body?: string;
    videoId?: string;
    actorId?: string;
  }) {
    // 1. Insert into notifications table
    // 2. Emit via Socket.io to user's room
    this.io.to(`user:${userId}`).emit("notification", notification);
  }

  async markRead(userId: string, notificationId: string) {
    await this.db.update(notifications)
      .set({ isRead: true })
      .where(and(eq(notifications.id, notificationId), eq(notifications.userId, userId)));
  }

  async markAllRead(userId: string) {
    await this.db.update(notifications)
      .set({ isRead: true })
      .where(and(eq(notifications.userId, userId), eq(notifications.isRead, false)));
  }
}

// Socket.io room structure:
// user:{userId} — personal notifications
// video:{videoId} — live comment updates while viewing
```

**Testing**:
- `Unit: notify inserts notification row and emits socket event`
- `Unit: markRead sets is_read = true for matching notification`
- `Unit: markAllRead updates all unread notifications for user`
- `Integration (Socket.io): client connected to user room receives notification events`
- `Integration: comment created → video creator receives notification in <1s`
- `E2E: notification bell shows unread count`
- `E2E: new comment on user's video → bell count increments without page refresh`
- `E2E: click notification → navigates to video at comment timestamp`

---

## Phase 7: Analytics and Engagement Tracking

### Purpose

Implement viewer analytics with view tracking, watch-time measurement, engagement heatmaps, and drop-off detection. After this phase, video creators can see who watched their videos, how much they watched, and where they lost attention.

### Tasks

#### 7.1 — View Tracking

**What**: Record view sessions with viewer metadata, watch duration, and completion status.

**Design**:

```typescript
// Client-side: beacon API for reliable view event delivery
// apps/web/src/hooks/useViewTracker.ts
interface ViewTrackerOptions {
  videoId: string;
  shareToken?: string;
  viewerEmail?: string;
}

export function useViewTracker({ videoId, shareToken, viewerEmail }: ViewTrackerOptions) {
  // 1. On first play: POST /api/v1/views/start → returns viewId
  // 2. Every 5 seconds during playback: batch engagement events
  // 3. On pause/seek/speed-change: add to event batch
  // 4. Every 15 seconds OR on page unload: send batch via navigator.sendBeacon
  // 5. On video end: send completion event
}

// packages/api/src/routers/analytics.ts
export const analyticsRouter = router({
  startView: publicProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      shareToken: z.string().optional(),
      viewerEmail: z.string().email().optional(),
      sessionId: z.string(),
      deviceType: z.enum(["desktop", "mobile", "tablet"]),
    }))
    .mutation(async ({ ctx, input }) => {
      // 1. Validate access (share token or authenticated)
      // 2. Insert video_views row
      // 3. Increment videos.view_count
      // Returns: { viewId }
    }),

  recordEngagement: publicProcedure
    .input(z.object({
      viewId: z.string().uuid(),
      watchDurationMs: z.number(),
      percentWatched: z.number().min(0).max(100),
      completed: z.boolean(),
      engagement: z.object({
        events: z.array(z.object({
          type: z.enum(["play", "pause", "seek", "buffer", "speed_change", "cta_click"]),
          tsMs: z.number(),
          at: z.string().datetime(),
        })),
        segmentsWatched: z.array(z.tuple([z.number(), z.number()])),
      }),
    }))
    .mutation(/* update video_views row with latest engagement data */),

  getVideoAnalytics: protectedProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      dateRange: z.object({
        from: z.string().datetime(),
        to: z.string().datetime(),
      }).optional(),
    }))
    .query(async ({ ctx, input }) => {
      // Aggregate from video_views:
      // - total views, unique viewers, avg watch %, completion rate
      // - views over time (by day)
      // - device breakdown
      // - top referrers
      // - viewer list with watch duration
      // Returns: VideoAnalytics
    }),
});

interface VideoAnalytics {
  totalViews: number;
  uniqueViewers: number;
  avgPercentWatched: number;
  completionRate: number;
  totalWatchTimeMs: number;
  viewsByDay: { date: string; views: number }[];
  deviceBreakdown: { desktop: number; mobile: number; tablet: number };
  topReferrers: { url: string; count: number }[];
  viewers: {
    email: string | null;
    name: string | null;
    watchDurationMs: number;
    percentWatched: number;
    completed: boolean;
    viewedAt: string;
  }[];
}
```

**Testing**:
- `Unit: startView creates video_views row with correct viewer metadata`
- `Unit: startView increments videos.view_count`
- `Unit: recordEngagement updates view row with latest watch duration and engagement`
- `Unit: recordEngagement with completed=true marks view as completed`
- `Unit: getVideoAnalytics aggregates correctly across multiple views`
- `Unit: getVideoAnalytics respects date range filter`
- `Integration: full view lifecycle — start → engagement updates → completion`
- `E2E: watch video → analytics page shows the view`

---

#### 7.2 — Engagement Heatmap

**What**: Generate per-segment engagement heatmaps showing which parts of a video are most/least watched, including rewatch and drop-off data.

**Design**:

```typescript
// packages/api/src/services/analytics-service.ts
interface HeatmapSegment {
  startMs: number;
  endMs: number;
  playCount: number;
  rewatchCount: number;
  dropOffCount: number;
  intensity: number;     // 0.0-1.0 normalised engagement score
}

export async function generateHeatmap(
  videoId: string,
  segmentDurationMs: number = 1000  // 1-second granularity
): Promise<HeatmapSegment[]> {
  // 1. Query video_views for all engagement data
  // 2. Extract segments_watched and rewatch_segments from JSONB
  // 3. Count plays, rewatches, drop-offs per segment
  // 4. Normalise to 0-1 intensity scale
  // 5. Return array of segments
}

// apps/web/src/components/analytics/EngagementHeatmap.tsx
interface EngagementHeatmapProps {
  segments: HeatmapSegment[];
  durationMs: number;
  onSegmentClick: (startMs: number) => void;
}
// Renders a colored bar under the video player:
// - Green (high intensity): most watched
// - Yellow (medium): moderate engagement
// - Red (low): drop-off points
```

**Testing**:
- `Unit: generateHeatmap with no views → all segments have 0 playCount`
- `Unit: generateHeatmap with one complete view → all segments have playCount 1`
- `Unit: generateHeatmap with rewatch → rewatchCount incremented for rewatched segments`
- `Unit: generateHeatmap with drop-off at 50% → dropOffCount > 0 at midpoint`
- `Unit: intensity normalisation — highest-play segment has intensity 1.0`
- `Fixture-based: heatmap from sample engagement data matches expected output`
- `E2E: analytics page shows colored heatmap bar under video thumbnail`

---

## Phase 8: Integrations — CRM, Slack, and Webhooks

### Purpose

Connect the platform to external systems — CRM (Salesforce, HubSpot) for sales attribution, Slack for sharing notifications, and webhooks for custom automation. After this phase, sales teams can track video engagement in their CRM pipeline and teams receive Slack notifications.

### Tasks

#### 8.1 — OAuth Integration Framework

**What**: Build a generic OAuth 2.0 integration framework for connecting external services, with encrypted token storage and automatic refresh.

**Design**:

```typescript
// packages/api/src/services/integration-service.ts
interface IntegrationProvider {
  name: string;
  authorizationUrl: string;
  tokenUrl: string;
  scopes: string[];
  clientId: string;
  clientSecret: string;
}

const PROVIDERS: Record<string, IntegrationProvider> = {
  salesforce: {
    name: "Salesforce",
    authorizationUrl: "https://login.salesforce.com/services/oauth2/authorize",
    tokenUrl: "https://login.salesforce.com/services/oauth2/token",
    scopes: ["api", "refresh_token"],
    clientId: process.env.SALESFORCE_CLIENT_ID!,
    clientSecret: process.env.SALESFORCE_CLIENT_SECRET!,
  },
  hubspot: {
    name: "HubSpot",
    authorizationUrl: "https://app.hubspot.com/oauth/authorize",
    tokenUrl: "https://api.hubapi.com/oauth/v1/token",
    scopes: ["contacts", "timeline"],
    clientId: process.env.HUBSPOT_CLIENT_ID!,
    clientSecret: process.env.HUBSPOT_CLIENT_SECRET!,
  },
  slack: {
    name: "Slack",
    authorizationUrl: "https://slack.com/oauth/v2/authorize",
    tokenUrl: "https://slack.com/api/oauth.v2.access",
    scopes: ["chat:write", "channels:read"],
    clientId: process.env.SLACK_CLIENT_ID!,
    clientSecret: process.env.SLACK_CLIENT_SECRET!,
  },
};

export class IntegrationService {
  // getAuthorizationUrl(provider, organisationId) → redirect URL with state param
  // handleCallback(provider, code, state) → exchange code for tokens, encrypt, store
  // getAccessToken(integrationId) → decrypt, check expiry, refresh if needed, return
  // disconnect(integrationId) → delete tokens, set status = "disconnected"
}

// Token encryption: AES-256-GCM using INTEGRATION_ENCRYPTION_KEY env var
// Follows ISO/IEC 27001 A.8.24 (Cryptography)
```

**Testing**:
- `Unit: getAuthorizationUrl returns valid URL with correct scopes and state`
- `Unit: handleCallback exchanges code for tokens and encrypts before storing`
- `Unit: getAccessToken with valid non-expired token → returns decrypted token`
- `Unit: getAccessToken with expired token → refreshes, stores new token, returns new`
- `Unit: getAccessToken with refresh failure → marks integration as "errored"`
- `Unit: disconnect removes tokens from DB`
- `Unit: token encryption round-trip — encrypt then decrypt yields original`
- `Integration (mocked OAuth): full OAuth flow — authorize → callback → token stored`

---

#### 8.2 — CRM Sync (Salesforce and HubSpot)

**What**: Sync video view events to CRM as timeline activities, enabling sales pipeline attribution.

**Design**:

```typescript
// packages/jobs/src/processors/crm-sync.ts
export async function processCrmSync(
  job: Job<{ videoId: string; viewId: string; integrationId: string }>
) {
  const { videoId, viewId, integrationId } = job.data;
  const integration = await db.query.integrations.findFirst({
    where: eq(integrations.id, integrationId),
  });
  const view = await db.query.videoViews.findFirst({ where: eq(videoViews.id, viewId) });
  const video = await db.query.videos.findFirst({ where: eq(videos.id, videoId) });

  const viewerEmail = view!.viewer?.email;
  if (!viewerEmail) return; // Can't attribute without email

  if (integration!.provider === "salesforce") {
    await syncToSalesforce(integration!, video!, view!);
  } else if (integration!.provider === "hubspot") {
    await syncToHubspot(integration!, video!, view!);
  }

  // Log sync in crm_sync_log
  await db.insert(crmSyncLog).values({
    integrationId,
    direction: "outbound",
    entityType: "video_view",
    payload: { videoTitle: video!.title, viewerEmail, percentWatched: view!.percentWatched },
    status: "completed",
    syncedAt: new Date(),
  });
}

// Salesforce: Create Task or custom VideoView__c object
// HubSpot: Create timeline event on contact record
```

**Testing**:
- `Integration (mocked Salesforce API): view event → Task created on matching Lead/Contact`
- `Integration (mocked HubSpot API): view event → timeline event created on contact`
- `Unit: view without email → job completes without API call`
- `Unit: CRM API failure → job retries, crm_sync_log status = "failed"`
- `Unit: crm_sync_log records payload and timestamps`

---

#### 8.3 — Slack Integration

**What**: Send Slack notifications when videos are recorded, shared, viewed, or receive comments.

**Design**:

```typescript
// packages/jobs/src/processors/slack-notify.ts
import { WebClient } from "@slack/web-api";

export async function processSlackNotification(
  job: Job<{ organisationId: string; event: SlackEvent }>
) {
  const integration = await db.query.integrations.findFirst({
    where: and(
      eq(integrations.organisationId, job.data.organisationId),
      eq(integrations.provider, "slack"),
    ),
  });
  if (!integration) return;

  const slack = new WebClient(decrypt(integration.credentials.bot_token));
  const channel = integration.config.channel_mappings?.[job.data.event.workspaceId] ?? "#general";

  await slack.chat.postMessage({
    channel,
    blocks: buildSlackBlocks(job.data.event),
    unfurl_links: false,
  });
}

type SlackEvent =
  | { type: "video_recorded"; videoId: string; title: string; creatorName: string; workspaceId: string }
  | { type: "video_shared"; videoId: string; title: string; shareUrl: string; workspaceId: string }
  | { type: "view_milestone"; videoId: string; title: string; viewCount: number; workspaceId: string };
```

**Testing**:
- `Integration (mocked Slack API): video_recorded event → message posted with video title and creator`
- `Integration (mocked Slack API): video_shared event → message posted with share link`
- `Unit: no Slack integration configured → job completes without API call`
- `Unit: Slack API error → job retries`

---

#### 8.4 — Outgoing Webhooks

**What**: Implement configurable outgoing webhooks with HMAC signing, retry logic, and delivery logging.

**Design**:

```typescript
// packages/api/src/routers/webhook.ts (new router)
export const webhookRouter = router({
  create: protectedProcedure
    .input(z.object({
      organisationId: z.string().uuid(),
      url: z.string().url(),
      events: z.array(z.enum([
        "video.created", "video.ready", "video.deleted",
        "video.viewed", "video.completed",
        "comment.created", "comment.resolved",
        "share.created", "share.revoked",
      ])).min(1),
    }))
    .mutation(async ({ ctx, input }) => {
      // Generate HMAC signing secret (crypto.randomBytes(32))
      // Insert webhooks row
      // Return { webhookId, secret } — secret shown once
    }),

  list: protectedProcedure
    .input(z.object({ organisationId: z.string().uuid() }))
    .query(/* ... */),

  delete: protectedProcedure
    .input(z.object({ webhookId: z.string().uuid() }))
    .mutation(/* ... */),

  getDeliveries: protectedProcedure
    .input(z.object({ webhookId: z.string().uuid(), limit: z.number().default(20) }))
    .query(/* ... */),
});

// Webhook delivery format (per OWASP API Security — signed payloads)
// POST <webhook_url>
// Headers:
//   Content-Type: application/json
//   X-Signature: sha256=<hmac_hex>
//   X-Event-Type: video.viewed
//   X-Delivery-ID: <uuid>
//   X-Timestamp: <unix_timestamp>
// Body:
// {
//   "event": "video.viewed",
//   "timestamp": "2026-05-25T10:00:00Z",
//   "data": { ... event-specific payload ... }
// }

// Retry policy: 3 attempts, exponential backoff (10s, 60s, 300s)
```

**Testing**:
- `Unit: webhook.create generates 32-byte HMAC secret`
- `Unit: webhook delivery includes correct HMAC signature`
- `Unit: webhook delivery retries on 5xx response (up to 3 attempts)`
- `Unit: webhook delivery logs response status and body`
- `Unit: webhook delivery skips inactive webhooks`
- `Integration: video.created event → matching webhook receives POST with correct payload`

---

## Phase 9: Public REST API

### Purpose

Expose a versioned REST API (v1) for programmatic video management, enabling third-party integrations and custom workflows. The API follows OpenAPI 3.1 spec and provides API key authentication.

### Tasks

#### 9.1 — API Key Management

**What**: Implement API key creation, rotation, and scoped permissions for the public API.

**Design**:

```typescript
// API keys stored in a new api_keys table
// packages/db/src/schema/api-keys.ts
export const apiKeys = pgTable("api_keys", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  keyHash: varchar("key_hash", { length: 255 }).notNull(),   // SHA-256 hash of key
  keyPrefix: varchar("key_prefix", { length: 10 }).notNull(), // first 8 chars for identification
  scopes: text("scopes").array().notNull(),
  // scopes: ["videos:read", "videos:write", "analytics:read", "webhooks:write"]
  lastUsedAt: timestamp("last_used_at", { withTimezone: true }),
  expiresAt: timestamp("expires_at", { withTimezone: true }),
  createdBy: uuid("created_by").notNull().references(() => users.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

// Key format: avm_live_<32-random-bytes-base64url>
// Prefix "avm_live_" for production, "avm_test_" for sandbox
```

**Testing**:
- `Unit: API key generation produces key in correct format (avm_live_...)`
- `Unit: key hash stored, plaintext NOT stored`
- `Unit: authenticate with valid key → resolationId resolved`
- `Unit: authenticate with expired key → 401`
- `Unit: authenticate with revoked key → 401`
- `Unit: scope check — videos:read key cannot call write endpoints`

---

#### 9.2 — REST Endpoints

**What**: Implement core CRUD endpoints for videos, organisations, and analytics with OpenAPI 3.1 documentation.

**Design**:

```
GET    /api/v1/videos                    → list videos (paginated)
POST   /api/v1/videos                    → create video (initiate upload)
GET    /api/v1/videos/:id                → get video details
PATCH  /api/v1/videos/:id                → update video metadata
DELETE /api/v1/videos/:id                → delete video
GET    /api/v1/videos/:id/analytics      → get video analytics
GET    /api/v1/videos/:id/transcript     → get transcript (WebVTT or JSON)
POST   /api/v1/videos/:id/share-links    → create share link
GET    /api/v1/organizations/:id         → get organisation details
GET    /api/v1/organizations/:id/usage   → get storage and video usage
```

Rate limiting: 100 requests/minute per API key (per OWASP API Security Top 10 — API4 Unrestricted Resource Consumption).

```typescript
// Response format
interface ApiResponse<T> {
  data: T;
  meta?: {
    cursor?: string;
    hasMore?: boolean;
    total?: number;
  };
}

interface ApiError {
  error: {
    code: string;           // machine-readable: "video_not_found"
    message: string;        // human-readable
    details?: unknown;
  };
}
```

**Testing**:
- `Integration: GET /api/v1/videos with valid API key → returns paginated video list`
- `Integration: GET /api/v1/videos without API key → 401`
- `Integration: GET /api/v1/videos/:id → returns video with media, ai, sharing fields`
- `Integration: DELETE /api/v1/videos/:id → soft deletes video`
- `Integration: GET /api/v1/videos/:id/transcript?format=webvtt → returns WebVTT content`
- `Integration: GET /api/v1/videos/:id/transcript?format=json → returns structured transcript`
- `Integration: rate limit exceeded → 429 with Retry-After header`
- `Integration: API key with videos:read scope → GET succeeds, POST returns 403`

---

## Phase 10: Advanced AI Features

### Purpose

Implement the differentiating AI-native features: filler-word removal, real-time recording coaching, semantic video search, and transcript-based editing. These features move AI from "add-on" to "baseline capability."

### Tasks

#### 10.1 — Filler-Word and Silence Removal

**What**: Automatically detect and remove filler words (um, uh, ah, like, you know) and extended silences from recordings.

**Design**:

```typescript
// packages/jobs/src/processors/filler-removal.ts
interface FillerSegment {
  startMs: number;
  endMs: number;
  type: "filler_word" | "silence";
  text?: string;          // the filler word detected
  confidence: number;
}

export async function processFillerRemoval(
  job: Job<{ videoId: string }>
) {
  // 1. Get word-level timestamps from Whisper transcript
  // 2. Identify filler words by matching against pattern list
  //    Patterns: /^(um|uh|ah|eh|hmm|like|you know|I mean|basically|actually|so|right)$/i
  // 3. Identify silences > 1.5 seconds between words
  // 4. Generate FFmpeg edit list (concat filter removing filler segments)
  // 5. Process video through FFmpeg → new file without fillers
  // 6. Upload cleaned version as additional rendition
  // 7. Update video.ai.filler_removal JSONB:
  //    { removed_count: 12, saved_ms: 8500, processed_at: "..." }
}
```

**Testing**:
- `Unit: filler detection identifies "um" at word boundaries`
- `Unit: filler detection does NOT flag "umbrella" or "human"`
- `Unit: silence detection finds gaps > 1.5s between word timestamps`
- `Unit: FFmpeg edit list correctly excludes filler segments`
- `Fixture-based: sample transcript with fillers → correct segments identified`
- `Integration: processed video duration = original - saved_ms (within 100ms tolerance)`

---

#### 10.2 — Semantic Video Search

**What**: Enable natural-language search across video transcripts using vector embeddings, finding specific moments within videos (not just matching filenames).

**Design**:

```typescript
// Vector storage: pgvector extension for PostgreSQL
// packages/db/src/schema/video-embeddings.ts
export const videoEmbeddings = pgTable("video_embeddings", {
  id: uuid("id").primaryKey().defaultRandom(),
  videoId: uuid("video_id").notNull().references(() => videos.id, { onDelete: "cascade" }),
  chunkIndex: integer("chunk_index").notNull(),
  startMs: integer("start_ms").notNull(),
  endMs: integer("end_ms").notNull(),
  text: text("text").notNull(),
  embedding: vector("embedding", { dimensions: 1536 }).notNull(),  // text-embedding-3-small
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

// CREATE INDEX idx_embeddings_vector ON video_embeddings
//   USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

// packages/api/src/services/search-service.ts
interface SearchResult {
  videoId: string;
  videoTitle: string;
  thumbnailUrl: string;
  startMs: number;
  endMs: number;
  text: string;           // matching transcript segment
  score: number;          // cosine similarity
}

export async function semanticSearch(
  organisationId: string,
  query: string,
  limit: number = 20
): Promise<SearchResult[]> {
  // 1. Embed the query using text-embedding-3-small
  // 2. Query pgvector for nearest neighbours within organisation
  // 3. Return matching segments with video metadata and timestamps
}

// Indexing: when transcript is generated, chunk into ~30-second segments
// and generate embeddings for each chunk
```

**Testing**:
- `Unit: transcript chunking produces ~30s segments with overlap`
- `Unit: each chunk has correct startMs and endMs`
- `Integration (mocked embeddings API): semanticSearch returns results sorted by similarity`
- `Integration: search "product roadmap" in org with matching video → returns segment with timestamp`
- `Integration: search scoped to organisation — cannot find videos from other orgs`
- `E2E: search bar with natural language query → results show video segments with click-to-seek`

---

#### 10.3 — AI Recording Coach

**What**: Display real-time coaching suggestions during recording — pacing alerts, filler-word warnings, key-point reminders, and time management.

**Design**:

```typescript
// apps/web/src/hooks/useRecordingCoach.ts
interface CoachingAlert {
  type: "pacing" | "filler" | "silence" | "time" | "key_point";
  message: string;
  severity: "info" | "warning";
  timestamp: number;
}

interface RecordingCoachOptions {
  goalDescription?: string;      // "Explain Q2 roadmap changes to the team"
  targetDurationMs?: number;     // desired recording length
  keyPoints?: string[];          // points the user wants to cover
}

export function useRecordingCoach(options: RecordingCoachOptions) {
  // Real-time analysis during recording:
  // 1. Use Web Speech API (SpeechRecognition) for live transcription
  // 2. Monitor speech rate (words per minute) — alert if < 100 or > 180 WPM
  // 3. Detect filler words in real-time — show gentle reminder
  // 4. Track silence duration — nudge after 3+ seconds
  // 5. Compare elapsed time vs target duration — warn at 80% and 100%
  // 6. Check off key points as they're mentioned (fuzzy matching)
  //
  // Returns: { alerts: CoachingAlert[], coveredKeyPoints: string[], speechRateWpm: number }
}

// apps/web/src/components/recorder/CoachingOverlay.tsx
// Renders a subtle sidebar during recording:
// - Current speech rate (wpm)
// - Key points checklist (checked off as covered)
// - Recent alerts (fade out after 5 seconds)
// - Time remaining bar
```

**Testing**:
- `Unit: speech rate calculation — 150 words in 60 seconds → 150 WPM`
- `Unit: pacing alert triggered at < 100 WPM`
- `Unit: pacing alert triggered at > 180 WPM`
- `Unit: filler detection in live stream — "um" detected → alert`
- `Unit: silence alert after 3+ seconds of no speech`
- `Unit: key point matching — "roadmap" in speech matches "Q2 roadmap changes" key point`
- `Unit: time warning at 80% of target duration`
- `E2E: start recording with goal → coaching overlay appears with key points`

---

## Phase 11: Personalisation and CTA Overlays

### Purpose

Enable sales teams to personalise videos with recipient-specific details (name, company) and add interactive call-to-action overlays (buttons, forms, calendar links). After this phase, a single recording can be customised for multiple recipients without re-recording.

### Tasks

#### 11.1 — CTA Overlay System

**What**: Build interactive overlay components that appear at specified timestamps during video playback.

**Design**:

```typescript
// packages/api/src/routers/cta.ts (new router)
export const ctaRouter = router({
  create: protectedProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      ctaType: z.enum(["button", "form", "poll", "calendar_link"]),
      label: z.string().max(255),
      url: z.string().url().optional(),
      startMs: z.number().int().nonnegative(),
      endMs: z.number().int().nonnegative().optional(),
      position: z.enum(["top_left", "top_right", "bottom_left", "bottom_right", "center"]).default("bottom_right"),
      style: z.object({
        bgColor: z.string().regex(/^#[0-9A-F]{6}$/i).default("#3B82F6"),
        textColor: z.string().regex(/^#[0-9A-F]{6}$/i).default("#FFFFFF"),
        borderRadius: z.number().default(8),
      }).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Store in video.sharing.cta_overlays JSONB array
    }),

  trackClick: publicProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      ctaLabel: z.string(),
      viewId: z.string().uuid(),
      timestampMs: z.number(),
    }))
    .mutation(/* record CTA click in engagement events */),
});

// apps/web/src/components/player/CTAOverlay.tsx
interface CTAOverlayProps {
  overlays: CTAConfig[];
  currentTimeMs: number;
  onCtaClick: (cta: CTAConfig) => void;
}
// Renders visible overlays based on currentTimeMs, animates in/out
```

**Testing**:
- `Unit: CTA appears when currentTimeMs >= startMs`
- `Unit: CTA disappears when currentTimeMs > endMs`
- `Unit: CTA with no endMs stays visible until video ends`
- `Unit: CTA click calls onCtaClick and opens URL`
- `Unit: CTA click tracked in analytics`
- `E2E: add CTA at 30s → play video → CTA appears at 30s mark`

---

#### 11.2 — Video Personalisation Engine

**What**: Allow text overlays with dynamic variables (recipient name, company) that are rendered per-viewer without re-recording.

**Design**:

```typescript
// Personalisation works by:
// 1. Creator defines variable positions on the video (text overlays)
// 2. When sharing, creator provides per-recipient values
// 3. Player renders text overlays dynamically using Canvas API

// packages/api/src/routers/personalisation.ts (new router)
export const personalisationRouter = router({
  defineVariables: protectedProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      variables: z.array(z.object({
        name: z.string(),              // "recipient_name", "company"
        type: z.literal("text_overlay"),
        position: z.object({ x: z.number(), y: z.number() }),
        style: z.object({
          font: z.string().default("Inter"),
          size: z.number().default(24),
          color: z.string().default("#FFFFFF"),
          backgroundColor: z.string().optional(),
        }),
        startMs: z.number().default(0),
        endMs: z.number().optional(),
      })),
    }))
    .mutation(/* store in video.sharing.personalisation JSONB */),

  createPersonalisedLink: protectedProcedure
    .input(z.object({
      videoId: z.string().uuid(),
      recipientEmail: z.string().email(),
      variables: z.record(z.string()),  // { "recipient_name": "Jane", "company": "Acme" }
    }))
    .mutation(async ({ ctx, input }) => {
      // Create share link with personalisation data embedded
      // Store variable values in share_links.config JSONB
      // Return personalised share URL
    }),
});

// Player renders personalised text overlays at defined positions
// using an overlay <canvas> element synchronised with video playback
```

**Testing**:
- `Unit: defineVariables stores variable definitions in video.sharing.personalisation`
- `Unit: createPersonalisedLink creates share link with variable values`
- `Unit: personalised player renders "Hello Jane" overlay at defined position`
- `Unit: different share links for same video show different names`
- `E2E: create personalised link → open → see recipient name overlay on video`

---

## Phase 12: Enterprise Features — Audit, Compliance, and Self-Hosting

### Purpose

Add enterprise-grade audit logging (ISO 27001), GDPR compliance features (right to erasure, data retention), and self-hosted deployment packaging. After this phase, the platform is ready for regulated industry deployment.

### Tasks

#### 12.1 — Audit Logging

**What**: Implement comprehensive audit logging per ISO/IEC 27001:2022 Annex A requirements for all significant actions.

**Design**:

```typescript
// packages/api/src/services/audit-service.ts
interface AuditEntry {
  organisationId: string;
  actorId: string | null;
  action: string;
  // Actions: video.created, video.deleted, video.shared, video.viewed,
  //          member.invited, member.removed, member.role_changed,
  //          settings.changed, integration.connected, integration.disconnected,
  //          api_key.created, api_key.revoked, share_link.created, share_link.revoked
  resourceType: string;
  resourceId: string;
  details: Record<string, unknown>;
  ipAddress: string;
  userAgent: string;
}

export class AuditService {
  async log(entry: AuditEntry): Promise<void> {
    await this.db.insert(auditLog).values({
      organisationId: entry.organisationId,
      actorId: entry.actorId,
      action: entry.action,
      resourceType: entry.resourceType,
      resourceId: entry.resourceId,
      details: entry.details,
      occurredAt: new Date(),
    });
  }

  async query(filters: {
    organisationId: string;
    action?: string;
    actorId?: string;
    resourceType?: string;
    from?: Date;
    to?: Date;
    limit?: number;
    cursor?: string;
  }): Promise<{ entries: AuditEntry[]; nextCursor: string | null }> {
    // Query audit_log with filters, cursor pagination
    // Partitioned table — must include date range for efficient queries
  }
}
```

**Testing**:
- `Unit: audit.log inserts entry with all fields`
- `Unit: audit.query with action filter returns only matching entries`
- `Unit: audit.query with date range uses partition pruning`
- `Unit: all video CRUD operations generate audit entries`
- `Unit: all membership changes generate audit entries`
- `Integration: audit log query performance < 100ms with 1M+ entries (partition pruning)`
- `E2E: settings page → audit log tab shows recent actions with actor and timestamp`

---

#### 12.2 — GDPR Compliance (Right to Erasure)

**What**: Implement GDPR Article 17 right-to-erasure workflow including video deletion, analytics purging, CDN cache invalidation, and CRM data removal.

**Design**:

```typescript
// packages/api/src/services/gdpr-service.ts
interface DeletionRequest {
  id: string;
  organisationId: string;
  subjectEmail: string;
  status: "pending" | "processing" | "completed" | "failed";
  videosDeleted: number;
  viewsDeleted: number;
  cdnPurged: boolean;
}

export class GdprService {
  async requestErasure(
    organisationId: string,
    subjectEmail: string,
    requestedBy: string
  ): Promise<DeletionRequest> {
    // 1. Create deletion_request record (status = pending)
    // 2. Enqueue erasure job
    return request;
  }

  async processErasure(requestId: string): Promise<void> {
    // 1. Find all videos created by user → soft delete → enqueue hard delete
    // 2. Find all video_views by user → delete viewer data
    // 3. Find all comments by user → anonymise (set body to "[deleted]", user to null)
    // 4. Find all CRM sync data → delete
    // 5. Purge CDN caches for deleted videos (Mux API, Cloudflare API)
    // 6. Remove user from all memberships
    // 7. Anonymise user record (clear email, name, avatar)
    // 8. Update deletion_request: status = completed, counts
    // 9. Audit log: gdpr.erasure_completed
  }
}
```

**Testing**:
- `Unit: requestErasure creates pending deletion request`
- `Unit: processErasure deletes all user's videos`
- `Unit: processErasure anonymises comments (body = "[deleted]")`
- `Unit: processErasure clears user PII (email, name, avatar)`
- `Unit: processErasure purges CDN caches`
- `Unit: processErasure logs audit entry`
- `Integration: full erasure flow — request → process → verify all data removed`
- `Unit: erasure of non-existent user → completes with zero counts`

---

#### 12.3 — Self-Hosted Deployment Package

**What**: Create a production Docker image and docker-compose configuration for self-hosted deployment with MinIO (S3), PostgreSQL, and Redis.

**Design**:

```dockerfile
# Dockerfile — multi-stage production build
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
RUN corepack enable && pnpm install --frozen-lockfile && pnpm turbo build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/apps/web/.next/standalone ./
COPY --from=builder /app/apps/web/.next/static ./apps/web/.next/static
COPY --from=builder /app/apps/web/public ./apps/web/public
EXPOSE 3000
CMD ["node", "apps/web/server.js"]
```

```yaml
# docker-compose.production.yml
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgresql://app:${DB_PASSWORD}@postgres:5432/async_video
      REDIS_URL: redis://redis:6379
      S3_ENDPOINT: http://minio:9000
      S3_BUCKET: videos
      S3_ACCESS_KEY: ${MINIO_ACCESS_KEY}
      S3_SECRET_KEY: ${MINIO_SECRET_KEY}
      NEXTAUTH_URL: ${APP_URL}
      NEXTAUTH_SECRET: ${AUTH_SECRET}
    depends_on: [postgres, redis, minio]

  worker:
    build: .
    command: ["node", "packages/jobs/dist/worker.js"]
    environment: # same as app
    depends_on: [postgres, redis, minio]

  postgres:
    image: postgres:16-alpine
    volumes: ["pgdata:/var/lib/postgresql/data"]
    environment:
      POSTGRES_DB: async_video
      POSTGRES_PASSWORD: ${DB_PASSWORD}

  redis:
    image: redis:7-alpine
    volumes: ["redisdata:/data"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    volumes: ["miniodata:/data"]
    environment:
      MINIO_ROOT_USER: ${MINIO_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${MINIO_SECRET_KEY}

volumes:
  pgdata:
  redisdata:
  miniodata:
```

**Testing**:
- `Integration: docker compose build succeeds`
- `Integration: docker compose up → all services healthy`
- `Integration: app reachable on port 3000`
- `Integration: worker connects to Redis and processes jobs`
- `Integration: MinIO accessible, bucket created on startup`
- `E2E: full recording → playback flow in containerised environment`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Video Recording & Upload     ─── requires Phase 1
    │
Phase 3: Video Processing Pipeline    ─── requires Phase 2
    │
Phase 4: Video Playback & Sharing     ─── requires Phase 3
    │
    ├── Phase 5: Library & Search      ─── requires Phase 4
    │       │
    ├── Phase 6: Comments & Collab     ─── requires Phase 4 (can parallel Phase 5)
    │       │
    └── Phase 7: Analytics & Engage    ─── requires Phase 4 (can parallel Phase 5, 6)
            │
Phase 8: Integrations (CRM/Slack)     ─── requires Phase 7 (analytics data for CRM sync)
    │
Phase 9: Public REST API              ─── requires Phase 5 (can parallel Phase 8)
    │
Phase 10: Advanced AI Features        ─── requires Phase 3, 5 (can parallel Phases 8, 9)
    │
Phase 11: Personalisation & CTA       ─── requires Phase 4 (can parallel Phase 10)
    │
Phase 12: Enterprise & Self-Hosting   ─── requires Phase 8 (all integrations in place)
```

### Parallelism Opportunities

- **Phases 5, 6, 7** can be developed concurrently after Phase 4 completes.
- **Phases 8, 9, 10, 11** can be developed concurrently after their respective dependencies.
- **Phase 12** should be last as it packages everything for production.

---

## Definition of Done (per phase)

1. All tasks implemented with production-quality code (no TODO stubs in shipped logic).
2. All unit tests pass (`pnpm turbo test`).
3. All integration tests pass (against local Docker services).
4. ESLint passes with zero errors and zero warnings (`pnpm turbo lint`).
5. TypeScript strict mode passes with zero errors (`pnpm turbo typecheck`).
6. Docker build succeeds (`docker build -t async-video .`).
7. Database migrations run cleanly on a fresh database.
8. New tRPC routes include Zod input validation with meaningful error messages.
9. New REST API endpoints return proper HTTP status codes (201 for creates, 204 for deletes, 4xx for client errors, 5xx for server errors).
10. Sensitive data (tokens, passwords) encrypted at rest; never logged.
11. Audit log entries created for all state-changing operations.
12. New configuration options documented in `.env.example` with comments.
