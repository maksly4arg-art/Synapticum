# Synapticum

**A personal multi-agent AI company: 8 autonomous Telegram agents + an orchestrating Builder, running on one Mac Studio.**

Synapticum - это личная мультиагентная AI-система: команда автономных агентов, каждый со своей ролью, личностью, памятью и Telegram-ботом. Builder (Claude Code) строит и чинит систему; агенты делают работу.

## Агенты

| Агент | Роль | Репозиторий |
|-------|------|-------------|
| **Bob** | Личный ассистент: медиа, поиск, vision | [ClaudeClaw](https://github.com/maksly4arg-art/ClaudeClaw) |
| **Alice** | CFO и монетизация, коучит остальных агентов | [Alice](https://github.com/maksly4arg-art/Alice) |
| **Eva** | Контент и SMM: генерация визуала, публикация в Telegram и Instagram | [Eva](https://github.com/maksly4arg-art/Eva) |
| **Freddy** | Deep research: мульти-итерационный анализ, OpenRouter Fusion | [Analyst](https://github.com/maksly4arg-art/Analyst) |
| **Joe** | Журналист (AI / сингулярность): статьи, авто-комментарии в X | [Joe](https://github.com/maksly4arg-art/Joe) |
| **Aleks** | Маркетинг-исполнение: кампании, слайды | [Aleks](https://github.com/maksly4arg-art/Aleks) |
| **Finn** | Локальный оператор ПК: shell + screen + GUI на локальной LLM ($0) | [Finn](https://github.com/maksly4arg-art/Finn) |
| **Jesse** | Трейдер (VSA/Elliott/SMC) + индикатор уровней | закрытый код |
| **Archivarius** | Пассивный слушатель и хранилище контекста (SQLite + FTS5) | [Archivarius](https://github.com/maksly4arg-art/Archivarius) |

Общие модули (A2A-сервер, транспорт движков, медиа-хендлеры): [Shared](https://github.com/maksly4arg-art/Shared). Рантайм и лаунчер: [ClaudeClaw](https://github.com/maksly4arg-art/ClaudeClaw).

## Ключевые идеи архитектуры

- **Topology B: движок = сменный слот.** Обёртка агента (личность, память, Telegram, A2A) - «дом»; под ним один чокпоинт `generate()`, который переключается между Claude Agent SDK (подписка), OpenRouter (5 режимов, включая Fusion из нескольких моделей) и локальной LLM через Ollama. Смена движка не трогает личность и знания агента.
- **A2A-связь.** Агенты общаются только по протоколу Google A2A (HTTP JSON-RPC 2.0), у каждого свой порт. Никаких общих шин сообщений.
- **Двухслойная память.** Obsidian Vault - граф знаний и единственный источник истины (wiki, правила, уроки, архитектура); SQLite с FTS5 - быстрый runtime-кеш и память переписки.
- **Self-learning.** У каждого агента вечная таблица уроков, которая инжектится в каждый промпт: агенты запоминают поправки юзера и друг друга.
- **Агент > скрипт.** Никаких keyword-гардов и re-prompt обвязок: ответ агента всегда доходит до юзера как есть.

## Документация

Вся архитектура - в [docs/](docs/):

- [architecture.md](docs/architecture.md) - полная карта системы (агенты, движки, A2A, память, интеграции)
- [how-to-build-an-agent.md](docs/how-to-build-an-agent.md) - как собирается агент
- [agent-hierarchy.md](docs/agent-hierarchy.md) - иерархия Senior/Junior и экономика
- [build-spec-topology-b-rollout.md](docs/build-spec-topology-b-rollout.md) - раскатка сменных движков
- [build-spec-delta.md](docs/build-spec-delta.md) - Delta: агент-контур как продукт (SaaS)
- [build-spec-boxbot.md](docs/build-spec-boxbot.md) - воронка Telegram-канала
- build-спеки Finn, чекпоинты и остальное - там же

## Что закрыто

Торговый агент Jesse и TTM-индикатор уровней (Pine Script) не публикуются. Секреты, ключи и базы памяти агентов в публичных репозиториях отсутствуют - конфигурация через `.env` (см. `.env.template` в каждом репо).

---

*Built by one human + Claude. Модель: 1000 клиентов × $1000.*
