# Тестовое ИИ менеджер задач

## 1. Краткое описание проекта и реализованных функций
Проект: full-stack приложение управления задачами с AI-помощником.

Функции по ТЗ:
- US-1 CRUD:
  - `POST /api/v1/tasks`
  - `GET /api/v1/tasks`
  - `GET /api/v1/tasks/:id`
  - `PATCH /api/v1/tasks/:id`
  - `DELETE /api/v1/tasks/:id`
- US-2 фильтрация/поиск:
  - query-параметры: `status`, `priority`, `q`, `dueDateFrom`, `dueDateTo`, `isOverdue`, `page`, `pageSize`, `sortBy`, `sortOrder`
  - фильтрация выполняется на backend (Prisma/PostgreSQL), не на клиенте.
- US-3 LLM категоризация:
  - `POST /api/v1/tasks/:id/ai/category-suggestion` -> suggestion
  - apply через `PATCH /api/v1/tasks/:id` (ручное подтверждение).
- US-4 LLM декомпозиция:
  - `POST /api/v1/tasks/:id/ai/decompose` -> suggestion
  - apply через `POST /api/v1/tasks/:id/subtasks/bulk` (ручное подтверждение).
- US-5 LLM приоритет:
  - `POST /api/v1/tasks/:id/ai/priority-suggestion` -> suggestion
  - apply через `PATCH /api/v1/tasks/:id` (ручное подтверждение).
- US-6 LLM сводка нагрузки:
  - `GET /api/v1/ai/workload-summary?upcomingDays=3|7|14`.

AI-часть:
- Prompt layer по сценариям US-3..US-6.
- LangGraph runtime (`TaskAIStateGraph`) с pipeline:
  `load_context -> build_prompt -> invoke_model -> validate_schema -> normalize_result`.
- Safety: AJV schema validation, error mapping (`AI_INVALID_SCHEMA`, `AI_TIMEOUT`, `AI_UNAVAILABLE`).

## 2. Пошаговые инструкции настройки среды (зависимости, установка, переменные окружения)
Требования:
- Node.js >= 24
- Docker + Docker Compose (для PostgreSQL)

Шаги:
1. Клонировать репозиторий и перейти в папку проекта.
2. Установить зависимости:
```bash
npm install
```
3. Поднять PostgreSQL:
```bash
docker compose up -d
```
4. Создать `.env`:
```bash
cp .env.example .env
```
5. Применить миграции:
```bash
npm run db:migrate:deploy --workspace backend
```
6. (Опционально) проверить статус миграций:
```bash
npm run db:migrate:status --workspace backend
```

Переменные окружения:
- Обязательные:
  - `DATABASE_URL` (PostgreSQL connection string)
  - `PORT` (по умолчанию `3001`)
  - `NEXT_PUBLIC_API_BASE_URL` (по умолчанию `http://localhost:3001/api/v1`)
- Для AI runtime:
  - `AI_PROVIDER_DEFAULT=openai|mock`
  - `OPENAI_API_KEY` (обязателен только для live `openai`; если пустой, fallback на `mock`)
  - `AI_MODEL_PRIMARY` (по умолчанию `gpt-4`)
  - `AI_TIMEOUT_MS`, `AI_RETRY_COUNT`
  - `AI_MAX_OUTPUT_TOKENS_US3/US4/US5/US6`.

## 3. Инструкции запуска приложения (фронтенд и бэкенд)
Backend:
```bash
npm run dev:backend
```

Frontend:
```bash
npm run dev:frontend
```

Адреса:
- Backend API: `http://localhost:3001/api/v1`
- Frontend: `http://localhost:3000`

Проверка, что backend поднялся:
```bash
curl http://localhost:3001/api/v1/health
```

Локальные проверки качества:
```bash
npm run ci:local
```

## 4. Описание принятых архитектурных решений
- Монорепо (npm workspaces):
  - `backend/` — NestJS + Prisma
  - `frontend/` — Next.js (App Router).
- API-дизайн:
  - версия API через префикс `/api/v1`
  - единый формат ошибок (error envelope)
  - idempotency для AI apply-операций (`Idempotency-Key`).
- Данные:
  - PostgreSQL
  - Prisma migrations
  - сущность `Task` с parent-child связью для подзадач.
- Разделение ответственности:
  - `tasks` модуль — бизнес-логика задач
  - `ai` модуль — prompts, runtime, provider adapters, schema validation
  - `common` — middleware/interceptors/filters.
- AI runtime:
  - prompt profiles в `backend/src/ai/prompts/*`
  - orchestration в `backend/src/ai/runtime/task-ai-state.graph.ts`
  - adapter pattern для provider (`openai`/`mock`)
  - post-validation AI ответа через AJV.
- HITL (human-in-the-loop):
  - US-3/US-4/US-5 возвращают только предложения
  - фактическая мутация данных только через отдельный apply-запрос из UI.

## 5. Известные проблемы, ограничения или компромиссы
- Приложение в формате demo: single-user, без auth/roles/tenant isolation.
- Отсутствует production-контур эксплуатации: нет CI/CD окружений, секрет-менеджмента, централизованного мониторинга и SLA-процессов.
- Хранилище idempotency in-memory: после рестарта backend ключи не сохраняются (для production нужен Redis/PostgreSQL store).
- AI-оценка в текущей версии упрощенная: `scripts/eval-placeholder.mjs` дает synthetic-проверку, а не live-eval на реальных пользовательских запросах.
- AI-функции реализованы в синхронном request/response формате: нет streaming-ответов и фоновой очереди для долгих операций.
- Нет enterprise-безопасности и governance: отсутствуют audit trail действий пользователей, RBAC/policy enforcement и лимиты по организациям/командам.

## 6. Список функций, которые кандидат добавил бы при наличии дополнительного времени
- AuthN/AuthZ (JWT/OAuth2), multi-user и разграничение доступа к задачам.
- Перенос idempotency-key storage в Redis/PostgreSQL.
- Production live-eval + regression gates в CI/CD на реальных LLM-вызовах.
- Улучшение LLM-контура:
  - prompt versioning (версионирование промптов и rollback),
  - A/B тесты промптов по сценариям US-3..US-6,
  - retrieval-augmented prompts (подмешивание релевантного контекста задач/истории),
  - response streaming для долгих AI-операций.
- Улучшение prompt engineering:
  - отдельные prompt-профили по типам задач (работа/личное/учеба),
  - автоматические prompt-eval тесты в CI с quality/cost метриками,
  - fallback-стратегии при низкой уверенности модели.
- Интеграция с мессенджерами (Telegram / Max):
  - бот-команды для CRUD (`/new`, `/list`, `/done`, `/priority`),
  - AI-команды (`/categorize`, `/decompose`, `/workload`),
- webhook/long-polling gateway + привязка чата к пользователю,
- уведомления о дедлайнах и ежедневная сводка в мессенджер.
- Расширенная observability: метрики, алерты, трассировка AI pipeline.
- UI улучшения: канбан, календарь, пакетные операции, audit log/история изменений.

## 7. Production Readiness (дополнительно)

### Инфраструктура
- [ ] Есть отдельные окружения: `dev`, `staging`, `prod`.
- [ ] Все сервисы запускаются контейнерами (backend/frontend/db).
- [ ] Настроен reverse proxy (Nginx/Caddy) и HTTPS/TLS.

### Безопасность
- [ ] Секреты не хранятся в репозитории (используется secret manager / env в CI).
- [ ] Настроены CORS, security headers, rate limiting.
- [ ] Включен dependency/container scanning.

### База данных
- [ ] Миграции выполняются автоматически в релиз-пайплайне.
- [ ] Настроены регулярные backup’ы PostgreSQL.
- [ ] Проверена процедура восстановления (restore test).

### Приложение
- [ ] Реализованы auth + RBAC (не single-user).
- [ ] Idempotency storage вынесен в Redis/PostgreSQL (не in-memory).
- [ ] Долгие AI-операции вынесены в очередь (worker).

### AI-контур
- [ ] Prompt versioning + rollback стратегия.
- [ ] Live-eval (не synthetic placeholder) на реальных кейсах.
- [ ] Cost/quality guardrails и алерты по деградации.

### Наблюдаемость
- [ ] Централизованные логи (с requestId/correlationId).
- [ ] Метрики и дашборды (latency/error rate/cost).
- [ ] Alerting по SLO (5xx, timeout, AI unavailable).

### Релизы и эксплуатация
- [ ] CI/CD: lint + tests + build + deploy + smoke tests.
- [ ] Есть rollback-процедура (версия приложения + миграции).
- [ ] Описан runbook инцидентов.
