import { useState } from 'react';
import { useController, useUi } from '../app/hooks';
import { Button } from '../components/controls';
import { Icon } from '../components/Icon';
import { t } from '../i18n/strings';

/** Лаунчер не смог прочитать конфиг/папки: показываем причину вместо пустого окна. */
export function FatalScreen() {
  const controller = useController();
  const fatal = useUi((state) => state.fatal) ?? t.unknownError;
  const [copied, setCopied] = useState(false);
  const report = `Valkyrie Launcher\n${navigator.userAgent}\n\n${fatal}`;
  return (
    <div className="fatal" data-tauri-drag-region>
      <section className="fatal-card" role="alert">
        <span className="fatal-icon">
          <Icon name="alert" size={28} />
        </span>
        <h1>{t.fatalTitle}</h1>
        <pre>{fatal}</pre>
        <div className="fatal-actions">
          <Button
            variant="tonal"
            icon={copied ? 'check' : 'copy'}
            onClick={() => void controller.copyText(report).then(setCopied)}
          >
            {copied ? t.fatalCopied : t.fatalCopy}
          </Button>
          <Button variant="filled" onClick={() => controller.closeWindow()}>
            {t.close}
          </Button>
        </div>
      </section>
    </div>
  );
}
