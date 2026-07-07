---
name: "cloudinary"
displayName: "Cloudinary"
description: "Upload, transform, optimize, and manage images and videos with Cloudinary. Search and analyze your media library with AI, and deliver optimized assets worldwide."
keywords: ["cloudinary", "image", "video", "upload", "transformation", "optimization", "asset management"]
author: "Cloudinary"
---

# Cloudinary Power

## Overview

Connect Kiro to Cloudinary — the media platform for uploading, storing, transforming, optimizing, and delivering images and videos. This power wires up four of Cloudinary's remote MCP servers and provides steering for the most common media workflows.

**Key capabilities:**

- **Upload and manage assets**: upload images/videos/raw files, search with Lucene-like expressions, organize with folders, tags, and metadata
- **Transform and optimize**: build delivery URLs with resizing, cropping, overlays, `f_auto`/`q_auto` optimization, and generative AI edits
- **Configure the environment**: upload presets, named transformations, streaming profiles, triggers
- **AI media analysis**: auto-tagging, captioning, moderation, object detection, quality scoring
- **OAuth authentication**: no API keys pasted into config — the user signs in to Cloudinary in a browser and selects a product environment
- **Agentic account provisioning**: users without a Cloudinary account can have one created from their email, claimed via email verification

## Onboarding

### Step 1: Check connection state

Check which Cloudinary tools are available:

- **Connected**: tools like `search-assets` and `upload-asset` are present — skip to Step 3.
- **Not connected**: no `cloudinary-*` tools are available — the server needs authentication (Step 2). If the user has no Cloudinary account and wants you to create one, go to Step 2b first.

### Step 2: Authenticate (existing account)

When Kiro connects to the `cloudinary-asset-management` server it opens the browser OAuth flow automatically — ask the user to complete it. (If it doesn't trigger, have the user connect/re-authenticate the server from Kiro's MCP panel, or run `/mcp auth`.) In the browser the user signs in to Cloudinary, consents, and **selects the product environment (cloud)** the tokens will act on.

Notes:

- Authentication is handled by the OAuth flow — you never paste or handle raw API keys. (Cloudinary's non-Analysis MCP servers also accept API-key authentication via `CLOUDINARY_URL`/header credentials, but this power standardizes on OAuth so no secrets enter the config.)
- Each of the four servers requires its own one-time consent. Authorize `cloudinary-asset-management` now; authorize the others when a workflow first needs them rather than all upfront.
- The product environment was chosen at consent time. If assets seem missing, the user may have selected a different environment — re-authorize to switch.

### Step 2b: Provision a new account (no Cloudinary account yet)

If the user has no Cloudinary account, you can create one from just their email address. Load `steering/account-provisioning.md` and follow it exactly — including its credential-redaction rules. In short: register via the agent provisioning endpoint, have the user claim the account through the verification email (they set a password), then return to Step 2 and authenticate via OAuth. Never surface or store the root credentials from the provisioning response unless the user explicitly chooses the headless fallback documented there.

### Step 3: Verify with a read-only call

Confirm the connection with a **read-only** call: `get-usage-details`, or `search-assets` with a small `max_results`. Never upload a test asset as a connectivity check — don't write to the user's media library uninvited.

Then tell the user what they can ask for: uploads, transformations and optimization, asset search and organization, metadata, environment configuration, and AI media analysis.

## Available Steering Files

- **upload** — Getting files into Cloudinary: public IDs, folders, presets, upload-time metadata, eager transformations, large files
- **transformations** — Delivery URLs: resizing/cropping, `f_auto`/`q_auto`, responsive images, overlays, video, generative AI edits
- **asset-management** — Search expressions, tags vs context vs structured metadata, folders, rename/delete, environment configuration
- **analysis** — AI analysis: tagging, captioning, moderation, async task pattern, feeding results back into searchability
- **account-provisioning** — Creating a Cloudinary account for a user who doesn't have one (agentic registration + claim ceremony)

## When to Load Steering Files

Load the steering file that matches the user's task. Load more than one when the task spans areas (e.g., "upload and tag product images" → upload + asset-management).

- Uploading images or videos (via MCP tools or in app code), upload presets, public IDs, eager transformations → `upload.md`
- Transforming, resizing, cropping, optimizing media; delivery URLs; responsive images; overlays; generative AI edits; video thumbnails/streaming → `transformations.md`
- Searching, organizing, tagging assets; folders; renaming/deleting; structured metadata; presets, named transformations, triggers, streaming profiles → `asset-management.md`
- AI analysis of media: auto-tagging, captioning, moderation, object detection, quality scoring → `analysis.md`
- Creating a Cloudinary account for a user without one; claim/verification flow; provisioning errors → `account-provisioning.md`

## Available MCP Servers

### cloudinary-asset-management

**Connection:** `https://asset-management.mcp.cloudinary.com/mcp` — OAuth (browser sign-in)
**Covers:** uploading and managing assets (Upload API + Admin API capabilities)

| Tool | Purpose |
|------|---------|
| `upload-asset` | Upload images, videos, or raw files with optional transformations |
| `search-assets` | Query assets using Lucene-like search expressions |
| `visual-search-assets` | Find visually similar images by URL, asset ID, or description |
| `get-asset-details` | Fetch full details of a single asset |
| `asset-update` | Modify an asset's metadata, tags, and attributes |
| `asset-rename` | Change an asset's public ID |
| `delete-asset` | Delete an asset by asset ID |
| `delete-derived-assets` | Remove derived (transformed) versions |
| `list-images` / `list-videos` / `list-files` | List assets by type with basic filtering |
| `list-tags` | List tags in use |
| `create-folder` / `move-folder` / `delete-folder` / `search-folders` | Folder management |
| `create-asset-relations` / `delete-asset-relations` | Link/unlink related assets |
| `generate-archive` | Create downloadable ZIP/TGZ archives of assets |
| `download-asset-backup` | Retrieve a backed-up asset version |
| `transform-asset` | Generate derived transformations (explicit/eager) |
| `get-tx-reference` | Load official transformation syntax reference (once per session) |
| `get-usage-details` | Account usage metrics — good read-only connectivity check |

### cloudinary-environment-config

**Connection:** `https://environment-config.mcp.cloudinary.com/mcp` — OAuth (browser sign-in)
**Covers:** product-environment configuration

Tools follow a create/list/update/delete pattern for **transformations** (named), **upload-presets**, **upload-mappings**, **triggers**, and **streaming-profiles** (e.g., `create-transformation`, `list-upload-presets`, `update-streaming-profile`, `delete-upload-mapping`). The read/detail tools are not uniform: `get-transformation-details`, `get-upload-preset-details`, and `get-streaming-profile` — while upload-mappings and triggers have no `get`. Triggers also add `test-trigger`.

### cloudinary-structured-metadata

**Connection:** `https://structured-metadata.mcp.cloudinary.com/mcp` — OAuth (browser sign-in)
**Covers:** structured metadata schema

Tools follow the same CRUD pattern for **metadata-fields** (`create-metadata-field`, `list-metadata-fields`, `get-metadata-field`, `update-metadata-field`, `delete-metadata-field`), **metadata-datasource-values** (update/delete), and **metadata-rules** (create/list/update/delete).

### cloudinary-analysis

**Connection:** `https://analysis.mcp.cloudinary.com/sse` — OAuth (browser sign-in)
**Covers:** AI media analysis

| Tool | Purpose |
|------|---------|
| `analyze-ai-vision-general` | Open-ended questions about an image |
| `analyze-ai-vision-tagging` | Tag images against a custom taxonomy |
| `analyze-ai-vision-moderation` | Yes/no policy-compliance questions |
| `analyze-captioning` | Generate descriptive captions |
| `analyze-coco` / `analyze-lvis` / `analyze-unidet` | Object detection (80 / thousands / unified categories) |
| `analyze-google-tagging` / `analyze-google-logo-detection` | Google label and logo detection |
| `analyze-cld-fashion` / `analyze-cld-text` / `analyze-human-anatomy` | Fashion attributes, text presence, anatomy |
| `analyze-shop-classifier` | Studio product shot vs natural photo |
| `analyze-image-quality` | Image quality score |
| `analyze-watermark-detection` | Detect watermarks |
| `tasks-get-status` | Check status of an async analysis task |

## MCP Configuration

```json
{
  "mcpServers": {
    "cloudinary-asset-management": {
      "url": "https://asset-management.mcp.cloudinary.com/mcp"
    },
    "cloudinary-environment-config": {
      "url": "https://environment-config.mcp.cloudinary.com/mcp"
    },
    "cloudinary-structured-metadata": {
      "url": "https://structured-metadata.mcp.cloudinary.com/mcp"
    },
    "cloudinary-analysis": {
      "url": "https://analysis.mcp.cloudinary.com/sse"
    }
  }
}
```

## Best Practices

- **Prefer MCP tools for operating on the user's Cloudinary account** (uploading, searching, configuring); **generate SDK code** when the user is building application features (in-app uploads, URL generation in their codebase).
- Always recommend `f_auto` and `q_auto` in delivery URLs unless the user has a specific reason not to.
- Never ask the user for their `api_key`/`api_secret` — OAuth handles authentication. If application code needs credentials (e.g., `CLOUDINARY_URL` for an SDK), direct the user to copy them from their Cloudinary Console into their own environment/secrets manager; never inline secrets in code or commit them.
- Asset operations like delete, rename, and overwrite affect delivered URLs and cached CDN copies; warn the user and use `invalidate: true` when the change must be visible immediately.
- For version-sensitive specifics (rate limits, add-on catalogs, new transformation parameters), verify against current docs via https://cloudinary.com/documentation/llms.txt rather than relying solely on steering content.

## Troubleshooting

### No Cloudinary tools available

**Cause:** the MCP server isn't connected or authorized.
**Solution:** have the user connect the server so Kiro opens the browser OAuth flow — from Kiro's MCP panel (connect / Re-authenticate) or via `/mcp auth`. The browser sign-in must complete, including product-environment selection.

### Authentication errors (401 / token expired)

**Cause:** the OAuth session behind the grant ended (e.g., the user signed out of Cloudinary).
**Solution:** re-authenticate the server; completing the OAuth sign-in again re-establishes the grant.

### Assets the user expects are missing

**Cause:** the token acts on the product environment selected at consent, which may not be the one the user is thinking of.
**Solution:** confirm which cloud they intended; re-authorize the server and select the right environment.

### Analysis tool fails with an add-on or subscription error

**Cause:** most analysis models require the corresponding add-on (AI Content Analysis, Google Auto Tagging, Amazon Rekognition, etc.).
**Solution:** tell the user which add-on to enable in the Cloudinary Console; don't retry until it's enabled.

### Need direct REST API access

**Cause:** the MCP servers cover agent-driven operations on the account, not your application's runtime traffic.
**Solution:** for application code, use a Cloudinary SDK with credentials the user copies from their Console. Don't route app runtime traffic through MCP.

### Provisioning returns 400 (account already exists)

**Cause:** a Cloudinary account already exists for that email.
**Solution:** skip provisioning; authenticate via OAuth (Onboarding Step 2).

## License and Support

**License:** MIT (SPDX-License-Identifier: `MIT`) — see the `LICENSE` file at the repository root.

This power integrates with [Cloudinary](https://cloudinary.com); these MCP servers are operated by Cloudinary, so the links below cover both the power and the servers.

- [Documentation](https://cloudinary.com/documentation)
- [Support](https://support.cloudinary.com)
- [Privacy Policy](https://cloudinary.com/privacy)
