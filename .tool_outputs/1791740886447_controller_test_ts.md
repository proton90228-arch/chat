import { describe, expect, it } from 'vitest';
import { LauncherController, TOAST_ERROR_MS, TOAST_EXIT_MS, TOAST_MS, type Effects } from '../src/app/controller';
import { VALKYRIE_VERSION } from '../src/core/versions';
import type { Backend, LauncherUpdate } from '../src/ipc/api';
import type { Bootstrap, GameExit, GameVersion, Settings, StatusReport, TaskEvent } from '../src/ipc/types';

const VANILLA: GameVersion = { id: '1.20.1', type: 'release', metadataUrl: 'https://piston-meta/1.20.1.json', metadataSha1: 'ab', valkyrie: false };

const settings = (patch: Partial<Settings> = {}): Settings => ({
  nickname: 'Sakura_Valk', manifestUrl: 'https://example/manifest.json', memoryMb: 4096, automaticMemory: true,
  selectedVersion: '1.21.10', selectedValkyrie: true, javaMode: 'automatic', customJava: '', theme: 'light',
  reducedMotion: false, uiSounds: true, changelogOpen: false, palette: 'sakura', ...patch,
});

const status = (patch: Partial<StatusReport> = {}): StatusReport => ({
  state: 'READY', detail: 'Valkyrie 0.9.0-beta', manifest: null, manifestInstalled: true, changelog: null, warning: null,
  installedLabel: 'Valkyrie 0.9.0-beta', ...patch,
});

class FakeClock {
  now = 0;
  private timers: { at: number; run: () => void }[] = [];
  setTimeout(run: () => void, ms: number): unknown {
    this.timers.push({ at: this.now + ms, run });
    return this.timers.length;
  }
  advance(ms: number): void {
    this.now += ms;
    const due = this.timers.filter((timer) => timer.at <= this.now).sort((a, b) => a.at - b.at);
    this.timers = this.timers.filter((timer) => timer.at > this.now);
    due.forEach((timer) => timer.run());
  }
}

function setup(overrides: Partial<Backend> = {}, boot: Partial<Bootstrap> = {}) {
  const calls: string[] = [];
  const exitHandlers: ((exit: GameExit) => void)[] = [];
  const effects = { gusts: [] as number[], sounds: [] as string[] };
  let current = settings(boot.settings ? boot.settings : {});
  const snapshot = () => ({ settings: current, effectiveMemoryMb: 2048 });
  const backend: Backend = {
    bootstrap: async () => ({
      settings: current, selected: VALKYRIE_VERSION, launcherVersion: '2.0.0', systemMemoryMb: 16384, effectiveMemoryMb: 4096,
      paths: { root: 'R', game: 'R/game', instance: 'R/game', logs: 'R/logs' }, accent: '#0078D4',
      defaultChangelog: ['a', 'b'], bundledClientVersion: '0.9.0-beta', runningPid: null, ...boot,
    }),
    loadVersions: async () => ({ versions: [VALKYRIE_VERSION, VANILLA], fromCache: false }),
    selectVersion: async (version) => {
      calls.push(`select:${version.valkyrie ? 'valkyrie' : version.id}`);
      current = { ...current, selectedVersion: version.id, selectedValkyrie: version.valkyrie };
      return snapshot();
    },
    checkStatus: async () => {
      calls.push('status');
      return status();
    },
    updateSettings: async (patch) => {
      calls.push(`settings:${Object.keys(patch).join(',')}`);
      current = { ...current, ...patch };
      return snapshot();
    },
    setJava: async (mode, path) => {
      calls.push(`java:${mode}:${path ?? ''}`);
      current = { ...current, javaMode: mode, customJava: path ?? '' };
      return snapshot();
    },
    installAndLaunch: async (nickname, onEvent) => {
      calls.push(`launch:${nickname}`);
      onEvent({ kind: 'status', message: 'Скачиваем Minecraft' });
      onEvent({ kind: 'progress', fraction: 0.5 });
      onEvent({ kind: 'stage', message: 'Скачиваем Java 21' });
      return { pid: 1, logPath: 'R/game/logs/minecraft-process.log', gameDirectory: 'R/game' };
    },
    repair: async (onEvent) => {
      calls.push('repair');
      onEvent({ kind: 'progress', fraction: null });
      return { ready: true, installedLabel: 'Valkyrie 0.9.0-beta' };
    },
    openFolder: async (kind) => {
      calls.push(`open:${kind}`);
    },
    onGameExit: async (handler) => {
      exitHandlers.push(handler);
      return () => undefined;
    },
    pickJava: async () => 'C:/jdk/bin/javaw.exe',
    systemAccent: async () => '#107C10',
    copyText: async () => undefined,
    log: () => undefined,
    checkLauncherUpdate: async () => null,
    window: { startDragging: async () => undefined, minimize: async () => undefined, close: async () => undefined, show: async () => undefined },
    ...overrides,
  };
  const clock = new FakeClock();
  const fx: Effects = { gust: (s) => effects.gusts.push(s), sound: (n) => effects.sounds.push(n) };
  const controller = new LauncherController(backend, fx, clock);
  return { controller, calls, exitHandlers, effects, clock };
}

describe('старт', () => {
  it('bootstrap → каталог → выбор сохранённой версии → проверка статуса', async () => {
    const { controller, calls } = setup();
    await controller.start();
    const state = controller.state;
    expect(state.phase).toBe('ready');
    expect(calls).toEqual(['select:valkyrie', 'status']);
    expect(state.versions).toHaveLength(2);
    expect(state.launch).toBe('READY');
    expect(state.detail).toBe('Valkyrie 0.9.0-beta');
    expect(state.changes).toEqual(['a', 'b']);
  });

  it('восстанавливает ванильную версию из настроек', async () => {
    const { controller, calls } = setup({}, { settings: settings({ selectedValkyrie: false, selectedVersion: '1.20.1' }) });
    await controller.start();
    expect(calls[0]).toBe('select:1.20.1');
    expect(controller.state.selected.id).toBe('1.20.1');
  });

  it('каталог недоступен → только Valkyrie и тост', async () => {
    const { controller } = setup({ loadVersions: async () => Promise.reject(new Error('offline')) });
    await controller.start();
    expect(controller.state.versions).toEqual([VALKYRIE_VERSION]);
    expect(controller.state.toasts[0]).toMatchObject({ message: 'Каталог версий недоступен', error: true });
  });

  it('ошибка bootstrap → экран фатальной ошибки', async () => {
    const { controller } = setup({ bootstrap: async () => Promise.reject('нет доступа к %APPDATA%') });
    await controller.start();
    expect(controller.state.phase).toBe('fatal');
    expect(controller.state.fatal).toBe('нет доступа к %APPDATA%');
  });

  it('предупреждение сервера обновлений → validation + не-ошибочный тост', async () => {
    const warning = 'Сервер обновлений недоступен, используется локальная сборка';
    const { controller } = setup({ checkStatus: async () => status({ warning }) });
    await controller.start();
    expect(controller.state.validation).toEqual({ text: warning, error: false });
    expect(controller.state.toasts[0]).toMatchObject({ message: 'Сервер обновлений недоступен', error: false });
  });

  it('игра уже запущена при перезагрузке интерфейса', async () => {
    const { controller, calls } = setup({}, { runningPid: 77 });
    await controller.start();
    expect(controller.state.launch).toBe('RUNNING');
    expect(calls).toEqual([]);
  });
});

describe('запуск', () => {
  it('невалидный никнейм не запускает игру', async () => {
    const { controller, calls } = setup();
    await controller.start();
    controller.setNickname('ab');
    expect(controller.state.nicknameInvalid).toBe(true);
    expect(controller.state.validation).toBeNull();
    await controller.launch();
    expect(controller.state.validation).toEqual({ text: '3–16 латинских букв, цифр или _', error: true });
    expect(calls).not.toContain('launch:ab');
  });

  it('успешный запуск: события задачи, RUNNING, тост, порыв ветра; выход 0 → READY', async () => {
    const { controller, calls, exitHandlers, effects } = setup();
    await controller.start();
    controller.setNickname('  Sakura_Valk ');
    const seen: string[] = [];
    controller.store.subscribe(() => seen.push(controller.state.launch));
    await controller.launch();
    expect(calls).toContain('launch:Sakura_Valk');
    expect(seen).toContain('STARTING');
    expect(seen).toContain('INSTALLING');
    expect(controller.state.launch).toBe('RUNNING');
    expect(controller.state.toasts[0]?.message).toBe('Minecraft успешно запущен');
    expect(effects.gusts).toEqual([1]);
    exitHandlers[0]!({ code: 0, logPath: 'x', ready: true, installedLabel: 'Valkyrie 0.9.0-beta' });
    expect(controller.state.launch).toBe('READY');
  });

  it('обновление клиента → UPDATING', async () => {
    const { controller } = setup({
      checkStatus: async () => status({
        state: 'UPDATE_AVAILABLE', manifestInstalled: false,
        manifest: { version: '1.0.0', minecraftVersion: '1.21.10', forgeProfile: 'f', jarUrl: 'u', sha256: 's', changelog: ['x'] },
        changelog: ['Новое'],
      }),
    });
    await controller.start();
    expect(controller.state.changes).toEqual(['Новое']);
    const seen: string[] = [];
    controller.store.subscribe(() => seen.push(controller.state.launch));
    await controller.launch();
    expect(seen.find((state) => state === 'UPDATING' || state === 'STARTING')).toBe('UPDATING');
  });

  it('ошибка запуска и аварийный выход игры', async () => {
    const { controller, exitHandlers } = setup({ installAndLaunch: async () => Promise.reject(new Error('SHA-1 mismatch')) });
    await controller.start();
    await controller.launch();
    expect(controller.state.launch).toBe('ERROR');
    expect(controller.state.detail).toBe('Ошибка запуска');
    expect(controller.state.validation).toEqual({ text: 'SHA-1 mismatch', error: true });
    exitHandlers[0]!({ code: 1, logPath: 'R/game/logs/minecraft-process.log', ready: true, installedLabel: 'L' });
    expect(controller.state.detail).toBe('Minecraft завершился с кодом 1');
    expect(controller.state.validation?.text).toBe('Minecraft завершился с ошибкой. Лог: R/game/logs/minecraft-process.log');
    expect(controller.state.toasts[0]?.message).toBe('Minecraft завершился с ошибкой 1');
  });

  it('починка закрывает настройки и сообщает о готовности', async () => {
    const { controller, calls } = setup();
    await controller.start();
    controller.toggleSettings();
    await controller.repair();
    expect(calls).toContain('repair');
    expect(controller.state.settingsOpen).toBe(false);
    expect(controller.state.launch).toBe('READY');
    expect(controller.state.toasts[0]?.message).toBe('Minecraft и Forge готовы');
  });
});

describe('панели, тосты и настройки', () => {
  it('тосты: максимум три, авто-скрытие 3.4 с / 5 с', async () => {
    const { controller, clock } = setup();
    controller.toast('1', false);
    controller.toast('2', true);
    controller.toast('3', false);
    controller.toast('4', false);
    expect(controller.state.toasts.map((toast) => toast.message)).toEqual(['4', '3', '2']);
    clock.advance(TOAST_MS);
    expect(controller.state.toasts.filter((toast) => toast.leaving).map((toast) => toast.message)).toEqual(['4', '3']);
    clock.advance(TOAST_EXIT_MS);
    expect(controller.state.toasts.map((toast) => toast.message)).toEqual(['2']);
    clock.advance(TOAST_ERROR_MS - TOAST_MS);
    clock.advance(TOAST_EXIT_MS);
    expect(controller.state.toasts).toEqual([]);
  });

  it('взаимоисключающие панели и порядок ESC', async () => {
    const { controller, calls } = setup();
    await controller.start();
    controller.toggleDrawer();
    expect(calls).toContain('settings:changelogOpen');
    controller.toggleSettings();
    expect(controller.state.drawerOpen).toBe(false);
    controller.setLibrary(true);
    expect(controller.state.settingsOpen).toBe(false);
    expect(controller.closeTopmost()).toBe(true);
    expect(controller.state.libraryOpen).toBe(false);
    expect(controller.closeTopmost()).toBe(false);
  });

  it('библиотека версий недоступна во время установки', async () => {
    let release: () => void = () => undefined;
    const { controller } = setup({
      installAndLaunch: (_nickname: string, _onEvent: (event: TaskEvent) => void) =>
        new Promise((resolve) => {
          release = () => resolve({ pid: 1, logPath: 'l', gameDirectory: 'g' });
        }),
    });
    await controller.start();
    const pending = controller.launch();
    controller.setLibrary(true);
    expect(controller.state.libraryOpen).toBe(false);
    release();
    await pending;
  });

  it('Java: выбор файла и возврат к автоматической', async () => {
    const { controller, calls } = setup();
    await controller.start();
    await controller.chooseJava();
    expect(controller.state.settings.javaMode).toBe('custom');
    await controller.chooseJava();
    expect(calls.filter((call) => call.startsWith('java:'))).toEqual(['java:custom:C:/jdk/bin/javaw.exe', 'java:automatic:']);
  });

  it('память привязывается к шагу 1024 МБ', async () => {
    const { controller, calls } = setup();
    await controller.start();
    await controller.setMemory(5000);
    expect(controller.state.settings.memoryMb).toBe(5120);
    expect(calls).toContain('settings:memoryMb');
  });

  it('обновление лаунчера: найдено → установка', async () => {
    let installed = false;
    const update: LauncherUpdate = { version: '2.1.0', notes: '', install: async (progress) => { progress(0.5); installed = true; } };
    const { controller } = setup({ checkLauncherUpdate: async () => update });
    await controller.start();
    await controller.checkLauncherUpdate(true);
    expect(controller.state.launcherUpdate).toMatchObject({ status: 'available', version: '2.1.0' });
    await controller.installLauncherUpdate();
    expect(installed).toBe(true);
  });

  it('ошибка папки → тост', async () => {
    const { controller } = setup({ openFolder: async () => Promise.reject(new Error('no')) });
    await controller.openFolder('logs');
    expect(controller.state.toasts[0]).toMatchObject({ message: 'Не удалось открыть папку', error: true });
  });
});
