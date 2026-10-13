import { useEffect, useMemo, useRef, useState, type KeyboardEvent } from 'react';
import { useController, usePresence, useUi } from '../app/hooks';
import { IconButton, Segmented } from '../components/controls';
import { Icon } from '../components/Icon';
import { filterVersions, rowBadge, rowMeta, triggerTitle, versionKey } from '../core/versions';
import { t } from '../i18n/strings';
import type { GameVersion } from '../ipc/types';

const ROW = 60;
const OVERSCAN = 5;
type Filter = 'releases' | 'snapshots';

/** Библиотека версий: поиск, Releases/Snapshots, виртуальный список, управление с клавиатуры. */
export function VersionLibrary() {
  const open = useUi((state) => state.libraryOpen);
  const reduced = useUi((state) => state.settings.reducedMotion);
  const { mounted, state } = usePresence(open, reduced ? 100 : 200);
  return mounted ? <LibraryDialog state={state} /> : null;
}

function LibraryDialog({ state }: { state: string }) {
  const controller = useController();
  const versions = useUi((s) => s.versions);
  const selected = useUi((s) => s.selected);
  const [query, setQuery] = useState('');
  const [filter, setFilter] = useState<Filter>(!selected.valkyrie && selected.type === 'snapshot' ? 'snapshots' : 'releases');
  const items = useMemo(() => filterVersions(versions, query, filter === 'snapshots'), [versions, query, filter]);
  const selectedIndex = useMemo(() => items.findIndex((v) => versionKey(v) === versionKey(selected)), [items, selected]);
  // null — «активна выбранная версия»; сбрасывается при смене поиска или фильтра.
  const [pointer, setActive] = useState<number | null>(null);
  const active = pointer ?? Math.max(0, selectedIndex);
  const [scrollTop, setScrollTop] = useState(0);
  const [viewport, setViewport] = useState(320);
  const listRef = useRef<HTMLDivElement>(null);
  const searchRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    searchRef.current?.focus();
  }, []);

  // Выбранная версия видна сразу и после смены фильтра.
  useEffect(() => {
    if (selectedIndex >= 0 && listRef.current) listRef.current.scrollTop = Math.max(0, selectedIndex * ROW - ROW * 2);
  }, [selectedIndex]);

  useEffect(() => {
    const list = listRef.current;
    if (!list) return;
    const observer = new ResizeObserver(() => setViewport(list.clientHeight));
    observer.observe(list);
    return () => observer.disconnect();
  }, []);

  const reveal = (index: number) => {
    const list = listRef.current;
    if (!list) return;
    if (index * ROW < list.scrollTop) list.scrollTop = index * ROW;
    else if ((index + 1) * ROW > list.scrollTop + list.clientHeight) list.scrollTop = (index + 1) * ROW - list.clientHeight;
  };

  const choose = (version: GameVersion | undefined) => {
    if (version) void controller.selectVersion(version);
  };

  const onKeyDown = (event: KeyboardEvent) => {
    const last = items.length - 1;
    const move = (index: number) => {
      event.preventDefault();
      const next = Math.max(0, Math.min(last, index));
      setActive(next);
      reveal(next);
    };
    if (event.key === 'ArrowDown') move(active + 1);
    else if (event.key === 'ArrowUp') move(active - 1);
    else if (event.key === 'PageDown') move(active + Math.floor(viewport / ROW));
    else if (event.key === 'PageUp') move(active - Math.floor(viewport / ROW));
    else if (event.key === 'Enter') {
      event.preventDefault();
      choose(items[active]);
    }
  };

  const first = Math.max(0, Math.floor(scrollTop / ROW) - OVERSCAN);
  const last = Math.min(items.length, Math.ceil((scrollTop + viewport) / ROW) + OVERSCAN);

  return (
    <>
      <div className="scrim scrim-modal" data-state={state} onClick={() => controller.setLibrary(false)} aria-hidden="true" />
      <div className="library-layer" data-state={state}>
        <section className="library" role="dialog" aria-modal="true" aria-labelledby="library-title" onKeyDown={onKeyDown}>
          <header className="panel-header">
            <div>
              <p className="eyebrow">{t.libraryEyebrow}</p>
              <h2 className="panel-title" id="library-title">
                {t.libraryTitle}
              </h2>
              <p className="panel-subtitle">{t.libraryDescription}</p>
            </div>
            <IconButton icon="close" label={t.close} onClick={() => controller.setLibrary(false)} />
          </header>
          <div className="library-tools">
            <label className="search">
              <Icon name="search" size={20} />
              <input
                ref={searchRef}
                value={query}
                placeholder={t.searchPlaceholder}
                spellCheck={false}
                aria-controls="library-list"
                aria-activedescendant={items[active] ? `version-${active}` : undefined}
                onChange={(event) => {
                  setQuery(event.currentTarget.value);
                  setActive(null);
                }}
              />
            </label>
            <Segmented<Filter>
              label="Тип версий"
              value={filter}
              onChange={(next) => {
                setFilter(next);
                setActive(null);
              }}
              options={[
                { id: 'releases', label: t.releases },
                { id: 'snapshots', label: t.snapshots },
              ]}
            />
          </div>
          <div
            ref={listRef}
            className="library-list"
            id="library-list"
            role="listbox"
            aria-label={t.libraryTitle}
            onScroll={(event) => setScrollTop(event.currentTarget.scrollTop)}
          >
            {items.length === 0 ? (
              <p className="library-empty">{t.libraryEmpty}</p>
            ) : (
              <div className="library-spacer" style={{ height: items.length * ROW }}>
                {items.slice(first, last).map((version, offset) => {
                  const index = first + offset;
                  const isSelected = versionKey(version) === versionKey(selected);
                  return (
                    <div
                      key={versionKey(version)}
                      id={`version-${index}`}
                      role="option"
                      tabIndex={-1}
                      aria-selected={isSelected}
                      className="version-row"
                      data-active={index === active}
                      data-valkyrie={version.valkyrie}
                      data-snapshot={version.type === 'snapshot'}
                      style={{ transform: `translateY(${index * ROW}px)` }}
                      onPointerMove={() => setActive(index)}
                      onClick={() => choose(version)}
                    >
                      <span className="version-icon">
                        <Icon name={version.valkyrie ? 'sakura' : 'cube'} size={20} />
                      </span>
                      <span className="version-text">
                        <span className="version-title">{triggerTitle(version)}</span>
                        <span className="version-meta">{rowMeta(version)}</span>
                      </span>
                      <span className="badge" data-kind={version.valkyrie ? 'brand' : version.type}>
                        {rowBadge(version)}
                      </span>
                      {isSelected && <Icon name="check" size={20} className="version-check" />}
                    </div>
                  );
                })}
              </div>
            )}
          </div>
        </section>
      </div>
    </>
  );
}
