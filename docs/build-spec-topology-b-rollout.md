---
title: Build-spec - Topology-B раскатка на флот Synapticum
date: 2026-06-18
author: builder
type: build-spec
status: approved-architecture, pending-build
related:
  - build-spec-finn-fleet-and-freddy.md
  - "[[project_b_rollout_all_agents]]"
  - "[[feedback_engine_toggle_clean_separation]]"
---

# Topology-B раскатка на флот - канонический build-документ

Сгенерировано воркфлоу-аудитом (8 агентов, читан реальный код каждого + живой тест `openclaw agent --json`). Эталон кода: `/Users/synapticum/Synapticum/Analyst/src/{engine.ts,memory.ts,llm-provider.ts,probe.ts}`.

> **ИСКЛЮЧЕНИЕ (2026-06-18):** **Bob - вне этой раскатки.** У него отдельный план, DeepSeek/ORmode ему ничего не даёт (личный ассистент, остаётся SDK-only). Не трогать. Пилот раскатки → **Aleks** (или тест транспорта напрямую через `openclaw agent`, без агента).
> **Канон как строить агента:** [[how-to-build-an-agent]] (Память + A2A + Руки + параллель OpenClaw/SDK + Vault + Голос + Telegram, без решений по умолчанию).

## Сводка аудита по агентам

| Агент | Чокпоинт сейчас | Память | Усилие B | Главная сложность |
|---|---|---|---|---|
| **Bob** | `runAgent()` единый, `agent.ts:126` | ✅ Archivarius | low-med | session-resume, onTyping |
| **Aleks** | `query()` `agent.ts:60` (1 сайт) | ✅ Archivarius (write-only) | low | headless (бот=null) |
| **Joe** | `query()` `agent.ts:115` (1 сайт) | ❌ локальный db (без agent_id) | high | миграция памяти |
| **Eva** | `query()` `agent.ts:104` (1 сайт) | ❌ локальный eva.db, 50+ сайтов записи | high | дуал-райт памяти |
| **Alice** | `query()` + 7×`runAgent` (8 сайтов) | ❌ split local/Archivarius | high | 8 сайтов + унификация |
| **Jesse** | `query()`×2 + 12 сайтов + **vision** | ❌ decoupled own db | high | vision-гард + память |
| **Finn** | `query()` `agent.ts:55`, **native OpenClaw** | ❌ только vault-lessons | low(смысл)/med(объём) | нет grammy-фронта |

---

## ⚠️ КЛЮЧЕВОЕ РАСХОЖДЕНИЕ (читать первым - меняет дизайн)

Эталон на Analyst в ORmode **бьёт в OpenRouter напрямую по HTTP** (`freddy.env:LLM_ENDPOINT=https://openrouter.ai/api/v1/...`, `llm-provider.ts:runLLM()` делает `fetch`). **OpenClaw тут НЕ участвует** - значит у Freddy ORmode сейчас text-only, без тулов.

Залоченное решение для флота другое: ORmode = **OpenClaw → OpenRouter** через `openclaw agent --json` (gateway :18789), чтобы получить реальные тулы/веб. Значит у нас **две разные ветки ORmode-транспорта**:
- **Транспорт A (Freddy/Fusion, существует):** HTTP `fetch` в OpenRouter, Fusion-плагин, без тулов. Оставляем для Freddy/прожарки.
- **Транспорт B (флот, строим):** `spawn('openclaw',['agent','--agent',id,'--model',slug,'--message',prompt,'--json'])`, парсим `.result.payloads[0].text`. Нового кода на Analyst НЕТ.

Форма вывода (ПЕРЕПРОВЕРЕНО живьём 2026-06-19 - `status` на ВЕРХНЕМ уровне, не в .result):
```
.status                                        -> "ok"   ← верхний уровень envelope!
.result.payloads[0].text                       -> ответ агента
.result.meta.agentMeta.usage.{input,output}    -> токены
.result.meta.agentMeta.{provider,model}        -> что реально отработало
```
Гоча: ранний план писал `.result.status` - НЕВЕРНО. Проверено: `--agent finn --model openrouter/deepseek/deepseek-v4-pro` → "PONG", provider=openrouter, 4.8с. Плюс OpenClaw инжектит свой workspace-персонаж (~26k токенов promptTokens у finn) - для флот-агентов их openclaw-определение должно нести МИНИМАЛЬНЫЙ persona (идентичность несёт НАШ промпт), иначе двойная личность + лишние токены.
Гоча: тот же вызов занял **113с** (`durationMs:113492`) на локальном qwen. Латентность раннера - реальный риск (раздел 5).

## 1. Shared-модуль vs пер-агентные копии

**Рекомендация: гибрид. Транспорт - в Shared, тонкая обвязка - пер-агентная.**

`/Users/synapticum/Synapticum/Shared/agent-engine/` (новый):
- `openclaw-runner.ts` - единственная реализация транспорта B (spawn + парсинг `--json` + таймаут + ошибки). Самый рисковый и одинаковый код - обязан быть один.
- `memory.ts` - текущий `Analyst/src/memory.ts`, но `AGENT_ID`/`ARCHIVARIUS_DB_PATH` параметризованы (сейчас хардкод `'freddy'`).
- `llm-provider.ts` - переносим как транспорт A (HTTP OpenRouter, для Freddy/Fusion).
- `probe.ts` - переносим (его импортит memory.ts).

Пер-агентно (`<Agent>/src/engine.ts`, ~40 строк): `getEngine()` читает `<AGENT>_MODE`; `generate()` - чокпоинт: `builder`→локальный `agent.ts:runAgent()` (SDK), `openrouter`→`Shared/agent-engine/openclaw-runner.ts`. Тонкий, агентоспецифичен (свой AGENT_ID, свой набор тулов, vision у Jesse) - дублировать дёшево и правильно.

Обоснование: проект уже так живёт (`agent-name-filter`, `vault-lessons`, `video-note-handler`, `a2a-server` - shared). Транспорт/память - того же класса (инфраструктура, не личность). Один баг в парсере = один фикс. `engine.ts` пер-агентный, т.к. builder-ветка зовёт локальный agent.ts (свой usage/maxTurns/vision).

Гоча сборки: Eva/Joe бегут из `dist/`. Shared собирать как нынешние shared; после правок Shared - пересобрать Shared И потребителей.

## 2. Пер-агентные шаги (по усилию)

Шаблон каждого (кроме Finn): создать `engine.ts`; добавить `<AGENT>_MODE`+ids в `config.ts`; память→shared `memory.ts`; все call-sites→`generate()`; `<agent>.env`+`<AGENT>_MODE=builder`+закомм. ORmode-блок; зарегистрировать в `openclaw.json`.

**LOW:**
- **Aleks** - 1 сайт `agent.ts:60`→`generate()`. Память уже в Archivarius (write-only) - добавить `recallContext()`-инъекцию. Гоча: бот может быть `null` (headless).
- **Bob** - `bot.ts:1480`+`index.ts:208`→`generate()`; `runAgent()` остаётся builder-реализацией. Память в Archivarius - добавить `recallContext()`. ДВА гоча (общие для всех): **session-resume** (SDK sessionId живёт ТОЛЬКО в builder-ветке, наружу из generate() не торчит; в ORmode континьюити даёт recallContext); **onTyping** (обе ветки дёргают typing; в openclaw-runner `setInterval(onTyping)` на время spawn - вызов 100с+).

**HIGH** (общая причина: локальная память, не Archivarius):
- **Alice** - 8 сайтов (`agent.ts:72`; `index.ts:301,377,432,492,541,573,620`). Унифицировать память (recordTurn/recallContext), каждый сайт через recallContext.
- **Jesse** - 12 сайтов + **vision** (`trade-review.ts:308 runAgentVision`, `agent.ts:290`). **РЕКОМЕНДАЦИЯ: vision держим на `builder` ВСЕГДА** (гард `if(images)→builder` независимо от JESSE_MODE - графики нельзя терять на дешёвой модели). Decoupled-память → перевести на shared. Тестировать в петле с трейдером.
- **Eva** - 1 чокпоинт (`agent.ts:104`), но 50+ сайтов записи (`bot.ts`). Дуал-райт через ОДНУ функцию (recordTurn пишет локально И в Archivarius), не плодить 50. `model-resolver.ts` не трогать.
- **Joe** - `agent.ts:115`→generate(). Локальный db без agent_id; добавить recordTurn рядом с saveConversation.

## 3. Finn - спецслучай (native OpenClaw → B-обёртка)

Сейчас Finn раздвоен: native OpenClaw-агент (`openclaw.json:agents.list id=finn default:true model=ollama/qwen3.6:27b`) И TS-обёртка с `query()` (`agent.ts:55`). Залок: привести В проект как B-обёртку.

Особенность: ORmode-ветка Finn по дефолту = **локальный ollama qwen ($0)**, у остальных = OpenRouter/DeepSeek. Один транспорт B, разные `--model`.

Шаги: создать `engine.ts`+`FINN_MODE`; `agent.ts:55`→generate(); добавить shared `memory.ts` (turns в Archivarius сейчас НЕ пишутся); vault-lessons оставить.
**Главный нетривиальный кусок - Telegram-фронт:** у Finn НЕТ grammy-фронта, Telegram даёт сам OpenClaw (`channels.telegram`+bindings). Чтобы стал B-агентом: поднять минимальный grammy в `Finn/src/index.ts` с тем же ботокеном И **снять telegram-binding из openclaw.json** (иначе два процесса дерутся за getUpdates → конфликт). Снять `default:true` с finn-агента.

## 4. OpenClaw wiring

**4.1 Провайдер OpenRouter в `~/.openclaw/openclaw.json`** (сейчас только ollama+consilium):
```json
"openrouter": {
  "baseUrl": "https://openrouter.ai/api/v1", "api": "openai-completions",
  "apiKey": "sk-or-...",
  "models": [{ "id": "deepseek/deepseek-v4-pro", "name": "Fleet ORmode", "input": ["text"] }]
}
```
Базовая модель флота = DeepSeek v4 pro. OR-режимы и Fusion/Fusion Max - КАНОН в [[how-to-build-an-agent]] §2.1 (Fusion = точечный режим через офиц. плагин `openrouter/fusion`, доступен агентам по запросу, НЕ always-on; не строить кастомно).

**4.2 Пер-агентные определения** в `agents.list`: `{id, model:"openrouter/deepseek/deepseek-v4-pro", tools:{profile:"full"}}` для bob/eva/jesse/alice/joe/aleks; finn=`ollama/qwen3.6:27b`($0). Снять `default:true` с finn.

**4.3 `openclaw-runner.ts`** (форма проверена):
```ts
const args = ['agent','--agent',id,'--model',slug,'--message',prompt,'--json','--timeout','600']
if (sessionKey) args.push('--session-key', sessionKey)
// НЕ --deliver: ответ доставляет НАША grammy. typing: setInterval на время spawn.
// парсинг: out.result.payloads[0].text ; usage=out.result.meta.agentMeta.usage
// HARD RULE: ORmode упал → text:null → ошибка юзеру, НИКАКОГО тихого фолбэка на Claude.
```
`--local` (встроенный раннер, нужны ключи в env процесса) vs gateway :18789 (тёплый раннер). **РЕКОМЕНДАЦИЯ: gateway** (не холодный старт на каждый вызов), но вводит зависимость "gateway жив".

**4.4 Память Vault** - сборка промпта остаётся в НАШЕЙ обёртке. `generate()` (одинаков для обеих веток): recallContext(chatId) из Archivarius → buildVaultContext/lessons → identity → склейка prompt → builder|openclaw → recordTurn(user)+recordTurn(assistant). OpenClaw-агент НЕ источник истины памяти; канон = наш Archivarius. Смена движка не форкает мозг.

## 5. Порядок, верификация, риски

**Пилот: транспорт сначала тестируется напрямую** (`openclaw agent --json`, без агента), затем первый агент = **Aleks** (Bob исключён). Aleks - 1 чокпоинт, память в Archivarius; гоча onTyping/session-resume обкатать там, где есть бот.

**Порядок:** 1) Shared-модуль + тест runViaOpenclaw напрямую. 2) openclaw.json (провайдер+агенты). 3) **Aleks (пилот)**, оба режима. 4) Joe, Eva (паттерн дуал-райта). 5) Alice. 6) Jesse (тяжёлый, последним из TS). 7) Finn (Telegram-фронт отдельно). **Bob - НЕ в раскатке (отдельный план).**

**Верификация на шаге:** builder отвечает как раньше + recordTurn в Archivarius; переключить env→openrouter через лаунчер→рестарт→ответ через OpenClaw (meta.provider=openrouter, model=deepseek); **тест общей памяти**: спросить в builder, переключить на openrouter, рестарт, "что я спрашивал?" - должен помнить (континьюити через Archivarius); typing крутится в обеих ветках; Eva/Joe - `npx tsc` (не --noEmit) + рестарт через лаунчер.

**Риски:** латентность раннера (113с замер - gateway+typing обязательны; vision Jesse на openclaw ещё медленнее → держать на builder); зависимость от gateway :18789 (health-карточка в лаунчере); двойной getUpdates у Finn (снять binding атомарно); стоимость OpenRouter пер-агент (per-agent budget, раскатывать по одному, смотреть счёт); Fusion = точечный режим (не always-on), канон [[how-to-build-an-agent]] §2.1; vision-деградация Jesse (жёсткий гард); дрейф dist (пересборка Shared→агент); два транспорта в Shared (документировать: флот→openclaw-runner, Freddy/Fusion→llm-provider).

## Файлы-якоря
- Эталон: `/Users/synapticum/Synapticum/Analyst/src/{engine,memory,llm-provider,probe}.ts`
- Новый shared: `/Users/synapticum/Synapticum/Shared/agent-engine/` (создать)
- OpenClaw: `/Users/synapticum/.openclaw/openclaw.json`
- Env: `/Users/synapticum/Synapticum/Vault/_config/<agent>.env`
