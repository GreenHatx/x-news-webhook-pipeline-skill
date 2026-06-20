# Example News-to-X Setup

## Live Intent

Pipeline watches Hacker News RSS hourly, filters AI/security items, sends one unseen item to an OpenClaw webhook, and produces a chat approval draft for X. X auto-posting is off by default.

## Webhook

- Public base: https://your-gateway.example.com
- Hook path: /hooks/news-to-x
- Local endpoint: http://127.0.0.1:18789/hooks/news-to-x
- Auth: Authorization: Bearer <OpenClaw hooks token>
- Mapping ID: news-to-x-draft
- Mapping location: ~/.openclaw/openclaw.json
- Required mode: approval draft only unless the operator explicitly authorizes public auto-post.

Do not expose the raw token in chat.

## RSS Source

- Feed: https://hnrss.org/frontpage
- Poll interval: 3600 seconds
- LaunchAgent label: com.openclaw.news-to-x-hn
- Wrapper: automations/news-to-x/news-to-x-hn.sh
- Poller: automations/news-to-x/news-to-x-rss.py
- State: automations/news-to-x/state-hn.json

## Filter

Default keywords include:

- AI: ai, artificial intelligence, llm, language model, openai, anthropic, claude, gpt, gemini, mistral, llama, agent, rag, embedding, transformer, machine learning, deep learning, inference, training
- Security: security, cybersecurity, infosec, appsec, devsecops, vulnerability, exploit, malware, ransomware, phishing, breach, cve, zero-day, supply chain attack, authentication, authorization, encryption, privacy, threat, attack

Override for a one-off dry-run:

    NEWS_TO_X_KEYWORDS="ai,llm,agent,security,cve" automations/news-to-x/news-to-x-hn.sh --dry-run

## Known Lessons

- {{payload}} in OpenClaw hook messageTemplate rendered empty for this mapping. Use field variables such as {{title}}, {{url}}, {{summary}}, or {{payload.title}}.
- twitter post --json can appear quiet when run through shell wrappers but still write success JSON to redirected stdout. Check output files before retrying.
- After deleting a duplicate tweet, verify by fetching direct status URL and expecting 404.

## Last Known Good Checks

- Gateway health after template fix: OK.
- LaunchAgent verified with run interval = 3600 seconds.
- Record live webhook run IDs in private operational notes, not in the public skill.
