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

Field constraints: `email` required (must be a valid address and **not** a disposable-email provider); `agent_framework` and `agent_llm_model` required, 2–100 chars; `agent_goal` required, 2–300 chars; `sdk_framework` optional, 2–100 chars.

A `200` response includes the new account's product environment with root `api_key` / `api_secret` and a `CLOUDINARY_URL` string, plus a `guidance` block.

**Redact the credentials at the tool boundary.** The response contains a full-access root secret; anything you print lands in the chat transcript and possibly in logs. Unless the user has explicitly chosen the headless fallback (Step 3), strip the credentials before the response enters the conversation — e.g.:

```sh
curl -s -X POST https://api.cloudinary.com/v1_1/provisioning/agents/accounts \
  -H 'Content-Type: application/json' -d @request.json \
  | jq 'del(.product_environments[].api_key, .product_environments[].api_secret)
        | del(.product_environments[].api_environment_variable)'
```

Never echo, display, or write the unredacted response to a file. These credentials are **inert** anyway — the environment is created disabled and nothing works until the user completes Step 2 — and the preferred path after the claim is OAuth (Step 3), which never needs them. Surface the `guidance` text to the user.

## Step 2 — Claim ceremony (email verification)

Cloudinary emails the user a verification link. Tell the user to:

1. Open the email Cloudinary sent to their address.
2. Click the verification link, **set a password**, and confirm.

This activates the product environment and gives the user a normal Cloudinary login. The link expires in ~24 hours — if it lapses, the user can request a new one from the sign-in page.

There is **no pollable completion signal** for the claim. The user completing the OAuth sign-in in Step 3 is itself the confirmation. (Only if you must use root credentials directly — see fallback — detect activation by retrying a real API call with backoff until it succeeds.)

## Step 3 — Connect via OAuth (preferred)

Once the user has claimed the account, return to the power's onboarding Step 2: connect the `cloudinary-*` MCP servers. The user signs in with the password they just set and selects the product environment. You receive scoped, short-lived tokens and never handle the root secret.

**Fallback — root credentials.** Only if the user explicitly needs direct REST API access beyond what the MCP servers expose, the root `api_key`/`api_secret` from Step 1 can be used with HTTP Basic auth or via `CLOUDINARY_URL` in an SDK. Treat the secret as full-access and long-lived: never log it, never put it in client-side code, and have the user store it in their own secrets manager. Prefer OAuth for everything else.

## Errors

Cloudinary returns an error envelope of the form `{ "error": { "category", "message" } }` (observed `category`: `user_error`). The documented failure conditions are:

| Status | Meaning | What to do |
| --- | --- | --- |
| 400 | A required parameter is missing, the email format is invalid, the email domain is a disposable-email provider, or an account already exists for the email. | Read the message. If an account already exists, don't retry — connect via OAuth instead (Onboarding Step 2). Otherwise fix the request body. |
| 403 | Cloudinary's abuse controls blocked the request. The message is **intentionally generic** (e.g., IP or region gating), so don't infer a specific cause. | Don't probe or retry tightly; if it persists, ask the user to sign up at cloudinary.com themselves. |
| 429 | Too many account-creation requests from the same IP address. | Back off and retry later. |

## Further reference

The endpoint contract above is the authority for this flow. For the surrounding account/OAuth concepts, see [`llms.txt`](https://cloudinary.com/documentation/llms.txt); if the provisioning endpoint's field constraints or error envelope appear to have changed, trust the live API response over this file.
