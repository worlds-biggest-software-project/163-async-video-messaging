# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Async Video Messaging · Created: 2026-05-20

## Philosophy

This model follows a traditional normalized relational design where every domain concept has its own table with well-defined foreign key relationships. The schema separates concerns cleanly: users and teams live in identity tables, videos and their processing pipeline in media tables, engagement tracking in analytics tables, and integrations in their own bounded context.

The normalized approach mirrors how mature video platforms like Mux and Cloudflare Stream structure their internal data — assets, playback configurations, tracks, and captions are all distinct entities with clear ownership chains. This model prioritises data integrity, referential consistency, and the ability to run complex cross-entity SQL queries (e.g., "which videos created by team X had the highest completion rate in Q1?").

This is the safest choice for a team building a long-lived product where schema evolution will be managed through migrations, and where the operational database also serves reporting needs.

**Best for:** Teams that prioritise data integrity, relational querying, and a well-understood migration path over schema flexibility.

**Trade-offs:**
- Pro: Strong referential integrity — orphaned records are impossible with proper FK constraints
- Pro: Standard PostgreSQL tooling — pg_dump, pgAdmin, any ORM works out of the box
- Pro: Complex queries (JOINs across videos, comments, analytics) are straightforward
- Pro: Well-understood by most backend engineers
- Con: Higher table count increases migration complexity
- Con: Adding jurisdiction-specific or per-customer fields requires ALTER TABLE or EAV patterns
- Con: Analytics queries on large tables may require read replicas or materialised views
- Con: Schema changes for new video metadata fields require migrations and deployments

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 27001:2022 | Encryption columns on `videos` and `video_files` tables; `audit_logs` table for access tracking |
| ISO/IEC 27018:2019 | `data_retention_policies` table enforces PII retention rules for video recordings |
| W3C WebVTT | `captions` table stores WebVTT content with language codes per ISO 639-1 |
| RFC 8216 (HLS) | `video_files` table tracks HLS manifest URLs and segment metadata |
| ISO 23009-1 (DASH) | `video_files` table supports DASH MPD URLs alongside HLS |
| RFC 6749 (OAuth 2.0) | `oauth_tokens` and `integrations` tables store OAuth credentials for CRM sync |
| oEmbed | `videos` table includes fields needed for oEmbed response generation |
| WCAG 2.2 | `captions` table ensures every video can have associated accessibility tracks |
| GDPR Art. 17 | `deletion_requests` table tracks right-to-erasure requests with propagation status |

---

## Core Identity & Multi-Tenancy

```sql
-- Organisations (tenants)
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, business, enterprise
    storage_limit_bytes BIGINT,
    custom_domain   VARCHAR(255),
    logo_url        TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
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
    password_hash   TEXT,           -- NULL if SSO-only
    auth_provider   VARCHAR(50),    -- 'email', 'google', 'saml'
    auth_provider_id VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT false,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_users_email ON users (email);

-- Organisation memberships (many-to-many with roles)
CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, member, viewer
    invited_by      UUID REFERENCES users(id),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);
CREATE INDEX idx_org_members_org ON organisation_members (organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members (user_id);

-- Workspaces (subdivisions within an organisation)
CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);
CREATE INDEX idx_workspaces_org ON workspaces (organisation_id);

-- Workspace memberships
CREATE TABLE workspace_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- admin, member, viewer
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, user_id)
);
CREATE INDEX idx_ws_members_ws ON workspace_members (workspace_id);
```

## Video & Media

```sql
-- Videos (core entity — one row per recording)
CREATE TABLE videos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    workspace_id    UUID REFERENCES workspaces(id) ON DELETE SET NULL,
    creator_id      UUID NOT NULL REFERENCES users(id),
    title           VARCHAR(500),
    description     TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'uploading',
    -- status: uploading, processing, ready, failed, deleted
    recording_type  VARCHAR(50) NOT NULL DEFAULT 'screen_camera',
    -- recording_type: screen_only, camera_only, screen_camera
    duration_ms     INTEGER,                    -- duration in milliseconds
    thumbnail_url   TEXT,
    share_token     VARCHAR(64) UNIQUE,         -- for link-based sharing without auth
    visibility      VARCHAR(50) NOT NULL DEFAULT 'workspace',
    -- visibility: private, workspace, organisation, public
    password_hash   TEXT,                       -- optional password protection
    allow_download  BOOLEAN NOT NULL DEFAULT false,
    allow_comments  BOOLEAN NOT NULL DEFAULT true,
    view_count      INTEGER NOT NULL DEFAULT 0,
    external_id     VARCHAR(255),               -- reference in external system (Mux asset ID, etc.)
    deleted_at      TIMESTAMPTZ,                -- soft delete
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_videos_org ON videos (organisation_id);
CREATE INDEX idx_videos_workspace ON videos (workspace_id);
CREATE INDEX idx_videos_creator ON videos (creator_id);
CREATE INDEX idx_videos_share_token ON videos (share_token);
CREATE INDEX idx_videos_status ON videos (status) WHERE status != 'deleted';
CREATE INDEX idx_videos_created ON videos (created_at DESC);

-- Video files (transcoded renditions — HLS, DASH, MP4 download)
CREATE TABLE video_files (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    file_type       VARCHAR(50) NOT NULL,
    -- file_type: source, hls_manifest, dash_manifest, mp4_720p, mp4_1080p, thumbnail, gif_preview
    storage_provider VARCHAR(50) NOT NULL,       -- s3, cloudflare_stream, mux
    storage_key     TEXT NOT NULL,                -- object key / asset ID in provider
    url             TEXT,                         -- CDN URL for playback
    mime_type       VARCHAR(100),
    file_size_bytes BIGINT,
    width           INTEGER,
    height          INTEGER,
    frame_rate      NUMERIC(5,2),
    bitrate_kbps    INTEGER,
    codec           VARCHAR(50),                 -- h264, h265, vp9, av1
    encryption      VARCHAR(50),                 -- none, aes_128, drm
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_video_files_video ON video_files (video_id);

-- Audio tracks (separate from video for multi-track support)
CREATE TABLE audio_tracks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    track_type      VARCHAR(50) NOT NULL DEFAULT 'primary',
    -- track_type: primary, secondary, commentary, music
    language        VARCHAR(10),                 -- ISO 639-1: en, es, fr
    codec           VARCHAR(50),
    bitrate_kbps    INTEGER,
    channels        INTEGER DEFAULT 2,
    storage_key     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audio_tracks_video ON audio_tracks (video_id);

-- Captions / transcripts (WebVTT, SRT)
CREATE TABLE captions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    language        VARCHAR(10) NOT NULL DEFAULT 'en',  -- ISO 639-1
    format          VARCHAR(20) NOT NULL DEFAULT 'webvtt', -- webvtt, srt
    source          VARCHAR(50) NOT NULL DEFAULT 'auto',   -- auto, manual, imported
    content         TEXT NOT NULL,                -- full WebVTT/SRT content
    is_default      BOOLEAN NOT NULL DEFAULT false,
    word_count      INTEGER,
    confidence      NUMERIC(4,3),                -- ASR confidence score 0.000-1.000
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_captions_video ON captions (video_id);
CREATE INDEX idx_captions_language ON captions (video_id, language);

-- Chapters (auto-generated or manual segment markers)
CREATE TABLE chapters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    start_ms        INTEGER NOT NULL,
    end_ms          INTEGER NOT NULL,
    source          VARCHAR(50) NOT NULL DEFAULT 'auto',  -- auto, manual
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_chapters_video ON chapters (video_id);

-- AI-generated summaries and show notes
CREATE TABLE video_summaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    summary_type    VARCHAR(50) NOT NULL,         -- summary, show_notes, key_points, action_items
    content         TEXT NOT NULL,
    model_version   VARCHAR(50),                  -- LLM model used
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_video_summaries_video ON video_summaries (video_id);
```

## Tags & Collections

```sql
-- Tags
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    color           VARCHAR(7),                   -- hex color #FF5733
    UNIQUE (organisation_id, name)
);

-- Video-tag assignments
CREATE TABLE video_tags (
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (video_id, tag_id)
);

-- Collections / folders
CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    workspace_id    UUID REFERENCES workspaces(id) ON DELETE SET NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    parent_id       UUID REFERENCES collections(id) ON DELETE CASCADE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_collections_org ON collections (organisation_id);
CREATE INDEX idx_collections_parent ON collections (parent_id);

-- Video-collection assignments
CREATE TABLE collection_videos (
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (collection_id, video_id)
);
```

## Collaboration & Comments

```sql
-- Comments (threaded, anchored to timestamps)
CREATE TABLE comments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE,  -- threading
    timestamp_ms    INTEGER,                     -- anchor point in video (NULL for general comments)
    body            TEXT NOT NULL,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_comments_video ON comments (video_id);
CREATE INDEX idx_comments_parent ON comments (parent_id);
CREATE INDEX idx_comments_video_ts ON comments (video_id, timestamp_ms);

-- Reactions (emoji reactions on videos or comments)
CREATE TABLE reactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    target_type     VARCHAR(20) NOT NULL,         -- 'video' or 'comment'
    target_id       UUID NOT NULL,
    emoji           VARCHAR(20) NOT NULL,          -- emoji shortcode: thumbsup, heart, etc.
    timestamp_ms    INTEGER,                       -- position in video for video reactions
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, target_type, target_id, emoji)
);
CREATE INDEX idx_reactions_target ON reactions (target_type, target_id);
```

## Sharing & Access Control

```sql
-- Share links (granular sharing beyond visibility settings)
CREATE TABLE share_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    created_by      UUID NOT NULL REFERENCES users(id),
    token           VARCHAR(64) NOT NULL UNIQUE,
    requires_email  BOOLEAN NOT NULL DEFAULT false,  -- viewer must enter email
    password_hash   TEXT,
    expires_at      TIMESTAMPTZ,
    max_views       INTEGER,
    view_count      INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_share_links_token ON share_links (token);
CREATE INDEX idx_share_links_video ON share_links (video_id);

-- Video access grants (explicit per-user access)
CREATE TABLE video_access_grants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    grantee_email   VARCHAR(255) NOT NULL,        -- may be a non-user (external viewer)
    grantee_user_id UUID REFERENCES users(id),
    permission      VARCHAR(50) NOT NULL DEFAULT 'view',  -- view, comment, edit
    granted_by      UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (video_id, grantee_email)
);
CREATE INDEX idx_access_grants_video ON video_access_grants (video_id);
```

## Analytics & Engagement

```sql
-- Video views (one row per unique view session)
CREATE TABLE video_views (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    viewer_user_id  UUID REFERENCES users(id),     -- NULL for anonymous viewers
    viewer_email    VARCHAR(255),                   -- captured if share link requires email
    share_link_id   UUID REFERENCES share_links(id),
    session_id      VARCHAR(64),
    ip_address      INET,
    user_agent      TEXT,
    country_code    VARCHAR(2),                    -- ISO 3166-1 alpha-2
    device_type     VARCHAR(20),                   -- desktop, mobile, tablet
    referrer_url    TEXT,
    watch_duration_ms INTEGER NOT NULL DEFAULT 0,
    percent_watched NUMERIC(5,2) NOT NULL DEFAULT 0,
    completed       BOOLEAN NOT NULL DEFAULT false,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at        TIMESTAMPTZ
);
CREATE INDEX idx_views_video ON video_views (video_id);
CREATE INDEX idx_views_viewer ON video_views (viewer_user_id);
CREATE INDEX idx_views_started ON video_views (started_at);

-- Engagement events (granular player events for heatmap generation)
CREATE TABLE engagement_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    view_id         UUID NOT NULL REFERENCES video_views(id) ON DELETE CASCADE,
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    event_type      VARCHAR(50) NOT NULL,
    -- event_type: play, pause, seek, buffer, speed_change, fullscreen,
    --             cta_click, comment_open, reaction, replay_segment
    timestamp_ms    INTEGER,                       -- position in video
    event_data      JSONB,                         -- event-specific payload
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_engagement_video ON engagement_events (video_id);
CREATE INDEX idx_engagement_view ON engagement_events (view_id);
CREATE INDEX idx_engagement_type ON engagement_events (video_id, event_type);
-- Partitioning by month recommended for production:
-- CREATE TABLE engagement_events (...) PARTITION BY RANGE (occurred_at);
```

## CRM & Integrations

```sql
-- Integration connections (OAuth tokens for external services)
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,          -- salesforce, hubspot, slack, jira, gmail
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    access_token    TEXT,                          -- encrypted at rest
    refresh_token   TEXT,                          -- encrypted at rest
    token_expires_at TIMESTAMPTZ,
    scopes          TEXT[],
    config          JSONB NOT NULL DEFAULT '{}',
    connected_by    UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, provider)
);

-- CRM contacts (synced from Salesforce/HubSpot)
CREATE TABLE crm_contacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    integration_id  UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
    external_id     VARCHAR(255) NOT NULL,         -- Salesforce Contact ID, HubSpot Contact ID
    email           VARCHAR(255),
    name            VARCHAR(255),
    company         VARCHAR(255),
    title           VARCHAR(255),
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_crm_contacts_org ON crm_contacts (organisation_id);
CREATE INDEX idx_crm_contacts_email ON crm_contacts (email);

-- Video-to-CRM attribution
CREATE TABLE crm_video_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    crm_contact_id  UUID NOT NULL REFERENCES crm_contacts(id) ON DELETE CASCADE,
    event_type      VARCHAR(50) NOT NULL,          -- viewed, completed, cta_clicked, replied
    view_id         UUID REFERENCES video_views(id),
    synced_to_crm   BOOLEAN NOT NULL DEFAULT false,
    synced_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_crm_events_video ON crm_video_events (video_id);
CREATE INDEX idx_crm_events_contact ON crm_video_events (crm_contact_id);

-- Webhook subscriptions (outgoing notifications)
CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    secret          TEXT NOT NULL,                  -- HMAC signing secret
    events          TEXT[] NOT NULL,                -- video.created, video.viewed, etc.
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Webhook delivery log
CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id      UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    response_status INTEGER,
    response_body   TEXT,
    attempt         INTEGER NOT NULL DEFAULT 1,
    delivered_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_webhook_del_webhook ON webhook_deliveries (webhook_id);
```

## Call-to-Action & Personalisation

```sql
-- Call-to-action overlays
CREATE TABLE cta_overlays (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    cta_type        VARCHAR(50) NOT NULL,          -- button, form, poll, calendar_link
    label           VARCHAR(255) NOT NULL,
    url             TEXT,
    start_ms        INTEGER NOT NULL,              -- when to show
    end_ms          INTEGER,                       -- when to hide (NULL = until end)
    position        VARCHAR(50) NOT NULL DEFAULT 'bottom_right',
    style           JSONB,                         -- color, size, etc.
    click_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cta_video ON cta_overlays (video_id);

-- Personalisation templates (variable insertion for sales outreach)
CREATE TABLE personalisation_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    variable_name   VARCHAR(100) NOT NULL,         -- recipient_name, company, title
    variable_type   VARCHAR(50) NOT NULL,          -- text_overlay, audio_insert, thumbnail_text
    position        JSONB,                         -- x, y, width, height for overlays
    style           JSONB,                         -- font, color, size
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_personalisation_video ON personalisation_templates (video_id);
```

## Audit & Compliance

```sql
-- Audit log (all significant actions)
CREATE TABLE audit_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    actor_id        UUID REFERENCES users(id),
    action          VARCHAR(100) NOT NULL,
    -- action: video.created, video.deleted, video.shared, member.invited,
    --         settings.changed, integration.connected, etc.
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    details         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_org ON audit_logs (organisation_id);
CREATE INDEX idx_audit_action ON audit_logs (action);
CREATE INDEX idx_audit_resource ON audit_logs (resource_type, resource_id);
CREATE INDEX idx_audit_occurred ON audit_logs (occurred_at);

-- Data retention policies
CREATE TABLE data_retention_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    resource_type   VARCHAR(50) NOT NULL,          -- video, view, audit_log
    retention_days  INTEGER NOT NULL,
    auto_delete     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- GDPR deletion requests
CREATE TABLE deletion_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    requested_by    UUID NOT NULL REFERENCES users(id),
    subject_email   VARCHAR(255) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- status: pending, processing, completed, failed
    videos_deleted  INTEGER NOT NULL DEFAULT 0,
    views_deleted   INTEGER NOT NULL DEFAULT 0,
    cdn_purged      BOOLEAN NOT NULL DEFAULT false,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Notification & Processing Queue

```sql
-- Notifications
CREATE TABLE notifications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    notification_type VARCHAR(50) NOT NULL,
    -- notification_type: video_viewed, comment_added, reaction_added,
    --                    mention, video_ready, share_accepted
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    video_id        UUID REFERENCES videos(id),
    actor_id        UUID REFERENCES users(id),
    is_read         BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notifications_user ON notifications (user_id, is_read);
CREATE INDEX idx_notifications_created ON notifications (created_at DESC);

-- Processing jobs (video transcoding, transcript generation, AI analysis)
CREATE TABLE processing_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    video_id        UUID NOT NULL REFERENCES videos(id) ON DELETE CASCADE,
    job_type        VARCHAR(50) NOT NULL,
    -- job_type: transcode, transcript, summary, chapter_detect,
    --           filler_removal, thumbnail_generate
    status          VARCHAR(50) NOT NULL DEFAULT 'queued',
    -- status: queued, processing, completed, failed, cancelled
    provider        VARCHAR(50),                   -- mux, cloudflare, whisper, openai
    external_job_id VARCHAR(255),
    progress        INTEGER DEFAULT 0,             -- 0-100
    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jobs_video ON processing_jobs (video_id);
CREATE INDEX idx_jobs_status ON processing_jobs (status) WHERE status IN ('queued', 'processing');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 5 | organisations, users, organisation_members, workspaces, workspace_members |
| Video & Media | 6 | videos, video_files, audio_tracks, captions, chapters, video_summaries |
| Tags & Collections | 4 | tags, video_tags, collections, collection_videos |
| Collaboration | 2 | comments, reactions |
| Sharing & Access | 2 | share_links, video_access_grants |
| Analytics & Engagement | 2 | video_views, engagement_events |
| CRM & Integrations | 5 | integrations, crm_contacts, crm_video_events, webhooks, webhook_deliveries |
| CTA & Personalisation | 2 | cta_overlays, personalisation_templates |
| Audit & Compliance | 3 | audit_logs, data_retention_policies, deletion_requests |
| Notifications & Processing | 2 | notifications, processing_jobs |
| **Total** | **33** | |

---

## Key Design Decisions

1. **Shared-schema multi-tenancy with `organisation_id` foreign keys.** Every tenant-scoped table carries an `organisation_id` column. Row-Level Security (RLS) policies can be layered on for defence-in-depth. This keeps the schema simple while supporting thousands of tenants.

2. **Soft deletion for videos.** The `deleted_at` column on `videos` supports GDPR right-to-erasure workflows that require a grace period before permanent deletion and CDN cache purging.

3. **Separate `video_files` table for renditions.** A single recording produces multiple files (source, HLS manifest, DASH manifest, MP4 downloads at various resolutions). The one-to-many relationship keeps the `videos` table clean.

4. **Engagement events designed for partitioning.** The `engagement_events` table will grow fastest. The schema supports PostgreSQL range partitioning by `occurred_at` to keep queries performant as data scales.

5. **Comments use adjacency list threading.** A self-referencing `parent_id` column provides simple threaded replies capped at 3-4 levels. Recursive CTEs handle rendering.

6. **CRM attribution as a first-class entity.** `crm_contacts` and `crm_video_events` provide pipeline attribution without polluting the core video tables — a key differentiator for sales-focused deployments.

7. **Processing jobs as a dedicated table.** Video processing is asynchronous and multi-step (transcode, transcript, summary, chapters). Tracking each job independently enables progress reporting and retry logic.

8. **OAuth tokens stored encrypted.** The `integrations` table stores access and refresh tokens. These must be encrypted at rest using application-level encryption (e.g., AES-256-GCM) per ISO 27001 requirements.
