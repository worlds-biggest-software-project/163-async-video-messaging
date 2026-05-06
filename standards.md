# Standards & API Reference

> Project: Async Video Messaging · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management; governs access controls and encryption for video recordings containing confidential business communications, customer demos, and internal communications; Annex A A.8.3 (Information access restriction) and A.8.24 (Cryptography). URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 27018:2019 — Protection of PII in Public Clouds** — Governs processing of personal data in video recordings (faces, voices, screen contents with personal data); requires retention policy transparency and data minimisation for cloud-hosted async video platforms. URL: https://www.iso.org/standard/76559.html

### W3C & IETF Standards

- **W3C Media Capture and Streams API (MediaDevices)** — W3C specification defining `getUserMedia()` and `getDisplayMedia()` for accessing camera, microphone, and screen capture in browsers; the foundational browser API for recording in async video messaging platforms. URL: https://www.w3.org/TR/mediacapture-streams/

- **W3C MediaStream Recording API** — W3C specification defining `MediaRecorder` for recording audio/video streams in browsers; used to capture webcam + screen composite recordings for async video messages without browser plugins. URL: https://www.w3.org/TR/mediastream-recording/

- **RFC 8825 — WebRTC Overview** — Used for peer-to-peer real-time preview during recording and for any synchronous video reply features; WebRTC's DTLS-SRTP provides encryption for live capture transport. URL: https://datatracker.ietf.org/doc/html/rfc8825

- **RFC 8216 — HLS: HTTP Live Streaming** — Apple's IETF-standardised adaptive bitrate streaming protocol; the most widely adopted video delivery protocol for async video message playback on iOS, macOS, and web; uses MPEG-TS or fMP4 segments over HTTP with M3U8 manifests. URL: https://datatracker.ietf.org/doc/html/rfc8216

- **ISO/IEC 23009-1 — MPEG-DASH: Dynamic Adaptive Streaming over HTTP** — ISO standard for adaptive bitrate video streaming; provides codec-independent ABR streaming via MPD manifests and fragmented MP4 segments; used as a universal alternative to HLS especially on Android and desktop. URL: https://www.iso.org/standard/83314.html

- **CMAF — Common Media Application Format (ISO/IEC 23000-19)** — ISO standard unifying fMP4 packaging for both HLS and DASH; eliminates duplicate packaging workflows; enables a single video encode to be delivered via either protocol; increasingly adopted by video infrastructure platforms. URL: https://www.iso.org/standard/71975.html

- **RFC 6455 — WebSocket Protocol** — Used for real-time status updates during video upload, processing progress notifications, and viewer presence signalling in async video platforms. URL: https://datatracker.ietf.org/doc/html/rfc6455

- **RFC 6749 — OAuth 2.0** — Authorization framework used by Loom, Vidyard, and video infrastructure APIs (Mux, Cloudflare Stream) for third-party app authorization and CRM/team tool integrations. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — Used for signed embed tokens providing time-limited access to specific video recordings without requiring viewer authentication. URL: https://datatracker.ietf.org/doc/html/rfc7519

### Data Model & API Specifications

- **OpenAPI 3.1** — Used by Mux, Vidyard, and Cloudflare Stream to describe their REST management APIs; enables SDK code generation and automated integration testing. URL: https://spec.openapis.org/oas/latest.html

- **oEmbed** — Open standard for allowing URLs to be embedded in web pages; Loom implements the oEmbed standard for rich video previews in Notion, Slack, Confluence, and other platforms that consume oEmbed endpoints. URL: https://oembed.com/

- **MP4 / ISO Base Media File Format (ISO 14496-12)** — The universal container format for async video recordings; ISO standard for MPEG-4 media; used for upload, download, and archival of recorded videos across all async video platforms. URL: https://www.iso.org/standard/83102.html

- **WebVTT — Web Video Text Tracks** — W3C standard for timed text tracks (subtitles, captions, metadata) in HTML5 video; used by async video platforms for auto-generated captions and transcript overlays; required for WCAG accessibility compliance. URL: https://www.w3.org/TR/webvtt1/

- **SRT — SubRip Text Format** — De facto standard subtitle/transcript format; widely supported for export and import of transcript captions from async video platforms; companion to WebVTT for broader tool compatibility.

- **MoQ — Media over QUIC (IETF Draft, RFC expected 2026)** — Emerging IETF standard for real-time media delivery over QUIC transport; Cloudflare launched the first MoQ relay network (330+ cities) in 2025; OpenMOQ consortium formed; MOQT RFC expected to be finalised 2026; will become relevant to next-generation video upload and delivery pipelines. URL: https://datatracker.ietf.org/wg/moq/about/

### Security & Authentication Standards

- **GDPR Article 9 — Biometric Data** — Async video messages containing recordings of individuals' faces and voices may be classified as biometric data under GDPR Article 9 when used for speaker identification; requires explicit consent for AI face/voice recognition features in async video platforms. URL: https://gdpr-info.eu/art-9-gdpr/

- **GDPR Article 17 — Right to Erasure** — Users can request deletion of video messages containing their personal data; async video platforms must support video deletion with propagation to CDN edge caches and viewer analytics. URL: https://gdpr-info.eu/art-17-gdpr/

- **GDPR Article 32 — Security of Processing** — Video recordings must be encrypted at rest and in transit; access controls must prevent unauthorised viewing of private recordings; audit logs required for enterprise deployments. URL: https://gdpr-info.eu/art-32-gdpr/

- **WCAG 2.2 — Web Content Accessibility Guidelines** — Requires auto-generated captions (Success Criterion 1.2.2 Captions Prerecorded) and audio descriptions for video content in async video platforms; drives adoption of automated transcription and caption features. URL: https://www.w3.org/TR/WCAG22/

- **ADA / Section 508 (US)** — US federal accessibility requirements requiring video captions for content shared with federal agencies or in public-facing business contexts; drives caption requirements in async video messaging used for customer communications. URL: https://www.access-board.gov/ict/

- **SOC 2 Type II** — Required enterprise compliance certification for SaaS async video platforms handling confidential business communications; Loom (Atlassian), Vidyard, and Veed.io maintain SOC 2 Type II reports. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **OWASP API Security Top 10 (2023)** — Governs REST API security for async video management APIs; API1 (Broken Object Level Authorization) is critical for ensuring viewers can only access recordings they are authorised to watch. URL: https://owasp.org/API-Security/

### MCP Server Specifications

Async video messaging is an emerging integration target for AI-native workflows:

- **Loom MCP Integration** — Loom (now part of Atlassian) integrates with the Atlassian ecosystem; community patterns exist for referencing Loom video recordings in AI workflows via Atlassian's MCP infrastructure; no dedicated standalone Loom MCP server identified as of May 2026.

- **AI Transcription + MCP Pattern** — Emerging architecture: async video platforms generate transcripts (via Whisper or platform-native ASR) which are then exposed via MCP server to enable AI agents to query and summarise meeting recordings, product demos, and training videos.

---

## Similar Products — Developer Documentation & APIs

### Loom (Atlassian)

- **Description:** Leading async video messaging platform for workplace communication; acquired by Atlassian in 2023; SDK for embedding recording in third-party apps; REST API for video management; deep integration with Atlassian suite (Jira, Confluence, Trello).
- **API Documentation:** https://dev.loom.com/
- **Record SDK Reference:** https://dev.loom.com/docs/record-sdk/details/api
- **SDK Getting Started:** https://dev.loom.com/docs/record-sdk/getting-started
- **SDKs/Libraries:** @loomhq/record-sdk (npm, JavaScript/TypeScript); oEmbed API
- **Developer Guide:** https://dev.loom.com/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, oEmbed, HLS/DASH playback, WebVTT captions
- **Authentication:** API key (developer portal); OAuth 2.0 for user-level integrations

### Vidyard

- **Description:** Sales-focused async video messaging platform; strong CRM integrations (Salesforce, HubSpot); Player API for embedded video + analytics; Upload API for programmatic video ingestion; Analytics API for viewer engagement data.
- **API Documentation:** https://developer.vidyard.com/
- **Player API:** https://knowledge.vidyard.com/hc/en-us/articles/360019034753-Using-the-Vidyard-Player-API
- **SDKs/Libraries:** REST API (JSON); Player JavaScript API; webhook events
- **Developer Guide:** https://www.vidyard.com/developers/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, HLS/DASH playback, WebVTT captions
- **Authentication:** API token (Vidyard Dashboard → Integrations tab)

### Veed.io

- **Description:** Browser-based video editing and async video messaging platform; strong in video creation, subtitling, and translation; REST API for video upload and processing; used for short-form async messages with rich editing.
- **API Documentation:** https://www.veed.io/developers (requires account)
- **SDKs/Libraries:** REST API; Webhooks; Zapier integration
- **Developer Guide:** Veed developer portal (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, MP4/WebM
- **Authentication:** API key; OAuth 2.0 for integrations

### Mux (Video Infrastructure API)

- **Description:** Developer-focused video API platform for building custom async video applications; REST API for video upload, encoding, delivery, and analytics; Mux Data for real-time video quality metrics; open-source React player components.
- **API Documentation:** https://docs.mux.com/api-reference
- **SDKs/Libraries:** mux-node (Node.js); mux-python; mux-go; mux-ruby; @mux/mux-player-react; @mux/mux-uploader-react
- **Developer Guide:** https://docs.mux.com/
- **Standards:** REST/JSON (OpenAPI 3.1), OAuth 2.0 (token-based), HLS (fMP4/CMAF), MPEG-DASH, WebVTT captions, MP4
- **Authentication:** API token (Basic Auth with Token ID + Secret); JWT signed playback tokens for private/signed URLs

### Cloudflare Stream

- **Description:** Serverless video storage, encoding, and delivery platform integrated with Cloudflare's global CDN (330+ cities); per-minute pricing; REST API and Player API; stream-react for React embedding; supports live stream recording.
- **API Documentation:** https://developers.cloudflare.com/stream/
- **REST API:** https://developers.cloudflare.com/api/resources/stream/
- **Player API:** https://developers.cloudflare.com/stream/viewing-videos/using-the-stream-player/using-the-player-api/
- **SDKs/Libraries:** cloudflare-stream (npm); stream-react (GitHub: cloudflare/stream-react); Cloudflare Workers integration
- **Developer Guide:** https://developers.cloudflare.com/stream/
- **Standards:** REST/JSON, OpenAPI 3.1, OAuth 2.0, HLS (CMAF), MPEG-DASH, WebVTT, MP4 download
- **Authentication:** Cloudflare API token; signed URLs (JWT) for private video access

### Bunny Stream (bunny.net)

- **Description:** Cost-effective video hosting and streaming CDN with HLS delivery, per-title encoding, and a REST API; increasingly popular for cost-sensitive async video embedding use cases; integrated with bunny.net's edge CDN network.
- **API Documentation:** https://docs.bunny.net/reference/get_-libraryid-videos
- **SDKs/Libraries:** REST API (JSON); bunnynet-php; bunnynet-python (community); JavaScript embed player
- **Developer Guide:** https://docs.bunny.net/
- **Standards:** REST/JSON, OpenAPI, API token auth, HLS, MP4
- **Authentication:** AccessKey header (API key per storage zone)

### OBS Studio (Open Source Screen Recording)

- **Description:** Open-source (GPL v2) screen recording and live streaming software; the de facto standard for high-quality screen + webcam recording used by async video creators; supports WebSocket control API for programmatic recording triggers.
- **API Documentation:** https://obsproject.com/wiki/Remote-Control-Guide
- **WebSocket API:** obs-websocket plugin (WebSocket remote control protocol, obsproject/obs-websocket)
- **SDKs/Libraries:** obs-websocket-js (JavaScript); obs-websocket-py (Python); OBS Go
- **Developer Guide:** https://obsproject.com/wiki/
- **Standards:** WebSocket (obs-websocket protocol), JSON-RPC style, GPL v2 licence
- **Authentication:** WebSocket password authentication; optional TLS

---

## Notes

- **MediaRecorder API as the standard browser recording primitive**: W3C's MediaStream Recording API (`MediaRecorder`) is now universally supported across Chrome, Firefox, Edge, and Safari; it eliminates the need for browser extensions for screen + webcam recording in async video platforms built with web technology.

- **CMAF unifying HLS and DASH (2025-2026)**: The Common Media Application Format (ISO 23000-19) has enabled the industry to converge on a single fMP4 packaging workflow for both HLS and DASH delivery; new video infrastructure deployments should use CMAF as the packaging standard.

- **MoQ as the next-generation streaming transport (2026)**: IETF's Media over QUIC (MoQ) protocol — with Cloudflare's relay network already deployed — is set to replace older streaming transports for real-time media; async video platforms should monitor the MOQT RFC finalisation expected in 2026.

- **WebVTT captions as accessibility and AI baseline**: WCAG 2.2 Success Criterion 1.2.2 requires captions for prerecorded video; all major async video platforms now offer auto-generated WebVTT captions via ASR (Whisper or proprietary); captions also serve as the transcript layer for AI summarisation and search indexing.

- **Biometric data risk in AI async video features**: GDPR Article 9 applies to face and voice recognition features in async video platforms; any AI-powered speaker identification, face detection, or emotion recognition feature requires explicit opt-in consent and a Data Processing Agreement (DPA).

- **Open-source landscape**: OBS Studio (GPL v2) is the leading open-source screen/webcam recording tool; there is no dominant open-source async video messaging platform equivalent to Loom; self-hosted alternatives can be built using Mux/Cloudflare Stream APIs + MediaRecorder + a WebVTT caption pipeline (Whisper, MIT).
