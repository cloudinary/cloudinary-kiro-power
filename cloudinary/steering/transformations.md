# Cloudinary Transformations & Optimization

**When this file applies:** Use this guidance whenever you are constructing Cloudinary delivery URLs, writing code that renders Cloudinary-hosted images or videos (img/video tags, SDK calls, srcset attributes), or advising on media optimization in a project connected to Cloudinary via MCP. It covers URL-based transformations for images and video, optimization defaults, responsive delivery, and generative AI edits.

## Delivery URL anatomy

```
https://res.cloudinary.com/<cloud_name>/<asset_type>/<delivery_type>/<transformations>/<version>/<public_id>.<ext>
```

- `asset_type`: `image`, `video`, or `raw`. `delivery_type`: usually `upload`.
- `<transformations>` is optional. `<version>` (e.g. `v1712345678`) is optional and busts CDN cache.
- Extension is optional; with `f_auto` you can omit it (or keep the original — `f_auto` overrides it).
- Public IDs are **case-sensitive** and may contain folder paths (`products/shoes/red-sneaker`). Never change their casing or guess an extension.

Example: `https://res.cloudinary.com/demo/image/upload/c_fill,w_300,h_200,g_auto/q_auto/f_auto/sample`

## Transformation syntax rules

- **Comma-separate** parameters within one component: `c_fill,w_300,h_200,g_auto`.
- **Slash-separate** chained components; each applies to the output of the previous one, left to right. **Order matters** — cropping then rounding corners is not the same as the reverse.
- **One action per component.** Action parameters (`c_`, `e_`, `l_`, `a_`) each get their own component. Qualifiers (`w_`, `h_`, `g_`, `ar_`, `x_`, `y_`, `co_`) must sit in the same component as the action they modify — a `g_auto` in its own component does nothing.
- Identical URLs hit the same cached derived asset; keep parameter order consistent (SDKs sort alphabetically) to maximize cache hits.

## Optimization defaults — always apply

Append `q_auto/f_auto` (or `q_auto,f_auto` as the final component) to **every** delivery URL unless the user explicitly needs a fixed format/quality:

- `f_auto` — serves AVIF/WebP/JPEG XL/etc. per browser support. Transparent images fall back to PNG when needed.
- `q_auto` — content-aware compression. Levels: `q_auto:best`, `q_auto:good` (default), `q_auto:eco`, `q_auto:low`. Honors the browser `Save-Data` header (switches to eco).
- Don't set a numeric quality (`q_80`) alongside or before `f_auto` — it defeats per-format tuning. Let `q_auto` decide; only override with a level, not a number.
- `f_auto` is ineffective when buried inside a named transformation — keep it in the URL itself.

## Resize & crop modes — choosing the right one

| Mode | Behavior | Use when |
|---|---|---|
| `c_fill` | Scale + crop to exactly fill w×h | Fixed-size slots (cards, heroes); pair with `g_auto` |
| `c_lfill` | Like fill, but never upscales | Fill without quality loss on small originals |
| `c_fit` | Scale to fit within w×h, no crop | Whole image must be visible; container tolerates letterbox |
| `c_limit` | Like fit, but only scales down — never upscales | Responsive max-size delivery; capping dimensions safely — good default for user content |
| `c_scale` | Force to w×h (or one dim, other auto) | Simple resizing; may distort if both dims given |
| `c_crop` | Extract region, no scaling | Cutting a region at original resolution |
| `c_thumb` | Crop toward detected face(s), then scale | Avatars/profile photos; use with `g_face` and optional `z_` zoom |
| `c_pad` / `c_lpad` | Fit then pad with background to exact w×h | Exact dimensions with full image visible; set `b_` (or `b_gen_fill`) |
| `c_auto` | AI picks the best crop strategy | When unsure and content varies |

**Gravity** (qualifier for fill/crop/thumb/pad):
- `g_auto` — AI keeps the most interesting content in frame. Default choice for `c_fill`.
- `g_face` / `g_faces` — center on one/all faces. Best for people.
- Compass: `g_north`, `g_south_east`, `g_center`, etc. — for deterministic positioning and overlay placement.

**Aspect ratio:** `ar_16:9`, `ar_1:1`, or decimal `ar_1.5`. Give `ar_` plus one dimension instead of hard-coding both: `c_fill,ar_1:1,w_400,g_auto`.

## Responsive sizing & DPR

- Always deliver images at their final rendered size — resize server-side, never with CSS alone.
- `dpr_2` (or `dpr_auto` with client hints) serves high-density variants: `c_fill,w_300,dpr_2` renders 600px wide for a 300px CSS slot.
- For `srcset`, generate width variants by changing only `w_` and keep everything else identical:
  ```html
  <img src=".../c_limit,w_800/q_auto/f_auto/hero"
       srcset=".../c_limit,w_400/q_auto/f_auto/hero 400w,
               .../c_limit,w_800/q_auto/f_auto/hero 800w,
               .../c_limit,w_1600/q_auto/f_auto/hero 1600w"
       sizes="(max-width: 600px) 100vw, 800px">
  ```
- 3–5 breakpoints is usually enough; each distinct URL is a billable derived transformation.

## Named transformations

- `t_<name>` applies a preset saved in the account: `/t_product_card/sample.jpg`.
- Prefer them for transformations reused across a codebase — one place to update, shorter URLs, and they can hide transformation details in strict-transformation setups.
- Can be chained with inline components: `/t_product_card/f_auto/...` (keep `f_auto` outside the named transformation).

## Overlays & text basics

- Image overlay: `l_<overlay_public_id>` — use `:` instead of `/` for folders (`l_logos:acme`). Position and close with a separate `fl_layer_apply` component when styling the layer:
  `/l_logos:acme,w_100/fl_layer_apply,g_south_east,x_20,y_20/`
- Text overlay: `l_text:<font>_<size>:<URL-encoded text>` with styling qualifiers:
  `/l_text:Arial_60_bold:Sale%2050%25/co_white,g_south,y_40/`
- URL-encode overlay text (spaces `%20`, `%` as `%25`, commas as `%2C`).

## Video transformations

Base: `https://res.cloudinary.com/<cloud>/video/upload/<transformations>/<public_id>.<ext>`

- Resize/crop works like images: `c_fill,w_640,h_360,g_auto`. `q_auto`/`f_auto` apply too.
- **Adaptive streaming:** use a streaming profile + HLS/DASH format: `sp_auto/f_m3u8` (or `sp_hd`, and `.m3u8`/`.mpd` extensions). Prefer ABR streaming over progressive MP4 for anything longer than a short clip.
- **Thumbnails from video:** request an image format with a start offset:
  `/video/upload/so_2.5,c_fill,w_400,h_225,g_auto/q_auto/f_auto/my-video.jpg` — `so_` picks the frame (seconds, or `so_auto`).
- **Trimming:** `so_` (start), `eo_` (end), `du_` (duration): `so_5,du_10`.
- On-the-fly limits: ~60 min output for ABR, ~30 min for progressive; longer runs asynchronously.

## Generative AI transforms (consume extra credits)

These count against quota at a **higher special rate** than standard transformations — mention this when suggesting them, and prefer eager/pre-generated derivatives since first requests may return HTTP 423/420 while processing.

- `e_gen_remove:prompt_<object>` — remove an object and inpaint: `e_gen_remove:prompt_the%20stick`. Multiple: `prompt_(phone;keyboard)`. Avoid on faces/hands.
- `e_gen_replace:from_<x>;to_<y>` — swap an object: `e_gen_replace:from_the%20lamp;to_a%20plant`.
- `b_gen_fill` — generative background used with padding crops to extend an image to a new aspect ratio: `ar_16:9,b_gen_fill,c_pad,w_1200`. Great for portrait→landscape.
- Also available: `e_gen_background_replace[:prompt_...]`, `e_gen_recolor:prompt_<obj>;to-color_<hex>`, `e_gen_restore`. Add `;seed_<n>` for reproducible variations. Not supported on animated or fetched images; most require non-transparent input.

## Common mistakes to avoid

1. **Wrong chain order.** Components run left to right — `e_grayscale` before an overlay grays only the base; after `fl_layer_apply` it grays everything.
2. **Splitting one action across components.** `c_fill,w_300/g_auto` silently ignores the gravity; qualifiers must live with their action: `c_fill,w_300,g_auto`.
3. **Cramming chained actions into one component.** Two effects like `e_blur,e_grayscale` in one component is invalid — chain them: `e_blur:200/e_grayscale`.
4. **Fixed quality with `f_auto`.** `q_80,f_auto` locks quality tuned for one format; use `q_auto,f_auto`.
5. **Changing public_id casing or extension.** IDs are case-sensitive; `Sample.jpg` ≠ `sample.jpg`. Don't append an extension you haven't verified — with `f_auto` omit it.
6. **Forgetting URL encoding** in text overlays and gen-AI prompts (spaces, commas, slashes).
7. **Upscaling small originals** with `c_fill`/`c_scale` — use `c_limit`/`c_lfill` when the source size is unknown.
8. **CSS-only resizing** — shipping a 4000px original into a 300px slot; always resize in the URL.
9. **`f_auto` inside a named transformation** — it won't negotiate formats there; keep it in the delivery URL.
10. **Unbounded gen-AI usage** — generative transforms are billed at a premium; cache/eagerly generate rather than composing unique prompts per request.

## Further reference

The syntax and gotchas above are stable, but the full parameter catalog and premium/credit specifics change. For authoritative, current detail:

- **`get-tx-reference`** (asset-management MCP server) — official transformation syntax reference; pull it once per session before composing non-trivial URLs.
- [Transformation reference](https://cloudinary.com/documentation/transformation_reference) and [Generative AI transformations](https://cloudinary.com/documentation/generative_ai_transformations) — full parameter lists, current limits, and credit rates.
- [`llms.txt`](https://cloudinary.com/documentation/llms.txt) — index to the rest of the docs when you need to go deeper.
