import { useController, usePresence, useUi } from '../app/hooks';
import { IconButton } from '../components/controls';
import { Icon } from '../components/Icon';
import { t } from '../i18n/strings';

/** Боковая панель «Что нового» (changelog из манифеста или встроенный список). */
export function ChangelogDrawer() {
  const controller = useController();
  const open = useUi((state) => state.drawerOpen);
  const reduced = useUi((state) => state.settings.reducedMotion);
  const changes = useUi((state) => state.changes);
  const version = useUi((state) => state.manifestVersion ?? state.bundledClientVersion);
  const { mounted, state } = usePresence(open, reduced ? 100 : 220);
  if (!mounted) return null;
  return (
    <>
      <div className="scrim" data-state={state} onClick={() => controller.setDrawer(false)} aria-hidden="true" />
      <aside className="panel drawer" data-state={state} role="dialog" aria-modal="true" aria-label={t.changelogTitle}>
        <header className="panel-header">
          <div>
            <p className="eyebrow">{t.changelogEyebrow}</p>
            <h2 className="panel-title">{t.changelogTitle}</h2>
            {version && <p className="panel-subtitle">Valkyrie {version}</p>}
          </div>
          <IconButton icon="close" label={t.close} onClick={() => controller.setDrawer(false)} />
        </header>
        <ul className="changes">
          {changes.map((change, index) => (
            <li key={`${index}-${change}`} style={{ ['--i' as string]: index }}>
              <Icon name="sakura" size={16} />
              <span>{change}</span>
            </li>
          ))}
        </ul>
      </aside>
    </>
  );
}
