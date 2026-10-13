package javafx.scene.canvas;
import javafx.scene.paint.Color;
import java.util.ArrayList;
import java.util.List;
/** Stub recording one entry per petal: translate, rotate (deg), scale, fill, rects. */
public class GraphicsContext {
    public static final class Op { public double x, y, deg, scale, r, g, b, a; public List<double[]> rects = new ArrayList<>(); }
    public final List<Op> frame = new ArrayList<>();
    private Op current;
    private Color fill;
    public void clearRect(double x, double y, double w, double h) { frame.clear(); }
    public void save() { current = new Op(); }
    public void restore() { frame.add(current); current = null; }
    public void translate(double x, double y) { current.x = x; current.y = y; }
    public void rotate(double deg) { current.deg = deg; }
    public void scale(double sx, double sy) { current.scale = sx; }
    public void setFill(Color c) { fill = c; current.r = c.getRed(); current.g = c.getGreen(); current.b = c.getBlue(); current.a = c.getOpacity(); }
    public void fillRect(double x, double y, double w, double h) { current.rects.add(new double[]{x, y, w, h}); }
}
