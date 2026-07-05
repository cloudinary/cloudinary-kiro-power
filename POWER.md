---
name: "cloudinary"
displayName: "Cloudinary"
description: "Upload, manage, transform, optimize, and analyze images and videos with Cloudinary. Connects to Cloudinary's remote MCP servers via OAuth — no API keys to paste — and can even provision a new Cloudinary account for you."
keywords: ["cloudinary", "image", "video", "media", "upload", "transformation", "optimization", "cdn", "dam", "thumbnail", "resize", "crop", "responsive images", "asset management"]
author: "Cloudinary"
---

# Cloudinary Power

This power connects Kiro to Cloudinary — the media platform for uploading, storing, transforming, optimizing, and delivering images and videos. It wires up Cloudinary's four remote MCP servers (asset management, environment configuration, structured metadata, and media analysis) and provides steering for the most common media workflows.

Authentication is OAuth: the user signs in to Cloudinary in a browser and picks a product environment. No API keys or secrets are ever stored in this power's configuration.

# Onboarding

## Step 1 — Determine whether the user has a Cloudinary account

Ask the user if they already have a Cloudinary account (or check: if any `cloudinary-*` MCP server is already connected and authorized, they do — skip to Step 3).

- **Has an account (or is willing to sign up in the browser)** → go to Step 2.
- **No account, and wants you to set one up for them** → go to Step 2b (agentic provisioning).

## Step 2 — Connect via OAuth (existing account)

The four Cloudinary MCP servers in `mcp.json` are OAuth 2.1 protected resources supporting dynamic client registration. When Kiro first connects, the user is sent to the browser to sign in to Cloudinary, consent, and **select the product environment (cloud)** the tokens will act on.

1. Trigger a connection to `cloudinary-asset-management` first — it covers upload plus asset administration and is enough for most workflows.
2. Have the user complete the browser sign-in and pick their product environment.
3. Verify the connection with a lightweight call (e.g., list a few assets or fetch usage).
4. Connect the other servers (`cloudinary-environment-config`, `cloudinary-structured-metadata`, `cloudinary-analysis`) lazily — only when a workflow needs them, since each requires its own consent.

Notes:
- Tokens are scoped and short-lived, refreshed automatically via `offline_access`. They authenticate **MCP calls only** — they cannot be used against Cloudinary's REST APIs directly.
- If the user works with multiple product environments, the environment was chosen at consent time; to switch environments, re-authorize the server.

## Step 2b — Provision a new account (no Cloudinary account yet)

If the user has no Cloudinary account, you can create one for them from just their email address. Load `steering/account-provisioning.md` and follow it. In short:

1. `POST https://api.cloudinary.com/v1_1/provisioning/agents/accounts` with the user's email and your agent metadata (no authentication required).
2. Tell the user to open the verification email from Cloudinary, click the link, and **set a password** (the link expires in ~24 hours). The account and its credentials stay **inert** until they do.
3. Once claimed, **return to Step 2** and connect via OAuth — the user signs in with the password they just set. Do not use the root `api_key`/`api_secret` from the provisioning response unless the user explicitly needs direct REST access; OAuth delegation is the preferred path.
4. If provisioning returns a 400 saying the email has already been taken, the user already has an account — go to Step 2 instead.

## Step 3 — Confirm setup

Run one simple end-to-end check: upload a small test asset or list existing assets via `cloudinary-asset-management`, and show the user the result. Mention that they can ask for uploads, transformations, optimization advice, asset search, metadata management, and media analysis.

# When to Load Steering Files

Load the steering file that matches the user's task. Load more than one when the task spans areas (e.g., "upload and tag product images" → upload + asset management).

- Uploading images or videos (from disk, URLs, or in app code), upload presets, public IDs, eager transformations → `steering/upload.md`
- Transforming, resizing, cropping, optimizing media; building delivery URLs; responsive images; overlays; generative AI edits; video thumbnails/streaming → `steering/transformations.md`
- Searching, organizing, tagging assets; folders; renaming/deleting; structured metadata fields; upload presets, named transformations, and webhooks configuration → `steering/asset-management.md`
- AI analysis of media: auto-tagging, captioning, moderation, content analysis tasks → `steering/analysis.md`
- Creating a Cloudinary account for a user who doesn't have one; claim/verification flow; provisioning errors → `steering/account-provisioning.md`

# General Guidance

- **Prefer MCP tools for operating on the user's Cloudinary account** (uploading assets, searching, configuring); **generate SDK code** when the user is building application features (in-app uploads, URL generation in their codebase).
- Always recommend `f_auto` and `q_auto` in delivery URLs unless the user has a specific reason not to.
- Never ask the user for their `api_key`/`api_secret` — OAuth handles authentication. If application code needs credentials (e.g., `CLOUDINARY_URL` for an SDK), direct the user to copy them from their Cloudinary Console into their own environment/secrets manager; never inline secrets in code or commit them.
- Asset operations like delete and rename affect delivered URLs; warn the user and consider `invalidate: true` for CDN cache invalidation.
