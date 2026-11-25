# Lawyer Chess Club

Закрытая платформа шахматного клуба юристов Москвы: Telegram Mini App и PWA (одна кодовая база, один аккаунт по `telegram_id`), Telegram бот и веб панель администратора. Требования: [docs/product/tz.md](docs/product/tz.md) (версия 1.1, коды S, P, A, UC, BR, NFR). Референсы дизайна: [docs/design/references/](docs/design/references/), карта экранов и компонентов: [docs/design/screen-map.md](docs/design/screen-map.md). Открытые решения: [docs/product/open-questions.md](docs/product/open-questions.md). Термины: [GLOSSARY.md](GLOSSARY.md).

## Stack

<to be filled> (решается в `/architect стек и архитектура`, вопрос Q08; ТЗ задаёт React + TypeScript, NestJS или FastAPI, PostgreSQL в РФ)

## Build approach

Tracer Bullet (каждый срез сквозной: БД → API → Mini App / бот / админка, узкий, но настоящий).

## Commands

```bash
# <to be filled после каркаса>
```

## Workflow

- План: `docs/scope/index.md`. Следующий шаг всегда первый неотмеченный пункт.
- Порядок: `/architect` → `/develop` → `/check verify` → `/test` → `/check review` → `/document` → `/sync`. Уровень по умолчанию GA.
- Спеки: `docs/specs/NNNN-title.md` (владелец `/architect`). ADR: `docs/adr/` (только для трудно отменяемых решений).
- Вопрос со статусом `open` в `open-questions.md` не решай сам: спроси или запиши допущение как `Assumed` в спеке.

## Rules

- Безопасность на сервере, не в интерфейсе: каждая закрытая операция проверяет роль и статус членства (RBAC). Скрытая кнопка не является защитой.
- Личность только из данных Telegram, проверенных на сервере: initData в Mini App, Telegram Login в PWA, со свежим `auth_date`. `initDataUnsafe` не доказательство.
- Mini App и PWA делят экраны и логику; отличия (вход, тема, кнопки, уведомления, установка) живут только в слое платформы.
- PWA: service worker кэширует только оболочку, никогда ответы закрытого API и ПДн; сессия только в `HttpOnly` cookie; выход очищает кэш и push подписку.
- Секреты (токен бота, ключи) только в серверной конфигурации. Никогда в клиенте, репозитории, логах.
- Минимизация ПДн по 152-ФЗ: не собирать то, чего нет в ТЗ. Видимость полей профиля определяет владелец.
- Моки, сиды и тестовые данные только с вымышленными людьми; имена и фото со скриншотов референса никуда не переносить.
- Регистрация на мероприятие атомарна и идемпотентна; вместимость и уникальность (мероприятие, член) гарантирует БД.
- Статусы меняются только по переходам ТЗ 3.3, права только по матрице ТЗ 3.2, значения сроков из BR01–BR14 в конфигурации, а не в коде.
- Результаты и рейтинг не перезаписываются: новая версия и пересчёт от точки изменения. Критические действия пишутся в аудит.
- Внутренний рейтинг называется LCC Rating, никогда «официальный ФШР/FIDE».
- Интерфейс на русском, все строки в словарях (под будущий английский). Имена в коде на английском по `GLOSSARY.md`.
- Логику рейтинга, записи и прав разрабатывай test first (скилл `tdd`), ожидаемые значения из утверждённых примеров, а не пересчётом.

## Agent skills

- [scope](.agents/skills/scope/), [architect](.agents/skills/architect/), [develop](.agents/skills/develop/), [check](.agents/skills/check/), [test](.agents/skills/test/), [debug](.agents/skills/debug/), [document](.agents/skills/document/), [sync](.agents/skills/sync/), [audit](.agents/skills/audit/): `jsmastery-pro/skills`, основной процесс от плана до сдачи
- [tdd](.agents/skills/tdd/): `mattpocock/skills`, red green цикл на согласованных seams для рейтинга, записи, RBAC
- [domain-modeling](.agents/skills/domain-modeling/): `mattpocock/skills`, ведение `GLOSSARY.md` и ADR при появлении новых терминов

## Context files

<!-- Nested AGENTS.md files are listed here as they are created -->

_Drafted by /audit from the repo, worth a quick human pass. Edit freely: once a line stops matching this draft, later runs treat it as curated and will flag rather than overwrite it._
