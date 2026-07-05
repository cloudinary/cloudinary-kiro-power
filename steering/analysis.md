# Cloudinary Media Analysis

**When this file applies:** Load this guidance when the user wants to understand, tag, caption, moderate, classify, or extract information from images and videos stored in (or reachable by) Cloudinary — e.g. "tag these product images", "generate captions", "check assets for inappropriate content", "detect watermarks/logos/text", or "make our media searchable". It covers the Cloudinary **Analysis MCP server** (`https://analysis.mcp.cloudinary.com/sse`, scopes: `media_analysis`, `query_analysis_tasks`), which wraps Cloudinary's Analyze API, and how it relates to upload-time add-ons handled by the Asset Management server.

## What the Analysis server offers

The Analyze API runs AI models against an asset identified by `uri` (any public URL) or `asset_id` (an asset already in the Cloudinary account). Main analysis types:

- **AI Vision (prompt-driven, most flexible):**
  - `ai_vision_general` — ask open-ended questions about an image ("what brand is this?", "describe the scene").
  - `ai_vision_tagging` — you supply candidate tag names with natural-language definitions; the model decides which apply.
  - `ai_vision_moderation` — you supply yes/no policy questions ("does this contain alcohol?"); returns per-question verdicts.
- **Cloudinary content-analysis models:** `captioning` (descriptive alt-text style caption), `coco` / `lvis` / `unidet` (object detection: 80 common / thousands of / unified categories), `cld_fashion` (garment attributes), `cld_text` (text presence and location), `human_anatomy`, `shop_classifier` (product shot vs. natural photo), `watermark_detection`, `image_quality` (IQA score 0–1 with low/medium/high band).
- **Third-party models:** `google_tagging` (general labels), `google_logo_detection` (brand logos).

Decision guide: prefer `ai_vision_*` when the user's need is expressed in natural language or is domain-specific; prefer a dedicated model when it matches exactly (captions → `captioning`, quality gating → `image_quality`, generic labels → `google_tagging`/`lvis`).

## Async task pattern

Analysis can run synchronously (result in the response) or asynchronously:

1. Start the analysis with `async: true` (or when the tool defaults to async). You get back a `task_id` with status `pending`.
2. Poll with the **`query_analysis_tasks`** capability (GET task by `task_id`) until status is `completed` or `failed`. Statuses: `pending` → `processing` → `completed` / `failed`.
3. Alternatively pass a `notification_url` to receive a webhook on completion instead of polling.

Rules:
- Do not busy-poll; check status between other work or at modest intervals, and surface the `task_id` to the user for long jobs.
- For batches, start all tasks first, then poll — don't serialize start/wait/start/wait.
- On `failed`, report the error; don't silently retry more than once.

## Analysis server vs. upload-time add-ons

There are two ways to get the same classes of insight — pick deliberately:

- **Use the Analysis server (Analyze API)** for assets that already exist, for ad-hoc/exploratory analysis, for external URLs not yet uploaded, for prompt-driven AI Vision tasks, and for backfilling metadata across an existing library.
- **Use upload-time parameters** (via the Asset Management / Upload API, not this server) when analysis should happen automatically as part of ingestion:
  - `detection` + `auto_tagging: <0–1 confidence>` — AI Content Analysis add-on models (`cld-fashion`, `coco`, `lvis`, `unidet`, `cld-text`, `captioning`, `iqa`, `watermark-detection`, ...); auto-applies tags above the confidence threshold. Also available retroactively via the Admin API `update` method.
  - `moderation` — `aws_rek` (image), `aws_rek_video`, `google_video_moderation`, `webpurify`, `perception_point` (malware), `duplicate:<threshold>`, `manual`. Chain with pipes (e.g. `"aws_rek|duplicate:0"`); `manual` must be last. Moderation lifecycle: `queued` → `pending` → `approved` / `rejected` (or `aborted` after an earlier rejection). Override with Admin API `update` + `moderation_status`.

Rule of thumb: continuous pipeline → upload-time add-on parameters; one-off, retroactive, or question-shaped analysis → this Analysis server.

## Feeding results back into searchability

Analysis results are returned to you — they are not automatically stored on the asset (except upload-time `auto_tagging`/`detection`, which persist tags and detection data). To make results durable and searchable:

- Write tags from tagging/object-detection output onto the asset (Asset Management server: add tags / update asset).
- Store captions in `context` metadata (commonly `alt` or `caption`) so they serve as alt text and search fodder.
- Store quality scores, moderation verdicts, or classifications in structured metadata fields when the account defines them.
- After writing, the Search API can find assets by those tags/metadata — this is the standard "analyze → enrich → search" loop.

## Add-ons, credits, and limits

- Most analysis types require an **active add-on subscription** for the corresponding feature (AI Content Analysis, Google Auto Tagging, Amazon Rekognition, WebPurify, Perception Point, Duplicate Image Detection, etc.). A request for an unsubscribed feature fails — tell the user which add-on to enable in the Cloudinary Console rather than retrying.
- Analyze API responses include a `limits` object (`used_by_request`, `remaining`, `limit`, `reset_time`). Check it before large batches; a 429 means quota exhaustion — report remaining quota and reset time instead of hammering.
- Add-on operations consume plan credits / tiered add-on units. For bulk jobs over many assets, warn the user about consumption and confirm before proceeding.
- Delivery URLs relying on AI Content Analysis results (e.g. object-aware cropping) must generally be signed or eagerly generated.

## Quick reference

| Goal | Do this |
|---|---|
| Describe / caption an image | `captioning` analysis; store result in `context.alt` |
| Custom domain tags | `ai_vision_tagging` with tag definitions; write tags back |
| Policy check on existing asset | `ai_vision_moderation` with yes/no questions |
| Moderate all future uploads | Upload-time `moderation` parameter (add-on) — not this server |
| Auto-tag at upload | Upload-time `detection` + `auto_tagging` threshold |
| Check a running job | `query_analysis_tasks` with the `task_id` |
