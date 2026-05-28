# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Async Video Messaging · Created: 2026-05-20

## Philosophy

This model uses a reduced set of relational tables for core entities (users, organisations, videos) with well-typed columns for frequently queried fields, while pushing variable, evolving, and domain-specific data into PostgreSQL JSONB columns. The result is a schema that balances structure with flexibility — you get relational integrity and fast indexed queries on the fields that matter most, combined with schemaless extensibility for everything else.

This approach is inspired by how modern SaaS platforms like Stripe, Shopify, and Slack store core operational data relationally but attach JSONB "metadata" or "properties" bags for customer-specific or feature-flag-driven fields. For an async video messaging platform, JSONB is particularly valuable for: video processing pipeline metadata (which varies by provider — Mux vs. Cloudflare Stream vs. self-hosted), per-organisation customisation settings, CRM sync configurations that differ between Salesforce and HubSpot, and analytics event payloads whose structure evolves with new player features.

The key discipline is deciding what goes in columns vs. JSONB. The rule: if you query it in a WHERE clause, JOIN on it, or enforce a constraint on it, it gets a column. Everything else goes in JSONB.

**Best for:** Teams building an MVP with evolving requirements, multi-provider architectures, and platforms where deployment flexibility (cloud vs. self-hosted) means schema variation.

**Trade-offs:**
- Pro: Fewer tables (~18 vs. ~33) — faster to build, fewer migrations to manage
- Pro: Adding new fields does not require ALTER TABLE — just update the application code
- Pro: JSONB GIN indexes enable fast containment queries on nested structures
- Pro: Multi-provider support (Mux, Cloudflare, self-hosted) without separate tables per provider
- Pro: Ideal for rapid iteration during early product development
- Con: No database-level constraints on JSONB fields — validation must be in the application
- Con: JSONB fields are harder to document and discover — requires disciplined API docs
- Con: Complex JSONB queries can be slower than relational JOINs on indexed columns
- Con: ORM support for JSONB varies — some frameworks treat it as an opaque blob

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 27001:2022 | `audit_log` table with JSONB `details` captures arbitrary security events |
| ISO/IEC 27018:2019 | Organisation `settings` JSONB includes data retention and PII handling configuration |
| W3C WebVTT | Transcript content stored in `videos.media` JSONB alongside other media metadata |
| RFC 8216 (HLS) / ISO 23009-1 (DASH) | Playback URLs stored in `videos.media` JSONB — provider-agnostic structure |
| RFC 6749 (OAuth 2.0) | Integration credentials in `integrations.credentials` JSONB — schema varies by provider |
| oEmbed | Generated from `videos` columns — title, duration, thumbnail fields are relational |
| GDPR Art. 17 | Soft delete with `deleted_at`; JSONB fields cleared on permanent deletion |
| WCAG 2.2 | Captions stored in `videos.media.captions` JSONB array |

---

## Core Tables

```sql
-- Organisations (tenants)
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "storage_limit_bytes": 53687091200,
    --   "custom_domain": "videos.acme.com",
    --   "branding": {"logo_url": "...", "accent_color": "#3B82F6", "player_theme": "dark"},
    --   "retention": {"video_days": 365, "analytics_days": 90, "audit_days": 730},
    --   "features": {"ai_summaries": true, "crm_sync": true, "personalisation": false},
    --   "security": {"sso_provider": "okta", "enforce_2fa": true, "ip_allowlist": ["10.0.0.0/8"]}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_organisations_slug ON organisations (slug);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    auth            JSONB NOT NULL DEFAULT '{}',
    -- auth example: {
    --   "provider": "google",
    --   "provider_id": "google-oauth-id",
    --   "password_hash": null,
    --   "email_verified": true,
    --   "mfa_enabled": false
    -- }
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example: {
    --   "notification_email": true,
    --   "notification_slack": true,
    --   "default_visibility": "workspace",
    --   "default_recording_type": "screen_camera",
    --   "timezone": "America/New_York"
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_email ON users (email);

-- Memberships (covers both org and workspace membership in one table)
CREATE TABLE memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    workspace_ids   UUID[] NOT NULL DEFAULT '{}',  -- workspaces this user belongs to
    org_role        VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, member
    workspace_roles JSONB NOT NULL DEFAULT '{}',
    -- workspace_roles example: {
    --   "ws-uuid-1": "admin",
    --   "ws-uuid-2": "member"
    -- }
    invited_by      UUID REFERENCES users(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, organisation_id)
);
CREATE INDEX idx_memberships_org ON memberships (organisation_id);
CREATE INDEX idx_memberships_user ON memberships (user_id);
CREATE INDEX idx_memberships_workspaces ON memberships USING gin (workspace_ids);

-- Workspaces
CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "default_visibility": "workspace",
    --   "auto_transcript": true,
    --   "auto_chapters": true
    -- }
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);
CREATE INDEX idx_workspaces_org ON workspaces (organisation_id);
```

## Videos (Core + JSONB Extensions)

```sql
-- Videos: relational columns for query-critical fields, JSONB for everything else
CREATE TABLE videos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    workspace_id    UUID REFERENCES workspaces(id) ON DELETE SET NULL,
    creator_id      UUID NOT NULL REFERENCES users(id),
    
    -- Relational columns: fields used in WHERE, ORDER BY, and JOINs
    title           VARCHAR(500),
    status          VARCHAR(50) NOT NULL DEFAULT 'uploading',
    recording_type  VARCHAR(50) NOT NULL DEFAULT 'screen_camera',
    visibility      VARCHAR(50) NOT NULL DEFAULT 'workspace',
    duration_ms     INTEGER,
    share_token     VARCHAR(64) UNIQUE,
    view_count      INTEGER NOT NULL DEFAULT 0,
    comment_count   INTEGER NOT NULL DEFAULT 0,
    
    -- JSONB columns: flexible, evolving, provider-specific data
    media           JSONB NOT NULL DEFAULT '{}',
    -- media example: {
    --   "source": {
    --     "provider": "cloudflare_stream",
    --     "asset_id": "cf-stream-uid",
    --     "storage_key": "uploads/abc.webm",
    --     "size_bytes": 52428800,
    --     "mime_type": "video/webm"
    --   },
    --   "playback": {
    --     "hls_url": "https://cdn.example.com/v/abc/master.m3u8",
    --     "dash_url": "https://cdn.example.com/v/abc/manifest.mpd",
    --     "thumbnail_url": "https://cdn.example.com/v/abc/thumb.jpg",
    --     "gif_preview_url": "https://cdn.example.com/v/abc/preview.gif",
    --     "mp4_downloads": {
    --       "720p": "https://cdn.example.com/v/abc/720p.mp4",
    --       "1080p": "https://cdn.example.com/v/abc/1080p.mp4"
    --     }
    --   },
    --   "encoding": {
    --     "codec": "h264",
    --     "width": 1920, "height": 1080,
    --     "frame_rate": 30.0,
    --     "bitrate_kbps": 2500
    --   },
    --   "captions": [
    --     {"language": "en", "format": "webvtt", "url": "https://cdn.example.com/v/abc/en.vtt",
    --      "source": "auto", "confidence": 0.94}
    --   ],
    --   "audio_tracks": [
    --     {"type": "primary", "language": "en", "codec": "aac", "channels": 2}
    --   ]
    -- }
    
    ai              JSONB NOT NULL DEFAULT '{}',
    -- ai example: {
    --   "transcript": {
    --     "content": "WEBVTT\n\n00:00.000 --> 00:03.500\nHey team...",
    --     "language": "en",
    --     "word_count": 487,
    --     "model": "whisper-large-v3",
    --     "generated_at": "2026-05-20T09:05:00Z"
    --   },
    --   "summary": "Quick update on Q2 product roadmap covering...",
    --   "show_notes": "## Key Topics\n- Feature X launch timeline...",
    --   "key_points": ["Launched feature X", "Timeline is Q3"],
    --   "action_items": ["Review PR #456", "Schedule demo"],
    --   "chapters": [
    --     {"title": "Intro", "start_ms": 0, "end_ms": 15000},
    --     {"title": "Feature X Demo", "start_ms": 15000, "end_ms": 85000},
    --     {"title": "Q&A", "start_ms": 85000, "end_ms": 145000}
    --   ],
    --   "filler_removal": {
    --     "removed_count": 12,
    --     "saved_ms": 8500,
    --     "processed_at": "2026-05-20T09:06:00Z"
    --   }
    -- }
    
    sharing         JSONB NOT NULL DEFAULT '{}',
    -- sharing example: {
    --   "password_hash": null,
    --   "allow_download": false,
    --   "allow_comments": true,
    --   "require_email": false,
    --   "embed_allowed": true,
    --   "embed_domains": ["*.acme.com"],
    --   "cta_overlays": [
    --     {"type": "button", "label": "Book Demo", "url": "https://cal.com/acme/demo",
    --      "start_ms": 120000, "position": "bottom_right",
    --      "style": {"bg_color": "#3B82F6", "text_color": "#FFFFFF"}}
    --   ],
    --   "personalisation": {
    --     "enabled": true,
    --     "variables": [
    --       {"name": "recipient_name", "type": "text_overlay", "position": {"x": 50, "y": 900},
    --        "style": {"font": "Inter", "size": 24, "color": "#FFFFFF"}}
    --     ]
    --   }
    -- }
    
    tags            TEXT[] NOT NULL DEFAULT '{}',   -- denormalised tag names as array
    
    deleted_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core query indexes
CREATE INDEX idx_videos_org ON videos (organisation_id);
CREATE INDEX idx_videos_workspace ON videos (workspace_id);
CREATE INDEX idx_videos_creator ON videos (creator_id);
CREATE INDEX idx_videos_status ON videos (status) WHERE deleted_at IS NULL;
CREATE INDEX idx_videos_created ON videos (created_at DESC);
CREATE INDEX idx_videos_share_token ON videos (share_token) WHERE share_token IS NOT NULL;

-- JSONB indexes for common queries
CREATE INDEX idx_videos_media_provider ON videos USING btree (
    (media->'source'->>'provider')
) WHERE media->'source'->>'provider' IS NOT NULL;

-- GIN index for tag array searches
CREATE INDEX idx_videos_tags ON videos USING gin (tags);

-- Full-text search on title + transcript
CREATE INDEX idx_videos_search ON videos USING gin (
    to_tsvector('english', coalesce(title, '') || ' ' || coalesce(ai->>'summary', ''))
);
```

## Collaboration

```sql
-- Comments (threaded, timestamp-anchored)
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,
    timestamp_ms    INTEGER,
    body            TEXT NOT NULL,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    reactions       JSONB NOT NULL DEFAULT '{}',
    -- reactions example: {"thumbsup": ["user-uuid-1", "user-uuid-2"], "heart": ["user-uuid-3"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_comments_video ON comments (video_id);
CREATE INDEX idx_comments_parent ON comments (parent_id);

-- Share links
CREATE TABLE share_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    created_by      UUID NOT NULL REFERENCES users(id),
    token           VARCHAR(64) NOT NULL UNIQUE,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {
    --   "requires_email": true,
    --   "password_hash": null,
    --   "expires_at": "2026-06-20T00:00:00Z",
    --   "max_views": 100,
    --   "allowed_domains": ["@acme.com"]
    -- }
    view_count      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_share_links_token ON share_links (token);
CREATE INDEX idx_share_links_video ON share_links (video_id);
```

## Analytics

```sql
-- Video views (one per session — high volume)
CREATE TABLE video_views (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    organisation_id UUID NOT NULL,
    viewer          JSONB NOT NULL DEFAULT '{}',
    -- viewer example: {
    --   "user_id": "uuid-or-null",
    --   "email": "viewer@example.com",
    --   "share_link_id": "link-uuid",
    --   "session_id": "sess-abc",
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "Mozilla/5.0...",
    --   "country_code": "US",
    --   "device_type": "desktop",
    --   "referrer_url": "https://mail.google.com"
    -- }
    watch_duration_ms INTEGER NOT NULL DEFAULT 0,
    percent_watched NUMERIC(5,2) NOT NULL DEFAULT 0,
    completed       BOOLEAN NOT NULL DEFAULT false,
    engagement      JSONB,
    -- engagement example: {
    --   "events": [
    --     {"type": "play", "ts_ms": 0, "at": "2026-05-20T10:00:01Z"},
    --     {"type": "pause", "ts_ms": 45000, "at": "2026-05-20T10:00:46Z"},
    --     {"type": "seek", "ts_ms": 12000, "at": "2026-05-20T10:00:48Z"},
    --     {"type": "cta_click", "ts_ms": 120000, "at": "2026-05-20T10:02:01Z",
    --      "cta_label": "Book Demo"}
    --   ],
    --   "segments_watched": [[0, 45000], [12000, 89000]],
    --   "rewatch_segments": [[12000, 25000]]
    -- }
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ
) PARTITION BY RANGE (started_at);

-- Monthly partitions
CREATE TABLE video_views_2026_05 PARTITION OF video_views
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE video_views_2026_06 PARTITION OF video_views
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_views_video ON video_views (video_id);
CREATE INDEX idx_views_org ON video_views (organisation_id, started_at);
CREATE INDEX idx_views_started ON video_views (started_at);
```

## Integrations & CRM

```sql
-- Integrations (one row per connected service per org)
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    credentials     JSONB NOT NULL DEFAULT '{}',
    -- credentials example (Salesforce): {
    --   "access_token": "encrypted:...",
    --   "refresh_token": "encrypted:...",
    --   "instance_url": "https://acme.my.salesforce.com",
    --   "token_expires_at": "2026-05-20T11:00:00Z",
    --   "scopes": ["api", "refresh_token"]
    -- }
    -- credentials example (Slack): {
    --   "bot_token": "encrypted:xoxb-...",
    --   "channel_mappings": {"workspace-uuid-1": "#engineering-updates"}
    -- }
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example (HubSpot): {
    --   "sync_contacts": true,
    --   "auto_log_views": true,
    --   "pipeline_id": "hubspot-pipeline-id",
    --   "deal_stage_on_view": "Video Watched"
    -- }
    connected_by    UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, provider)
);

-- CRM sync log (tracks what has been synced to/from CRM)
CREATE TABLE crm_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
    direction       VARCHAR(10) NOT NULL,          -- inbound, outbound
    entity_type     VARCHAR(50) NOT NULL,          -- contact, video_view, deal_activity
    external_id     VARCHAR(255),
    payload         JSONB NOT NULL,
    -- payload example (outbound video_view): {
    --   "contact_email": "prospect@example.com",
    --   "video_title": "Q2 Product Update",
    --   "watch_percent": 85.5,
    --   "completed": true,
    --   "cta_clicked": true
    -- }
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    error           TEXT,
    synced_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_crm_sync_integration ON crm_sync_log (integration_id, created_at);
CREATE INDEX idx_crm_sync_status ON crm_sync_log (status) WHERE status = 'pending';
```

## Processing & Notifications

```sql
-- Processing jobs
CREATE TABLE processing_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    job_type        VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',
    provider        VARCHAR(50),
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example (transcode): {
    --   "provider": "mux",
    --   "external_job_id": "mux-asset-id",
    --   "requested_renditions": ["hls", "mp4_720p", "mp4_1080p"],
    --   "input_url": "https://storage.example.com/uploads/abc.webm"
    -- }
    result          JSONB,
    -- result example (transcode complete): {
    --   "hls_url": "https://stream.mux.com/abc/master.m3u8",
    --   "duration_ms": 145000,
    --   "renditions": [...]
    -- }
    progress        INTEGER DEFAULT 0,
    error           TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jobs_video ON processing_jobs (video_id);
CREATE INDEX idx_jobs_status ON processing_jobs (status) WHERE status IN ('queued', 'processing');

-- Notifications
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type            VARCHAR(50) NOT NULL,
    data            JSONB NOT NULL,
    -- data example: {
    --   "video_id": "uuid",
    --   "video_title": "Q2 Product Update",
    --   "actor_id": "uuid",
    --   "actor_name": "Jane Smith",
    --   "comment_id": "uuid",
    --   "comment_preview": "Great update! Can you clarify..."
    -- }
    is_read         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notifications_user ON notifications (user_id, is_read);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    actor_id        UUID,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example: {
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "Mozilla/5.0...",
    --   "changes": {"visibility": {"from": "private", "to": "public"}},
    --   "share_link_token": "abc123"
    -- }
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_audit_org ON audit_log (organisation_id, occurred_at);
CREATE INDEX idx_audit_action ON audit_log (action);
CREATE INDEX idx_audit_resource ON audit_log (resource_type, resource_id);
```

## Collections

```sql
-- Collections / folders (lightweight — hierarchy via parent_id)
CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    workspace_id    UUID REFERENCES workspaces(id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    parent_id       UUID REFERENCES collections(id) ON DELETE CASCADE,
    video_ids       UUID[] NOT NULL DEFAULT '{}',  -- ordered list of video IDs
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {
    --   "description": "Sales demo library",
    --   "cover_image_url": "...",
    --   "sort_order": 0
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_collections_org ON collections (organisation_id);
CREATE INDEX idx_collections_parent ON collections (parent_id);
CREATE INDEX idx_collections_videos ON collections USING gin (video_ids);
```

---

## Example Queries

### Full-text search across titles and AI summaries

```sql
SELECT id, title, ai->>'summary' AS summary, view_count, created_at
FROM videos
WHERE organisation_id = 'org-uuid'
  AND deleted_at IS NULL
  AND to_tsvector('english', coalesce(title, '') || ' ' || coalesce(ai->>'summary', ''))
      @@ plainto_tsquery('english', 'product roadmap Q2')
ORDER BY created_at DESC
LIMIT 20;
```

### Find videos by media provider

```sql
SELECT id, title, media->'source'->>'provider' AS provider,
       media->'source'->>'asset_id' AS asset_id
FROM videos
WHERE organisation_id = 'org-uuid'
  AND media->'source'->>'provider' = 'mux'
  AND deleted_at IS NULL;
```

### Aggregate engagement heatmap from JSONB events

```sql
-- Generate a heatmap by extracting segment data from the engagement JSONB
SELECT
    v.id AS video_id,
    v.title,
    segment.start_ms,
    segment.end_ms,
    COUNT(*) AS times_watched
FROM video_views vv
JOIN videos v ON v.id = vv.video_id
CROSS JOIN LATERAL (
    SELECT
        (elem->>0)::INTEGER AS start_ms,
        (elem->>1)::INTEGER AS end_ms
    FROM jsonb_array_elements(vv.engagement->'segments_watched') AS elem
) segment
WHERE vv.video_id = 'video-uuid'
  AND vv.started_at >= '2026-05-01'
GROUP BY v.id, v.title, segment.start_ms, segment.end_ms
ORDER BY segment.start_ms;
```

### Find videos with specific tags

```sql
SELECT id, title, tags, view_count
FROM videos
WHERE organisation_id = 'org-uuid'
  AND tags @> ARRAY['sales', 'demo']
  AND deleted_at IS NULL
ORDER BY view_count DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 4 | organisations, users, memberships, workspaces |
| Videos | 1 | Single `videos` table with JSONB for media, AI, sharing |
| Collaboration | 2 | comments, share_links |
| Analytics | 1 | video_views (partitioned, JSONB engagement payload) |
| Integrations & CRM | 2 | integrations, crm_sync_log |
| Processing & Notifications | 3 | processing_jobs, notifications, audit_log |
| Collections | 1 | collections (with UUID array for video ordering) |
| **Total** | **14** | Plus monthly partitions for video_views and audit_log |

---

## Key Design Decisions

1. **Single `videos` table with 3 JSONB columns (`media`, `ai`, `sharing`).** Instead of 6+ relational tables for files, captions, chapters, summaries, CTAs, and personalisation, a single row holds all video data. The JSONB columns are semantically grouped: `media` for storage/playback, `ai` for ML-generated content, `sharing` for access control and CTAs.

2. **Reactions embedded in comments as JSONB.** Rather than a separate `reactions` table, reactions are stored as `{"emoji": ["user-id-1", "user-id-2"]}` inside each comment. This eliminates a high-write, low-value join table and makes fetching a comment with its reactions a single read.

3. **Tags as a PostgreSQL array with GIN index.** Tags are denormalised into a `TEXT[]` column on `videos` rather than requiring a `tags` + `video_tags` junction table. The GIN index on the array supports `@>` containment queries efficiently.

4. **Collections use a `UUID[]` array for video ordering.** Instead of a junction table with `sort_order`, the ordered list of video IDs is stored as an array. This makes reordering a single array update rather than multiple row updates.

5. **Engagement events embedded in view sessions.** The `video_views.engagement` JSONB column stores the full array of player events (play, pause, seek, CTA click) for each session. This avoids a separate high-volume events table while retaining full granularity. The trade-off is that per-event querying requires JSONB array extraction (see heatmap query example).

6. **Multi-provider support without provider-specific tables.** The `media.source.provider` field distinguishes between Mux, Cloudflare Stream, and self-hosted storage. Provider-specific fields (Mux asset IDs, Cloudflare Stream UIDs, S3 bucket paths) all live in the same JSONB structure. Adding a new provider requires no schema changes.

7. **CRM sync via a generic log table.** Instead of separate tables for Salesforce contacts, HubSpot contacts, and deal activities, a single `crm_sync_log` table records all sync operations as JSONB payloads. This supports adding new CRM providers without schema changes.

8. **Partitioned `video_views` and `audit_log`.** These are the highest-volume tables. Monthly range partitioning keeps query performance stable and enables cost-effective archival of old data.
