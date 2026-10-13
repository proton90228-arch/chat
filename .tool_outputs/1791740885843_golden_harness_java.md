package com.valkyrie.launcher;

import javafx.animation.AnimationTimer;
import javafx.scene.canvas.GraphicsContext;
import java.util.Locale;

/** Runs the ORIGINAL SakuraBackground with a deterministic frame schedule and prints JSON snapshots. */
public final class GoldenHarness {
    public static void main(String[] args) {
        SakuraBackground bg = new SakuraBackground();
        bg.setWidth(1040);
        bg.setHeight(650);
        AnimationTimer timer = AnimationTimer.RUNNING.get(0);
        GraphicsContext gc = bg.getGraphicsContext2D();
        long now = 1_000_000_000L;
        long[] pattern = {16_666_667L, 16_666_666L, 16_666_667L, 33_333_333L, 8_000_000L, 70_000_000L, 16_666_667L};
        int[] snapshots = {1, 2, 10, 31, 60, 91, 150, 151, 152, 200, 201, 230, 260};
        StringBuilder out = new StringBuilder("{\"seed\":1447119947,\"frames\":[");
        StringBuilder events = new StringBuilder();
        timer.handle(now); // first frame only stores lastFrame
        boolean first = true;
        for (int frame = 1; frame <= 260; frame++) {
            if (frame == 30) { bg.gust(1.0); events.append("gust1.0@30 "); }
            if (frame == 90) { bg.gust(0.35); bg.gust(0.35); events.append("gust0.35x2@90 "); }
            if (frame == 150) { bg.setWidth(820); bg.setHeight(560); }
            if (frame == 200) { bg.setReducedMotion(true); bg.gust(1.0); }
            if (frame == 230) { bg.setReducedMotion(false); bg.setDark(true); }
            long delta = pattern[frame % pattern.length];
            now += delta;
            timer.handle(now);
            boolean snap = false;
            for (int s : snapshots) if (s == frame) snap = true;
            if (!snap) continue;
            if (!first) out.append(',');
            first = false;
            out.append("{\"frame\":").append(frame).append(",\"petals\":[");
            for (int i = 0; i < gc.frame.size(); i++) {
                GraphicsContext.Op op = gc.frame.get(i);
                if (i > 0) out.append(',');
                out.append(String.format(Locale.ROOT, "[%.17g,%.17g,%.17g,%.17g,%.17g,%.6f]", op.x, op.y, Math.toRadians(op.deg), op.scale, op.a, op.r));
            }
            out.append("]}");
        }
        out.append("],\"pattern\":[");
        for (int i = 0; i < pattern.length; i++) out.append(i > 0 ? "," : "").append(pattern[i]);
        out.append("],\"rects\":[");
        GraphicsContext.Op op = gc.frame.get(0);
        for (int i = 0; i < op.rects.size(); i++) {
            double[] r = op.rects.get(i);
            out.append(i > 0 ? "," : "").append(String.format(Locale.ROOT, "[%s,%s,%s,%s]", r[0], r[1], r[2], r[3]));
        }
        out.append("]}");
        System.out.println(out);
    }
}
