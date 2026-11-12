# PlacidWay Daily Facebook Post (n8n)

An n8n workflow that picks one fresh PlacidWay page per run, has an LLM write a short Facebook post, checks the post with two guardrail layers, and publishes it to a test Facebook Page (or sends it to a human for approval first). Every run is logged.

Built and tested on n8n 2.45.7 (self-hosted). All logic lives in n8n nodes: no external server or script is needed. Details on the stack: DeepSeek models through OpenRouter, Telegram for review and alerts, an n8n Data Table as the post log, Facebook Graph API v23.

## Assessment checklist: where to find each requirement

| Requirement from the brief | Where it lives | Evidence |
|---|---|---|
| Daily schedule + manual run | Nodes `Daily 09:00 MYT`, `Manual run` | [workflows/placidway_daily_post.json](workflows/placidway_daily_post.json) |
| Extract title, URL, text, image (method justified) | Official sitemap + `HTML` node, see *Decisions* | `Fetch sitemap`, `Extract title, text, image` |
| One item per run, never the same URL twice | Data Table `pw_post_log`, node `Skip already used URLs` | [docs/samples.md](docs/samples.md): every run picks a new URL |
| LLM post: hook, 2-4 lines, CTA, link, 3-5 hashtags, under 500 chars before the link | `LLM: write post` + `Assemble post + rule checks` | Rule check enforces every part |
| Guardrails: no medical claims, no invented prices, no facts outside the source | Rules in code + a separate LLM audit, one retry, then block + alert | *Guardrails in detail* |
| Publish to a test Page, with image | `Facebook: photo post` / `Facebook: text post` | Post IDs in [docs/samples.md](docs/samples.md) |
| Config flag: auto vs approval | `Config.mode`; approval = Telegram message + n8n form | `Send draft to reviewer`, `Wait for reviewer decision` |
| Log each run (item, text, post ID, status) | `pw_post_log` | Columns listed under *Logs and dashboard* |
| Retry and alert, never fail silently | Retry On Fail (3x), `Describe failure`, Telegram alert, execution marked failed | Failure test in [docs/samples.md](docs/samples.md) |
| Stretch: variety rules | Weekday rotation articles / Q&A / news | `Pick content type` |
| Stretch: simple dashboard | The Data Tables `pw_post_log` and `pw_engagement` | Built into n8n |
| Stretch: engagement read-back | Workflow `PlacidWay Engagement Read-back` | [workflows/placidway_engagement.json](workflows/placidway_engagement.json) |
| Stretch: image handling, cross-posting | Featured image (`og:image`) is posted; no generation, no Instagram/LinkedIn | See *Limitations* |

## What it does

1. Runs daily at 09:00 (Asia/Kuala_Lumpur) or on demand with **Manual run**.
2. Reads robots.txt and an official XML sitemap, picks the newest URL not used before.
3. Fetches that one page and extracts title, description, featured image, canonical URL and main text.
4. LLM writes: hook, 2-4 value lines, CTA, 3-5 hashtags (under 500 characters before the link). The workflow appends the link.
5. Guardrails check the draft. One regeneration is allowed, then the run is blocked and an alert is sent.
6. `auto` mode publishes straight away; `approval` mode asks a reviewer on Telegram first.
7. The result (item, post text, Facebook post ID, status) is written to a Data Table. Failures send a Telegram alert.

## Architecture

```
Manual run ─┐
Daily 09:00 ─┴► Config ► Pick content type (weekday rotation)
   ► Fetch robots.txt ► Wait 1s ► Fetch sitemap
   ► Parse sitemap + robots rules ► Skip already used URLs (Data Table)
   ► Take newest unused ► Wait 1s ► Fetch page
   ► Extract title, text, image ► Build source record
   ► Build writer prompt ► LLM: write post
   ► Assemble post + rule checks ► Build audit prompt ► LLM: audit claims
   ► Combine verdict ► Guardrails passed?
        │ no ► Retry left? ─ yes ► Add feedback ► (back to Build writer prompt)
        │                  └ no  ► Log blocked draft ► Telegram alert
        │ yes
        ▼
   Log draft (reserve URL) ► Approval mode?
        │ yes ► Ask reviewer on Telegram ► Approved? ─ no ► Log rejected
        │ no / approved
        ▼
   Publish to Facebook? ─ simulation ► Simulated result ┐
        │ facebook                                       │
        ▼                                                │
   Has image? ─ yes ► Facebook: photo post ┐             │
              └ no  ► Facebook: text post  ┴► Facebook result
                                                 ▼       ▼
                                              Log result ► Run summary

Any HTTP / LLM / Telegram / Facebook error output
   ► Describe failure ► Log failure ► Telegram alert ► Stop and Error
```

A second workflow, **PlacidWay Error Alerts** (Error Trigger), is a safety net for scheduled runs that crash outside the handled paths.

## Decisions and trade-offs

| Decision | Why |
|---|---|
| **Official XML sitemaps** (articles, Q&A answers, news) instead of scraping listing pages or RSS | Official, lightweight, lists images, needs no headless browser. RSS returned 403. |
| **Weekday rotation of content type** | Simple variety rule so the Page does not post only one kind of content. `source_override` forces a type. |
| **Polite crawling**: robots.txt parsed every run, fail-closed if unreadable; Wait nodes keep at least 1.1 s between requests; identifiable User-Agent; 3 requests per run | Respects the site and keeps load tiny. |
| **n8n Data Table `pw_post_log`** as dedup store and log | Built in, no extra service. `Skip already used URLs` filters on `url`. The URL is reserved (row inserted) before approval or publishing, so a pending or rejected item is never picked twice. Failed runs log an empty URL so the item can be retried. The table also acts as the dashboard. |
| **Two-layer guardrails** | Rules catch what is measurable; a second LLM call catches what rules cannot (see below). |
| **Link appended by the workflow, not the LLM** | The LLM cannot invent or alter URLs. |
| **Telegram message + n8n Wait form** for approval | The draft is sent to the reviewer on Telegram with a link to an n8n form (Approve / Reject). I first tried the native "send and wait" buttons, but Telegram rejects inline buttons that point to `localhost`. The form link needs no public URL. 12 h timeout counts as rejected. |
| **Failure releases the URL** | If a run fails after reserving a URL (for example a Facebook error), the log row is marked `failed` and its URL is cleared, so the item can be retried by a later run. |
| **Retry On Fail** (3 tries, 5 s) on HTTP, LLM and Facebook nodes | Handles transient errors; persistent errors go to the failure path. |

### Guardrails in detail

1. **Rule checks (Code node, deterministic):** valid structure, 2-4 value lines, 3-5 hashtags, under 500 characters before the link, banned claim words (for example "cure", "guaranteed", "painless", "success rate"), no currency symbols or price words, every number in the post must appear in the source text, no links from the LLM.
2. **LLM audit:** a second call checks each claim in the post against the source and returns PASS or FAIL with evidence.

On failure, the issues are sent back for **one** regeneration. If it fails again: status `blocked_guardrail`, nothing is published, Telegram alert sent.

## Setup

1. Import the four JSON files from the `workflows/` folder: `placidway_setup.json`, `placidway_daily_post.json`, `placidway_error_alerts.json`, `placidway_engagement.json`.
2. Run **PlacidWay Setup (run once)** to create the `pw_post_log` and `pw_engagement` Data Tables.
3. Create the credentials below and attach them to the matching nodes.
4. Open **Config** in the daily workflow and fill the fields.
5. In **PlacidWay Error Alerts**, fill the chat ID in **Alert config**.
6. Publish (activate) the daily workflow to enable the 09:00 schedule, and the engagement workflow to enable the 6-hourly read-back.

### Credentials (none are stored in the exported JSON)

| Credential | Type | Used by |
|---|---|---|
| OpenRouter API | Header Auth: `Authorization: Bearer <key>` | both LLM nodes |
| Telegram bot | Telegram API (token from BotFather) | alerts and approval |
| Facebook Graph API | Page access token with `pages_manage_posts`, `pages_read_engagement` | Facebook nodes |

### Config fields

| Field | Values | Meaning |
|---|---|---|
| `mode` | `auto` / `approval` | Publish directly, or ask a human on Telegram first |
| `publish_target` | `facebook` / `simulation` | Real post, or write `SIMULATED-<run_id>` as the ID |
| `fb_page_id` | Page ID | Test Page to post to |
| `telegram_chat_id` | chat ID | Where alerts and approvals go |
| `llm_model` | default `deepseek/deepseek-v4-pro-0813` | Writer model on OpenRouter (better wording, about a minute per run) |
| `llm_audit_model` | default `deepseek/deepseek-v4.1-flash` | Auditor model (fast; a different model from the writer on purpose) |
| `source_override` | empty / `articles` / `answers` / `news` | Force a content type |
| `run_id`, `run_at` | automatic | Execution ID and Malaysia-time timestamp |

## How to run

- **Manual test:** set `mode=auto`, `publish_target=simulation`, click **Manual run**. Inspect the node outputs, then check the new row in `pw_post_log`. Switch to `publish_target=facebook` for a real post on the test Page.
- **Approval test:** set `mode=approval`, run, then open the link in the Telegram message and choose **Approve** (or **Reject**) in the n8n form. Rejected or timed-out items are logged as rejected.
- **Failure test:** put an invalid token in the Facebook credential and run. The run logs a failed row, sends a Telegram alert naming the cause, and the execution is marked failed.

Failure classes: Facebook token expired or invalid (OAuthException / code 190), site unreachable, LLM API error, extraction failed.

## Logs and dashboard

Data Table `pw_post_log` columns: `run_id, run_at, mode, publish_target, source_type, url, title, post_text, image_url, fb_post_id, status, attempts, audit_verdict, error, updated_at`. Filter by `status` to see scheduled, published, rejected, blocked or failed items.

## Engagement read-back (stretch goal)

The workflow **PlacidWay Engagement Read-back** runs every 6 hours (or manually). It reads every `published` row of `pw_post_log`, asks the Facebook Graph API for reactions, comments and shares of that post, and upserts one row per post into the Data Table `pw_engagement` (`fb_post_id, url, title, reactions, comments, shares, status, checked_at`). A failed read is stored as `read_failed` instead of being hidden. It uses the same Page token (`pages_read_engagement`).

## Sample outputs

See [docs/samples.md](docs/samples.md): three simulated runs plus real posts on the test Page, with source URLs, post IDs and the failure test.

## Limitations and next steps

- Single instance on a laptop: the schedule only runs while n8n is running.
- The Telegram approval link opens the local n8n webhook, so the reviewer must be on the same machine or network in this demo. Production: host n8n with a public URL.
- A Data Table is fine for a demo; production would use Postgres.
- The LLM audit is a screening layer, not proof. Approval mode is recommended for medical content.
- The Facebook app is in development mode.
- Not implemented: Instagram and LinkedIn cross-posting (needs Meta app review). Engagement is read as plain counts; a next step would be Page insights (reach, clicks) and feeding the results back into topic selection.

## Files

- `workflows/placidway_daily_post.json`: main workflow
- `workflows/placidway_error_alerts.json`: error-alert workflow
- `workflows/placidway_engagement.json`: engagement read-back workflow
- `workflows/placidway_setup.json`: one-time Data Table creation
- `docs/samples.md`: sample posts with source URLs and Facebook post IDs
