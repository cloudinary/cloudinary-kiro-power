# Provisioning a Cloudinary Account for the User

When this file applies: the user wants to use Cloudinary but has no account, and has asked you (or agreed) to set one up on their behalf. This is Cloudinary's **agentic registration** flow: you register an account from the user's email, the user claims it by verifying their email and setting a password, and then you connect via OAuth like any existing account. If the user already has an account, skip this file entirely and connect the MCP servers via OAuth.

## Step 1 — Register

`POST` the agent account-creation endpoint. No authentication is required.

```http
POST https://api.cloudinary.com/v1_1/provisioning/agents/accounts
Content-Type: application/json

{
  "email": "user@example.com",
  "agent_framework": "kiro",
  "agent_llm_model": "<your model & version>",
  "agent_goal": "<one sentence: why this account is being created>",
  "sdk_framework": "<optional, e.g. node — tailors the returned guidance>"
}
```

Field constraints: `email` required; `agent_framework` and `agent_llm_model` required, 2–100 chars; `agent_goal` required, 2–300 chars; `sdk_framework` optional, 2–100 chars.

A `200` response includes the new account's product environment with root `api_key` / `api_secret` and a `CLOUDINARY_URL` string, plus a `guidance` block.

**Important:** these credentials are **inert** — the environment is created disabled and nothing works until the user completes Step 2. You generally do **not** need to store them: the preferred path after the claim is OAuth (Step 3). Surface the `guidance` text to the user.

## Step 2 — Claim ceremony (email verification)

Cloudinary emails the user a verification link. Tell the user to:

1. Open the email Cloudinary sent to their address.
2. Click the verification link, **set a password**, and confirm.

This activates the product environment and gives the user a normal Cloudinary login. The link expires in ~24 hours — if it lapses, the user can request a new one from the sign-in page.

There is **no pollable completion signal** for the claim. The user completing the OAuth sign-in in Step 3 is itself the confirmation. (Only if you must use root credentials directly — see fallback — detect activation by retrying a real API call with backoff until it succeeds.)

## Step 3 — Connect via OAuth (preferred)

Once the user has claimed the account, return to the power's onboarding Step 2: connect the `cloudinary-*` MCP servers. The user signs in with the password they just set and selects the product environment. You receive scoped, short-lived tokens and never handle the root secret.

**Fallback — root credentials.** Only if the user explicitly needs direct REST API access beyond what the MCP servers expose (MCP tokens do not work against `https://api.cloudinary.com/...`), the root `api_key`/`api_secret` from Step 1 can be used with HTTP Basic auth or via `CLOUDINARY_URL` in an SDK. Treat the secret as full-access and long-lived: never log it, never put it in client-side code, and have the user store it in their own secrets manager. Prefer OAuth for everything else.

## Errors

Cloudinary's standard error envelope: `{ "error": { "category", "message", "code?", "details?" } }`. `code` is optional — the validation and duplicate-email 400s carry only `category` + `message`.

| Status | Meaning | What to do |
| --- | --- | --- |
| 400 (validation) | Missing/oversized field or invalid UTF-8 | Fix the request body. |
| 400 (email taken) | Message contains `{"email":["has already been taken"]}` | The user already has an account — don't retry; connect via OAuth instead. |
| 403 `agent_registration_disabled` | Agent signup temporarily off | Don't retry tightly; ask the user to sign up at cloudinary.com themselves. |
| 403 `geo_location_not_permitted` | Region not allowed | Inform the user; do not retry. |
| 403 (generic "Invalid request") | Request blocked (e.g., IP gating) | Do not probe further. |
| 429 `ip_rate_limit_exceeded` | Per-IP signup cap (default 10/day) | Back off; retry later. |
| 5xx | Transient | Retry with exponential backoff. |
