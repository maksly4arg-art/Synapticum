---
title: CHECKPOINT 2026-06-19 - где встали, что дальше (OR билд)
date: 2026-06-19
author: builder
type: checkpoint
status: STOPPED - продолжить в новой сессии
related:
  - how-to-build-an-agent.md
  - build-spec-topology-b-rollout.md
  - "[[project_or_modes_standard]]"
  - "[[project_b_rollout_all_agents]]"
  - "[[project_freddy_openrouter]]"
---

# CHECKPOINT - стоп-кадр перед «OR билд»

> ## ✅ ОБНОВЛЕНИЕ 2026-06-21 — §5 FINN: ОПЕРАТОР ПЕРЕВЕДЁН НА ОБЁРТКУ (Канон-чисто)
> Боевой путь оператора (`finn-call.ts`) теперь идёт через `generate()` → `runViaOR({model:<тир>})` → `openclaw agent finn` (НЕ сторонний exec): finn.db получает §3-память (conversations/engine_runs), руки целы (тот же агент finn), `FINN_MODE=openrouter` дефолт, OpenClaw-workspace finn нейтрализован (личность → `Finn/src/instructions.ts`). Старый декомпозитор в `src/` снят, дубль-A2A :41008 убран. Проверено live (junior). Канонизировано: how-to-build-an-agent.md §5/§12/§11/§0.2-реестр. **Part C (`sync-operator-memory` траектории) для диалога больше НЕ нужен** — finn.db пишется напрямую обёрткой. ОСТАЛОСЬ (юзер): рестарт «Finn Calls», боевой hands-прогон, per-agent `operate`/`finn_result` rollout.
>
> ## ✅ ОБНОВЛЕНИЕ 2026-06-20 — BOB ВКЛЮЧЁН В РАСКАТКУ (по запросу юзера)
> Старая пометка «Bob - вне раскатки / SDK-only» ОТМЕНЕНА: по прямому запросу Bob получил полную OR-обвязку Topology B, как флот. tsc=0, dist собран, НЕ перезапущен, НЕ закоммичен.
> - **`Bob/src/engine.ts`** (новый) — чокпоинт `getEngine()`+`generate()`: builder→`runAgent` (Claude SDK, без изменений), openrouter→`runViaOR({agentId:'bob', mode:BOB_OR_MODEL})`. HARD RULE без тихого фолбэка. OR-ветка препендит **краткую личность Bob** (`BOB_OR_IDENTITY`) — в builder личность даёт SDK (CLAUDE.md+скиллы), OpenClaw их не читает (build-spec §1).
> - **`config.ts`** +`AGENT_ID='bob'`, `BOB_MODE` (дефолт builder), `BOB_OR_MODEL` (дефолт deepseek). **`env.ts`** переведён на центральный `engines.env` + `Vault/_config/bob.env` (он БАЙТ-в-байт = старый Bob/.env, токены целы; Bob auth = OAuth `~/.claude`, не ANTHROPIC_API_KEY).
> - **Call-sites→`generate()`:** `bot.ts:1480` (юзер-чат) + `index.ts:208` (межагентное). **`scheduler.ts` и `vision.ts` ОСТАВЛЕНЫ на `runAgent` (Claude SDK)** — vision передаёт ПУТИ кадров, читаемые SDK из cwd; scheduled = фон. Не деградируем на слепой/дешёвый OR.
> - **OpenClaw:** агент `bob` в `openclaw.json` (deepseek-v4-pro), workspace `~/.openclaw/workspaces/bob` нейтрализован (6 comment-only .md как у флота). CLI `openclaw agents list` видит bob. **Транспорт deepseek-solo уже проверен на флоте — живой вызов (деньги) НЕ гонял, это шаг юзера.**
> - **package.json** +`agent-engine` dep + симлинк `node_modules/agent-engine`. **engines.env** +`BOB_MODE=builder`/`BOB_OR_MODEL=deepseek`. **Лаунчер:** `Bob`+`bob` добавлены в `FLEET_OR_PREFIX`/`AGENT_ID_PREFIX` → у карточки Bob появляется OR-ряд (DS/Kimi/Gem/Fus/Max); кнопка «Builder» (be-bob мост живой Claude) СОХРАНЕНА (она в Claude-ряду, не зависит от OR). py_compile OK.
> - **ОСТАЛОСЬ юзеру:** живой тест (лаунчер → OR + DS → рестарт Bob → Telegram «помнишь?»-континьюити через локальную bob.db). Гоча: латентность OpenClaw-раннера; gateway :18789 должен быть жив.
>
> ## ✅ ОБНОВЛЕНИЕ 2026-06-19 (OR билд, заход 1) - ШАГ 1 ИЗ ПЛАНА СДЕЛАН
> **OR-каталог (5 режимов) + `<AGENT>_OR_MODEL` построены и собраны (tsc=0), слаги верифицированы на живом OpenRouter.**
> - `Shared/agent-engine`: добавлены `OR_MODES` (5 режимов), `FUSION_CONFIG`/`FUSION_MAX_CONFIG` (стандарт §2.1: судья **Gemini 3.1 Pro**, панель Gemini 3.5 Flash + Kimi K2.6 + DeepSeek, финал DeepSeek), `runFusion()` (транспорт A, прямой HTTP OpenRouter, self-resolve ключа из env→openclaw.json), `runViaOR()` диспетчер (solo→OpenClaw, fusion→HTTP). `runViaOpenclaw` не тронут. MODELS расширен deepseek/kimi/gemini.
> - `openclaw.json`: в провайдер `openrouter` добавлены `moonshotai/kimi-k2.6` (text+image) и `google/gemini-3.5-flash` (text+image).
> - Флот (Aleks/Joe/Eva/Alice/Jesse): `config.ts` +`<AGENT>_OR_MODEL` (дефолт `deepseek` = поведенчески-нейтрально), `engine.ts` `runViaOpenclaw`→`runViaOR(mode)`, env +видимый переключатель. builder-ветка байт-в-байт. **dist пересобран** (агенты стартуют из dist).
> - Верификация: 6 слагов (deepseek-v4-pro, kimi-k2.6, gemini-3.5-flash, gemini-3.1-pro-preview, opus-4.8, gpt-5.5) присутствуют в live `openrouter/api/v1/models`.
> - **НЕ коммичено. Требуется ЖИВОЙ тест юзера** (рестарт через лаунчер → `<AGENT>_MODE=openrouter` + `<AGENT>_OR_MODEL=kimi/fusion/...`).
> - **Осталось из плана:** §2 общая память-шина, §3 Freddy к стандарту (он ещё на старом Fusion-судье DeepSeek!), §4 launcher-тогглы, §5 Finn B, §6 vision. Fusion-режим работает только при наличии OpenRouter-ключа (есть в openclaw.json → подхватится).
>
> ### ✅ ДОБАВЛЕНО (заход 1, по фидбеку юзера про «двойные пути» и симлинки):
> - **Единый пульт движков `Vault/_config/engines.env`** - ВСЕ `<AGENT>_MODE`+`<AGENT>_OR_MODEL` флота в ОДНОМ файле «что к чему». Дубль ликвидирован: тоггл-строки убраны из персональных `<agent>.env` (там остался только закомментированный override-escape-hatch).
> - Каждый `<agent>/src/env.ts` (5 шт) теперь грузит `engines.env` (база) → персональный файл (override), через хелпер `parseEnvInto`. dist пересобран, рантайм-резолв проверен (все 5 видят свой режим из центрального файла).
> - **Симлинки:** лишних/архитектурных НЕТ. Единственные - стандартные npm `file:`-депы Shared-модулей в `node_modules` (НУЖНЫ). Vault/_config - одна реальная папка, дублей путей нет.
> - **§4 launcher-тоггл теперь пишет в `engines.env`** (единая точка), не в 5 файлов.
>
> ### ⚠️ §2 МОДЕЛЬ ПАМЯТИ — ИСПРАВЛЕНА по фидбеку юзера (2026-06-19)
> «Общая шина» юзера = НЕ одна общая таблица. Это общее ПОЛЕ (Vault + наш SQLite-слой), но **у каждого агента СВОЯ локальная база (свой `db.ts`), контексты НЕ смешиваются**, одна и та же база во всех режимах, провайдер памятью НЕ владеет (особенно OpenClaw — Finn увести с дефолта в наше поле). Прожарка Freddy читает чужие базы СНАРУЖИ и только по явному запросу (не авто, не жечь токены).
> - **Мой Shared-модуль `agent-memory` УДАЛЁН** — он писал в общую `conversations` Archivarius = ровно отвергнутая «свалка». Был не подключён, удалён без следов.
> - **Правильная реализация:** каждый агент использует СВОЙ локальный `db.ts` (как Joe/Eva/Jesse уже: `saveConversation`/`getRecentConversations` → свой `*.db`). Никакого дуал-райта в Archivarius.
> - **Состояние агентов (проверено прямым чтением):**
>   - Joe/Eva/Jesse/aleks — уже на своей локальной базе, НЕ трогаю (только проверить, что континьюити берётся из db, а не из сессии движка — read-only).
>   - **Bob — ГОТОВ:** пишет локально (`bot.ts`→bob.db); запись в Archivarius (`archiveSaveConversation`) = мёртвый код. 75 легаси-строк удалены. Навыки (lessons/trade_journal/...) не тронуты.
>   - **Finn — СТРОИТЬ:** в finn.db НЕТ `conversations`; континьюити висит на сессии движка (getSession/setSession+sessionId) = анти-паттерн §3.1. Нужна локальная база диалога (таблица + save/recall в index.ts, движок-агностик). 1 шальная строка из Archivarius удалена.
>   - **Alice — ДВОИТ:** 294 локально (её база цела) + 294 зеркало в Archivarius. Развести = выключить зеркало-запись (ждёт подтверждения юзера).
> - **Archivarius `conversations` для диалога флота выводится из игры** (служебная для самого Archivarius, но агент-диалоги туда не льют).
> - **Аудит 2-6 застрял** (2/9), остановлен — на прямом чтении кода.
>
> ### ✅ §2 ЗАКРЫТ ДЛЯ ФЛОТА (2026-06-19, заход 1)
> - **Bob** — готов (локальная база, легаси-зеркало (75) удалено).
> - **Alice** — РАЗВЕДЕНА: `archiveSaveConversation` занеткана (no-op, 1 точка), 294 зеркальные строки удалены из Archivarius, локальные 294 в alice.db целы. tsc=0.
> - **Joe** — ок без правок (его `buildEnrichedPrompt` инжектит локальную историю ВСЕГДА → OR-память работает).
> - **Eva + Jesse** — ФИКС OR-памяти: их `buildEnrichedPrompt` инжектил историю только при `!sessionId` (для SDK-resume); в OR-ветке передавал stale sessionId → OR терял контекст (§3.1). Теперь OR-ветка передаёт `sessionId=undefined` (OpenClaw stateless, история всегда). builder-ветка не тронута. tsc=0, dist собран.
> - **aleks** — per-chat истории по дизайну нет (маркетинг), память не добавляю (как просил юзер «не трогать»).
> - **Archivarius `conversations` = ПУСТ** — флот-диалог полностью выведен из общей таблицы. Модель «общее поле / своя база у каждого / контексты не смешиваются» достигнута.
> - **Shared-модуль `agent-memory` удалён** (он был под отвергнутую «общую таблицу»).
> - **Finn** — db-база диалога добавлена в finn.db (`conversations` + saveConversation/getRecentConversations, движок-агностик, тай-брейк id), tsc=0. НО проводка в поток + Topology B + развязка с native OpenClaw = это §5 (Finn ещё на Claude SDK с session-resume, своего engine.ts нет).
>
> ### ✅ FINN PART B ГОТОВ (тоггл движка TS-Finn) + PART C СКОУП (2026-06-19)
> Решение юзера по Finn = **1 и 3** (НЕ grammy-захват токена; руки не трогаем):
> - **Part B (готово):** TS-Finn (A2A-воркер) получил Topology B. `engine.ts` (builder→runAgent SDK / openrouter→runViaOR), `FINN_MODE`+`FINN_OR_MODEL` (дефолт OR = `local` qwen $0), env.ts на центральном `engines.env`, `buildEnrichedPrompt` вынесен, index.ts через `generate()`, dep `agent-engine`+симлинк, режим `local` добавлен в `OR_MODES`. tsc=0, дефолт builder (поведение не меняется). FINN-строки в engines.env.
>   - **Открытие:** TS-Finn НЕ grammy-бот, это A2A-воркер (декомпозиция стратегий, без юзер-чата). Пользовательский Telegram-Finn + руки = **native OpenClaw** (`channels.telegram` botToken 8835590251, `bindings: route telegram→finn`, локальный qwen + hands-MCP). Эти ДВА Finn — разные роли; getUpdates-конфликта нет (grammy у TS нет).
> - **Part C (скоуп, не начат):** память оператора (native OpenClaw Finn) лежит как **JSONL-траектории** `~/.openclaw/agents/finn/sessions/*.trajectory.jsonl` (state.sqlite по диалогам пуст). «В наше поле» = **мост**: парсить траектории → finn.db `conversations` (дормантная таблица уже добавлена, тай-брейк id). OpenClaw остаётся файловым рантайм-источником, мы зеркалим. Реализация Part C = отдельный синк (parse trajectory jsonl → saveConversation), запускать периодически/по запросу.
>
> ### ✅ §3 FREDDY К СТАНДАРТУ — движок мигрирован (2026-06-19)
> - Freddy OR-путь теперь через **Shared `runViaOR`** (не свой llm-provider). `FREDDY_OR_MODEL` (engines.env, дефолт `fusion`) = **СТАНДАРТНЫЙ Fusion** (панель Gemini 3.5 Flash + Kimi K2.6 + DeepSeek, судья **Gemini 3.1 Pro**, финал DeepSeek). Старый судья-DeepSeek/панель gemini-3-flash-preview УБРАНЫ. Solo-режимы = транспорт B (OpenClaw, реальные тулы/веб) → «оба транспорта» через каталог.
> - `env.ts` Freddy → центральный `engines.env`; `freddy.env` LLM_* блок УДАЛЁН (ключ self-resolve из openclaw.json); dep `agent-engine`+симлинк; tsc=0, getEngine()=openrouter резолвится. generate() — единственный путь к модели (проверено). images (видео-кружки) идут в Fusion (панель Gemini/Kimi видит); web → web-плагин.
> - **Freddy своя память** (memory.ts) остаётся на Archivarius `conversations` (agent_id=freddy) — его собственная база, не смешана. Не трогал.
> - **⚠️ ПРОЖАРКА — нужен рефактор (следствие §2-модели):** `probe.ts` читает чужие турны из общей Archivarius `conversations`, которую мы ОЧИСТИЛИ (флот ушёл в локальные db). Сейчас прожарка флот-агентов вернёт пусто (не крашит). По решению юзера (Q2) прожарка должна читать СОБСТВЕННУЮ базу агента СНАРУЖИ и ТОЛЬКО по запросу. Нужен реестр agentId→{dbPath, схема} (схемы разнятся: Alice timestamp, прочие created_at). Отдельный фокус-таск.
>
> ### ✅ §6 VISION — построен (нативный OpenClaw infer, НЕ HTTP) 2026-06-19
> По фидбеку юзера отказались от base64-HTTP. Транспорт = **нативный `openclaw infer image describe(-many)`** (файл из workspace → OpenClaw сам зовёт провайдера). Проверено живьём (фото Атакамы описано; парсер чинён под реальный JSON `{ok, outputs:[{text}]}`).
> - Хелпер `runVision` (Shared) + врезан в `runViaOR`: картиночный ход НЕ идёт через OpenClaw-CLI (он текстовый) — уходит на vision-модель.
> - §2.2 маршрут: `kimi/gemini` — сами (родное зрение); `deepseek/fusion/fusion-max` — Gemini 3.5 Flash (облако); **`local` — `ollama/qwen2.5vl:32b` ($0, НЕ облако)** — поправка юзера, поле `vision` в каталоге.
> - **Finn локальное зрение = qwen2.5-VL 32B** (RAM 96GB, текст-мозг 29GB; 32b чтоб параллель-лоад влез; 72b впритык). Качается (`ollama pull`), объявлен в openclaw.json (input text,image).
> - **Картинки = файлы в workspace** (телеграм-дроп или вручную), не base64. **Workspace переезжают на /Volumes/Synapticum Memory** (диск 3.3TB своб): НОВЫЕ туда, старые не трогаем, БЕЗ симлинков (агенты стартуют ws с нуля в новом месте) — решение юзера.
> - **Осталось по §6:** дождаться pull → $0-тест локального зрения; per-agent проводка (агенты передают путь файла из ws в `runViaOR(images:[...])` — video-note кадры, Eva); workspace-релокейшн (config WORKSPACE_DIR → внешний диск).
>
> ### ✅ ПОЛНЫЙ БИЛД ОСТАТКА ЗАВЕРШЁН (2026-06-20, без тестов — по запросу юзера)
> Все 8 тронутых агентов: tsc=0, dist собран. НЕ закоммичено, НЕ перезапущено.
> - **§4 Launcher-тогглы:** в `launcher.pyw` у каждого OR-агента (Aleks/Joe/Eva/Alice/Jesse) два тоггла — MODE (builder/OR) + OR_MODEL (DS/Kimi/Gem/Fus/Max), пишут в `engines.env`, рестарт карточки при смене. Хелперы `_read/_write_engines_kv`, `_add_fleet_or_toggles`. Freddy свой mode_row сохранён. py_compile OK.
> - **§6 Vision:** ядро (`runVision` через нативный `openclaw infer image describe`, §2.2 маршрут, local→qwen2.5vl:72b $0) + thread-through `images` в `generate()`→`runViaOR` всех 5 флот + **caller-hookup**: video-кружки (Joe/Alice/Jesse) шлют `frames`, Jesse фото-чарт шлёт `[localPath]` (бесплатный сканер графиков в local/OR). builder-ветка игнорит images (SDK читает путь из промпта).
> - **Workspace-переезд:** все 7 агентов (Aleks/Joe/Eva/Alice/Bob/Jesse/Finn) `WORKSPACE_DIR`→`/Volumes/Synapticum Memory/agent-workspaces/<agent>`, без симлинков, с нуля. **ОТКАЗОУСТОЙЧИВОСТЬ (2026-06-20):** `existsSync('/Volumes/Synapticum Memory') ? внешний : resolve(PROJECT_ROOT,'workspace')` — DAS упал → фолбэк на СТАРЫЙ локальный ./workspace (резерв), агент не крашит, только storage деградирует. Старые ws НЕ перенесены (юзер сам).
> - **Прожарка-рефактор:** `Analyst/src/probe.ts` — реестр `AGENT_BASES` (per-agent dbPath+table+timeCol), читает базу агента read-only по запросу; Freddy остаётся на Archivarius. ГОЧА: Bob db.ts говорит created_at, на диске timestamp — взят живой timestamp (если Bob пересоздать — сломается).
> - **Finn Part C:** мост `Finn/scripts/sync-operator-memory.ts` — парсит `*.trajectory.jsonl` (`model.completed`→messagesSnapshot, telegram chat_id из untrusted-metadata, идемпотентность маркером `src:openclaw:<sid>:<seq>:<role>`) → finn.db conversations. ПОСТРОЕН, НЕ запущен/не расписан (cron) — это тест-фаза.
> - **qwen2.5vl:72b** скачана; `OLLAMA_MAX_LOADED_MODELS=1` в plist (применится на рестарте Ollama).
>
> ### 🧪 ОСТАЛОСЬ НА ФАЗУ ТЕСТОВ (не билд) → ПРОТОКОЛ: `test-plan-or-build-and-consilium.md`
> Полный план тестов готов (2026-06-20): смоук OR-билда (A1-A6) + **тест способностей Finn в 4 прохода** (Middle → Senior → Консилиум+синтез-Senior → Консилиум+синтез-Builder). Две задачи: **Задача 1 (ОСНОВНАЯ) — серьёзный аналитический промпт** (SaaS-метрики, многоходовый вердикт, лестница по глубине); Задача 2 — вероятностный парадокс (7/15, лестница по числу) + альт. модулярка (343). Запускать в следующей сессии.
>
> ### 🛠 СЛЕДУЮЩАЯ СЕССИЯ — ОТДЕЛЬНАЯ ЗАДАЧА: правка лаунчера (правки юзера ПЕРВИЧНЫ → логика под них)
> **ПЕРВИЧНЫ ПРАВКИ ЮЗЕРА в лаунчере (визуал/кнопки) — это спека, НЕ логика под ними.** Порядок: юзер вносит/хочет правки → под каждой правкой ПРОВЕРИТЬ логику → если логика НЕ соответствует правке, КОРРЕКТИРОВАТЬ ЛОГИКУ под правку юзера (логика подгоняется к правкам, не наоборот). + проверка соответствия (кнопка ↔ что реально делает). (Помнить: Tk тред-безопасность — UI только из UI_QUEUE, SIGSEGV-история; внешняя правка +PID Alice/Aleks не клоберить.)
> 1. Рестарт Ollama (активировать MAX_LOADED=1) + `$0`-тест локального зрения (qwen2.5vl:72b).
> 2. Запустить/расписать Finn Part C sync (против реальных траекторий).
> 3. Живой тест флота в OR (переключить в лаунчере → рестарт → Telegram). Eva/Aleks vision-хендлеров не имеют (Eva генерит, Aleks текст) — норма.
> 4. Решение по коммиту (теги отката `pre-b-rollout-2026-06-18` целы).


> **Команда продолжения (новая сессия):** `начинай OR билд`.
> **Цель:** привести ВСЕХ агентов (включая Freddy) к ЕДИНОМУ OR-стандарту [[how-to-build-an-agent]] §2.1: транспорт B (OpenClaw, реальные тулы/веб) + полный каталог OR-режимов (5) + общая память-шина (Archivarius) + руки + vision.
> **ЖЕЛЕЗНЫЙ ИНВАРИАНТ:** Claude SDK (builder) НЕ ломать. Дефолт `<AGENT>_MODE=builder` = `runAgent` (SDK) байт-в-байт. Тоггл лишь ПЕРЕКЛЮЧАЕТ на OR. Проверено: все 5 флот-агентов на builder отвечают.

---

## ✅ ЧТО СДЕЛАНО (проверено живьём)

1. **Shared-модуль `Shared/agent-engine/`** (транспорт B): `runViaOpenclaw()` (spawn `openclaw agent --json`, парсинг `.status`(верхний уровень!)+`.result.payloads[0].text`) + центральный реестр `MODELS` (правится одной строкой; `pilot=openrouter/deepseek/deepseek-v4-pro`). ESM index.js+index.d.ts, без build-шага.
2. **OpenClaw обвязка:** провайдер `openrouter` (DeepSeek) в `~/.openclaw/openclaw.json`; агенты `aleks/joe/eva/alice/jesse` зарегистрированы (`openclaw agents add --workspace ... --model openrouter/deepseek/deepseek-v4-pro`, БЕЗ --bind), workspace **нейтрализованы** (дефолтные BOOTSTRAP/IDENTITY/SOUL забиты минимальной директивой - иначе модель болтает про файлы).
3. **Topology B врезан + проверен ОБА режима живьём:** **Aleks, Joe, Eva, Alice, Jesse** (5/6). У каждого: `engine.ts` (getEngine+generate), `buildEnrichedPrompt` вынесен (общий для обеих веток), все call-sites→`generate()`, `<AGENT>_MODE=builder` в env, `agent-engine` dep, tsc=0, builder-путь байт-в-байт (нет регрессии). Личности верные в обоих движках.
4. **Jesse vision-инвариант:** `runAgentVision` (графики) ВСЕГДА Claude SDK, структурно не уходит на DeepSeek (адверсари-проверено).
5. **Каноны-доки:** `how-to-build-an-agent.md` (+§2.1 OR-стандарт + Fusion-конфиг), `build-spec-topology-b-rollout.md`, [[project_or_modes_standard]].
6. **Freddy (ранее, 2026-06-18):** чистая развязка `engine.ts`, reason/probe/chat через чокпоинт. **НО на СТАРОМ подходе** (см. раздел Freddy).

## 🛑 ГДЕ ВСТАЛО (не начато / не доделано)

- **Launcher-тоггл** (кнопки `<AGENT>_MODE` per agent) - **НЕ начат**. Остановился на чтении паттерна `ToggleRow` + `_write_freddy_mode` в launcher.pyw. Сейчас режим только правкой env + рестарт. (Для флота тоггл ПРОЩЕ Freddy: просто писать `<AGENT>_MODE=` в env, без LLM_*-блока - ключ в openclaw.json.)
- **Полный каталог OR-режимов (5)** - **НЕ построен.** Сейчас только solo DeepSeek через OpenClaw (pilot). `<AGENT>_OR_MODEL` (выбор режима) не реализован. Fusion/Fusion Max (транспорт A, HTTP-плагин) НЕ в Shared-модуле.
- **Общая память-шина** (`recordTurn`/`recallContext` на Archivarius) - **НЕ применена к флоту.** Joe/Eva/Alice/Jesse читают ЛОКАЛЬНУЮ память в `buildEnrichedPrompt` (континьюити в пределах движка ОК, но НЕ общая шина → кросс-агентная видимость/прожарка неполная).
- **Finn B-интеграция** (Telegram-фронт ре-хоум) - **НЕ начат.**
- **Vision через OR** - не построен (только builder). **Руки** - Finn-специфичны (MCP), не обобщены на флот.

## ⚠️ Freddy - «не совсем верно начали» (привести к стандарту)

Freddy сейчас = **транспорт A ТОЛЬКО** (прямой HTTP OpenRouter, Fusion, БЕЗ тулов), своя `memory.ts` (не Shared), нет OpenClaw/тулов/vision. Стандарт требует:
1. Freddy на `Shared/agent-engine` (как флот) + транспорт B (тулы/веб через OpenClaw) ВДОБАВОК к транспорту A (Fusion для прожарки).
2. Freddy память → общая шина (он пишет в Archivarius, но через свой memory.ts; унифицировать на Shared).
3. **Fusion-конфиг Freddy на СТАРОМ судье (DeepSeek)** - стандарт §2.1: судья = **Gemini 3.1 Pro**, панель Gemini 3.5 Flash + Kimi K2.6 + DeepSeek, финал DeepSeek. Привести.
4. `FREDDY_OR_MODEL` (Fusion / solo) - как у флота.

## 🐞 БАГИ / открытые вопросы (НЕ проработаны)

1. **Память флота локальная**, не на общей шине Archivarius (кросс-агентная видимость/прожарка/общая память при смене движка - неполные).
2. **Freddy на старом транспорте A** + старый Fusion-судья (DeepSeek ≠ стандарт Gemini 3.1 Pro).
3. **OR-каталог (5 режимов) не построен**; `<AGENT>_OR_MODEL` не реализован; Fusion (транспорт A) не в Shared.
4. **Нет live-reload:** после тоггла режима ОБЯЗАТЕЛЕН рестарт агента (config.ts кеширует env на импорте).
5. **Латентность OpenClaw-раннера** (часть вызовов 60-113с); typing-индикатор обязателен в ORmode.
6. **`openclaw agents add` пересевает workspace** при пересоздании - нейтрализацию повторять.
7. **Finn раздвоен** (native OpenClaw-агент + TS-обёртка); при поднятии grammy → getUpdates-конфликт (снять telegram-binding атомарно).
8. **Launcher-тоггл отсутствует** (UI).
9. **recordTurn/recallContext не на флоте** (отложенный инкремент).
10. **Vision через OR не построен** (только builder); **руки** не обобщены.
11. OpenClaw инжектит свой workspace-persona (~токены) - держать workspace минимальным.

## 📦 Состояние репозиториев (НЕ коммитить до первичных тестов юзера)

Не закоммичено: Aleks 6, Joe 6, Eva 8, Alice 6, Jesse 16, Analyst 1 файл. Теги отката `pre-b-rollout-2026-06-18` целы (откат: `git -C <dir> reset --hard pre-b-rollout-2026-06-18`). `launcher.pyw` имеет внешнюю правку (+PID Alice/Aleks, НЕ моя - не трогать). Bob - вне раскатки.

## 🗺 ПЛАН новой сессии «начинай OR билд»

1. **OR-каталог** в `Shared/agent-engine` (ФИНАЛ §2.1): текст-мозг 5 режимов = **1 DeepSeek V4 Pro · 2 Kimi K2.6 (заменил Qwen, видит сам) · 3 Gemini 3.5 Flash · 4 Fusion · 5 Fusion Max**. Solo (1-3) via OpenClaw; Fusion (4-5) via HTTP-плагин `openrouter/fusion` (готовый, НЕ кастом). Ввести `<AGENT>_OR_MODEL`.
2. **Общая память-шина:** вынести `memory.ts` (recordTurn/recallContext, параметризованный agentId/dbPath) в Shared → подключить флоту + Freddy.
3. **Freddy к стандарту:** Shared-модуль, оба транспорта, Fusion судья = Gemini 3.1 Pro.
4. **Launcher-тогглы** per agent (MODE + OR_MODEL) + рестарт.
5. **Finn B** (grammy-фронт + снять telegram-binding).
6. **Vision** (ФИНАЛ, см §2.2): **родное зрение где есть** (Kimi #2 / Gemini #3 / Claude-builder видят сами, без отдельного вызова); **Gemini 3.5 Flash - костыль ТОЛЬКО слепым мозгам** (DeepSeek #1 / Fusion) на картиночный ход; per-ход, один вызов, не релай/не плюсом; через OpenRouter (без Google OAuth). Один хелпер в Shared, заменить стопгап Jesse «vision всегда Claude». **Hands** - Finn-специфичны (отдельно).

Каждый шаг: **builder не ломать**, тестировать ОБА режима живьём (как делали для Aleks/Joe/Eva/Alice/Jesse).
