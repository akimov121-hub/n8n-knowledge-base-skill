# n8n knowledge base — a Claude skill

A portable, installation-agnostic card-file of **battle-tested solutions, common
pitfalls, and working methodology for self-hosted n8n** — packaged as a
[Claude](https://claude.com/claude-code) skill.

It turns hard-won experience (deploying via REST API, Postgres quirks, Telegram bot
gotchas, AI-agent patterns, debugging) into a symptom-indexed reference an AI assistant
can consult **before** a risky action instead of re-learning the same mistakes.

> 🌍 **Bilingual.** Every reference exists in English (`references/en/`) and Russian
> (`references/ru/`). The skill itself is English-first; the Russian set is a full mirror.

---

## What's inside

- **`SKILL.md`** — the entry point: a **symptom index** (errors ⚠️ and solutions ✅)
  that routes you to the exact entry, plus a short methodology section.
- **`references/en/`** & **`references/ru/`** — the full card-file, one file per domain:

| File | Domain |
|---|---|
| `api-setup.md` | Connecting an AI assistant to n8n via the REST API (**start here**) |
| `rest-api-deploy.md` | Deploying workflows via REST API, JWT tokens, PUT pitfalls |
| `postgres.md` | Postgres nodes, SQL, JSONB, Chat Memory, idempotency |
| `telegram.md` | Telegram bots: messages, photos, buttons, voice, Mini Apps, Login Widget |
| `set-transforms.md` | Edit Fields (Set), Execute Workflow, item-loss pitfalls |
| `ai-agents.md` | AI Agent: model choice, memory, RAG without vectors, tool workflows |
| `debugging-sandbox.md` | Debugging order, sandbox benches, Code-node sandbox limits |
| `instagram.md` | Publishing to Instagram via the Graph API |
| `methodology.md` | Project structure, naming, architecture principles, deploy protocol |

Every entry has a stable code (`DEP-R2`, `TG-E1`, …): domain prefix + type
(`R` = solution, `E` = error) + number. Codes are identical across both languages.

## How to use it

### With Claude Code (as a skill)

1. Copy the `n8n-knowledge-base/` folder into your skills directory (e.g.
   `~/.claude/skills/` or your plugin's `skills/`).
2. Claude auto-discovers it via `SKILL.md`. When you work on n8n, Claude consults the
   symptom index and opens only the relevant reference.

The packaged `n8n-knowledge-base.skill` (a zip) is also provided for one-shot import.

### As plain documentation

You don't need Claude — `SKILL.md` + `references/` read perfectly well as a standalone
n8n troubleshooting handbook. Start from the symptom index in `SKILL.md`.

## Important: this is a template

The skill is **not tied to any specific n8n installation**. Everywhere you see a
`<placeholder>` — `<N8N_BASE_URL>`, `<workflow_id>`, `<BOT_TOKEN>`, `<telegram_id>`,
`<POSTGRES_CRED_ID>`, … — substitute your own value. No real domains, tokens, IDs or
credentials are included.

## Contributing

Found a new pitfall or a better fix? Add an entry to the relevant domain file (both
`en/` and `ru/` if you can), add a row to the symptom index in `SKILL.md`, and open a PR.
Record solutions only once they're confirmed to work; record errors as soon as you hit them.

## License

[MIT](LICENSE) © Anton Akimov

---

# База знаний n8n — скилл для Claude

Переносимая, не привязанная к конкретной инсталляции картотека **проверенных решений,
типовых граблей и методологии работы с self-hosted n8n** — в виде скилла для
[Claude](https://claude.com/claude-code).

Превращает накопленный опыт (деплой через REST API, особенности Postgres, грабли
Telegram-ботов, паттерны AI-агентов, отладка) в справочник с указателем по симптомам,
к которому ассистент обращается **перед** рискованным действием, а не наступает на те же
грабли заново.

> 🌍 **Двуязычно.** Каждый reference есть на английском (`references/en/`) и русском
> (`references/ru/`). Сам скилл — English-first; русский набор — полная копия.

## Что внутри

- **`SKILL.md`** — точка входа: **указатель по симптомам** (ошибки ⚠️ и решения ✅),
  ведущий к нужной записи, плюс краткая методология.
- **`references/en/`** и **`references/ru/`** — полная картотека, по файлу на домен
  (REST API деплой, Postgres, Telegram, Set, AI-агенты, отладка, Instagram, методология).

У каждой записи стабильный код (`DEP-R2`, `TG-E1`, …): префикс домена + тип
(`R` — решение, `E` — ошибка) + номер. Коды одинаковы в обоих языках.

## Как пользоваться

**С Claude Code (как скилл):** скопируй папку `n8n-knowledge-base/` в каталог скиллов
(например `~/.claude/skills/`). Claude сам найдёт её по `SKILL.md` и при работе с n8n
будет открывать только нужный reference. Для разового импорта есть собранный
`n8n-knowledge-base.skill` (zip).

**Как обычная документация:** `SKILL.md` + `references/` читаются и без Claude — как
самостоятельный справочник по n8n. Начинай с указателя по симптомам в `SKILL.md`.

## Важно: это шаблон

Скилл **не привязан к конкретной инсталляции**. Везде, где встречается `<плейсхолдер>`
(`<N8N_BASE_URL>`, `<workflow_id>`, `<BOT_TOKEN>`, `<telegram_id>` …) — подставь своё
значение. Реальных доменов, токенов, ID и credential'ов в репозитории нет.

## Лицензия

[MIT](LICENSE) © Anton Akimov
