# Cloudinary Power for Kiro

A [Kiro power](https://kiro.dev/docs/powers/) that connects Kiro to [Cloudinary](https://cloudinary.com) — upload, manage, transform, optimize, and analyze images and videos directly from your agent.

## What it does

- Wires up four of Cloudinary's remote MCP servers over **OAuth** (browser sign-in, no API keys pasted into config):
  - **Asset management** — upload assets and manage your media library (Upload + Admin API capabilities)
  - **Environment config** — upload presets, named transformations, streaming profiles, webhooks
  - **Structured metadata** — define and manage metadata fields
  - **Analysis** — AI media analysis (tagging, captioning, moderation)
- Ships steering files with Cloudinary best practices for uploads, transformations & optimization, asset management, and media analysis.
- Supports **agentic account provisioning**: if you don't have a Cloudinary account, Kiro can create one from your email — you claim it by verifying the email and setting a password, then sign in via OAuth. See [Cloudinary's AUTH.md](https://github.com/cloudinary/auth.md) for the underlying agentic-registration flow.

## Install

1. Open Kiro → **Powers** panel → **Add Custom Power**
2. **Import power from GitHub** using this repository's `cloudinary/` folder URL (or **Import power from a folder** and select the `cloudinary/` directory if you cloned it)
3. On first use, Kiro opens your browser to sign in to Cloudinary and choose a product environment

## Layout

```
.
├── README.md
├── LICENSE
└── cloudinary/       ← the power; import this folder in Kiro
    ├── POWER.md      ← power manifest: activation keywords, onboarding, steering routing
    ├── mcp.json      ← four remote Cloudinary MCP servers (OAuth)
    └── steering/
        ├── upload.md
        ├── transformations.md
        ├── asset-management.md
        ├── analysis.md
        └── account-provisioning.md
```

## Try it

Ask Kiro things like:

- "Upload everything in ./assets to Cloudinary and tag it `launch`"
- "Give me a responsive, optimized delivery URL for hero.jpg cropped to 16:9"
- "Find all videos larger than 50 MB uploaded this month"
- "Auto-caption the images in the products folder and store the captions as metadata"
