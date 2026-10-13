import { useEffect, useLayoutEffect, useRef } from 'react';
import { petalPath, type PetalPaint, type SakuraSim } from '../sakura/sim';

/**
 * Холст с лепестками поверх фона, под интерфейсом. Кадр пропускается после возврата окна
 * из фона (как `setPaused(false)` в Java), шаг ограничен 50 мс внутри симуляции.
 */
export function SakuraField({ sim, paint, reducedMotion }: { sim: SakuraSim; paint: PetalPaint; reducedMotion: boolean }) {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const paintRef = useRef(paint);
  useLayoutEffect(() => {
    paintRef.current = paint;
  }, [paint]);
  useEffect(() => {
    sim.setReducedMotion(reducedMotion);
  }, [sim, reducedMotion]);

  useEffect(() => {
    const canvas = canvasRef.current;
    const context = canvas?.getContext('2d', { alpha: true });
    if (!canvas || !context) return;
    const shape = petalPath();
    let scale = 1;
    let last = 0;
    let frame = 0;

    const resize = () => {
      const rect = canvas.getBoundingClientRect();
      scale = window.devicePixelRatio || 1;
      canvas.width = Math.max(1, Math.round(rect.width * scale));
      canvas.height = Math.max(1, Math.round(rect.height * scale));
      sim.resize(rect.width, rect.height);
      sim.draw(context, paintRef.current, scale, shape);
    };
    const loop = (now: number) => {
      if (last !== 0) sim.step((now - last) / 1000);
      last = now;
      sim.draw(context, paintRef.current, scale, shape);
      frame = requestAnimationFrame(loop);
    };
    const visibility = () => {
      last = 0;
    };
    const observer = new ResizeObserver(resize);
    observer.observe(canvas);
    resize();
    frame = requestAnimationFrame(loop);
    document.addEventListener('visibilitychange', visibility);
    return () => {
      cancelAnimationFrame(frame);
      observer.disconnect();
      document.removeEventListener('visibilitychange', visibility);
    };
  }, [sim]);

  return <canvas ref={canvasRef} className="sakura-field" aria-hidden="true" />;
}
