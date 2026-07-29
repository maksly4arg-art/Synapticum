---
title: Agent Hierarchy — Senior/Junior Cost Optimization
updated: 2026-05-21
updated_by: builder
status: planned
confidence: high
origin: user-idea
---

> TL;DR: Каждый "дорогой" агент получает дешёвого ассистента (DeepSeek/Hermes), который выполняет рутину по плану. Старший просыпается 1-2 раза в неделю для стратегии и проверки. Цель: $3000 → $200/мес.

## Проблема

| Источник расхода | Текущий $3000/мес |
|---|---|
| Бесконечные итерации настройки | ~60% |
| Русский промпт (2x токенов vs English) | ~15% |
| Отсутствие orchestrator (каждый агент грузит всё) | ~15% |
| Opus на рутинных задачах | ~10% |

## Оптимизация в 3 этапа

### Этап 1: Промпты + Orchestrator (сейчас → 15.06)
- English-only промпты (экономия ~30% токенов)
- Keyword-triggered blocks (сделано для Евы, ~80% экономия контекста)
- maxTurns на всех агентах
- Прекращение бесконечных кругов итераций

**Результат: $3000 → $900-1000/мес**

### Этап 2: Agent Hierarchy (после 15.06)
Архитектура "директор + ассистент":

```
┌─────────────────────────────────────────────────────┐
│                   SENIOR TIER                        │
│           Claude Opus/Sonnet (1-2x/week)            │
│                                                     │
│  Jesse-Sr    Alice-Sr    Eva-Sr    Freddy-Sr        │
│  (strategy)  (review)    (creative) (deep research) │
│      │            │          │          │           │
│      ▼            ▼          ▼          ▼           │
├─────────────────────────────────────────────────────┤
│                   JUNIOR TIER                        │
│          DeepSeek/Hermes (always-on, $0-2/day)      │
│                                                     │
│  Jesse-Jr    Alice-Jr    Eva-Jr    Freddy-Jr        │
│  (sync,scan) (reports)   (publish)  (collect)       │
│                                                     │
│  Работают по ПЛАНУ составленному Senior             │
│  НЕ думают, НЕ принимают решений                   │
│  При неопределённости → пишут в очередь Senior      │
└─────────────────────────────────────────────────────┘
```

### Этап 3: Max 20x Budget ($200/мес)
- Senior: Claude через подписку Max 20x ($200/мес = $200 agent credits)
- Junior: DeepSeek V4 Pro API ($2/$8 per 1M tok) или Hermes local ($0)
- Senior просыпается по расписанию (cron 1-2x/week) + по эскалации от Junior

**Результат: $900 → $200/мес**

## Роли Junior агентов

| Junior | Рутина (ежедневно) | Когда будит Senior |
|---|---|---|
| Jesse-Jr | Bybit sync, watchlist scan, журнал | Найден сетап, аномалия, нужна стратегия |
| Alice-Jr | Отчёты, метрики, tracking | Deadline, решение по монетизации |
| Eva-Jr | Публикация по расписанию, аналитика постов | Нужен новый контент, креативное решение |
| Freddy-Jr | Сбор данных, мониторинг источников | Найден важный сигнал, нужен deep analysis |

## Протокол Senior ↔ Junior

```
1. Senior составляет PLAN (checklist + triggers)
2. Junior исполняет план шаг за шагом
3. При trigger → Junior эскалирует (пишет в queue)
4. Senior просыпается (cron или escalation) → проверяет работу Junior
5. Senior корректирует план → Junior продолжает
```

## Стек Junior

- **Модель:** DeepSeek V4 Pro API (reasoning задачи) / Hermes 3 local (простая рутина)
- **Фреймворк:** Тот же Grammy + TypeScript, заменяется только agent.ts (adapter pattern)
- **Стоимость:** ~$0.50-2/день на всех Junior вместе
- **Формат плана:** Markdown checklist в Vault (Senior пишет, Junior читает и исполняет)

## Математика

| Компонент | Стоимость/мес |
|---|---|
| Max 20x подписка | $200 |
| Senior agents (из подписки) | ~$150 от credits (1-2x/week × 4 agents) |
| Junior agents (DeepSeek API) | ~$30-50 |
| **Итого** | **~$200-250** |

## Что НЕ меняется
- Vault, A2A, Archivarius — работают одинаково для обоих тиров
- Telegram боты — те же (Junior отвечает пользователю напрямую, Senior — через очередь)
- Self-learning — уроки общие, Junior применяет, Senior создаёт новые

## Зависимости
- [[multi-provider]] — adapter pattern для смены провайдера (уже спланирован)
- [[project_june15_deadline]] — дедлайн: до 15.06 строим, после — только оптимизация
- Adapter в agent.ts (одна точка замены провайдера) — подтверждена сегодня

## Следующий шаг
1. Закончить Eva context optimization (DONE)
2. Портировать keyword-blocks на Jesse и Alice
3. Написать adapter pattern в /Users/synapticum/Synapticum/Shared/ai-provider/
4. Создать первый Junior-прототип (Eva-Jr — самый простой кейс: публикация по расписанию)
