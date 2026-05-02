# Async Video Messaging

> Candidate #163 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| Loom | Screen and webcam recording with link-based sharing, AI summaries, and team workspaces | SaaS | Free; Business $15/user/mo; Business + AI $20/user/mo; Enterprise custom | Strengths: dominant mindshare, Atlassian integration, ease of use; Weaknesses: storage limits, no deep CRM tie-in |
| Vidyard | Sales-focused async video with CRM integrations, viewer analytics, and hosted video library | SaaS | Free tier; Plus $59/user/mo; Teams $99/user/mo | Strengths: CRM-native analytics, pipeline attribution; Weaknesses: expensive for general use |
| Tella | Polished screen recording with brand customisation, chapters, and embeddable video pages | SaaS | Free tier; Pro ~$19/mo | Strengths: output quality, customisable player; Weaknesses: limited collaboration features |
| Sendspark | Personalised video outreach with dynamic variable insertion and email integration | SaaS | Free tier; paid from ~$39/mo | Strengths: personalisation at scale, call-to-action overlays; Weaknesses: niche sales focus |
| Descript | Text-based video/audio editing, transcription, screen recording, and publishing | SaaS | Free; Creator $19/mo; Pro $35/mo; Enterprise $50/mo | Strengths: transcript-driven editing, AI filler word removal; Weaknesses: overkill for simple messaging |
| Riverside | HD remote recording with local capture, AI editing, transcription, and distribution | SaaS | Free; Standard $19/mo; Pro $29/mo; Business custom | Strengths: studio-quality multi-track capture; Weaknesses: heavier than pure messaging tools |
| Scribe | Automated step-by-step guide generation from screen actions | SaaS | Free; Pro $29/user/mo; Enterprise custom | Strengths: documentation automation; Weaknesses: static output, limited video depth |
| Zight (formerly CloudApp) | Screen recording, screenshot annotation, and GIF creation with link sharing | SaaS | Free; Pro $9.95/user/mo; Team $12.99/user/mo | Strengths: lightweight, fast sharing; Weaknesses: limited analytics |

## Relevant Industry Standards or Protocols

- **WebRTC** — browser-native real-time media capture used by most browser-based recorders for webcam and microphone input
- **HLS / DASH (Adaptive Bitrate Streaming)** — standard delivery protocols for progressive video playback over CDN
- **WebVTT / SRT** — caption and subtitle formats used for AI-generated transcripts overlaid on video
- **SCORM / xAPI** — learning management standards relevant when async video is used for training delivery

## Available Research Materials

1. Vidyard (2026). *B2B Video Messaging Platforms: 2026 Buyer's Guide*. Vidyard Blog. https://www.vidyard.com/blog/b2b-video-messaging-platforms-2026-buyers-guide/
2. Tella (2026). *How To Choose The Best Loom Alternative in 2026 — Top 20 Reviewed*. Tella Blog. https://www.tella.com/blog/loom-alternatives
3. ClickUp (2026). *13 Best Loom Alternatives and Competitors for 2026*. ClickUp Blog. https://clickup.com/blog/loom-alternatives/
4. Glitter AI (2026). *Best Loom Alternatives for 2026 — Top 8 Tools Compared*. Glitter AI Blog. https://www.glitter.io/blog/process-documentation/best-loom-alternatives
5. Costbench (2026). *Loom Pricing 2026: 4 Plans from Free–$20/user/month*. https://costbench.com/software/communication/loom/
6. Guidde (2026). *Vidyard vs. Loom: Which Video Messaging Tool Wins in 2026?* https://www.guidde.com/tool-comparison/vidyard-vs-loom-comparison-2026

## Market Research

**Market Size:** The async video messaging segment sits within the broader enterprise video market. Specific async-messaging-only market figures are not widely published; analysts bundle it with the enterprise video collaboration market, which is valued in the multi-billion-dollar range with strong YoY growth driven by hybrid work.

**Funding:** Loom was acquired by Atlassian in 2023 for approximately $975 million. Vidyard has raised over $70 million in venture funding. Smaller players such as Tella and Sendspark remain seed- to Series A-stage.

**Pricing Landscape:** Free tiers are near-universal as a user-acquisition mechanism. Paid plans typically cluster in the $15–$30/user/month range for teams; enterprise deals are negotiated separately. Sales-oriented tools (Vidyard) command significant premiums due to CRM attribution value.

**Key Buyer Personas:** Remote-first engineering and product teams (internal async updates), sales development representatives (personalised outreach), training and L&D teams (knowledge transfer), and customer success teams (onboarding walkthroughs).

**Notable Trends:** AI features — filler-word removal, auto-chapters, transcript-based editing, and automated show-note generation — are shifting from premium add-ons to baseline expectations. The market is bifurcating between lightweight internal-messaging tools and video-intelligence platforms with deep CRM and pipeline analytics.

## AI-Native Opportunity

- Automatic removal of filler words, silences, and off-topic tangents during recording or immediately post-capture, reducing editing to near-zero effort
- Real-time AI coaching overlay during recording that suggests pacing, tone, and key points to cover based on the video's stated goal
- Semantic video search across an entire workspace library, enabling teams to find moments within recordings rather than full files
- Personalisation engine that clones a speaker's voice and face for scaled outreach videos, inserting recipient name and context without re-recording
- Intelligent routing that analyses viewer engagement heatmaps and automatically follows up via email or Slack when a viewer drops off at a key moment
