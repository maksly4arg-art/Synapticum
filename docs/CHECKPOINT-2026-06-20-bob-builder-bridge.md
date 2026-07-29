# CHECKPOINT 2026-06-20 — Launcher redesign + Bob "Builder" MCP bridge

СТОП-КАДР. Следующая сессия: **довести обвязку Bob OR-панели** (шаги ниже) + **закончить тесты**.

---

## ✅ СДЕЛАНО в этой сессии

### Лаунчер (`ClaudeClaw/launcher.pyw`) — большой редизайн
- Панель агента = **2 строки**: строка1 `[Claude][Opus 4.8][Sonnet](+[Builder])`, строка2 `[OR][DS][Kimi][Gem][Fus][Max]`. Кнопки движка (`Claude`/`OR`) ведут строки.
- **Развязка движков (rule #2):** горит только активный движок, второй ряд погашен (`ToggleRow.set_dimmed`). `EnginePanel` класс.
- Режим называется **Claude** (вместо builder). GPT Deep/Light удалены. Opus 4.8 дефолт у всех.
- Кнопки `GPT*` убраны; микро-кнопки на карточке агента: 🧹 очистка + ↻ рестарт.
- Секции: слева **AGENTS** (все кроме Finn), справа **LOCAL** (Finn) + **PROCESS** (Claude Code restart, Orchestrator, Finn Calls, Polymarket, X Auto-Post). FinCalls спущен в Process. **Orchestrator-карточка УБРАНА** (по решению юзера; демон не стартует).
- Freddy унифицирован на `engines.env` (FLEET_OR_PREFIX += Analyst→FREDDY). Старый freddy.env-блок-механизм в лаунчере удалён.
- Окно **490x860** (шире+выше).

### Счётчик денег по движкам (req 12)
- Мелкий счётчик токенов убран. Деньги цветом: Claude=зелёный, DS=жёлтый, Kimi=тёмно-зелёный, Gem=голубой, Fus=синий, Max=розовый.
- **Формула в лаунчере** (не в скрипте, по решению юзера): `TEXT_RATES` (USD/токен на мозг) × токены. `_parse_model` парсит `or:<MODE>/<KIND>` → бакет(режим)+kind(реальная модель). `_bucket_display_cost` суммирует все kinds бакета → per-mode сумма (текст+vision).
- Агенты логируют `or:<MODE>/<KIND>` (mode=выбранный движок, kind=реально сработавшая модель). `runViaOR` теперь возвращает `kind` (vision-ход слепого мозга = `gemini`). Правки: `Shared/agent-engine/index.js` + 6× `engine.ts` (Alice/Eva/Jesse/Joe/Aleks/Analyst).
- **Генерация картинок в счётчик НЕ пишется** (по решению юзера) — Flux/fal.ai биллится отдельно; счётчик = только что едят модели (текст + vision). `IMG_RATES` и логирование генерации в Eva откатаны.

### Bob — фикс падения
- Bob крашился в `initDatabase`: таблица `conversations` имела старую колонку `timestamp`, код ждёт `created_at`. **Мигрировано** (0 строк, безопасно): `ALTER TABLE conversations RENAME COLUMN timestamp TO created_at` + `ALTER TABLE lessons ADD COLUMN context TEXT`. Bob стартует.

### Bob "Builder" режим = МОСТ (обернуть живого Claude в Telegram Боба)
- Кнопка **Builder** у Bob в лаунчере = ACTION (`EnginePanel.on_builder` → `ControlPanel._launch_builder_session`): гасит грамми-агента Боба + открывает claude-сессию через `osascript Terminal → be-bob.sh`.
- **MCP-мост** `/Users/synapticum/Synapticum/bob-mcp-bridge/` (зарегистрирован `claude mcp add bob-telegram --scope user`, ✔ Connected): инструменты `telegram_poll` / `telegram_send` / `telegram_status`. Токен = Bob (`@Bob_supernova_bot`).
- **Гейткипер `be-bob.py`** (финальная версия): дешёвый `curl getUpdates` long-poll 50с (**$0 в простое, без LLM**), `claude -p` (подписка, `ANTHROPIC_API_KEY` снят, `--dangerously-skip-permissions`, cwd=Vault=доверенная папка) дёргается ТОЛЬКО на реальное сообщение; сообщения передаются в prompt, Claude отвечает через `telegram_send`. `be-bob.sh` → `exec python3 be-bob.py`.
- Старый `claude -p` Builder-путь у Bob (`builder-cli.ts`) + ветка в `agent.ts` УДАЛЕНЫ. Builder у Alice/Freddy УБРАН (has_builder=False) — только у Bob.
- Скилл `~/.claude/skills/bob/SKILL.md` (one-pass, рекурсия внешняя).

---

## 🔑 КЛЮЧЕВЫЕ НАХОДКИ (durable)
1. **Биллинг:** `claude -p` и SDK = **ПОДПИСКА**, когда в env НЕТ `ANTHROPIC_API_KEY` (метрирует именно ключ, не сам `-p`). В шелле ключа нет; в `bob.env` ключ **пустой** → обычный Боб и так на подписке. Вывод: Builder-режим Бобу денег НЕ экономит, только возможности (полный Claude Code vs урезанный SDK).
2. **MCP односторонний:** Claude ДЁРГАЕТ инструменты, Telegram не толкает в Claude. Значит автономный Telegram требует, чтобы Claude кто-то вызвал (`claude -p`). Интерактивный TUI сам не запускается (висит на диалоге доверия/одобрении MCP) — для автономии негоден. `--print` = автономный режим.
3. **Чистая основная модель (схема юзера):** диалог/творчество — ЗДЕСЬ в Claude Desktop (подписка, полный Claude, интерактив, без поллинга), результат — НАРУЖУ по MCP (односторонне). Для контент/ресёрч-агентов (Eva, Freddy). Автономные в Telegram (Bob-ассистент, Joe по расписанию) остаются грамми-агентами.

---

## ⏭ ДЕФЕРНУТО → СЛЕДУЮЩАЯ СЕССИЯ: обвязка Bob OR-панели (как у флота)

Цель: у Боба 2-я строка (OR + 5 мозгов) реально работает, не фейк. Bob сейчас имеет НОЛЬ OR-обвязки. Точные шаги (surface уже разведан):

1. **Симлинк** agent-engine в Bob: `ln -s ../../Shared/agent-engine Bob/node_modules/agent-engine` (у Bob его нет).
2. **engines.env**: добавить `BOB_MODE=builder` + `BOB_OR_MODEL=deepseek`.
3. **Bob env**: `Bob/src/env.ts` имеет `readEnvFile` + `config.ts` его юзает — но ПРОВЕРИТЬ, читает ли он ЦЕНТРАЛЬНЫЙ `engines.env` (у флота env.ts читает CENTRAL_ENV_PATH). Если нет — добавить чтение engines.env. Затем экспорт `BOB_MODE`/`BOB_OR_MODEL` в `config.ts` (process.env || env || default), по образцу Alice/Eva config.ts.
4. **Bob engine.ts** (создать, по образцу Alice/src/engine.ts): `getEngine()` от `BOB_MODE`; `generate()` → если openrouter: `runViaOR({agentId, mode: BOB_OR_MODEL, ...})` + `logUsage({model:`or:${BOB_OR_MODEL}/${r.kind ?? BOB_OR_MODEL}`,...})`; иначе `runAgent`. **Переключить ОДИН call-site** `bot.ts:1480` (`runAgent(message, sessionId, sendTyping)`) на `generate(...)`. (Всего 1 call-site → риск низкий.)
5. **openclaw.json**: добавить агента `bob` (нужен для solo-мозгов DS/Kimi/Gem). Шаблон eva: ключи `{id,name,workspace,agentDir,model}`, `model: openrouter/deepseek/deepseek-v4-pro`. Патч через `openclaw config patch` + kickstart gateway (паттерн в launcher `_set_finn_tier`). Fusion работает и БЕЗ этого (прямой HTTP), solo — нет.
6. **launcher.pyw**: `FLEET_OR_PREFIX += "Bob":"BOB"`, `AGENT_ID_PREFIX += "bob":"BOB"`. Bob тогда автоматически получит 2-строчную панель + останется кнопка Builder (мост). has_builder Bob = True (оставить).
7. **tsc Bob** + живой тест OR-режимов.

**ИНВАРИАНТ:** НЕ ломать рабочий SDK-Telegram Боба (грамми). Builder-режим (гейткипер) НЕ трогать — он отдельный.

---

## 🧪 ТЕСТЫ (закончить в следующей сессии)
- Bob Builder live: выкл Bob-агента → кнопка Builder → написать `@Bob_supernova_bot` → проверить ответ (лог `bob-mcp-bridge/be-bob.log`, `.gk-offset`). Проверить, что в простое НЕТ вызовов `claude -p` (бесплатно).
- OR-счётчики денег live: после реальных OR-задач увидеть цветные per-mode суммы.
- Визуал лаунчера после релонча (2-строчные панели, секции).
- Bob OR-режимы (после обвязки): DS/Kimi/Gem/Fusion реально отвечают.

## ФАЙЛЫ
- `ClaudeClaw/launcher.pyw` (редизайн, EnginePanel, money, builder action)
- `bob-mcp-bridge/` (index.mjs мост, be-bob.py гейткипер, be-bob.sh)
- `Shared/agent-engine/index.js` (kind, токены)
- `{Alice,Eva,Jesse,Joe,Aleks,Analyst}/src/engine.ts` (лог `or:mode/kind`)
- `~/.claude/skills/bob/SKILL.md`, `~/.claude.json` (MCP bob-telegram)
