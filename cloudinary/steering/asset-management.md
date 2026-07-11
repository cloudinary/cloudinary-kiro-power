# Cloudinary Asset Management & Admin Operations

**When this file applies:** Load this guidance whenever the user asks to find, organize, tag, move, rename, or delete Cloudinary assets, define or edit structured metadata fields, or configure the product environment (upload presets, named transformations, streaming profiles, webhooks). These workflows map to three Cloudinary MCP servers: **asset-management** (Admin API: assets, search, folders, tags, settings), **structured-metadata** (metadata field definitions and values), and **environment-config** (presets, transformations, streaming profiles, webhook notifications). Prefer the most specific server for the task; do not use asset-management tools to define metadata fields or presets when a dedicated server exists.

## Finding assets: always prefer Search over listing

Use the Search method with a filter expression instead of paginating `resources` listings whenever the user's request implies *criteria* (tags, folder, size, date, metadata). Listing is only appropriate for "show me everything" browsing. Search is one API call; listing-then-filtering wastes rate limit and tokens.

### Search expression syntax (Lucene-like)

- `field:value` — tokenized/contains match; `field=value` — exact match. `tags:shirt` matches partial token hits; `tags=shirt` requires the exact tag.
- Boolean operators must be ALL CAPS: `AND`, `OR`, `NOT` (also `&&`, `||`, `!`, and `+`/`-` prefixes). Adjacent terms default to OR. Group with parentheses.
- Ranges: `width:{200 TO 1024}` (also `[ ]` inclusive), relative dates like `2d`, `4w`; comparison ops `>`, `>=`, `<` work for numbers/dates.
- Wildcards: `*` in values (e.g., `public_id:products/*`). Quote values containing spaces or special characters; escape `"` and `*` with `\`.
- Useful fields: `public_id`, `folder`/`asset_folder`, `tags`, `context`, `metadata`, `resource_type`, `format`, `width`, `height`, `aspect_ratio`, `bytes`, `duration`, `created_at`, `uploaded_at`, `transparent`, `moderation_status`.
- Structured metadata: query by field external_id, e.g. `metadata.department=marketing` or presence check `metadata=sku`.

Example expressions:

```
tags=sale AND resource_type:image
folder:products/* AND format:png AND transparent:true
resource_type:video AND duration:{30 TO 120} AND bytes>10000000
uploaded_at:[4w TO 1w] AND tags=hero AND width>=1920
context.alt:"summer campaign" OR tags=summer-2026
metadata.approval_status=approved AND NOT tags=archived
```

Notes: results exclude assets pending moderation unless you add `moderation_status:pending`; use `max_results` (cap 500) and `next_cursor` for paging; request `sort_by` and only needed `with_field` values to keep responses small.

## Tags vs context vs structured metadata — choosing the right one

| Mechanism | Nature | Use when |
|---|---|---|
| **Tags** | Flat labels, many per asset | Grouping, bulk operations (delete/list by tag), quick categorical filtering. No values, no validation. |
| **Context** | Free-form key-value strings per asset | Per-asset descriptive text (`alt`, `caption`, ad-hoc attributes). No schema, no validation, not shared across assets. |
| **Structured metadata** | Account-level typed schema, values per asset | Governed fields: validated, typed, searchable, mandatory-able, with predefined option lists. Use for anything a team standardizes on (SKU, department, rights, approval status). |

Rule of thumb: if the user wants controlled vocabulary, validation, or reporting — structured metadata. If they want quick grouping for bulk ops — tags. If it's one-off descriptive text — context.

## Structured metadata fields (structured-metadata server)

- Field types: `string`, `integer`, `date`, `enum` (single-select), `set` (multi-select). Max 100 fields per environment.
- Defining a field requires `external_id` (stable identifier — used in search expressions), `label`, `type`; optional `mandatory` and `default_value`.
- `enum`/`set` fields require a `datasource` of up to 3,000 entries, each with its own `external_id` and `value`. Update the datasource to add/remove options; never delete an option that assets still reference without confirming intent.
- Validation is supported: numeric ranges (`greater_than`/`less_than`), regex patterns for strings, and combined `and` rules. Conditional metadata rules can enable fields, activate values, or set mandatory based on other fields' values.
- Set values on assets via the metadata update tools (or at upload via presets). Changes to metadata trigger webhook notifications if configured.

## Folders: fixed vs dynamic mode

Determine the environment's folder mode before moving assets — behavior differs fundamentally:

- **Fixed folder mode:** the folder is part of the public ID (`products/shoes/sneaker-01`). Moving an asset means **renaming** its public ID, which changes its delivery URL (breaking change for consumers).
- **Dynamic folder mode:** `asset_folder` is independent of the public ID. Moving an asset updates `asset_folder` only; delivery URLs are unaffected. Also supports a separate `display_name`.
- Folder ops: create folders explicitly, delete only empty folders (folders with many assets require deleting/moving contents first).

## Renaming, moving, deleting — and the CDN caveat

- **Rename** changes an asset's public ID (fixed mode: also its folder/URL). Use `overwrite: true` only when the user explicitly accepts replacing an existing target.
- **Delete** by public IDs, asset IDs, prefix, or tag. Deleting by tag/prefix is bulk and irreversible — restate scope and confirm before running. `keep_original: true` deletes only derived versions.
- **CDN invalidation:** deleting or overwriting an asset does NOT purge CDN caches by default — old versions keep being served until cache TTL expires. Pass `invalidate: true` on rename/delete/overwrite when the user needs the change visible immediately, and warn that invalidation propagation across the CDN takes several minutes to about an hour.

## Environment configuration (environment-config server)

- **Upload presets:** named bundles of upload options (folder, tags, context/metadata defaults, incoming/eager transformations, allowed formats, moderation, auto-tagging). *Signed* presets support all parameters (backend use); *unsigned* presets are for client-side uploads — protect them with `allowed_formats`, `max_file_size`, and `disallow_public_id`. Prefer editing an existing preset over creating near-duplicates; list existing presets first.
- **Named transformations:** saved, reusable transformation definitions referenced as `t_<name>` in URLs. Changing a named transformation affects every URL using it — check usage and prefer creating a new name over mutating one in production. Avoid `f_auto` inside eager/incoming transformations; use `eager_async: true` for video.
- **Streaming profiles:** predefined ladder configurations for adaptive bitrate video; create/update via this server rather than hand-building per-asset transformations.
- **Webhook notifications:** configure notification URLs and event-type triggers (upload, delete, moderation, metadata changes). Advise users to verify notification signatures on their receiving endpoint. Metadata-change notifications indicate origin (API vs console) and change type.

## Rate limits and operational discipline

- Admin API operations are rate-limited per hour (500/hr on free plans, 2,000+/hr on paid). Every call counts; `versions=true` costs extra.
- Check `X-FeatureRateLimit-Remaining` / `X-FeatureRateLimit-Reset` headers when doing bulk work; batch operations (delete by tag/prefix, bulk tag updates) instead of looping per-asset calls.
- Never poll listings in a loop to "watch" for changes — recommend webhooks instead.
- For destructive or URL-breaking operations (bulk delete, rename in fixed folder mode, editing a shared named transformation or preset), summarize the blast radius and get user confirmation first.

## Further reference

Search field names, rate-limit numbers, and metadata/config options evolve — confirm against the docs before relying on specifics:

- [Search API](https://cloudinary.com/documentation/search_api) — full expression syntax and queryable fields.
- [Admin API reference](https://cloudinary.com/documentation/admin_api) and [Structured metadata](https://cloudinary.com/documentation/structured_metadata) — current limits, field types, and config options.
- [`llms.txt`](https://cloudinary.com/documentation/llms.txt) — index to the rest of the docs when you need to go deeper.
