---
name: receiving-inbound-email
description: >-
  Use when an application receives email rather than sends it — provisioning a Mailtrap
  inbound address, pointing an MX record at Mailtrap, or reading, replying to and
  forwarding arriving mail. Use when building an email-to-ticket flow or an AI agent with
  its own inbox. Use when verifying an inbound webhook signature or working with
  `@inbound-mailtrap.io` addresses.
---

# Receiving Email with Mailtrap Inbound

## Overview

**Inbound Email** gives an application hosted receiving addresses on **`inbound-mailtrap.io`** — no mail server and no DNS work to get started. It is a distinct product, **included with Email API/SMTP** rather than sold separately.

Resources nest: **folder → inbox → messages**, with **threads** grouping related messages. There are two ways to receive, usable together:

- **Webhook push** — Mailtrap POSTs parsed JSON to your endpoint as mail arrives.
- **Polling** — read the messages endpoint on your own schedule.

**Before generating SDK code:** read the README of the relevant SDK repository (see **SDKs** below) for the current inbound surface. Do not rely on memory.

**Related skills:** `authorizing-api-requests` (token scope, env vars, `account_id` resolution), `sending-emails` (outbound and reply delivery), `testing-with-sandbox` (capturing mail your own app sends — and the natural partner for rehearsing a receive-and-reply flow before it reaches production).

## When to use

- Your application is the **destination** for mail: support inboxes, email-to-ticket, parsers, agent inboxes.
- You need a **receiving address** without running an MX record or a mail server.
- You want to **list, read, reply to, or forward** messages that have arrived.
- You need to **verify an inbound webhook signature** before trusting a payload.

## When not to use

- **Sending** mail to recipients → `sending-emails`.
- **Capturing mail your own app sent** so it is never delivered → `testing-with-sandbox`. That is **Email Sandbox**, a different product. Sandbox addresses live on `inbox.mailtrap.io`; Inbound Email addresses live on `inbound-mailtrap.io`. The two are not interchangeable.
- Token scope, storage, or `account_id` resolution → `authorizing-api-requests`.

## Quick reference

### API base

| Service           | Base URL                            | Auth                                     |
| ----------------- | ----------------------------------- | ---------------------------------------- |
| Inbound Email API | `https://mailtrap.io/api/inbound`   | `Authorization: Bearer $MAILTRAP_API_TOKEN` |

### Tokens

Creating folders and inboxes requires an **account-level** token. A token scoped to a single sending domain can *read* inbound but returns **403** on create — if `create` fails with 403, this is why. See `authorizing-api-requests`.

### CLI

The [Mailtrap CLI](https://github.com/mailtrap/mailtrap-cli) is the most direct path and covers the whole inbound surface:

```
brew install mailtrap/cli/mailtrap
mailtrap configure --api-token YOUR_TOKEN
```

```
mailtrap inbound folders   list | get | create | update | delete
mailtrap inbound inboxes   list | get | create | update | delete
mailtrap inbound messages  list | get | delete | reply | reply-all | forward
mailtrap inbound threads   list | get | delete
```

**Always pass `-o json`**, or set `MAILTRAP_OUTPUT=json` once (works on v0.6.0, though no help text mentions it). Output defaults to `table`, which is meant for humans. Global flags: `--api-token` (env `MAILTRAP_API_TOKEN`), `--account-id` (env `MAILTRAP_ACCOUNT_ID`), `-o/--output`.

If the [Mailtrap MCP server](https://github.com/mailtrap/mailtrap-mcp) is already connected, it exposes the same operations as tools (`list-inbound-messages`, `reply-to-inbound-message`, and so on).

### Provision an inbox

```
mailtrap inbound folders create --name "Support" -o json
mailtrap inbound inboxes create --folder-id FOLDER_ID --name "Support inbox" -o json
```

The inbox response carries the generated `address`. Send mail to it, then read:

```
mailtrap inbound messages list --inbox-id INBOX_ID -o json
mailtrap inbound messages get --inbox-id INBOX_ID --id MESSAGE_ID -o json
```

Bodies come back as **`text_body`** and **`html_body`** — not `text`/`html`.

⚠️ **The CLI returns a reduced view of a message.** On v0.6.0, `get` surfaces 10 of the API's 22 fields and `list` 8 of 18. Silently dropped from both: **`attachments`**, `cc`, `bcc`, `reply_to`, `headers`, `in_reply_to`, `references`, `rfc_message_id`, `raw_message_url`, `text_size`, `html_size`. `list` also discards the `{data, last_id, total_count}` wrapper, so **`--last-id` pagination cannot be driven from CLI output** — the cursor is never returned. No flag changes this.

So for attachments, headers, cc/bcc, threading fields, raw MIME, or pagination, go to the API (or an SDK, or the MCP tools) — `GET /inboxes/{inbox_id}/messages` and `GET /inboxes/{inbox_id}/messages/{message_id}` off the base above. The CLI stays the shortest path for provisioning, replying, forwarding and deleting.

### Attachments

`list` rows carry the metadata — `attachment_id`, `filename`, `content_type`, `content_disposition`, `content_id`, `size`. `get` adds **`download_url`** and **`download_url_expires_at`**: a signed S3 link good for roughly an hour, so fetch it while handling the message rather than storing it for later. `raw_message_url` (also expiring) returns the full MIME. The API returns `attachments` as an empty array when a message has none; it is the CLI that drops the key on every message, so read attachments through the API.

**Size cap.** Inbound rejects anything over the account's maximum email size at SMTP time with `552 5.3.4 Message exceeds max size` — **10 MB on the lower Email API/SMTP tiers, up to 30 MB on Business and Enterprise** (pricing page, "Max email size"). The rejection happens before delivery, so nothing is stored and no webhook fires — the sender gets the bounce and the inbox stays empty. The ceiling applies to the *encoded* MIME message, and base64 adds roughly a third, so on a 10 MB plan a raw attachment much above ~7.5 MB will not fit. If a message someone insists they sent never appears, check its size first.

The `get` response includes a `domain_id` on **every** inbox, hosted ones included — it is not a reliable signal for whether an inbox is custom-domain. Judge that from the address or from how the inbox was created.

### Custom-domain catch-all

To receive at your own domain: verify the domain, enable inbound domain receiving under **Domains → Domain Verification**, add the MX record Mailtrap shows, then create the inbox with `--domain-id`:

```
mailtrap inbound inboxes create --folder-id FOLDER_ID --name "Catch-all" --domain-id DOMAIN_ID -o json
```

The result is a catch-all: every address at that domain lands in the inbox. Per-username routing rules are not available.

### Reply and forward

⚠️ **`reply`, `reply-all`, and `forward` send real email to real recipients.** They are not sandboxed. Never run them in a loop over an inbox without an explicit allowlist.

```
mailtrap inbound messages reply --inbox-id INBOX_ID --id MESSAGE_ID --text "Thanks for reaching out!" -o json
mailtrap inbound messages forward --inbox-id INBOX_ID --id MESSAGE_ID --to colleague@example.com -o json
```

`reply` also accepts `--html`, `--cc`, `--bcc`, `--reply-to`, and `--category`. `--from` works on **custom-domain inboxes only** — a hosted `@inbound-mailtrap.io` inbox replies from its own address.

### Webhooks

Register the endpoint, then verify every payload before acting on it.

- Payloads are signed **HMAC-SHA256**, carried in the **`mailtrap-signature`** header. HTTP header names are case-insensitive; there is **no `X-` prefix**.
- The signing secret is **viewable and resettable in the webhook's details in the Mailtrap UI**. The CLI returns it only from `webhooks create`; `webhooks get` and `list` do not include it.
- Failed deliveries are **retried every 5 minutes for about 3 hours, then the webhook is paused** (`active: false`) until you re-enable it. A slow handler earns a queue of duplicates; a dead one goes silent. Acknowledge with a 2xx quickly and move slow work to a queue.

**Verify the signature against the raw request body.** Deserializing and re-serializing the JSON reorders keys and normalizes whitespace, which changes the bytes the HMAC was computed over and makes every signature fail. In practice this means reading the body as text *before* any framework model-binding touches it.

Details: [Inbound webhooks](https://docs.mailtrap.io/inbound-email/webhooks.md).

### SDKs

Official SDKs cover the inbound resources — read the repository README for the current surface:
[Node.js](https://github.com/mailtrap/mailtrap-nodejs) · [Python](https://github.com/mailtrap/mailtrap-python) · [PHP](https://github.com/mailtrap/mailtrap-php) · [Ruby](https://github.com/mailtrap/mailtrap-ruby) · [Java](https://github.com/mailtrap/mailtrap-java) · [.NET](https://github.com/mailtrap/mailtrap-dotnet) · [Go](https://github.com/mailtrap/mailtrap-go) · [CLI](https://github.com/mailtrap/mailtrap-cli)

### Common mistakes

| Mistake                                              | Fix                                                                                                     |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| Verifying the signature against re-serialized JSON    | Use the **raw request body**. Re-serializing reorders keys and breaks every signature.                   |
| Writing `inbound.mailtrap.io`                         | The domain is **`inbound-mailtrap.io`**, with a hyphen.                                                  |
| Expecting an `X-Mailtrap-Signature` header            | The header is `mailtrap-signature` — no `X-` prefix.                                                     |
| `create` returns 403                                  | The token is domain-scoped. Folder and inbox creation needs an **account-level** token.                  |
| Parsing CLI output that arrived as a table            | Pass `-o json` or set `MAILTRAP_OUTPUT=json`; `table` is the default and is for humans.                  |
| Confusing Sandbox addresses with Inbound addresses    | `inbox.mailtrap.io` is Email Sandbox; `inbound-mailtrap.io` is Inbound Email.                            |
| Running `reply` or `forward` while exploring an inbox | Both deliver real mail. Use `list` and `get` to inspect.                                                 |
| Setting `--from` on a hosted inbox                    | `--from` is supported on **custom-domain inboxes only**.                                                 |
| Waiting on a large attachment that never arrives      | Plan size cap (10 MB lower tiers, 30 MB Business/Enterprise), SMTP `552 5.3.4`. No message, no webhook.  |
| Using `domain_id` to detect a custom-domain inbox     | Hosted inboxes return a `domain_id` too. Use the address instead.                                        |
| Concluding a message has no attachments from the CLI  | CLI `get`/`list` drop the `attachments` field entirely. Call the API for anything beyond body and subject.|
| Storing a `download_url` to fetch later               | Signed and expires in about an hour. Download during processing, or re-read the message.                 |

Full reference: [Receiving emails](https://docs.mailtrap.io/inbound-email/receiving-emails.md) and the [API docs](https://docs.mailtrap.io/developers/).
