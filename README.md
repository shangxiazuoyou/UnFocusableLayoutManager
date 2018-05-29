# UnFocusableLayoutManager
解决因RecyclerView的Header高度超出屏幕造成布局向上部分滚动的问题
代码如下：
```
import android.content.Context;
import android.graphics.Rect;
import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;
import android.util.AttributeSet;
import android.view.View;

public class UnFocusableLayoutManager extends LinearLayoutManager {

    public UnFocusableLayoutManager(Context context) {
        super(context);
    }

    public UnFocusableLayoutManager(Context context, int orientation, boolean reverseLayout) {
        super(context, orientation, reverseLayout);
    }

    public UnFocusableLayoutManager(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
    }

    @Override
    public boolean requestChildRectangleOnScreen(RecyclerView parent, View child, Rect rect, boolean immediate) {
        return this.requestChildRectangleOnScreen(parent, child, rect, immediate, false);
    }

    @Override
    public boolean requestChildRectangleOnScreen(RecyclerView parent, View child, Rect rect, boolean immediate, boolean focusedChildVisible) {
        return false;
    }
}
```
使用：
```
RecyclerView.setLayoutManager(new UnFocusableLayoutManager(context));
```
