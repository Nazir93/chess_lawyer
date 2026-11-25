# 0001. Стек и архитектура: TypeScript модульный монолит на NestJS, PostgreSQL, Valkey, Docker на VPS

**Date**: 2026-11-25
**Status**: Proposed

## Summary

Весь проект пишем на одном языке, TypeScript: клиент (Mini App и PWA), админка, сервер, бот и фоновые задачи живут в одном репозитории. Сервер это одно приложение NestJS, разделённое на модули по областям ТЗ, которое запускается в двух ролях: `api` (запросы) и `worker` (очередь). Данные в PostgreSQL, очередь и счётчики в Valkey (совместим с Redis), всё в Docker на двух VPS в РФ: production и отдельный staging. Права, атомарная запись, надёжная доставка уведомлений, наблюдаемость и бэкапы заложены с первого дня; ограничение одного production сервера записано честно и вынесено заказчику (Q31).

## Requirements

**User stories**:
- Как разработчик, я хочу один язык и общие схемы данных для клиента и сервера, чтобы ошибки на стыке API ловились при сборке, а не у пользователя.
- Как администратор клуба, я хочу, чтобы система переживала падение одного процесса и восстанавливалась из копии по NFR07, чтобы клуб не терял записи и рейтинг.
- Как заказчик, я хочу развернуть систему с нуля на любом VPS в РФ по инструкции, без привязки к одному облаку.

**Acceptance criteria**:
- **AC-1**: `pnpm dev:up` поднимает локально PostgreSQL, Valkey, `server` в ролях `api` и `worker`, `web` и `admin` через Docker Compose (`infra/compose/compose.local.yml`); команда завершается, когда все сервисы в состоянии healthy (таймаут 120 с). `GET /live` отвечает 200 без обращения к зависимостям, `GET /ready` отвечает 200 только при доступной PostgreSQL.
- **AC-2**: Контракт API задаётся только Zod схемами в `packages/contracts`. Из них генерируется `packages/contracts/openapi.json` (команда `pnpm contracts:openapi`), файл хранится в репозитории; CI падает, если сгенерированный файл отличается от сохранённого или если клиент не собирается против текущих схем.
- **AC-3**: `server` не хранит состояние в памяти процесса: сессии в PostgreSQL, очередь, rate limit и блокировки в Valkey или PostgreSQL. На production работают 2 экземпляра `api` и 2 экземпляра `worker`; остановка любого одного не прерывает вход, запись и обработку очереди (NFR12 на уровне процесса).
- **AC-4**: Модули сервера обращаются друг к другу только через публичный файл модуля; `dependency-cruiser` в CI запрещает импорт внутренностей чужого модуля, импорт `domain` слоя в зависимость от NestJS или БД и импорт серверного кода из клиентских приложений (NFR10).
- **AC-5**: В репозитории нет секретов открытым текстом: только `infra/secrets/*.sops.env` (зашифровано) и `.env.example`. `gitleaks` в CI и в pre-commit; секреты не попадают в слои Docker образов, в клиентскую сборку и в логи (NFR14).
- **AC-6**: Образы собираются один раз на коммит и продвигаются по digest: staging получает образ автоматически, production тот же digest после ручного подтверждения в GitHub Actions (NFR11).
- **AC-7**: Логи в JSON в stdout с `requestId`, без ПДн и секретов (список скрываемых полей в конфигурации логгера); ошибки сервера и клиента попадают в GlitchTip на production VPS; Uptime Kuma на staging VPS оповещает в Telegram чат администраторов о недоступности production, об отсутствии успешного бэкапа за сутки и о застрявших задачах очереди (NFR08).
- **AC-8**: Бэкап PostgreSQL вне сервера в S3 хранилище в РФ: полная копия ежедневно в 03:00 МСК, архив журналов не реже раза в 60 с, хранение 14 дней, шифрование на стороне WAL-G; восстановление на момент времени описано в runbook и проверяется раз в месяц (NFR07).

## Decision

**Chosen option**: Option 1: TypeScript модульный монолит на NestJS 12 с первыми пакетами Nest (`@nestjs/drizzle`, `@nestjs/authentication`, `@nestjs/authorization`, `@nestjs/bullmq`, `@nestjs/outbox`, `@nestjs/terminus`, `@nestjs/throttler`, `@nestjs/config`), PostgreSQL, Valkey, React клиенты, Docker Compose на двух VPS.

Один репозиторий pnpm + Turborepo; одно серверное приложение с двумя точками входа; развёртывание контейнерами без облачных управляемых сервисов.

**Implementation skills**: `nestjs` · `nestjs-auth` · `nestjs-data` · `nestjs-feature` · `nestjs-production` · `nestjs-testing` (`nestjs/agent-skills`, `.agents/skills/<name>/`) · `telegram-miniapps` · `telegram-bots` (`ssheleg/telegram-dev`, `.agents/skills/<name>/`) · `tdd` · `domain-modeling` (`mattpocock/skills`, `.agents/skills/<name>/`)

## Rationale

Обоснование и варианты: см. [rationale.md](./rationale.md).

## Proposed stack

Версии фиксируются точно в момент создания каркаса (`packageManager` в `package.json`, `.nvmrc`, теги образов с digest); ниже линии версий.

| Layer | Choice | Reason |
|---|---|---|
| Язык | TypeScript strict, ESM | Общие типы и схемы клиента и сервера; проекты Nest 12 работают в ESM |
| Среда выполнения | Node.js 24 LTS | Активная LTS на конец 2026 |
| Менеджер пакетов и монорепо | pnpm 10 workspaces + Turborepo | Общие пакеты без публикации, кэш сборки и тестов в CI |
| Архитектурный шаблон | Модульный монолит: `apps/server` с точками входа `main.api.ts` и `main.worker.ts`, один образ | Масштаб до 1 000 членов не оправдывает микросервисы; роли масштабируются отдельно, код общий |
| Серверный фреймворк | NestJS 12 на `@nestjs/platform-node` | Модули под области ТЗ; адаптер Node быстрее Express и совместим с сессиями и CSRF защитой первых пакетов Nest |
| Валидация и контракты | Zod 4 в `packages/contracts`, `@Body({ schema })` в Nest, OpenAPI через `@nestjs/swagger` | Одна схема даёт проверку входа, типы клиента и OpenAPI без расхождений |
| Конфигурация | `@nestjs/config` с проверкой Zod при старте | Нет нужной переменной: сервер не стартует |
| Основная БД | PostgreSQL 17 | ACID, уникальность и вместимость держит сама БД, частичные индексы, `SKIP LOCKED` |
| Работа с БД | `@nestjs/drizzle` + Drizzle ORM v1, миграции `drizzle-kit` | SQL остаётся видимым: условные `UPDATE`, транзакции, блокировки; маппинг `23505` в 409 |
| Очередь | `@nestjs/bullmq` + BullMQ на Valkey 8 | Отложенные задачи, повторы, лимит скорости отправки в Telegram |
| Надёжная доставка | `@nestjs/outbox`: запись события в той же транзакции, публикация в очередь отдельно | Событие не теряется между БД и очередью; доставка минимум один раз, обработчики идемпотентны |
| Вход и сессии | `@nestjs/authentication`: `SessionCookieProvider`, хранилище сессий в PostgreSQL, свой `AuthenticationProvider` для проверки Telegram initData и Telegram Login; TOTP пакета для 2FA админки | Проверенная реализация сессий, CSRF и TOTP вместо своей; Telegram проверку пишем по скиллу `telegram-miniapps` |
| Права | `@nestjs/authorization`: политики по матрице ТЗ 3.2, маршрут без проверки запрещён | Запрет по умолчанию встроен в пакет |
| Rate limit | `@nestjs/throttler` с хранилищем в Valkey, после guard входа | Общие счётчики для нескольких экземпляров, лимит на пользователя, а не на IP прокси |
| Здоровье | `@nestjs/terminus`: `/live`, `/ready`, мягкое завершение | Caddy выводит экземпляр из ротации до остановки |
| Плановые задачи | `@nestjs/schedule` + `@nestjs/locks` | Задача по расписанию выполняется одним экземпляром |
| Клиент (Mini App и PWA) | React 19, Vite, TanStack Router, TanStack Query; Telegram через официальный `telegram-web-app.js` за слоем платформы, обёртка `@telegram-apps/sdk-react` (библиотека сообщества) допустима только внутри слоя платформы | Быстрая сборка, контроль кэша данных, замена обёртки не затрагивает экраны |
| PWA | `vite-plugin-pwa` (Workbox), кэш только оболочки | Ответы API и ПДн не кэшируются (ТЗ §11); детали в спеке фичи 29 |
| Админка | Отдельное приложение React + Vite, TanStack Table | Тот же стек, свой хост и cookie |
| UI и стили | Tailwind CSS v4 с токенами Modern Heritage, Radix primitives в `packages/ui` | Одна дизайн система, доступность из коробки |
| Переводы | i18next в `packages/i18n` | Русский сейчас, английский без перестройки |
| Бот | grammY: webhook в `api`, отправка из `worker` | Зрелая библиотека для TypeScript |
| Web Push | `web-push` с VAPID | Без сторонних сервисов |
| Файлы | S3 совместимое хранилище в РФ у VPS провайдера (запасной вариант MinIO), `sharp` в `worker` | Файлы не в БД; детали в спеке фичи 31 |
| Обратный прокси и TLS | Caddy 2 | Автоматические сертификаты, заголовки, балансировка с активной проверкой здоровья |
| Хостинг | 2 VPS в РФ, провайдер позже (Assumed, Q17) | Переносимо; staging изолирован от данных production |
| Развёртывание | GitHub Actions + GHCR (приватный реестр), Docker Compose, скрипт выката по SSH | Один сценарий для обоих окружений без оркестратора |
| Секреты | SOPS + age | Нет облачного хранилища секретов на VPS |
| Резервные копии | WAL-G в S3 в РФ | NFR07 с восстановлением на момент времени |
| Ошибки | GlitchTip на production VPS (внутренняя сеть, доступ через `admin.` по VPN или SSH туннелю) | Данные об ошибках остаются на production и в РФ |
| Доступность и оповещения | Uptime Kuma на staging VPS, оповещения в Telegram чат администраторов | Взгляд на production снаружи без передачи данных |
| Тесты | Vitest; Testcontainers с PostgreSQL и Valkey; Playwright; k6 | Права, запись и рейтинг на настоящей БД; нагрузка в фиче 27 |
| Качество кода | ESLint 9 flat config, Prettier, `tsc --noEmit`, `dependency-cruiser`, `gitleaks`, Husky + lint-staged | AC-4, AC-5 и единый стиль до коммита |

## Architecture

**Структура репозитория**

```
apps/
  server/     NestJS: src/main.api.ts, src/main.worker.ts, src/modules/*
  web/        Mini App и PWA (слой платформы в src/platform/telegram и src/platform/pwa)
  admin/      админка
packages/
  contracts/  Zod схемы, коды ошибок, openapi.json
  ui/         дизайн система
  i18n/       словари
  config/     общие настройки ESLint, TypeScript, Tailwind
infra/
  compose/    compose.local.yml, compose.staging.yml, compose.production.yml
  caddy/      Caddyfile по окружениям
  secrets/    *.sops.env
  scripts/    deploy.sh, migrate.sh, restore.sh
docs/
```

**Модули сервера** (`apps/server/src/modules/`): `auth` · `access` · `invites` · `applications` · `membership` · `profiles` · `venues` · `files` · `events` · `tournaments` · `rating` · `notifications` · `content` · `audit` · `settings` (значения BR) · `admin`.

Каждый модуль делится на `http` (контроллеры, только в роли `api`), `application` (сценарии), `domain` (правила без NestJS и БД), `infra` (Drizzle, клиенты). Наружу модуль отдаёт только `index.ts`. Роль `worker` подключает те же модули без `http` и с обработчиками очередей.

**Маршруты** (домены Assumed до Q21)

| Хост и путь | Куда | Примечание |
|---|---|---|
| `app.<домен>/` | статика `web` | SPA, fallback на `index.html` |
| `app.<домен>/api/v1/*` | `api` (префикс `/api` снимается) | cookie `__Host-lcc_app` |
| `admin.<домен>/` | статика `admin` | |
| `admin.<домен>/api/v1/admin/*` | `api` | cookie `__Host-lcc_admin`; другие пути API с этого хоста запрещены |
| `app.<домен>/api/v1/telegram/webhook` | `api` | только с заголовком секрета |
| `/live`, `/ready` | `api` | только из внутренней сети и для Caddy, снаружи 404 |
| Staging | `staging.app.<домен>`, `staging.admin.<домен>` | отдельный бот |

Cookie обеих сессий: префикс `__Host-`, `Path=/`, без `Domain`, `Secure`, `HttpOnly`; `SameSite=Lax` для `app.`, `SameSite=Strict` для `admin.`. Сроки и продление по BR05 из модуля `settings`. Если первая нить (фича 6) покажет, что Telegram WebView на какой то платформе не держит cookie, клиент `miniapp` переходит на токен сессии в памяти и заголовок `Authorization`; решение и тест в спеке фичи 6.

**Процессы и надёжность**
- `api`: проверка сессии, прав и входа; запись в БД и строки outbox в одной транзакции; внутри транзакции нет сетевых вызовов.
- `worker`: публикует outbox в BullMQ (номер строки outbox как `jobId`, поэтому повторная публикация не создаёт дубль) и выполняет задачи. Политика по умолчанию: 5 попыток, пауза 10 с с удвоением, `removeOnComplete` 1 000 последних, `removeOnFail` 7 дней; упавшие после всех попыток задачи остаются в списке failed и вызывают оповещение. Обработчик каждой задачи идемпотентен: перед внешним действием проверяет отметку выполнения в БД.
- Webhook Telegram: проверяет `X-Telegram-Bot-Api-Secret-Token`, вставляет `update_id` в таблицу `telegram_updates` с уникальным ключом (повтор отбрасывается) вместе со строкой outbox в одной транзакции и только затем отвечает 200. Если PostgreSQL недоступна, отвечает 500, и Telegram повторит доставку.
- Отправка в Telegram через лимитер BullMQ: не больше 25 сообщений в секунду на бота.
- Valkey: `maxmemory-policy noeviction` (иначе BullMQ теряет задачи), AOF `everysec`, том на диске.
- Сбой Valkey: `/ready` остаётся 200 (зависит только от PostgreSQL); события копятся в outbox и уходят после восстановления; rate limit на вход, активацию приглашения, проверку QR и запись работает в закрытом режиме (отказ 503 с понятным сообщением), чтение данных продолжает работать.

**Топология и выкат**

| VPS | Сервисы | Ориентир |
|---|---|---|
| Production | Caddy, `api-1`, `api-2`, `worker-1`, `worker-2`, PostgreSQL 17, Valkey 8, WAL-G, GlitchTip | 4 vCPU, 8 ГБ RAM, 100 ГБ NVMe |
| Staging | Та же схема по одному экземпляру, отдельный бот; Uptime Kuma | 2 vCPU, 4 ГБ RAM |

Выкат (`infra/scripts/deploy.sh`, запускает GitHub Actions по SSH):
1. Скачать образ по digest.
2. Проверить, что последний бэкап моложе 24 часов, иначе остановиться.
3. Запустить миграции отдельным контейнером под advisory lock PostgreSQL, таймаут 5 минут. Миграции совместимы с предыдущей версией кода (сначала расширить, сужать в следующем релизе); откат миграций не делаем, только исправляющая миграция.
4. Перезапустить `api-1`, дождаться `/ready`, затем `api-2`; Caddy с активной проверкой здоровья не шлёт запросы в перезапускаемый экземпляр, `@nestjs/terminus` отвечает 503 перед остановкой. Затем `worker-1` и `worker-2` с мягким завершением задач.
5. Если `/ready` не стал 200 за 60 с, вернуть предыдущий digest.

Данные production на staging не копируются. Проверка восстановления выполняется в отдельном временном контейнере на production VPS и удаляется после проверки.

**Ограничение одного production VPS.** Отказ процесса переживаем, отказ сервера, диска или провайдера нет: недоступно всё до восстановления. NFR01 (99,5% в месяц, около 3,6 часа простоя) может быть нарушен одним таким инцидентом при восстановлении по NFR07 до 4 часов. Решение за заказчиком: Q31.

**Безопасность по умолчанию**
- HTTPS, HSTS, `X-Content-Type-Options`, `Referrer-Policy: strict-origin-when-cross-origin`.
- CSP базовая (уточняется в фичах 6 и 29): для `app.` `frame-ancestors https://web.telegram.org`, `script-src 'self' https://telegram.org`, `connect-src 'self'` плюс хост S3; для `admin.` `frame-ancestors 'none'`. Сначала режим только отчётов на staging.
- `@nestjs/authorization`: маршрут без политики запрещён. Роль и статус членства читаются из БД на каждый запрос, а не из сессии, поэтому приостановка действует сразу.
- CSRF защита `@nestjs/authentication` для cookie сессий, проверка `Origin`.
- PostgreSQL, Valkey, GlitchTip только во внутренней сети Docker; SSH по ключу, без root по паролю; firewall открывает 80, 443, SSH.
- Пользователь БД приложения без прав на DDL; миграции отдельным пользователем.
- Аудит критических действий в той же транзакции, что и действие.

**Запас по нагрузке**
- Пул соединений на экземпляр 10, всего не больше 50 при `max_connections` 100.
- Списки только с пагинацией по курсору; индексы под каждый список в спеке модели данных (фича 3).
- Тяжёлая работа (обработка изображений, экспорт, пересчёт рейтинга) только в `worker`.
- Кэш в Valkey добавляется только после измерения узкого места.

## Configuration required

- `APP_ENV` (`local`, `staging`, `production`), `NODE_ENV`, `PORT`, `ROLE` (`api` или `worker`)
- `DATABASE_URL`, `DATABASE_MIGRATION_URL`, `DATABASE_POOL_MAX`
- `VALKEY_URL`
- `SESSION_SECRET` (не короче 32 байт)
- `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET`, `TELEGRAM_BOT_USERNAME`
- `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT`
- `S3_ENDPOINT`, `S3_REGION`, `S3_BUCKET_FILES`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`
- `WALG_S3_PREFIX`, `WALG_S3_ACCESS_KEY`, `WALG_S3_SECRET_KEY`, `WALG_LIBSODIUM_KEY` (только у контейнера бэкапа, ключ с правом только на запись)
- `APP_ORIGIN`, `ADMIN_ORIGIN`
- `GLITCHTIP_DSN`, `VITE_GLITCHTIP_DSN` (клиентский, без секретов)
- `UPTIME_KUMA_PUSH_URL_BACKUP`, `UPTIME_KUMA_PUSH_URL_QUEUE`
- `LOG_LEVEL`

Секреты SOPS шифруются на три ключа age: владельца проекта, сервера окружения и CI. Значения бизнес правил BR01–BR14 не переменные окружения: они хранятся в таблице `business_rules` (модуль `settings`), начальные значения из утверждённых ответов Q22 записывает миграция, меняет суперадминистратор в A11 с записью в аудит, сервер проверяет их схемой при чтении.

## Consequences

**Positive**:
- Один язык, общие схемы и сгенерированный OpenAPI: меньше ошибок на стыке, проще передача (NFR09).
- Права, вход, сессии, outbox и проверки здоровья на первых пакетах Nest, а не самописные.
- Права, статусы и вместимость держат сервер и БД; клиент не может их обойти.
- Переносимость на любой VPS в РФ с Docker.

**Negative / tradeoffs**:
- PostgreSQL, Valkey, бэкапы, GlitchTip и обновления ОС сопровождаем сами; нужен runbook и ответственный.
- Production на одном VPS: отказ сервера останавливает всё (Q31).
- Пакеты `@nestjs/authentication` и `@nestjs/authorization` молодые (версии 0.1.x): возможны изменения API; версии фиксируются точно, обновления через тесты.
- Доставка уведомлений минимум один раз: при падении процесса в момент отправки возможен редкий дубль сообщения.
- Valkey как второе хранилище состояния добавляет компонент; принято инженером ради лимитера и общих счётчиков.

**Neutral**:
- Каркас (фича 1) строит `/develop` по этой спеке.
- `@nestjs/observe` отключён (внешний сервис, 152-ФЗ); вместо него GlitchTip и логи.
- Скилл `fastify-best-practices` удалён: при адаптере `@nestjs/platform-node` он не нужен.

## Follow-up

- [ ] Q17: выбрать VPS провайдера в РФ с S3 хранилищем; проверить условия хранения ПДн.
- [ ] Q21: подтвердить домены; до ответа действует схема `app.` и `admin.` (Assumed).
- [ ] Q31: заказчик принимает риск одного production VPS или добавляет третий.
- [ ] Фича 5 и 7: детали сессий (сроки, продление, отзыв, число сессий), вход администратора (первый фактор по Q09, первый суперадмин, восстановление доступа, резервные коды, шифрование секрета TOTP).
- [ ] Фича 6: защита от повторного использования initData и данных Telegram Login (хэш в БД с TTL), проверка cookie в Telegram WebView на iOS, Android, Desktop.
- [ ] Фича 29: жизненный цикл service worker, список кэшируемого, обновление версии, очистка при выходе.
- [ ] Фича 31: политика бакета, срок жизни ссылок не больше 5 минут и выдача только после проверки прав, удаление файлов.
- [ ] Фича 27: сценарии k6 (вход, календарь, пик записи на одно мероприятие), runbook обновлений, ротации секретов и восстановления.
- [ ] `/sync`: внести в `AGENTS.md` стек, команды, установленные скиллы (`nestjs/agent-skills`, `ssheleg/telegram-dev`) и MCP (Context7, GitHub); отказ от Drizzle скилла до каркаса; Observe MCP отклонён (152-ФЗ).
- [ ] После каркаса: `drizzle-kit skills` для официального скилла Drizzle.
