package javafx.animation;
import java.util.ArrayList;
import java.util.List;
/** Stub: the harness drives handle(now) manually. */
public abstract class AnimationTimer {
    public static final List<AnimationTimer> RUNNING = new ArrayList<>();
    public abstract void handle(long now);
    public void start() { if (!RUNNING.contains(this)) RUNNING.add(this); }
    public void stop() { RUNNING.remove(this); }
}
