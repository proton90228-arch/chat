# Архитектура

```
┌──────────────────────────── ui/ (WebView2) ────────────────────────────┐
│ React-экраны ── useUi(selector) ──► LauncherController (app/controller) │
│   PlayButton · VersionLibrary · SettingsPanel · ChangelogDrawer · Toasts│
│   SakuraField (canvas, sim.ts) · тема (theme/: HCT → токены M3)         │
│                       ▼ Backend (ipc/api.ts)                            │
│          tauri.ts (invoke / Channel / listen)  |  mock.ts (браузер)     │
└───────────────────────────────┬────────────────────────────────────────┘
                                │ IPC: команды, Channel<TaskEvent>, событие game-exited
┌───────────────────────── src-tauri/ (Rust) ─────────────────────────────┐
│ commands.rs: AppState { config, services, manifest, running, busy }     │
│ main.rs: single-instance, window-state, dialog, clipboard, updater       │
│ accent.rs: акцент Windows (DWM) для палитры «Windows»                    │
└───────────────────────────────┬────────────────────────────────────────┘
┌──────────────────────── crates/valkyrie-core ───────────────────────────┐
│ config · catalog · net · install · profile · java · launch · update     │
│ status (LaunchState, check_status / install_and_launch / repair)        │
│ reporter (TaskEvent) · log · paths · properties · uuid · system         │
└─────────────────────────────────────────────────────────────────────────┘
```

## IPC

| Команда | Что делает |
| --- | --- |
| `bootstrap` | конфиг, выбранная версия, пути, акцент, changelog по умолчанию, PID запущенной игры |
| `load_versions` | каталог Mojang (`cache/version_manifest_v2.json` при офлайне) |
| `select_version` | сохранить выбор, вернуть снимок настроек |
| `check_status` | `StatusReport`: состояние кнопки, манифест, changelog, предупреждение |
| `update_settings(patch)` | частичное обновление настроек, атомарная запись `launcher.properties` |
| `set_java(mode, path)` | автоматическая или своя Java (с проверкой файла) |
| `install_and_launch(nickname, onEvent)` | установка/обновление + запуск; прогресс через `Channel<TaskEvent>` |
| `repair(onEvent)` | переустановка с проверкой хэшей |
| `open_folder(kind)` | `game` или `logs` в Проводнике |
| `system_accent`, `log_ui` | акцент Windows; запись ошибок интерфейса в `logs/launcher.log` |

`TaskEvent`: `status` (текст), `stage` (новая фаза, прогресс с нуля), `progress` (`fraction` 0…1 или `null`).
Событие `game-exited`: `{ code, logPath, ready, installedLabel }`. Примеры JSON лежат в `contracts/`.

## Состояния кнопки

`CHECKING → READY | INSTALL_REQUIRED | UPDATE_AVAILABLE | UNAVAILABLE | ERROR`,
запуск: `STARTING / UPDATING → INSTALLING → RUNNING → READY | ERROR`. Тексты, иконки и флаги
(`actionable`, `busy`) совпадают с `LaunchState.java` (`ui/src/core/launchState.ts`).

## Файлы на диске

```
%APPDATA%\ValkyrieLauncher\
  launcher.properties        настройки (формат java.util.Properties)
  logs\launcher.log          журнал лаунчера (ротация после 2 МБ)
  game\                      Minecraft, Forge, mods, assets, libraries
    logs\                    лог процесса Minecraft
    runtimes\<major>\        скачанные Java
  cache\                     кэш каталога версий
```

## Тема

`theme/hct.ts` и `theme/scheme.ts` строят схемы Material You из seed-цвета (HCT, tonal palettes);
тест сверяет 8 схем с эталоном из Python-реализации (`ui/tests/fixtures/schemes.json`).
Статические токены (`styles/tokens.css`) содержат пресеты; палитра «Windows» вычисляется на лету.
