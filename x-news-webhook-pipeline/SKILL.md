---
name: x-news-webhook-pipeline
description: Build, maintain, debug, or extend an RSS/webhook-to-X content pipeline. Use when the user mentions X/Twitter posting automation, news/RSS/webhook feeds, Hacker News AI/security filters, chat approval drafts, X auto-post gating, or asks to add sources/filters/schedules for a news-to-X workflow.
---

# X News Webhook Pipeline

## Guardrail

Treat X/Twitter posting as a public external action.

- Default to approval-draft mode: generate a chat draft, then post only after the operator explicitly approves public posting.
- Do not enable direct auto-post unless the operator clearly asks for automatic public posting.
- Never print hook tokens or API secrets in chat. Use placeholders such as <OpenClaw hooks token>.
- If testing a real post, verify once and avoid retrying blindly. A silent CLI can still have posted.

## Current Setup

Read references/current-setup.md before changing a live pipeline.

Main files:

- automations/news-to-x/news-to-x-rss.py: RSS poller, filtering, duplicate state, webhook POST.
- automations/news-to-x/news-to-x-hn.sh: launchd wrapper.
- automations/news-to-x/com.openclaw.news-to-x-hn.plist: launchd schedule.
- automations/news-to-x/state-hn.json: seen URLs and last hook result.
- ~/.openclaw/openclaw.json: OpenClaw hook mapping news-to-x-draft.
- ~/Library/LaunchAgents/com.openclaw.news-to-x-hn.plist: active launchd copy.

## Workflow

1. Inspect current state:

    launchctl print "gui/$(id -u)/com.openclaw.news-to-x-hn" | rg 'state|runs|last exit code|run interval'
    sed -n '1,160p' automations/news-to-x/state-hn.json

2. Edit the poller or plist with apply_patch.

3. Validate locally:

    python3 -m py_compile automations/news-to-x/news-to-x-rss.py
    automations/news-to-x/news-to-x-hn.sh --dry-run

4. If the plist changed, reload launchd:

    PLIST_SRC="$PWD/automations/news-to-x/com.openclaw.news-to-x-hn.plist"
    PLIST_DST="$HOME/Library/LaunchAgents/com.openclaw.news-to-x-hn.plist"
    cp "$PLIST_SRC" "$PLIST_DST"
    chmod 644 "$PLIST_DST"
    launchctl bootout "gui/$(id -u)" "$PLIST_DST" >/dev/null 2>&1 || true
    launchctl bootstrap "gui/$(id -u)" "$PLIST_DST"
    launchctl enable "gui/$(id -u)/com.openclaw.news-to-x-hn"

5. If ~/.openclaw/openclaw.json hook mappings changed, restart gateway and verify:

    nohup sh -c 'sleep 2; openclaw gateway restart >> /tmp/openclaw-gateway-restart-news-to-x.log 2>&1' >/dev/null 2>&1 &
    sleep 8
    openclaw gateway health

## Hook Mapping Rules

OpenClaw hook templates resolve {{title}}, {{payload.title}}, {{url}}, and similar fields. They do not provide a whole-payload variable named {{payload}}.

Use field-by-field templates. Include a silence guard:

    Payload alanlari:
    - Baslik: {{title}}
    - Kaynak: {{source}}
    - URL: {{url}}
    - Ozet: {{summary}}
    - Tarih: {{published_at}}

    Eger Baslik veya URL bos ise kullaniciya mesaj atma; sadece NO_REPLY yaz.

## Adding Sources

For another RSS source:

1. Add feed URL support via NEWS_TO_X_RSS_FEED or create a new wrapper script.
2. Use a separate state file per source.
3. Keep a conservative keyword filter.
4. Start in dry-run, then one live webhook test.

## Troubleshooting

- Unexpected Telegram message saying payload is empty: inspect hook template; avoid {{payload}}.
- Duplicate X posts: check whether twitter post wrote success JSON to stdout before retrying.
- Repeated RSS items: inspect state-hn.json; ensure selected URL is inserted into seen.
- No messages: dry-run may show no_unseen_matching_items; that is fine.
- Hook 401: use Authorization: Bearer <token> or x-openclaw-token; query tokens are rejected.
