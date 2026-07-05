# Cloudinary: Uploading Assets

**When this file applies:** Load this guidance whenever the task involves getting files into Cloudinary — uploading images, videos, or raw files via the Cloudinary asset-management MCP tools (Upload API / Admin API), configuring upload presets, or writing application code that uploads to Cloudinary. It covers choosing identifiers, folders, presets, upload-time metadata, transformations, and large-file handling.

## MCP tools vs. generating SDK code

- **Use the MCP upload tools** for one-off or operator-driven tasks: seeding a media library, migrating a batch of files, uploading a test asset, fixing a specific asset. The user asks *you* to upload; you call the tool.
- **Generate application code** when the user is building an app that uploads at runtime (user uploads, server pipelines). Do not route their app's traffic through MCP. Write server-side SDK code (signed) or client-side unsigned uploads with an unsigned preset.

Server-side Node.js example (credentials come from the `CLOUDINARY_URL` env var, format `cloudinary://<api_key>:<api_secret>@<cloud_name>` — never hardcode secrets):

```js
import { v2 as cloudinary } from "cloudinary";
// CLOUDINARY_URL in env configures the SDK automatically

const result = await cloudinary.uploader.upload("local-or-remote-file.jpg", {
  asset_folder: "products",
  use_filename: true,
  unique_filename: false,
  overwrite: false,
  resource_type: "auto",
  tags: ["catalog", "summer"],
  context: { alt: "Red sneaker, side view" },
});
console.log(result.public_id, result.secure_url);
```

For Python: `cloudinary.uploader.upload(file, **options)` with the same option names.

## public_id: explicit vs. auto-generated

- If you omit `public_id`, Cloudinary assigns a random ID (e.g. `8jsb1xofxdqamu2rzwt9q`). Fine for user-generated content; bad for assets referenced by predictable URLs.
- Set an explicit `public_id` when the app or user needs stable, readable delivery URLs. Rules: max 255 chars, no leading/trailing space or `/`, no `? & # \ % < >`. Omit the file extension for image/video; **include** it for `raw`.
- `use_filename: true` derives the public_id from the uploaded filename. Combined with:
  - `unique_filename: true` (default) → filename + random suffix (collision-safe).
  - `unique_filename: false` → exact normalized filename (predictable, but can collide).
- Decision rule: explicit `public_id` for curated/app assets; `use_filename: true, unique_filename: false` for mirroring an existing file tree; defaults for untrusted user uploads.

## Overwrite behavior

- Default is effectively "don't clobber": uploading with an existing public_id only replaces the asset if `overwrite: true`.
- Overwriting may clear existing tags, context, and structured metadata — re-send them in the same call if they must survive.
- After overwriting, add `invalidate: true` to purge cached CDN copies (propagation takes seconds to minutes).

## Folders

Product environments run in one of two folder modes — check before choosing a strategy:

- **Dynamic folders (newer accounts):** set `asset_folder` to place the asset in the Media Library. Slashes in `public_id` do *not* create folders; display location and public_id are independent.
- **Fixed folders (legacy):** slashes in `public_id` (or the `folder` param) define both the URL path and the Media Library folder.

If unsure of the mode, query the environment config via the Admin/config tools or ask the user rather than guessing.

## Upload presets

- A preset is a named, server-stored bundle of upload options; pass `upload_preset: "<name>"` in the call.
- **Signed presets** (server-side): all parameters allowed. Precedence: parameters in the request override the preset, except `eager` and incoming `transformation`, which are **merged**.
- **Unsigned presets** (browser/client-side, no signature): only a small whitelist of request parameters is accepted; everything else must live in the preset. Preset values win, except `context`/`metadata` (merged) and `public_id` (request wins unless `disallow_public_id` is set).
- When generating client-upload code, recommend hardening the unsigned preset: `allowed_formats`, `max_file_size`, `disallow_public_id`, and an incoming transformation to cap dimensions.
- Default presets (e.g. `ml_default`) can auto-apply per resource type from console settings — changing them affects production workflows; warn before editing.

## Tags, context, metadata at upload time

Set these in the upload call rather than with follow-up API calls — it's atomic and cheaper:

- `tags: ["a", "b"]` — flat labels for grouping, search, and bulk operations.
- `context: { alt: "...", caption: "..." }` — free-form key-value pairs (contextual metadata).
- `metadata: { "field-id": "value" }` — structured metadata; values must conform to the fields defined in the environment, otherwise the upload errors.
- Auto-tagging: `categorization: "google_tagging", auto_tagging: 0.6` applies detected tags above the confidence threshold.

## Eager vs. on-the-fly transformations

- Default to **on-the-fly**: store the original untouched and put transformations in the delivery URL. This is Cloudinary's core model; don't pre-generate derivatives without a reason.
- Use `eager: [{ width: 400, height: 300, crop: "pad" }, ...]` when specific derived versions must exist immediately (strict first-request latency, or on-the-fly is restricted). For video, prefer `eager_async: true` with `eager_notification_url` — video derivation is slow.
- An **incoming transformation** (top-level `transformation`/`width`/`crop` params) permanently alters the stored original (e.g. cap resolution, strip data). Use only when the original should not be kept as-is.

## Large files and video (chunked upload)

- Files **over 100 MB must be uploaded in chunks**. SDKs expose `upload_large` (e.g. `cloudinary.uploader.upload_large(file, { chunk_size: 20_000_000 })`); default chunk size 20 MB, minimum 5 MB.
- Videos commonly exceed 100 MB — reach for `upload_large` for any sizable video, and set `resource_type: "video"` explicitly (SDK default is `image`).
- Intermediate chunk responses return `done: false`; the final response has `done: true`. For files over ~20 GB, also set `async: true`.
- If an MCP upload tool hits size limits on a huge file, generate a short SDK/CLI script for the user instead of retrying through MCP.

## Other options that matter

- `resource_type`: SDK default is `image` (also covers PDFs). Use `"auto"` when file types are mixed or unknown; use `"raw"` for non-media files (include the extension in the public_id).
- `notification_url`: webhook that receives the upload result — pair with `async: true` or eager-async processing so the app isn't blocked on long operations.
- `type`: `upload` (public, default) vs `private`/`authenticated` for access-restricted originals.
- Remote sources: the `file` argument accepts local paths, remote HTTPS URLs, and data URIs — for assets already hosted elsewhere, pass the URL instead of downloading first.

## Quick checklist before uploading

1. One-off task → MCP tool; app runtime → generate SDK code with `CLOUDINARY_URL`.
2. Decide public_id strategy (explicit / use_filename / random) and overwrite intent.
3. Fixed vs dynamic folder mode → `public_id` path vs `asset_folder`.
4. Attach tags/context/metadata in the same call.
5. Video or >100 MB → chunked upload, explicit `resource_type`, consider eager async + notification_url.
