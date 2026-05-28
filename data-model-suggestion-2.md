# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Async Video Messaging · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable domain event appended to an event store. The current state of any entity (video, comment, share link) is derived by replaying its event stream. Read-optimised projections (materialised views) serve the UI and API, while the event store serves as the authoritative source of truth.

This pattern is used by systems where full auditability is non-negotiable — financial ledgers, healthcare records, and compliance-heavy SaaS platforms. For an async video messaging platform, event sourcing provides a complete history of every video lifecycle event (uploaded, transcoded, shared, viewed, commented, deleted), enabling temporal queries ("who had access to this video on March 15th?"), regulatory compliance, and AI-powered analytics on behavioural patterns.

The CQRS (Command Query Responsibility Segregation) layer separates write commands (record, share, comment) from read queries (list my videos, show analytics). This allows independent scaling — analytics reads can hit a denormalised read store while writes append to the event store.

**Best for:** Regulated industries, enterprise deployments requiring full audit trails, and platforms where temporal queries and compliance reporting are core requirements.

**Trade-offs:**
- Pro: Complete audit trail — every change is recorded with timestamp, actor, and payload
- Pro: Temporal queries are trivial — replay events to any point in time
- Pro: Excellent for AI analytics — event streams are natural training data for engagement models
- Pro: GDPR compliance — deletion events can be appended without destroying history (crypto-shredding)
- Con: Higher complexity — developers must understand event replay and projection rebuilding
- Con: Eventual consistency between event store and read models
- Con: More storage — events accumulate; requires snapshot strategies for performance
- Con: Debugging requires tracing through event chains rather than inspecting a single row

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/IEC 27001:2022 | Event store is append-only — tamper-evident audit trail by design |
| ISO/IEC 27018:2019 | Crypto-shredding pattern: encrypt PII with per-user keys; destroy key for erasure |
| GDPR Art. 17 | Erasure via crypto-shredding — append `UserDataErased` event, destroy encryption key |
| GDPR Art. 32 | All events encrypted at rest; event payloads containing PII use envelope encryption |
| W3C WebVTT | Transcripts stored as `TranscriptGenerated` events; projected into read model |
| RFC 8216 (HLS) | `VideoTranscoded` events carry HLS manifest URLs in payload |
| OWASP API Security | Command handlers validate authorization before appending events |
| SOC 2 Type II | Event store provides immutable evidence for SOC 2 audit procedures |

---

## Event Store (Write Side)

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(50) NOT NULL,
    -- stream_type: Video, User, Organisation, Workspace, Comment,
    --              ShareLink, Integration, Subscription
    stream_id       UUID NOT NULL,                 -- aggregate root ID
    event_type      VARCHAR(100) NOT NULL,
    -- Examples:
    --   VideoUploadStarted, VideoUploadCompleted, VideoTranscoded,
    --   VideoTitleChanged, VideoShared, VideoDeleted,
    --   TranscriptGenerated, ChaptersDetected, SummaryGenerated,
    --   CommentAdded, CommentResolved, ReactionAdded,
    --   VideoViewed, ViewerEngagement, CTAClicked,
    --   ShareLinkCreated, ShareLinkRevoked,
    --   UserCreated, UserJoinedOrganisation, UserRoleChanged,
    --   IntegrationConnected, CRMContactSynced, CRMEventPushed
    version         INTEGER NOT NULL,              -- monotonic per stream for optimistic concurrency
    payload         JSONB NOT NULL,                -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example: {"actor_id": "uuid", "ip": "1.2.3.4", "user_agent": "...",
    --                    "correlation_id": "uuid", "causation_id": "uuid"}
    organisation_id UUID NOT NULL,                 -- tenant partition key
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)       -- optimistic concurrency control
) PARTITION BY RANGE (occurred_at);

-- Create monthly partitions (example for initial months)
CREATE TABLE events_2026_05 PARTITION OF events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE events_2026_06 PARTITION OF events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

-- Indexes for event retrieval
CREATE INDEX idx_events_stream ON events (stream_type, stream_id, version);
CREATE INDEX idx_events_type ON events (event_type, occurred_at);
CREATE INDEX idx_events_org ON events (organisation_id, occurred_at);
CREATE INDEX idx_events_correlation ON events ((metadata->>'correlation_id'))
    WHERE metadata->>'correlation_id' IS NOT NULL;
```

### Example Event Payloads

```sql
-- VideoUploadStarted
-- payload: {
--   "title": "Q2 Product Update",
--   "recording_type": "screen_camera",
--   "creator_id": "user-uuid",
--   "workspace_id": "ws-uuid",
--   "source_file": {"storage_key": "uploads/abc.webm", "size_bytes": 52428800}
-- }

-- VideoTranscoded
-- payload: {
--   "renditions": [
--     {"type": "hls_manifest", "url": "https://cdn.example.com/v/abc/master.m3u8", "codec": "h264"},
--     {"type": "mp4_720p", "url": "https://cdn.example.com/v/abc/720p.mp4", "size_bytes": 31457280}
--   ],
--   "duration_ms": 145000,
--   "width": 1920, "height": 1080,
--   "thumbnail_url": "https://cdn.example.com/v/abc/thumb.jpg"
-- }

-- TranscriptGenerated
-- payload: {
--   "language": "en",
--   "format": "webvtt",
--   "content": "WEBVTT\n\n00:00.000 --> 00:03.500\nHey team, quick update on...",
--   "word_count": 487,
--   "confidence": 0.94,
--   "model": "whisper-large-v3"
-- }

-- VideoViewed
-- payload: {
--   "viewer_user_id": "user-uuid-or-null",
--   "viewer_email": "viewer@example.com",
--   "share_link_id": "link-uuid",
--   "session_id": "sess-abc",
--   "country_code": "US",
--   "device_type": "desktop",
--   "watch_duration_ms": 89000,
--   "percent_watched": 61.38,
--   "completed": false
-- }

-- ViewerEngagement (granular player events — high volume)
-- payload: {
--   "view_id": "view-uuid",
--   "events": [
--     {"type": "play", "timestamp_ms": 0, "at": "2026-05-20T10:00:01Z"},
--     {"type": "pause", "timestamp_ms": 45000, "at": "2026-05-20T10:00:46Z"},
--     {"type": "seek", "timestamp_ms": 12000, "at": "2026-05-20T10:00:48Z"},
--     {"type": "play", "timestamp_ms": 12000, "at": "2026-05-20T10:00:48Z"}
--   ]
-- }
```

## Snapshots (Performance Optimisation)

```sql
-- Snapshots cache the current state of an aggregate to avoid replaying all events
CREATE TABLE snapshots (
    stream_type     VARCHAR(50) NOT NULL,
    stream_id       UUID NOT NULL,
    version         INTEGER NOT NULL,              -- event version this snapshot reflects
    state           JSONB NOT NULL,                -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id)
);

-- Example snapshot for a Video aggregate:
-- state: {
--   "id": "video-uuid",
--   "title": "Q2 Product Update",
--   "status": "ready",
--   "creator_id": "user-uuid",
--   "organisation_id": "org-uuid",
--   "workspace_id": "ws-uuid",
--   "duration_ms": 145000,
--   "visibility": "workspace",
--   "view_count": 47,
--   "created_at": "2026-05-20T09:00:00Z",
--   "updated_at": "2026-05-20T10:15:00Z"
-- }
```

## Read Models / Projections (Query Side)

Read models are rebuilt from the event stream. They are disposable — if corrupted, they can be rebuilt by replaying events.

```sql
-- Video read model (denormalised for fast queries)
CREATE TABLE rm_videos (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    workspace_id    UUID,
    creator_id      UUID NOT NULL,
    creator_name    VARCHAR(255),                  -- denormalised from User events
    creator_avatar  TEXT,
    title           VARCHAR(500),
    description     TEXT,
    status          VARCHAR(50) NOT NULL,
    recording_type  VARCHAR(50) NOT NULL,
    duration_ms     INTEGER,
    thumbnail_url   TEXT,
    hls_url         TEXT,                          -- denormalised from VideoTranscoded
    dash_url        TEXT,
    share_token     VARCHAR(64),
    visibility      VARCHAR(50) NOT NULL,
    view_count      INTEGER NOT NULL DEFAULT 0,
    comment_count   INTEGER NOT NULL DEFAULT 0,
    reaction_count  INTEGER NOT NULL DEFAULT 0,
    has_transcript  BOOLEAN NOT NULL DEFAULT false,
    has_chapters    BOOLEAN NOT NULL DEFAULT false,
    has_summary     BOOLEAN NOT NULL DEFAULT false,
    tags            TEXT[],                         -- denormalised tag names
    deleted_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    last_projected  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_videos_org ON rm_videos (organisation_id);
CREATE INDEX idx_rm_videos_workspace ON rm_videos (workspace_id);
CREATE INDEX idx_rm_videos_creator ON rm_videos (creator_id);
CREATE INDEX idx_rm_videos_created ON rm_videos (created_at DESC);
CREATE INDEX idx_rm_videos_search ON rm_videos USING gin (to_tsvector('english', coalesce(title, '') || ' ' || coalesce(description, '')));

-- Video detail read model (includes transcript, chapters, summaries)
CREATE TABLE rm_video_details (
    video_id        UUID PRIMARY KEY,
    transcript_webvtt TEXT,
    transcript_language VARCHAR(10),
    chapters        JSONB,
    -- chapters: [{"title": "Intro", "start_ms": 0, "end_ms": 15000}, ...]
    summary         TEXT,
    show_notes      TEXT,
    key_points      JSONB,
    -- key_points: ["Launched feature X", "Timeline is Q3", ...]
    action_items    JSONB,
    -- action_items: ["Review PR #456", "Schedule demo with client"]
    cta_overlays    JSONB,
    -- cta_overlays: [{"type": "button", "label": "Book Demo", "url": "...", "start_ms": 120000}]
    personalisation_vars JSONB,
    last_projected  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Analytics read model (pre-aggregated per video per day)
CREATE TABLE rm_video_analytics_daily (
    video_id        UUID NOT NULL,
    date            DATE NOT NULL,
    organisation_id UUID NOT NULL,
    views           INTEGER NOT NULL DEFAULT 0,
    unique_viewers  INTEGER NOT NULL DEFAULT 0,
    total_watch_ms  BIGINT NOT NULL DEFAULT 0,
    avg_percent_watched NUMERIC(5,2) NOT NULL DEFAULT 0,
    completions     INTEGER NOT NULL DEFAULT 0,
    cta_clicks      INTEGER NOT NULL DEFAULT 0,
    comments        INTEGER NOT NULL DEFAULT 0,
    reactions       INTEGER NOT NULL DEFAULT 0,
    top_countries   JSONB,
    -- top_countries: {"US": 23, "GB": 8, "DE": 5}
    device_breakdown JSONB,
    -- device_breakdown: {"desktop": 30, "mobile": 6}
    PRIMARY KEY (video_id, date)
);
CREATE INDEX idx_rm_analytics_org ON rm_video_analytics_daily (organisation_id, date);

-- Heatmap read model (engagement intensity per video segment)
CREATE TABLE rm_video_heatmap (
    video_id        UUID NOT NULL,
    segment_start_ms INTEGER NOT NULL,             -- segment start (e.g., every 1000ms)
    segment_end_ms  INTEGER NOT NULL,
    play_count      INTEGER NOT NULL DEFAULT 0,    -- times this segment was played
    rewatch_count   INTEGER NOT NULL DEFAULT 0,    -- replays of this segment
    drop_off_count  INTEGER NOT NULL DEFAULT 0,    -- viewers who stopped here
    PRIMARY KEY (video_id, segment_start_ms)
);

-- User read model (denormalised for display)
CREATE TABLE rm_users (
    id              UUID PRIMARY KEY,
    email           VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    organisations   JSONB,
    -- organisations: [{"id": "uuid", "name": "Acme", "role": "admin"}, ...]
    video_count     INTEGER NOT NULL DEFAULT 0,
    last_login_at   TIMESTAMPTZ,
    last_projected  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Organisation read model
CREATE TABLE rm_organisations (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    plan            VARCHAR(50) NOT NULL,
    member_count    INTEGER NOT NULL DEFAULT 0,
    video_count     INTEGER NOT NULL DEFAULT 0,
    storage_used_bytes BIGINT NOT NULL DEFAULT 0,
    storage_limit_bytes BIGINT,
    last_projected  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Comments read model (flattened for display)
CREATE TABLE rm_comments (
    id              UUID PRIMARY KEY,
    video_id        UUID NOT NULL,
    parent_id       UUID,
    user_id         UUID NOT NULL,
    user_name       VARCHAR(255),                  -- denormalised
    user_avatar     TEXT,                           -- denormalised
    timestamp_ms    INTEGER,
    body            TEXT NOT NULL,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    reply_count     INTEGER NOT NULL DEFAULT 0,
    reactions       JSONB,
    -- reactions: {"thumbsup": 3, "heart": 1}
    created_at      TIMESTAMPTZ NOT NULL,
    last_projected  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rm_comments_video ON rm_comments (video_id, created_at);

-- CRM attribution read model
CREATE TABLE rm_crm_attribution (
    video_id        UUID NOT NULL,
    contact_email   VARCHAR(255) NOT NULL,
    contact_name    VARCHAR(255),
    contact_company VARCHAR(255),
    crm_provider    VARCHAR(50) NOT NULL,
    external_contact_id VARCHAR(255),
    first_viewed_at TIMESTAMPTZ,
    last_viewed_at  TIMESTAMPTZ,
    total_views     INTEGER NOT NULL DEFAULT 0,
    total_watch_ms  BIGINT NOT NULL DEFAULT 0,
    completed       BOOLEAN NOT NULL DEFAULT false,
    cta_clicked     BOOLEAN NOT NULL DEFAULT false,
    synced_to_crm   BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (video_id, contact_email)
);
CREATE INDEX idx_rm_crm_contact ON rm_crm_attribution (contact_email);
```

## Projection Tracking

```sql
-- Tracks the last event processed by each projection for restart/rebuild
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_occurred_at TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Projections: rm_videos, rm_video_details, rm_video_analytics_daily,
--              rm_video_heatmap, rm_users, rm_organisations, rm_comments,
--              rm_crm_attribution
```

## Crypto-Shredding for GDPR Erasure

```sql
-- Per-user encryption keys for PII fields in events
CREATE TABLE user_encryption_keys (
    user_id         UUID PRIMARY KEY,
    key_id          VARCHAR(100) NOT NULL,         -- reference to key in KMS (AWS KMS, Vault)
    algorithm       VARCHAR(50) NOT NULL DEFAULT 'AES-256-GCM',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    destroyed_at    TIMESTAMPTZ                    -- set when GDPR erasure is executed
);
-- When a user exercises right-to-erasure:
-- 1. Append UserDataErased event to the event store
-- 2. Destroy the encryption key in KMS
-- 3. PII in event payloads becomes undecryptable
-- 4. Rebuild projections — PII fields become NULL/anonymised
```

---

## Example Queries

### Replay a video's complete history

```sql
SELECT event_type, version, payload, metadata, occurred_at
FROM events
WHERE stream_type = 'Video' AND stream_id = 'video-uuid'
ORDER BY version ASC;
```

### Reconstruct video state at a specific point in time

```sql
SELECT event_type, payload, occurred_at
FROM events
WHERE stream_type = 'Video'
  AND stream_id = 'video-uuid'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY version ASC;
-- Application replays these events to reconstruct the state as of March 15th
```

### Find all videos a specific viewer accessed in a date range

```sql
SELECT DISTINCT (payload->>'stream_id') AS video_id,
       payload->>'viewer_email' AS email,
       occurred_at
FROM events
WHERE event_type = 'VideoViewed'
  AND payload->>'viewer_email' = 'jane@example.com'
  AND occurred_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY occurred_at DESC;
```

### Rebuild analytics projection from events

```sql
-- Run by the projection engine to rebuild rm_video_analytics_daily
SELECT
    (payload->>'video_id')::UUID AS video_id,
    occurred_at::DATE AS date,
    organisation_id,
    COUNT(*) AS views,
    COUNT(DISTINCT payload->>'session_id') AS unique_viewers,
    SUM((payload->>'watch_duration_ms')::INTEGER) AS total_watch_ms,
    AVG((payload->>'percent_watched')::NUMERIC) AS avg_percent_watched,
    COUNT(*) FILTER (WHERE (payload->>'completed')::BOOLEAN) AS completions
FROM events
WHERE event_type = 'VideoViewed'
  AND occurred_at >= '2026-05-01'
GROUP BY 1, 2, 3;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | `events` (partitioned by month) |
| Snapshots | 1 | `snapshots` for aggregate state caching |
| Read Models | 8 | rm_videos, rm_video_details, rm_video_analytics_daily, rm_video_heatmap, rm_users, rm_organisations, rm_comments, rm_crm_attribution |
| Infrastructure | 2 | projection_checkpoints, user_encryption_keys |
| **Total** | **12** | Plus monthly partitions for events |

---

## Key Design Decisions

1. **Single `events` table as the sole source of truth.** All state is derived from events. This eliminates the "two sources of truth" problem and makes the audit trail complete by construction.

2. **Monthly partitioning on the events table.** Video platforms generate high event volumes (especially `ViewerEngagement` events). Monthly partitions enable efficient archival and pruning of old engagement data while retaining lifecycle events indefinitely.

3. **Optimistic concurrency via `(stream_type, stream_id, version)` unique constraint.** Concurrent writes to the same aggregate are detected and rejected, preventing lost-update anomalies without pessimistic locking.

4. **Separate read models per access pattern.** `rm_videos` serves the library list view, `rm_video_details` serves the player page, `rm_video_analytics_daily` serves the analytics dashboard, and `rm_video_heatmap` serves the engagement visualisation. Each is optimised for its query pattern.

5. **Crypto-shredding for GDPR compliance.** Rather than attempting to delete events from the immutable store, PII is encrypted with per-user keys stored in a KMS. Erasure destroys the key, making PII permanently unrecoverable while preserving the event structure for audit purposes.

6. **Projection checkpoints enable reliable rebuilds.** If a read model becomes corrupted or a new projection is added, it can be rebuilt from scratch by replaying events from the beginning, or incrementally from the last checkpoint.

7. **Engagement events batched in payload arrays.** The `ViewerEngagement` event carries an array of player events in its payload rather than one event-store row per play/pause/seek. This reduces write amplification for the highest-volume event type while preserving full granularity.

8. **Metadata envelope for correlation and causation tracking.** Every event carries a `correlation_id` (linking events triggered by the same user action) and `causation_id` (linking an event to the event that caused it), enabling distributed tracing across the event stream.
