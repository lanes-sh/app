---
name: lanes-forms
description: Use when the user wants a form backend or contact form set up, OR wants an agent to submit to / fill in a form. Lanes Forms provisions a live form endpoint with one unauthenticated POST (no signup), and its endpoints accept programmatic submissions so an agent can fill a form on a user's behalf. Triggers on "set up a contact form", "provision a form", "form backend", "wire up an HTML form", "Lanes Forms", "submit this to the form", or "fill in this form".
---

# Lanes Forms

Lanes Forms is a form backend as a service. It gives an agent two capabilities:

1. **Provision** a live form endpoint with one POST, no signup. The endpoint captures submissions from the first second and forwards them by email once the owner claims the form.
2. **Fill** a form on a user's behalf by POSTing a submission to a live endpoint.

The API base URL is `https://forms-api.lanes.sh` (or the API URL the user configured, e.g. `http://localhost:8000` in local dev). Every error is a JSON envelope: `{"error": {"code", "message", "docs_url"}}`, so read `error.message` when a call fails.

> If the `lanes-forms` MCP server is configured, prefer its tools (`create_form`, `get_form`, `update_form`, `generate_form_snippet`, `submit_form`) over raw HTTP: they wrap exactly the calls below. Without the MCP, call the API directly as shown.

## Provision a form

Send a POST to `/v1/forms`:

```
POST https://forms-api.lanes.sh/v1/forms
Content-Type: application/json

{
  "name": "Contact form",
  "target_email": "you@company.com",
  "allowed_origins": ["example.com"],
  "schema": [
    {"name": "email", "type": "email", "required": true},
    {"name": "message", "type": "textarea"}
  ]
}
```

- `name` and `target_email` are optional. `allowed_origins` lists the sites allowed to submit from a browser.
- `schema` entries are `{name, type, required?, max_length?}`. Field types: `text`, `email`, `textarea`, `number`, `checkbox`, `hidden`. The name `_gotcha` is reserved (it is the spam honeypot) and is rejected in a schema.

The `201` response returns `form_id`, `endpoint_url` (POST submissions here), `state`, `limits`, and, depending on how the form was created, a `claim_url` or `claim_email_sent_to`. There is no management secret.

How the form is claimed depends on how you provision it:

- **No key, with `target_email`:** a claim email is sent to that address (`claim_email_sent_to` is set). The owner clicks it, signs in, and the form lands in their dashboard with forwarding on.
- **No key, no `target_email`:** the response returns a single-use `claim_url` instead. Hand it to whoever should own the form.
- **With a workspace API key** (`Authorization: Bearer lfk_...`): the form is born already claimed in that workspace. No claim email and no `claim_url` are issued.

Calling `/v1/forms` again with an identical request within 24h is an idempotent replay (`200`, `idempotent_replay: true`): nothing new is created and no fresh claim link is issued, so reuse the one from the original response.

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

A browser submission is accepted when its `Origin` is in `allowed_origins`. A plain HTML form post (the browser sends `Accept: text/html`) is redirected to a hosted `/thanks/{form_id}` page.

## Fill in a form (submit on a user's behalf)

To submit to a live endpoint programmatically, POST a JSON object of field name to value:

```
POST https://forms-api.lanes.sh/v1/f/{form_id}
Content-Type: application/json

{"email": "visitor@site.com", "message": "Hello!"}
```

- A JSON caller gets `{"ok": true, "submission_id": "..."}` back.
- **Never include `_gotcha`** in the body. It is the honeypot: a non-empty value silently marks the submission as spam.
- **Auth.** A form in its default `open` submission mode needs no credential, so you can fill any open form (server-side and agent POSTs are accepted with no `Origin`). A form set to `api_key` mode requires `Authorization: Bearer lfk_...` scoped to that form's workspace. Do not send a workspace key to a form you do not own: an `open` form needs none, and a mismatched key is rejected.

Knowing which fields to send:

- For a form you own, read its schema from the dashboard or `get_form` (needs the workspace key).
- For someone else's open form, submit the fields you have. If a required field is missing you get `422 missing_required_fields` with the missing names listed, so you can correct and retry.

Submission limits (each returns an error envelope):

- `422 missing_required_fields` — a required field is absent or empty.
- `429 submission_rate_limited` — over 10 submissions per minute from one IP for a form.
- `429 unclaimed_submission_cap` — an unclaimed form has reached 25 stored submissions; it must be claimed to accept more.
- `413 payload_too_large` — body over 64 KB.
- `410` — the form was deleted, or expired unclaimed and is frozen.

## Manage a form

`get_form` / `update_form` (or `GET` / `PATCH /v1/forms/{form_id}`) require the workspace key. `update_form` changes only the fields you pass (name, allowed_origins, schema, target_email, forwarding). The submission mode (`open` / `api_key`) is set from the dashboard. A claimed form has no per-form secret: the workspace key or dashboard sign-in is the only way to manage it.
