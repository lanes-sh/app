---
name: lanes-forms
description: Use when the user wants a form backend or contact form set up (a live POST endpoint that captures submissions and emails them), OR wants an agent to fill in / submit to a form on their behalf. Lanes Forms creates and manages hosted form endpoints from Claude Code. Triggers on "set up a contact form", "provision a form", "form backend", "wire up an HTML form", "Lanes Forms", "submit this to the form", or "fill in this form".
---

# Lanes Forms

Lanes Forms is a form backend as a service. It gives an agent two capabilities:

1. **Provision** a live form endpoint that captures submissions from the first second and delivers them (email forwarding and/or stored in the dashboard).
2. **Fill** a form on a user's behalf by POSTing a submission to a live endpoint.

The API base URL is `https://api.lanes.sh` in production (or the API URL the user configured — the local dev default is `http://localhost:8080`). Every error is a JSON envelope: `{"error": {"code", "message", "docs_url"}}`, so read `error.message` when a call fails.

## Fastest path: the MCP server

The `lanes-forms` MCP server wraps every call below and is the easiest way to work. If it isn't connected yet, run **`/setup-forms-mcp`** to register it — it points at the hosted server (`https://api.lanes.sh/mcp`), authenticated with a workspace key. Its five tools are `create_form`, `get_form`, `update_form`, `generate_form_snippet`, and `submit_form`. With a workspace key configured, `create_form` provisions a form **born claimed** into that workspace, and the management tools (`get_form` / `update_form` / `generate_form_snippet`) require it. Without the MCP, call the API directly as shown below.

## Provision a form

Send a `POST` to `/v1/forms`. Only `schema` is required.

```
POST https://api.lanes.sh/v1/forms
Authorization: Bearer lfk_...          # workspace key — see "Get a workspace key"
Content-Type: application/json

{
  "name": "Contact form",
  "recipients": ["you@company.com"],
  "allowed_origins": ["example.com"],
  "schema": [
    {"name": "email", "type": "email", "required": true},
    {"name": "message", "type": "textarea"}
  ]
}
```

- `schema` is **required** (1–50 fields). Each entry is `{name, type, required?, max_length?}`. Field types: `text`, `email`, `textarea`, `number`, `checkbox`, `hidden`. Names must be unique, 1–80 chars; `_gotcha` is reserved (the spam honeypot) and is rejected in a schema.
- `name`, `allowed_origins`, and `recipients` are optional. `allowed_origins` (max 20) are the sites allowed to submit from a browser — give a hostname (`example.com`), full origin (`https://example.com`), or wildcard (`*.example.com`); `localhost` is always allowed for local testing. `recipients` (max 5) are the addresses submissions are emailed to; the **first** is the claim/owner address.

The `201` response returns `form_id`, `endpoint_url` (POST submissions here), `state`, `expires_at`, `limits`, `workflow`, `forward_email`, `store_submissions`, and — depending on how it was created — `claim_url` or `claim_email_sent_to`. There is no per-form management secret.

**How the form is created and claimed depends on the bearer:**

- **With a workspace API key** (`Authorization: Bearer lfk_...`) — the reliable path. The form is born **claimed** in that workspace, ready to manage immediately. No claim email and no `claim_url`; `expires_at` is null. This is what the MCP uses when `FORMS_API_KEY` is set.
- **Anonymous, with `recipients`:** a claim email is sent to the first address (`claim_email_sent_to` is set). The owner clicks it, signs in, and the form lands in their dashboard.
- **Anonymous, no `recipients`:** the response returns a single-use `claim_url` instead — hand it to whoever should own the form.

> **Anonymous provisioning is gated.** An unauthenticated `POST /v1/forms` returns `503 forms_public_disabled` on deployments where public provisioning is off (its current production default). When you can't sign the request anonymously, provision with a workspace `lfk_` key. Anonymous (unclaimed) forms store up to 25 submissions and **expire 7 days** after creation unless claimed.

Calling `/v1/forms` again with an identical **anonymous** request within 24h is an idempotent replay (`200`, `idempotent_replay: true`): nothing new is created and no fresh claim link is issued, so reuse the one from the original response. Keyed creates skip idempotency — every keyed call makes a new form.

## Delivery: where submissions go (workflow)

By default every form **stores** submissions in the dashboard, and if you passed `recipients` it also **emails** them there (each recipient is verified before delivery starts). For a normal contact form that's the whole setup — you don't need to send a `workflow` at all.

To change delivery, use the simple sugar **or** an explicit `workflow` array, never both (sending both returns `422 workflow_conflict`):

- **Sugar:** `forward_email` (bool) and `store_submissions` (bool) toggle the two defaults; `recipients` sets who email goes to. e.g. `"store_submissions": false` to email only.
- **Explicit `workflow`:** an array of `email` / `store` / `webhook` actions, e.g. `"workflow": [{"type": "email", "enabled": true, "to": ["you@co.com"]}, {"type": "store", "enabled": true}]`.

Enabling email with no recipient returns `422 email_action_requires_recipient`. **Webhook** delivery and **custom** (bring-your-own-database) storage are *coming soon* — enabling either returns `422 action_not_available`; use `destination: "lanes"` for storage for now.

## Get a workspace key (`lfk_`)

The keyed provision path and the MCP management tools need a workspace API key. The simplest way to mint one is the **Lanes dashboard's API keys page** (needs the **admin** role) — no bootstrap required. Programmatically, from an admin dashboard session:

1. `GET /v1/me` → read the `workspace_id` to mint under.
2. `POST /v1/workspaces/{workspace_id}/api-keys` with `{"name": "cli"}` → `201` returns `key` (the `lfk_...` plaintext) **once** — store it now; it is never shown again. A workspace allows up to 10 active keys.

## Wire it into a site

Point the site's form at `endpoint_url` and add a hidden `_gotcha` honeypot input so bots that fill it are dropped:

```html
<form action="ENDPOINT_URL" method="POST">
  <input type="email" name="email" required />
  <textarea name="message"></textarea>
  <input type="text" name="_gotcha" style="display:none" tabindex="-1" />
  <button type="submit">Send</button>
</form>
```

A browser submission is accepted when its `Origin` is in `allowed_origins`. A plain HTML form post (the browser sends `Accept: text/html`) is redirected to a hosted `/thanks/{form_id}` page. (`generate_form_snippet` produces this HTML — or a React component — for a form you own.)

## Fill in a form (submit on a user's behalf)

To submit to a live endpoint programmatically, POST a JSON object of field name to value:

```
POST https://api.lanes.sh/v1/f/{form_id}
Content-Type: application/json

{"email": "visitor@site.com", "message": "Hello!"}
```

- A JSON caller gets `{"ok": true, "submission_id": "..."}` back.
- **Never include `_gotcha`** in the body. It is the honeypot: a non-empty value silently marks the submission as spam.
- **Auth.** A form in its default `open` submission mode needs no credential, so you can fill any open form (server-side and agent POSTs are accepted with no `Origin`). A form set to `api_key` mode requires `Authorization: Bearer lfk_...` scoped to that form's workspace. Don't send a workspace key to a form you don't own: an `open` form needs none, and a mismatched key is rejected.

Knowing which fields to send: for a form you own, read its schema from the dashboard or `get_form`. For someone else's open form, submit the fields you have — if a required field is missing you get `422 missing_required_fields` with the missing names listed, so you can correct and retry.

## Manage a form

`get_form` / `update_form` (or `GET` / `PATCH /v1/forms/{form_id}`) take the workspace `lfk_` key (or a dashboard sign-in from a workspace member). `PATCH` changes only the fields you pass — `name`, `recipients`, `allowed_origins`, `schema`, `forward_email`, `store_submissions`, `workflow`, and `submission_auth` (`open` / `api_key`). Also available (dashboard sign-in): `DELETE /v1/forms/{form_id}` to delete, `GET /v1/forms/{form_id}/submissions` (add `?format=csv` to export), and `GET /v1/workspaces/{workspace_id}/forms` to list a workspace's forms.

## Errors and limits

Each returns the error envelope above:

- `422 missing_required_fields` — a required field is absent or empty (submission).
- `422 workflow_conflict` / `action_not_available` — sent both sugar and a `workflow` array, or enabled a coming-soon action.
- `429 submission_rate_limited` — over 10 submissions per minute from one IP for a form.
- `429 unclaimed_submission_cap` — an unclaimed form hit 25 stored submissions; it must be claimed to accept more.
- `413 payload_too_large` — submission body over 64 KB.
- `410` — the form was deleted, or expired unclaimed and is frozen.
- `503 forms_public_disabled` — anonymous provisioning is off on this deployment; use a workspace key.
- `401 invalid_api_key` — the `lfk_` key is unknown or revoked.
