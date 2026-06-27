# REST API деплой

Деплой и правка воркфлоу через REST Public API self-hosted n8n.

> Плейсхолдеры: `<N8N_BASE_URL>` — домен вашего инстанса; `<workflow_id>` — id воркфлоу;
> `<ERROR_WORKFLOW_ID>` — id общего Error Workflow; `<*_CRED_ID>` — id credential'ов.

---

## ✅ Решения

### DEP-Р1. Два типа JWT-токенов — не путать

На сервере могут быть два разных токена для двух разных API. Перепутаешь — 401 или 404.

| Тип | Audience | Header | Endpoint | Для чего |
|---|---|---|---|---|
| MCP Server API | `mcp-server-api` | `Authorization: Bearer <token>` | `/mcp-server/http` | SDK / MCP-операции |
| REST Public API | `public-api` | `X-N8N-API-KEY: <token>` | `/api/v1/...` | GET/PUT workflows, executions |

Статус: ✅ Подтверждено в проде.

### DEP-Р2. Порядок деплоя через PUT без потери credentials

**Контекст:** `PUT /api/v1/workflows/{id}` полностью заменяет все ноды. GET **не возвращает**
поле `credentials` (хранится в БД отдельно). Не добавишь credentials вручную перед PUT —
все привязки слетят.

**Обязательный порядок:**
1. `GET /api/v1/workflows/{id}` — получить актуальный JSON.
2. Хирургическая правка в Python, **сохранить все node ID** из исходника.
3. Добавить `credentials` в каждую ноду по типу (credential map ниже).
4. `PUT` с минимальными settings (см. DEP-Р3).
5. `GET /api/v1/executions?workflowId={id}&limit=3` — проверить, что нет ошибок credentials.

**Шаблон credential map** (подставь свои id и имена под типы нод, которые используешь):

```python
telegram_types = ["telegramTrigger", "telegram"]
postgres_types = ["postgres", "memoryPostgresChat"]
openai_types   = ["lmChatOpenAi", "n8n-nodes-base.openAi", "@n8n/n8n-nodes-langchain.openAi"]

CREDS = {
    "telegram":  {"telegramApi": {"id": "<TELEGRAM_CRED_ID>", "name": "<имя credential>"}},
    "postgres":  {"postgres":    {"id": "<POSTGRES_CRED_ID>", "name": "<имя credential>"}},
    "openai":    {"openAiApi":   {"id": "<OPENAI_CRED_ID>",   "name": "<имя credential>"}},
}
# Если разные воркфлоу используют разных ботов/БД — держи отдельную запись на каждый
# credential и подставляй нужный по контексту воркфлоу.
```

Узнать реальные id credential'ов: `GET /api/v1/credentials` (или из UI). Тип credential
(`telegramApi`, `postgres`, `openAiApi`) должен совпадать с тем, что ждёт нода.

Статус: ✅ Подтверждено в проде.

### DEP-Р3. Минимальные settings для PUT

При PUT убирай нестандартные поля settings (`availableInMCP`, `binaryMode`, `timeSavedMode`
и т.п.) — иначе валидация ругается (см. DEP-О7). Передавай только базовый набор:

```json
{
  "executionOrder": "v1",
  "callerPolicy": "workflowsFromSameOwner",
  "errorWorkflow": "<ERROR_WORKFLOW_ID>"
}
```

Сервер сохраняет существующие дополнительные settings сам — после PUT GET снова покажет
`binaryMode` и т.п. Терять их не страшно.

Статус: ✅ Подтверждено.

### DEP-Р4. ID нового воркфлоу — брать ТОЛЬКО из ответа сервера

При `POST /api/v1/workflows` (создание) сервер возвращает реальный `id`. Его и только его
использовать во всех последующих вызовах (activate / PUT / executions / delete).

**Правило:** не угадывать, не «помнить», не подставлять похожий по виду id. Сразу после POST
извлекать `id` из JSON-ответа в переменную. После создания/правки — сверяться со списком по
**имени**: `GET /api/v1/workflows?limit=200` → найти свой `name`.

Связано с DEP-О5. Статус: ✅ Правило.

### DEP-Р6. Ротация секрета сервиса — читать из service_tokens в рантайме, не самообновлять credential

- **Контекст:** долгоживущий токен внешнего сервиса (OAuth/Bearer, Meta/IG и т.п.)
  периодически продлевается воркфлоу. Соблазн — после продления записать его в n8n-credential
  через `PATCH /api/v1/credentials/{id}` (DEP-Р5), чтобы HTTP-ноды тянули из credential.
  На практике этот PATCH в проде падает (`invalid syntax`, см. DEP-Р5) — публичный API
  не предназначен для записи секретов credential.
- **Решение (надёжное):** **credential для ротируемого токена не использовать вообще.**
  Источник правды — таблица-зеркало `service_tokens(service, token, expires_at)`.
  1. Воркфлоу продления: `refresh → UPDATE service_tokens` (и всё, без узла обновления
     credential и без алёрта о его сбое).
  2. Воркфлоу-потребитель: в начало ветки — Postgres-нода
     `SELECT token FROM service_tokens WHERE service='<svc>'`; HTTP-ноды переводятся с
     `authentication: genericCredentialType` на `none` + ручной заголовок `Authorization`
     = `={{ "Bearer " + $('<нода-токен>').first().json.token }}`. Узлы, что читают вход по
     имени (`$('<нода-входа>')...`), от вставки токен-ноды в линию не ломаются — данные
     тянутся по имени, не из `$json`.
- **Почему лучше:** нет хрупкого PATCH, нет ручных обновлений UI, нет ложных алёртов;
  ротация = просто запись в БД, потребитель всегда берёт свежее.
- **Деплой:** менять активный суб-воркфлоу через REST PUT (DEP-Р2/Р3), сняв бэкап JSON;
  токен-нода = клон существующей Postgres-ноды (тот же credential). Проверка без публикации:
  дёрнуть read-эндпоинт сервиса токеном из `service_tokens` (200 = токен валиден); затем
  один реальный прогон.
- **Статус:** ✅ Подтверждено на практике (реальная публикация прошла).

---

## ⚠️ Ошибки

### DEP-О1. `Node does not have any credentials set` на первой ноде после PUT

- **Симптом:** сразу после PUT воркфлоу падает на первой же ноде.
- **Причина:** PUT перезаписал ноды без поля `credentials` (GET его не отдаёт).
- **Лечение:** повторить PUT, добавив credentials в каждую ноду по credential map (DEP-Р2).

### DEP-О2. SDK `update_workflow` через MCP ломает воркфлоу при топологических изменениях

- **Симптом:** после правки через MCP `update_workflow` слетают credentials и/или меняется
  поведение нод.
- **Причина:** перегенерирует node IDs → теряет привязку credentials; автоповышает
  typeVersion нод (Set 3.2→3.4, Merge 3.1→3.2) → меняет поведение.
- **Как избежать:** при любых топологических изменениях деплоить через REST PUT (DEP-Р2),
  а не через SDK `update_workflow`.

### DEP-О3. Запрос к n8n из изолированного браузерного контекста → 401

- **Симптом:** `fetch()` к n8n API из расширения браузера возвращает 401, хотя в самом
  браузере сессия активна.
- **Причина:** расширение работает в изолированном контексте и не шарит сессию n8n.
- **Как избежать:** не использовать браузерный `fetch()` для операций с n8n API — только
  REST API с токеном.

### DEP-О4. n8n молча выбрасывает неизвестные ключи parameters

- **Симптом:** параметр ноды передан через API, но поведение не изменилось.
- **Причина:** n8n при сохранении молча отбрасывает неизвестные/неверно названные ключи
  `parameters`, не выдавая ошибку. Частая ловушка — camelCase вместо snake_case
  (`callbackData` вместо `callback_data`), либо неверное имя параметра.
- **Как избежать:** после PUT обязательно делать **GET-сверку** — проверять, что параметр
  реально сохранился в JSON. Сверять точные имена (см. примеры в telegram.md: `file` вместо
  `photo`, parse_mode в snake_case).

### DEP-О5. Каскад 404 «You do not have permission to ... this workflow»

- **Симптом:** после создания воркфлоу все вызовы activate/PUT/delete/GET возвращают
  `HTTP 404` с текстом про права («Ask the owner to share it with you») — выглядит как
  проблема доступа.
- **Причина (реальная):** обращение идёт к **несуществующему id** — id был выдуман или
  перепутан, а не взят из ответа POST. n8n на чужой/несуществующий id отвечает 404 с
  формулировкой про права, что путает.
- **Лечение:** взять настоящий id из ответа на POST (или из `GET /workflows?limit=200` по
  `name`). См. DEP-Р4.

### DEP-О6. После PUT воркфлоу остаётся деактивированным; немедленный `/activate` отдаёт 400

- **Симптом:** `PUT` на активном воркфлоу публикует новую версию, в publishHistory виден
  `deactivated`. Сразу следом `POST /activate` иногда возвращает **HTTP 400** и НЕ
  активирует — воркфлоу остаётся выключенным (бот «молчит»).
- **Причина:** гонка — PUT сам инициирует деактивацию/активацию, и параллельный `/activate`
  прилетает в неудачный момент.
- **Лечение:** после PUT **не доверять одному `/activate`** — сделать GET, проверить `active`,
  и если `false` — повторить `/activate` с паузой (ретрай-цикл до `active:true`).
- **Профилактика:** в скрипте деплоя — обязательный пост-чек активности с ретраем (2–3
  попытки, пауза ~2 сек), отдельно от PUT.

### DEP-О7. PUT отвергает «лишние» ключи settings (400), но живой воркфлоу их сохраняет

- **Симптом:** `PUT` → **HTTP 400** `"request/body/settings must NOT have additional properties"`.
- **Причина:** схема Public API принимает ограниченный набор `settings` (`executionOrder`,
  `errorWorkflow`, `callerPolicy`, `saveManualExecutions`, `timezone`,
  `saveExecutionProgress`, …). Ключи, проставленные UI (`binaryMode`, `timeSavedMode`,
  `availableInMCP`), GET отдаёт, но PUT с ними падает.
- **Лечение:** в PUT передавать только минимальные settings (DEP-Р3). Сервер сохраняет
  существующие дополнительные settings сам.

### DEP-О8. PUT основного воркфлоу падает 400, пока суб-воркфлоу не «published»

- **Симптом:** `PUT /api/v1/workflows/{id}` с нодой Execute Workflow возвращает
  `400: Node "..." references workflow ... which is not published. Please publish all
  referenced sub-workflows first.`
- **Причина:** n8n требует, чтобы суб-воркфлоу, на который ссылается активный воркфлоу, был
  опубликован (activate) до деплоя ссылающегося.
- **Лечение:** порядок деплоя связки: POST суб → `POST /workflows/{subId}/activate` →
  PUT основного. Суб-воркфлоу с Execute Workflow Trigger активируется без вебхуков — это
  безопасно.
- **Статус:** ⚠️ Подтверждено на практике.

---

## 🟡 Хрупкие / нерекомендуемые пути

### DEP-Р5. PATCH /api/v1/credentials/{id} — самообновление credential (нестабильно в проде)

- **Паттерн:** на n8n существует `PATCH /api/v1/credentials/{id}` с телом
  `{"data": {...полные данные credential...}}` — теоретически позволяет воркфлоу
  самообновлять секреты (ротация токенов) без ручного входа в UI.
- **Нюансы:** GET данные credential не отдаёт — текущее значение надо хранить в зеркале
  (таблица `service_tokens`); для httpHeaderAuth в data передавать `name`, `value` и
  `allowedHttpRequestDomains` (`"all"` либо `"domains"`+`allowedDomains` — схема create
  требует согласованности); PATCH с пустым `{}` безвреден (200, данные не трогает).
- **Безопасность:** ключ Public API для таких вызовов хранить отдельным credential
  (httpHeaderAuth `X-N8N-API-KEY`), ограниченным доменом инстанса.
- **⚠️ Прод-опыт — паттерн ненадёжен, заменён.** В ручном тесте PATCH проходил, но первый же
  автопрогон по расписанию упал: узел обновления credential ушёл в error-выход с
  `error: "invalid syntax"` (воспроизводимость в проде не подтвердилась; подбирать рабочее
  тело на боевом секрете нельзя). **Правильное решение — НЕ самообновлять credential, а
  читать секрет из `service_tokens` в рантайме: см. DEP-Р6.** Запись оставлена как
  «известно-хрупкий» путь.
- **Статус:** 🟡 Проходит в ручном тесте, но в проде нестабилен → вытеснен DEP-Р6.
