---
title: Build-спец — локальный флот Finn + миграция Freddy на OpenRouter
updated: 2026-06-17
updated_by: builder
status: active
confidence: verified
origin: user
always_load: false
---

> TL;DR: Канонический build-документ для продолжения В НОВОМ КОНТЕКСТНОМ ОКНЕ. Два потока: (1) локальный флот Finn — 3 тира на Ollama/M3 Ultra; (2) миграция Freddy на OpenRouter — 2 равнозначных переключаемых пути (ORmode + BuilderMode-B). Архитектура ЗАФИКСИРОВАНА — осталось СТРОИТЬ по шагам. Модели скачаны, Finn на Q8. Память: [[project_openclaw_local]], [[project_freddy_openrouter]].

# ✅ BUILT (2026-06-17, builder session)

**Поток 2 (Freddy) — СОБРАН ПОЛНОСТЬЮ, tsc 0, dist собран:**
- ⚠️ **ПОПРАВКА К СПЕКУ:** тезис «ORmode = config, ноль кода» был НЕВЕРЕН — `llm-provider.ts` НЕ был подключён ни к чему; Freddy ходил к модели только через SDK `query()` (`agent.ts`), и пайплайн (`research-pipeline.ts`→`runClaude`→`runAgent`) тоже. Поэтому ORmode потребовал РЕАЛЬНОГО подключения чокпоинта.
- **Чокпоинт:** новый `Analyst/src/reason.ts` — ORmode-aware (`isProviderConfigured()` → `runLLM` OpenRouter, иначе `runAgent` SDK). `runClaude` пайплайна делегирует в него. Чат-путь (`agent.ts`) остаётся на SDK (нужны инструменты/MCP).
- **Fusion passthrough:** `llm-provider.ts` — `callOpenAI`/`streamOpenAI` мёржат `plugins`+`extraBody` из env (`LLM_PLUGINS`/`LLM_EXTRA_BODY` как JSON) + per-call override. Схема Fusion ПРОВЕРЕНА по доку OpenRouter: `model:"openrouter/fusion"` + `plugins:[{id:"fusion",analysis_models:[...],model:<judge>}]` (спековый `openrouter/openrouter/fusion` = опечатка двойного префикса).
- **ORmode env:** управляемый блок в `Vault/_config/freddy.env` (между маркерами `# === SYNAPTICUM FREDDY-MODE`), ВЫКЛЮЧЕН (LLM_* закомментированы), `FREDDY_MODE=builder`. Тумблер лаунчера (раз)комментирует строки.
- **Прожарка:** `Analyst/src/probe.ts` — reader последних N ходов любого агента из Archivarius `conversations` + ORmode-aware верификация (stale-data правило в промпте). Команда `/probe <agent> [фокус]` (+ алиасы `/прожарка`).
- **Мост BuilderMode-B:** `Analyst/src/builder-bridge.ts` — делегация ЖИВОМУ Builder через `builder_tasks` со статусом **`pending_live`** (headless-воркер берёт только `pending` → развязка; воркер к тому же не запущен и его BASH-путь мёртвый-Windows). Фоновый поллер авто-доставляет ответ в чат. Команды `/delegate`, `/delresult <id>`. Drain-CLI живого Builder: `Analyst/scripts/builder-inbox.mjs` (`list`/`show`/`answer`/`fail`).
- **Лаунчер:** под карточкой Freddy — тумблер **BuilderMode-B / ORmode** (пишет блок в freddy.env + авто-рестарт Freddy). Логика протестирована (round-trip идемпотентен, JSON Fusion цел).

**Поток 1 (Finn) — тоггл тира + SENIOR-КОНСИЛИУМ СОБРАНЫ:**
- В OpenClaw зарегистрированы 3 модели ollama (Junior `qwen3.6:27b-q8_0` / Middle `llama4:scout`) + провайдер `consilium`/`senior`. `config patch` применён.
- Лаунчер: под карточкой Finn — тумблер **Junior/Middle/Senior**. Junior/Middle = сырые ollama-теги; **Senior = `consilium/senior`** (не свап модели!). `_set_finn_tier` патчит `agents.list` (lossless) + `launchctl kickstart gui/501`. Off-UI-thread. Dry-run валиден. Карточка **Consilium** (health :11500).
- **Senior-консилиум СОБРАН и протестирован** (исправление спека: Senior ≠ одиночная модель, а Mixture-of-Agents): `Finn/scripts/consilium.ts` — локальный OpenAI-совместимый прокси на `127.0.0.1:11500`, который OpenClaw видит как обычную модель (провайдер `consilium`, api `openai-completions`). Пайплайн: Scout (`llama4:scout` `/api/generate`, digest-skip для малых вводов) → совет {qwen3.5:122b, gpt-oss:120b, llama4:scout} НЕЗАВИСИМО (`/api/chat`) → синтез (qwen3.5:122b автономно / Builder через `pending_live`). RAM: **строго одна модель за раз** — `keep_alive:0` (top-level) + поллинг `/api/ps` (`waitVramFree`) между загрузками + startup-sweep. Промежутки в Archivarius (`shared_knowledge` дайджест/синтез, `conversations` ответы совета, chat_id=`consilium:<runId>`).
- **Адверсариальный review (воркфлоу, 40 агентов, 9 подтв. findings) → исправлено:** (1) HIGH сериализация — глобальный single-flight, параллельные запросы в очередь (не 2 модели разом); (2) HIGH синтез на отдельном `SYNTH_CTX=32768` (не 8192); (3) MED Scout в try/catch (фолбэк на raw); (4) MED кап дайджеста `MAX_DIGEST_CHARS=12000`; (5) LOW startup-sweep + waitVramFree нудж каждую итерацию + GET `/v1/models`. tsc 0. Боевой лёгкий e2e на qwen3.6 прошёл (digest-skip→совет→посл. выгрузка/перезагрузка→синтез→валидный OpenAI-ответ + сериализация подтверждена depth=2).
- CAVEAT: статус `pending_live` в `builder_tasks` работает (живая таблица БЕЗ CHECK-констрейнта, в коде-схеме db.ts CHECK есть, но `CREATE IF NOT EXISTS` его не применил к существующей таблице). Если таблицу пересоздадут из db.ts — добавить `pending_live` в CHECK.

**ДОБАВЛЕНО 2026-06-17 (вторая итерация):**
- **4 тира вместо 3** (по запросу: «что в обычном Senior без consilium»): Junior `ollama/qwen3.6:27b-q8_0` / Middle `ollama/llama4:scout` / **Senior `ollama/qwen3.5:122b` (одиночная сильнейшая, ~81GB)** / **Council `consilium/senior` (MoA-прокси)**. Тоггл 4 кнопки (width 210). Senior снова = одиночная модель (не consilium); Council = совет. `_read_finn_tier` мапит по полному model-id.
- **RAM-кнопка/тоггл** под карточкой Consilium: «RAM норма (0)» / «RAM 90GB» (`iogpu.wired_limit_mb=92160`). Нужна для Senior-122b И Council (модели 65-81GB не влезут на GPU при дефолтном ~67GB капе). Механизм: `osascript ... with administrator privileges` → `sysctl -w iogpu.wired_limit_mb=92160 && purge` (один пароль), off-UI-thread. Сбрасывается при ребуте. Приложения НЕ закрывает (вторично +3-5GB, снесло бы Chrome-сессии Joe/Jesse). Полный отжим = `~/Synapticum/free-ram-for-llm.sh`. База: [[mac-llm-heavy-model-ram]] (`Vault/architecture/mac-llm-heavy-model-ram.md`): корень = лимит видеопамяти macOS, НЕ процессы.
- **КРАШ-ФИКС лаунчера (критично):** приложение Synapticum падало каждые 2-3 мин (SIGSEGV в Tcl). Корень: воркер-треды (health-проверки карточек + usage) вызывали `self.after()`/Tk из НЕ-главного потока (Tcl не потокобезопасен); карточка Consilium = 4-й тред усилила гонку. Фикс: воркеры пишут в `UI_QUEUE` (queue.Queue), главный поток разгребает через `_pump_ui` (`after(200,...)`). Проверено: 4.5 мин без крашей (раньше падал каждые 2-3). Урок в памяти [[launcher_tk_thread_safety]]. ПРАВИЛО: новые треды/health-карточки в launcher.pyw — из треда только `UI_QUEUE.put`, никогда Tk.
- finn возвращён на junior (безопасный дефолт; Council/Senior выбирать тогглом когда поднята RAM-кнопка).
- **КОНСОЛИДАЦИЯ UI (по запросу юзера — было два контрола консилиума):** большая карточка-панель Consilium УБРАНА. Остался ОДИН контрол — кнопка **Council** в ряду тиров: при выборе она сама авто-стартует прокси (`_start_consilium_proxy`, если `:11500` не отвечает) + ставит модель. RAM-тоггл переехал под карточку Finn (он для Senior-122b И Council). Прокси добавлен в «Exit & Kill All» (раз карточки-менеджера больше нет). Итог: Junior/Middle/Senior/Council + RAM-тоггл — всё под Finn, без дублей.
- **RAM idle-unload (keep_alive):** проблема — модель НЕ освобождала RAM в простое (Ollama default keep_alive 5мин; 87GB Senior пинил 96GB-машину, мешал переходу на Council). Корень: сервит **Ollama.app**-демон (не LaunchAgent), env подхватывается только при рестарте ВСЕГО дерева app после `launchctl setenv`. Фикс (юзер выбрал «app + env + login-item»): `OLLAMA_KEEP_ALIVE=60s` → простой >60с = выгрузка, активный чат держит (таймер сброс). Персист: LaunchAgent `com.synapticum.ollama-keepalive` + `~/Synapticum/ollama-keepalive-login.sh` (self-heal). Консилиум не затронут (per-request `keep_alive:0` перебивает). Плюс launcher выгружает модель на каждом переключении тира (`_ollama_unload_all`). Провайдер OpenClaw keep_alive слать НЕ умеет (нет в схеме). Деталь в памяти [[project_openclaw_local]] (CORRECTION 2026-06-17) + [[launcher_tk_thread_safety]] рядом.

# ✅ ДОБАВЛЕНО 2026-06-18 (Senior-тесты + контекст + 2-й режим синтеза)
- **Per-tier num_ctx (OpenClaw `models.providers.ollama.models[].params.num_ctx`, доходит до Ollama через gateway, проверено):** Junior `qwen3.6:27b-q8_0`=**262144 (256k, нативный max)**; Middle `llama4:scout`=**524288 (512k, практический RAM-max)**; Senior `qwen3.5:122b`=**131072 (128k, RAM-кап)**. 1M на Scout ПРОВЕРЕН и ОТКЛОНЁН (~30GB swap, модель вылетает из GPU vram 67→42GB → тресхинг). 512k = чистый потолок Scout. Менять = config patch массива models + kickstart.
- **Senior боевые замеры (qwen3.5:122b Q4_K_M, 80.7GB веса):** KV дешёвый (GQA) ~0.024GB/1k. 28k ctx = ~2 мин, 103k ctx = ~10 мин (load 22с + чтение 103k@266 tok/s + ген 26 tok/s), swap ПЛОСКИЙ, фриза НЕТ (вопреки первой опасливой прикидке — память не лимит, лимит = СКОРОСТЬ чтения). Качество: needle+reasoning 4/4 факта среди 980 дистракторов на 103k. Замеры в [[mac-llm-heavy-model-ram]].
- **Контекст-архитектура КОНСИЛИУМА (по схеме юзера; consilium.ts per-stage, т.к. модели идут ПО ОДНОЙ — RAM не делится, каждый этап = вся машина):** `DIGEST_CTX=524288` (Scout ингест большого внешнего входа + компактный дайджест) → `COUNCIL_CTX=131072` (каждый член {122b, gpt-oss, scout} думает на дайджесте, 128k = общий потолок, последовательно) → `SYNTH_CTX=131072` (синтез держит N ответов + финал). `MAX_DIGEST_CHARS=48000` (богатый, но компактный дайджест). Это ВНУТРЕННИЙ контекст консилиума, отдельный от per-tier num_ctx чата.
- **2 РЕЖИМА СИНТЕЗА консилиума (это и был вопрос «кнопка справа vs лампочка»):** кнопка **Council** в тирах = ВКЛ консилиум (сама стартует прокси), НЕ выбирает синтезатора. Старая большая карточка-лампочка = была только вкл/выкл процесса-прокси (УДАЛЕНА). Кто пишет ФИНАЛ — отдельный тумблер **Синтез:Senior / Синтез:Builder** под Finn: `local`=qwen3.5:122b синтезит автономно ($0) / `builder`=панель уходит ЖИВОМУ Builder (мне) через BuilderMode-B. Механизм: control-файл `OpenClaw/consilium-synth.txt`, прокси читает per-run (без рестарта) — `readSynthMode()` в consilium.ts. Проверено: смена файла → /health меняет режим без рестарта.
- ПРИМЕНИТЬ: релонч лаунчера (новый тумблер Синтез); consilium.ts-правки (per-stage ctx) подхватятся при следующем старте прокси (Council). tsx читает исходник, сборка не нужна.

**ЧТОБЫ ПРИМЕНИТЬ:** (1) рестарт Freddy через лаунчер (бежит из `dist/`, уже собран); (2) **релонч лаунчера** (`kill $(pgrep -f launcher.pyw)` + relaunch) — нет live-reload, новые тумблеры/карточки появятся только после переоткрытия; (3) для Senior-тира — включить карточку **Consilium** (стартует прокси на :11500 БЕЗ TEST_MODEL = реальный совет), иначе Finn на Senior упадёт (прокси не отвечает).

**ОСТАЛОСЬ (дни юзера, тяжёлый инференс):** Поток 1 шаги 3-4 (замер реального max-контекста Finn/Scout; GPU-кап LaunchDaemon если 122B/gpt-oss не идут 100% GPU), шаг 5 (compaction+Archivarius-ретрив). **Боевой тяжёлый прогон Senior-консилиума** (реальные 80GB-модели последовательно: запустить Consilium-карточку, дёрнуть Finn на Senior-тире вопросом — проверить что qwen3.5:122b/gpt-oss:120b/scout грузятся ПО ОДНОЙ без OOM; если упрутся в GPU-кап → шаг 4 LaunchDaemon). Боевой тест ORmode (тумблер → реальный запрос к Fusion) — когда юзер захочет потратить OpenRouter-биллинг.

# 0. Текущее состояние (что УЖЕ есть)

Всё на Mac Studio M3 Ultra, 96 GB, канон в `~/Synapticum/`.

- **OpenClaw 2026.6.6** — рантайм агентов. Бинарь `~/.local/bin/openclaw`. Конфиг `~/.openclaw/openclaw.json` — править ТОЛЬКО через `openclaw config patch --file <f.json5>`; для ЗАМЕНЫ массивов нужен флаг `--replace-path <dotpath>` (НЕ `--replace`/`--merge` — их не существует). Гейтвей = LaunchAgent `ai.openclaw.gateway`, дашборд `http://127.0.0.1:18789/`. Перезапуск: `launchctl kickstart -k gui/501/ai.openclaw.gateway`.
- **Ollama 0.30.8** — локальный движок, нативный API `http://127.0.0.1:11434` (к OpenClaw цеплять БЕЗ `/v1` — иначе ломается tool-calling). Автозапуск = LaunchAgent `com.synapticum.ollama` (RunAtLoad, KeepAlive=false). CLI `~/.local/bin/ollama`.
- **Telegram**: Finn = бот `@Finn_synapticumbot` (токен в OpenClaw config), привязка `finn <- telegram`.
- **Модели на диске**: `qwen3.6:27b-q8_0` (29GB), `qwen3.5:122b` (81GB), `gpt-oss:120b` (65GB), `llama4:scout` (67GB) + baked `finn` (Q8). OpenRouter ключ в `/Users/synapticum/Synapticum/OpenClaw/.env` (`sk-or-…`, аккаунт пополнен).
- **Лаунчер** `ClaudeClaw/launcher.pyw` (tkinter, titled «Synapticum») — есть карточка Finn (статус = health-проба `:11434`, клик = вкл/выкл `ollama serve`). НЕТ live-reload → перезапуск: `kill $(pgrep -f launcher.pyw)` + relaunch `~/.local/python-tk/python/bin/python3 .../launcher.pyw`.

---

# 1. ПОТОК 1 — Локальный флот Finn (3 тира)

Одна личность «Finn», 3 тира, грузятся по требованию ПО ОДНОЙ (Ollama выгружает через keep_alive 5 мин → в простое модель 0 RAM).

| Тир | Модель (тег) | RAM | Роль |
|---|---|---|---|
| 🐎 Junior | `qwen3.6:27b-q8_0` | ~33 GB | рутина, руки, лёгкое; лёгкий → реализует ~весь 262K контекста |
| 🧠 Middle | `llama4:scout` | ~67 GB | контекст-машина (10M нативный → ~256-512K на 96GB), задачи посложнее |
| 🎓 Senior | консилиум ↓ | по одной | редкие тяжёлые задачи |

**Senior = локальный Mixture-of-Agents (СОВЕТ независимых ответов — выбрано юзером, НЕ цепочка):**
```
Scout СОБИРАЕТ сырьё → сжатый ДАЙДЖЕСТ (не финальный ответ)
  → каждый из {qwen3.5:122b, gpt-oss:120b, llama4:scout} отвечает НЕЗАВИСИМО по дайджесту
    → СИНТЕЗ
```
Почему: независимость мнений = сила ансамбля (избегаем якорения цепочки). Дайджест мелкий → влезает всем (решает RAM-контекст). Scout первый, т.к. большой контекст держит сырьё; Qwen-синтез последний, т.к. вход уже сжат.

**Синтезатор — 2 режима:** (1) Builder/Claude (я) на высоких ставках; (2) локально-автономный — `qwen3.5:122b` финалит (сильнейший рассуждатель: GPQA 88.4, IFEval 92.6).

**Контекст («бесконечный контекст»):** реальное окно держим адекватным (Finn ~до 262K; Scout ~256K+; тяжёлые ~30-64K). Переполнение → compaction (OpenClaw `agents.defaults.compaction.memoryFlush`) в саммари. Полную историю → Archivarius `conversations` (FTS) для точного recall. Саммари держать МАЛЕНЬКИМ. Задачи юзера ~500K → схема обязательна, не опциональна.

**RAM (упрощённый вариант — решено юзером):** НЕТ кнопки/purge/sudoers. Модели по одной, прошлая выгружается. Дефолтный GPU-кап ~72GB; Finn(33)+Scout(67) влезают свободно → Claude можно держать открытым. Только ~80GB-модели (Qwen 122B 81, gpt-oss 65-82) могут превышать кап → ПОДГОТОВИТЬ (НЕ включать) разовый персистентный `iogpu.wired_limit_mb=88000` через LaunchDaemon, и то только ЕСЛИ тест покажет, что без него не идут 100% GPU. Инференс-тесты тяжёлых — в отдельные дни юзера; связь/пути тестить можно (0 RAM).

## Что построить (поток 1)
1. **Тоггл тира** на карточке Finn в лаунчере (Junior/Middle/Senior) по образцу `ToggleRow`; переключает model OpenClaw-агента finn (`config patch` на `ollama/<tier>` + kickstart гейтвея).
2. **Senior-консилиум**: механизм Scout-дайджест → 3 независимых прогона (ollama, по одной, малый контекст) → синтез (я / qwen3.5:122b); промежуток в Archivarius.
3. **Замер реального max контекста** Finn и Scout (грузить с растущим `num_ctx`, смотреть RAM + `ollama ps`) — отдельный день (инференс).
4. **GPU-кап**: загрузить Qwen 122B/gpt-oss, `ollama ps` (100% GPU?); если нет — LaunchDaemon `iogpu.wired_limit_mb=88000`. Отдельный день.
5. **Compaction + Archivarius-ретрив** под контекст-схему.

---

# 2. ПОТОК 2 — Миграция Freddy на OpenRouter

**КЛЮЧЕВОЙ ФАКТ:** все агенты УЖЕ на подписке (`ANTHROPIC_API_KEY` пуст → Agent SDK берёт `~/.claude.json` OAuth = Pro/Max). SDK ≠ платный API. Реальный смысл ORmode = СНЯТЬ Freddy с общей подписки на отдельный OpenRouter (отдельный биллинг, не ест Claude-лимит).

Freddy уже портативен: `Analyst/src/llm-provider.ts` — vendor-agnostic (`claude`|`openai` + кастомный endpoint + `claude -p` CLI fallback). ORmode = config, НОЛЬ кода ядра.

**ДВА РАВНОЗНАЧНЫХ ПЕРЕКЛЮЧАЕМЫХ ПУТИ в лаунчере (ни один не главнее — выбор юзера):**

## Путь A — ORmode (OpenRouter Fusion)
Env-профиль Freddy (ПОДГОТОВИТЬ ВЫКЛЮЧЕННЫМ — принцип «готовь но не включай»):
```
LLM_PROVIDER=openai
LLM_ENDPOINT=https://openrouter.ai/api/v1/chat/completions
LLM_API_KEY=<OpenRouter ключ из OpenClaw/.env>
LLM_MODEL_DEEP=openrouter/fusion   # НЕ openrouter/openrouter/fusion (двойной префикс = опечатка)
```
**Fusion-конфиг (панель / судья / синтез) - КАНОН в [[how-to-build-an-agent]] §2.1.** НЕ дублировать тут (разъедется). ⚠️ Прежний вариант «судья/синтез = DeepSeek» УСТАРЕЛ: судья = **Gemini 3.1 Pro** (разная модель от панелиста-Gemini → нет самосуждения), финал пишет DeepSeek. Есть и **Fusion Max** (премиум-панель Opus+GPT-5.5+Gemini 3.1 Pro, DRACO 68.3).
- **Малая доделка кода:** в `llm-provider.ts` функции `callOpenAI`/`streamOpenAI` добавить passthrough поля `plugins` в body (для Fusion-плагина). ~5 строк.
Назначение: анализ, прожарка, кросс-сетевые проверки.

## Путь B — BuilderMode (мост к ЖИВОМУ Builder) ← юзер выбрал B
- SDK-на-подписке (текущее) = тоже подписка, но headless. B = делегировать задачу Freddy ЖИВОЙ Builder-сессии (мне, с контекстом/инструментами) — «одеть Freddy как скилл».
- **Моста Freddy→живой Builder в коде НЕТ** — это реальная стройка. Дизайн: очередь задач (Archivarius `builder_tasks` или новый канал) → Builder подхватывает → результат назад. Проверить существующий builder-механизм / построить.

## Прожарка (верификация работы других агентов)
Новый коннектор: читать ПОСЛЕДНИЕ 2-3 хода ЛЮБОГО агента из Archivarius `conversations` (кросс-агентная таблица; НЕ грузить полный контекст агента) → анализ/верификация. «Джесси ответил X — проверь». Либо юзер спрашивает напрямую.

## Возможности (УЖЕ есть)
Коннекторы Freddy: `youtube`, `habr`, `hackernews`, `reddit`, `x`, `blogs` → «YouTube + глубокий поиск» встроены. Брифы (vault-briefs) — как раньше.

## Тумблер в лаунчере
Секция Freddy: выбор пути **ORmode / BuilderMode-B** (равнозначны, переключаемы). Механизм: переключение задаёт LLM_-переменные (ORmode) либо маршрутит на Builder (B).

## Что построить (поток 2)
1. ORmode env-профиль Freddy (ВЫКЛЮЧЕННЫМ).
2. Fusion `plugins`-passthrough в `llm-provider.ts`.
3. Прожарка-коннектор (reader Archivarius `conversations`).
4. Мост BuilderMode-B (Freddy→живой Builder; проверить/построить делегирование).
5. Тумблер ORmode/BuilderMode-B в панель.

**Джо = будущий порт:** у него голый `@anthropic-ai/claude-agent-sdk` `query()` (нет `llm-provider.ts`) → сначала вживить эту абстракцию, потом голый DeepSeek = config. (Делать после рабочего Freddy.)

---

# 3. Гочи / операционные заметки
- OpenClaw `config patch`: замена массивов — `--replace-path <dotpath>` (НЕ `--replace`/`--merge`).
- Ollama→OpenClaw: нативный URL `:11434` БЕЗ `/v1`.
- OpenClaw официальный `curl|bash`-installer багнут (`--min-release-age`) → ставить `npm i -g openclaw` напрямую.
- Ollama pull может зависнуть на финальном манифесте (73 KB/s) → kill + restart (резюмит с кэша, добивает за секунды).
- Лаунчер: нет live-reload → kill+relaunch (агенты не страдают, они detached).
- Гейтвей держит конфиг в памяти → `kickstart` для релоада после `config patch`.
- Все агенты на подписке (ANTHROPIC_API_KEY пуст); ORmode уводит Freddy на отдельный OpenRouter-биллинг.

# 4. Порядок сборки для нового контекста
- **Поток 2 (Freddy)** — облачный, железо НЕ трогает → строить/тестить свободно: ORmode (шаги 1-2-3) → мост B (4) → тумблер (5).
- **Поток 1 (локальный)** — пути/тоггл/консилиум строить можно; инференс-тесты тяжёлых (контекст-замер, GPU-кап) — в дни юзера.
- **Приложение Synapticum** = растить лаунчер: каждая фича → карточка/секция (анти-склероз юзера).

Память: [[project_openclaw_local]], [[project_freddy_openrouter]], [[project_analyst]], [[project_archivarius]], [[project_agent_hierarchy]], [[project_multi_provider]].
