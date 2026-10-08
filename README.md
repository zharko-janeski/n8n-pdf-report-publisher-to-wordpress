
# n8n PDF Report → WordPress Publisher

Drop a financial report PDF into a Google Drive folder → Gemini extracts it into
structured JSON → n8n creates a WordPress draft + sends a Telegram bulletin.
Failures alert admin Telegram via a two-layer error system.

![n8n workflow canvas](canvas.png)

## Demo

| WP Admin — auto-created draft | Rendered blog post |
|---|---|
| ![WP admin draft](admin-wp.png) | ![Blog post](wp.png) |

## Pipeline

```
Drive folder → PDF check → Download → Base64 → Gemini (responseSchema JSON)
    → Validate → WP draft post → Telegram bulletin

Layer 1: inline error branches (Gemini / WP)  → admin alert, flow stops clean
Layer 2: dedicated Error Handler workflow     → admin alert on ANY unhandled crash
```

## What it does

- **Drive trigger** — watches a folder, admits `application/pdf` only
- **Gemini structured extraction** — PDF as `inlineData`, output enforced by
  API-level `responseSchema`: title, ticker, broker, report type, semantic HTML
  article (H2/H3, financial tables, key takeaways), Telegram bulletin < 2000 chars
- **WordPress REST** — DRAFT post via Application Passwords (Basic Auth),
  theme-independent, returns post ID + preview link
- **Telegram** — sanitized HTML bulletin + draft preview link
- **Two-layer error handling** — inline branches for known-risk nodes + separate
  Error Handler workflow (Settings → Error Workflow) for everything unhandled.
  Every production n8n workflow should ship with both.

## Setup

1. Import `error-handler.json`, then `main-publisher.json`
2. Create credentials (n8n credential store — no hardcoded keys):
   - Google Drive OAuth2
   - Header Auth → name `x-goog-api-key`, value = Gemini API key
   - Basic Auth → WP username + Application Password
   - Telegram bot token
3. Edit the **Pipeline Config** node:

   | Field | Purpose |
   |---|---|
   | `wpBaseUrl` | WordPress site, no trailing slash |
   | `wpCategoryIds` | optional, comma-separated category IDs |
   | `geminiModel` | model name (see note below) |
   | `tgChannelId` | bulletin destination |
   | `adminChatId` | where failure alerts go |
   | `geminiSchema` / `geminiPrompt` | the extraction contract — tune here, not in the HTTP node |

4. Main workflow → Settings → **Error Workflow** → select the Error Handler
5. Self-hosted: `N8N_DEFAULT_BINARY_DATA_MODE=filesystem`
6. Activate → drop a fresh PDF in the folder

## Gemini model note

Google deprecated Gemini 2.5 for new API keys. The model is one config field —
tested on `gemini-3.8-flash` (identical structured-output support); grandfathered
keys can flip back to `gemini-2.5-pro`. Zero code changes.

## Roadmap (v2)

- [ ] AI-generated header image → WP featured media
- [ ] `wpPostStatus` config field (draft / publish)
- [ ] Social fan-out: LinkedIn + X (caption + image)
- [ ] Processed-files ledger (exactly-once posting)
