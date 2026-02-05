# Auto-Start для звонков (Native + Web): технический Plan of Action

## 1) Что уже есть в кодовой базе (baseline)

### Точка входа запуска записи
- Frontend вызывает Tauri-команды через `RecordingService`:
  - `start_recording`
  - `start_recording_with_devices_and_meeting`.
- В Rust-командах `frontend/src-tauri/src/lib.rs` запуск в итоге делегируется в `audio::recording_commands::start_recording_with_devices_and_meeting(...)`.

### Что делает backend при старте записи
- Проверяет, что запись еще не идет (`IS_RECORDING`/`is_recording`).
- Валидирует готовность модели транскрипции.
- Разрешает устройства (предпочтительные/дефолтные mic + system audio).
- Создает `RecordingManager`, который:
  - поднимает audio pipeline,
  - стартует стримы микрофона/системного звука,
  - запускает мониторинг аудио-девайсов,
  - запускает задачу транскрипции.

### Уже существующий сигнал для авто-детекта
- В проекте есть `SystemAudioDetector` (macOS-only реализация), который умеет сообщать:
  - `SystemAudioStarted(Vec<String>)` — список приложений, которые выводят звук,
  - `SystemAudioStopped`.
- На non-macOS сейчас заглушка.

## 2) Цель Auto-Start

Автоматически стартовать запись, когда пользователь **реально в звонке**, для двух классов источников:
1. Native apps (Zoom/Telegram/Slack desktop).
2. Web apps (Google Meet / Zoom Web / Teams Web в браузере).

Ключевое требование: минимизировать ложные срабатывания (музыка/YouTube/просто открытая вкладка) и не пропускать реальные звонки.

## 3) Предлагаемая архитектура

Ввести новый backend-модуль: `call_detection` (Rust, внутри `src-tauri/src`).

### 3.1 Компоненты

1. **CallDetectionOrchestrator** (основной координатор)
   - периодический цикл (например 1 Hz);
   - собирает сигналы от детекторов;
   - считает `confidence score`;
   - управляет state machine (`Idle -> Candidate -> InCallConfirmed -> Cooldown`).

2. **Signal Providers**
   - **Audio Activity Provider**
     - macOS: использовать существующий `SystemAudioDetector` + расширение метаданных;
     - Windows: WASAPI session scan (какие процессы имеют активные audio sessions);
     - Linux: PulseAudio/PipeWire clients (через `pactl`/native bindings) при наличии.
   - **Process Provider**
     - проверка наличия процессов `zoom`, `slack`, `telegram`, browser-процессов.
   - **Window/Title Provider**
     - поиск top-level окон и анализ title/class/executable.
     - особенно важно для web-call: title содержит `Meet`, `Microsoft Teams`, `Zoom Meeting` и т.п.
   - **(Опционально) Mic Usage Provider**
     - эвристика: процесс одновременно имеет output+input activity.

3. **Call Classifier**
   - На входе набор сигналов.
   - На выходе:
     - `CallKind`: `Native(Zoom|Slack|Telegram|Teams|Unknown)` или `Web(GoogleMeet|ZoomWeb|TeamsWeb|Unknown)`;
     - `confidence` (0..100);
     - `reason_codes` для дебага/телеметрии.

4. **AutoStart Controller**
   - При переходе в `InCallConfirmed` делает безопасный вызов существующей команды старта записи.
   - Предотвращает повторный старт (debounce + lock).

### 3.2 Новые настройки (store)

Расширить `RecordingPreferences`:
- `auto_start_enabled: bool` (default `false`)
- `auto_start_mode: "off" | "all_calls" | "native_only" | "web_only"`
- `auto_start_confidence_threshold: u8` (default 70)
- `auto_start_grace_period_sec: u16` (например 4–8 сек)
- `auto_start_cooldown_sec: u16` (например 60 сек)
- `auto_start_allowed_apps: Vec<String>` (whitelist)
- `auto_start_blocked_apps: Vec<String>` (blacklist)

## 4) Логика детекта для Native Apps

### 4.1 Сигналы

- `P1`: целевой процесс запущен (`zoom`, `slack`, `telegram`, `teams`).
- `P2`: у процесса есть активная audio output session (не просто процесс в трее).
- `P3`: у процесса есть top-level окно с call-like заголовком (например `Meeting`, `Call`, `Huddle`, `Voice`).
- `P4` (усилитель): одновременно есть input activity (микрофон) у этого же процесса.

### 4.2 Решение

Считать звонок подтвержденным, если, например:
- `P2` + (`P3` или `P4`) удерживаются > `grace_period`.

Это фильтрует:
- случай, когда Zoom открыт, но звонка нет;
- случай, когда приложение просто проигрывает уведомление.

## 5) Логика детекта для Web Apps

### Проблема
Браузер мультипроцессный, а Tauri/Rust из коробки не знает URL вкладки.

### 5.1 Практичная стратегия без расширения браузера (Phase 1)

Комбинировать:
- `W1`: активная аудио-сессия браузера (Chrome/Edge/Firefox).
- `W2`: наличие видимого browser window с call-паттерном в title:
  - Google Meet: `"Meet"`, `"Google Meet"`
  - Teams Web: `"Microsoft Teams"`, `"Teams"`
  - Zoom Web: `"Zoom"`, `"Zoom Meeting"`
- `W3`: устойчивый уровень audio activity > N секунд.

Подтверждать web-call при `W1 + W2 (+W3)`.

### 5.2 Надежная стратегия с browser extension (Phase 2, recommended)

Добавить optional companion extension:
- extension ловит `tabs.onUpdated`, `tab.url`, `tab.audible`, `tab.mutedInfo`;
- при URL матчах (`meet.google.com`, `teams.microsoft.com`, `app.zoom.us/wc`) отправляет событие в локальный `localhost` endpoint приложения;
- Rust получает точный сигнал `web_call_started/web_call_ended`.

Плюс: почти идеальная точность.
Минус: нужно установить extension (UX + support).

## 6) State machine и анти-флаппинг

Состояния:
- `Idle`
- `CandidateDetected` (порог не достигнут)
- `InCallConfirmed`
- `RecordingStartedByAutoStart`
- `Cooldown`

Правила:
- В `CandidateDetected` держать таймер подтверждения.
- Старт записи только один раз за сессию (`call_session_id`).
- После завершения/потери сигнала — вход в `Cooldown`, чтобы не триггериться от кратких шумов.
- Если пользователь вручную остановил запись, не перезапускать автостарт до следующей новой call-сессии.

## 7) Интеграция с текущим кодом

1. Добавить `call_detection` модуль и фоновую задачу инициализации при старте приложения.
2. В `lib.rs` при `setup`:
   - создать shared-state детектора;
   - старт/стоп детектора по настройке `auto_start_enabled`.
3. При событии `InCallConfirmed` вызывать уже существующий путь старта записи:
   - `start_recording_with_devices_and_meeting(...)`.
4. Meeting name для auto-start:
   - генерировать `"Auto: <app/provider> <timestamp>"` либо оставить существующий дефолт.
5. Добавить новые Tauri-команды:
   - `get_auto_start_status`
   - `set_auto_start_enabled`
   - `get_last_detection_reasons` (для дебага в UI).
6. В telemetry (уже есть analytics модуль):
   - `auto_start_candidate_detected`
   - `auto_start_triggered`
   - `auto_start_suppressed` (+reason).

## 8) Платформенная реализация

### macOS
- Использовать существующий системный аудио детектор как базу.
- Добавить Window enumeration (CGWindowList/AX API) для title-based сигналов.
- Для web-calls: связка Browser audio + Window title.

### Windows
- WASAPI session enumeration (через windows-rs/Win32 APIs) для `pid -> audio session active`.
- Foreground/top-level window enumeration (EnumWindows/GetWindowText/GetWindowThreadProcessId).

### Linux
- PipeWire/PulseAudio через pactl/native API для активных sink inputs.
- X11/Wayland окно-метаданные ограничены: сделать degrade path
  - если нет window-title, опираться на process+audio эвристику,
  - понизить confidence и требовать более длинный grace period.

## 9) Надежность, приватность, UX

- **Приватность**: не читать контент экрана/аудио как данные для детекта; только системные метаданные (процесс, наличие аудио-сессии, title).
- **Прозрачность**: в UI показывать "Почему авто-старт сработал" (reason codes).
- **Safe default**: feature выключена по умолчанию.
- **Kill switch**: быстрый toggle в tray/settings.

## 10) Этапы внедрения

### Phase 1 (MVP, без расширения)
- macOS + Windows:
  - process + audio-session + window-title scoring;
  - state machine + auto-start trigger;
  - UI toggle + debug reasons.

### Phase 2
- Linux parity.
- Улучшение словаря title-паттернов и allow/block lists.
- Аналитика качества (precision/false positives).

### Phase 3 (optional, max reliability)
- Browser extension companion для точного web-call детекта.

## 11) Acceptance критерии

- Zoom desktop в реальном звонке → автостарт ≤ 8 сек.
- Zoom открыт без звонка 30 мин → автостарт не происходит.
- Google Meet во вкладке с активным разговором → автостарт ≤ 8 сек.
- YouTube/Spotify в браузере → автостарт не происходит.
- После ручного stop во время звонка → нет немедленного повторного auto-start.

## 12) Риски и как снижать

- **Ложные срабатывания от мультимедиа** → требовать комбинированный сигнал (audio + title/process + time).
- **Различия платформенных API** → trait-based providers + per-OS impl.
- **Wayland ограничения** → graceful degradation + extension как fallback для web.
- **Регрессии UX** → feature flag + telemetry + staged rollout.

---

## Короткий вывод

В рамках текущего стека Tauri + Rust наиболее практичный путь — добавить **multi-signal detector** (process + audio session + window title) и запускать текущий pipeline записи через уже существующие команды. Для web-звонков это даст рабочий baseline; для «Krisp-level» надежности целесообразно предусмотреть optional browser extension как Phase 3.
