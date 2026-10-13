import { useMemo } from 'react';
import { useController, usePresence, useUi } from '../app/hooks';
import { Button, IconButton, SettingRow, Slider, Switch } from '../components/controls';
import { Icon } from '../components/Icon';
import { javaText, memoryText } from '../core/format';
import { LAUNCH_STATES } from '../core/launchState';
import { t } from '../i18n/strings';
import { buildScheme, PRESETS, type PresetId } from '../theme/scheme';

const MEMORY_MIN = 2048;
const MEMORY_MAX = 8192;

function Swatch(props: { id: string; seed: string; name: string; kanji: string; dark: boolean; selected: boolean; onSelect: () => void }) {
  const { id, seed, name, kanji, dark, selected, onSelect } = props;
  const scheme = useMemo(() => buildScheme(seed)[dark ? 'dark' : 'light'], [seed, dark]);
  return (
    <button
      type="button"
      role="radio"
      aria-checked={selected}
      className="swatch"
      title={`${name} ${kanji}`}
      aria-label={name}
      data-palette={id}
      onClick={onSelect}
      style={{
        ['--sw-primary' as string]: scheme.primary,
        ['--sw-secondary' as string]: scheme['secondary-container'],
        ['--sw-tertiary' as string]: scheme['tertiary-container'],
        ['--sw-on' as string]: scheme['on-primary'],
      }}
    >
      <span className="swatch-disc" aria-hidden="true">
        {selected && <Icon name="check" size={16} />}
      </span>
    </button>
  );
}

function LauncherUpdateRow() {
  const controller = useController();
  const update = useUi((state) => state.launcherUpdate);
  const busy = useUi((state) => LAUNCH_STATES[state.launch].busy);
  const hint =
    update.status === 'available' && update.version
      ? t.updateReady(update.version)
      : update.status === 'installing'
        ? `${t.updateInstalling}${update.fraction !== null ? ` ${Math.round(update.fraction * 100)}%` : ''}`
        : update.status === 'latest'
          ? t.upToDate
          : t.aboutRuntime;
  return (
    <SettingRow icon="update" title={t.launcherName} hint={hint}>
      {update.status === 'available' || update.status === 'installing' ? (
        <Button variant="filled" disabled={busy || update.status === 'installing'} onClick={() => void controller.installLauncherUpdate()}>
          {t.installUpdate}
        </Button>
      ) : (
        <Button variant="tonal" disabled={update.status === 'checking'} onClick={() => void controller.checkLauncherUpdate()}>
          {t.checkUpdates}
        </Button>
      )}
    </SettingRow>
  );
}

/** Настройки: выпадающая панель из строки заголовка (как в Java), стиль — M3 Expressive. */
export function SettingsPanel() {
  const open = useUi((state) => state.settingsOpen);
  const reduced = useUi((state) => state.settings.reducedMotion);
  const { mounted, state } = usePresence(open, reduced ? 100 : 160);
  return mounted ? <SettingsContent state={state} /> : null;
}

function SettingsContent({ state }: { state: string }) {
  const controller = useController();
  const settings = useUi((s) => s.settings);
  const effectiveMemoryMb = useUi((s) => s.effectiveMemoryMb);
  const accent = useUi((s) => s.accent);
  const launcherVersion = useUi((s) => s.launcherVersion);
  const blocked = useUi((s) => LAUNCH_STATES[s.launch].busy || s.launch === 'RUNNING');
  const dark = settings.theme === 'dark';
  const palettes: { id: string; seed: string; name: string; kanji: string }[] = Object.entries(PRESETS).map(
    ([id, preset]) => ({ id, seed: preset.seed, name: preset.name, kanji: preset.kanji }),
  );
  if (accent) palettes.push({ id: 'windows', seed: accent, name: t.paletteSystem, kanji: '⊞' });

  return (
    <>
      <div className="scrim scrim-light" data-state={state} onClick={() => controller.toggleSettings()} aria-hidden="true" />
      <section className="panel settings" data-state={state} role="dialog" aria-modal="true" aria-labelledby="settings-title">
        <header className="panel-header">
          <h2 className="panel-title" id="settings-title">
            {t.settings}
          </h2>
          <IconButton icon="close" label={t.close} onClick={() => controller.toggleSettings()} />
        </header>
        <div className="settings-scroll">
          <h3 className="section-label">{t.appearance}</h3>
          <div className="section">
            <SettingRow icon="moon" title={t.darkTheme} hint={t.darkThemeHint}>
              <Switch label={t.darkTheme} checked={dark} onChange={(value) => void controller.setDark(value)} />
            </SettingRow>
            <SettingRow icon="palette" title={t.palette} hint={t.paletteHint} />
            <div className="swatches" role="radiogroup" aria-label={t.palette}>
              {palettes.map((palette) => (
                <Swatch
                  key={palette.id}
                  {...palette}
                  dark={dark}
                  selected={settings.palette === palette.id || (palette.id === 'sakura' && !(settings.palette in PRESETS) && settings.palette !== 'windows')}
                  onSelect={() => void controller.setPalette(palette.id as PresetId | 'windows')}
                />
              ))}
            </div>
            <SettingRow icon="motion" title={t.reducedMotion} hint={t.reducedMotionHint}>
              <Switch label={t.reducedMotion} checked={settings.reducedMotion} onChange={(value) => void controller.setReducedMotion(value)} />
            </SettingRow>
          </div>

          <h3 className="section-label">{t.sound}</h3>
          <div className="section">
            <SettingRow icon="volume" title={t.uiSounds} hint={t.uiSoundsHint}>
              <Button variant="text" className="sound-preview" onClick={() => controller.sound('click')}>
                {t.preview}
              </Button>
              <Switch label={t.uiSounds} checked={settings.uiSounds} onChange={(value) => void controller.setUiSounds(value)} />
            </SettingRow>
          </div>

          <h3 className="section-label">{t.minecraft}</h3>
          <div className="section">
            <SettingRow icon="memory" title={t.memory} hint={memoryText(settings, effectiveMemoryMb)} />
            <SettingRow icon="check" title={t.automaticAllocation}>
              <Switch
                label={t.automaticAllocation}
                checked={settings.automaticMemory}
                onChange={(value) => void controller.setAutomaticMemory(value)}
              />
            </SettingRow>
            <Slider
              label={t.memory}
              min={MEMORY_MIN}
              max={MEMORY_MAX}
              step={1024}
              value={Math.max(MEMORY_MIN, Math.min(MEMORY_MAX, settings.automaticMemory ? effectiveMemoryMb : settings.memoryMb))}
              disabled={settings.automaticMemory}
              format={(value) => `${value / 1024} GB`}
              onChange={(value) => void controller.setMemory(value)}
            />
            <SettingRow icon="java" title={t.java} hint={javaText(settings)}>
              <Button variant="tonal" onClick={() => void controller.chooseJava()}>
                {t.change}
              </Button>
            </SettingRow>
            <SettingRow icon="repair" title={t.gameInstallation} hint={t.verifyRepair}>
              <Button variant="tonal" disabled={blocked} onClick={() => void controller.repair()}>
                {t.repair}
              </Button>
            </SettingRow>
          </div>

          <h3 className="section-label">{t.about}</h3>
          <div className="section">
            <LauncherUpdateRow />
            <p className="about-line">
              {t.launcherName} {launcherVersion} · {t.aboutRuntime}
            </p>
          </div>
        </div>
      </section>
    </>
  );
}
