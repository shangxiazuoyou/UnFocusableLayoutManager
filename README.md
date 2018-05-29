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
原因解析：
在调用RecyclerView的notifyDataSetChange方法的时候本身进行一段位置的滑动，首先我确认了代码中没有任何的地方会调用RecyclerView的定位以及滚动方法。网上也没有查到方法，只好给recyclerView设置滚动监听断点跟踪源码。在跟踪了多个方法后找到了异常调用源码的关键方法，下面是它的源码
```
/**
         * Requests that the given child of the RecyclerView be positioned onto the screen. This
         * method can be called for both unfocusable and focusable child views. For unfocusable
         * child views, focusedChildVisible is typically true in which case, layout manager
         * makes the child view visible only if the currently focused child stays in-bounds of RV.
         * @param parent The parent RecyclerView.
         * @param child The direct child making the request.
         * @param rect The rectangle in the child's coordinates the child
         *              wishes to be on the screen.
         * @param immediate True to forbid animated or delayed scrolling,
         *                  false otherwise
         * @param focusedChildVisible Whether the currently focused view must stay visible.
         * @return Whether the group scrolled to handle the operation
         */
        public boolean requestChildRectangleOnScreen(RecyclerView parent, View child, Rect rect,
                boolean immediate,
                boolean focusedChildVisible) {
            int[] scrollAmount = getChildRectangleOnScreenScrollAmount(parent, child, rect,
                    immediate);
            int dx = scrollAmount[0];
            int dy = scrollAmount[1];//<span style="color:#FF6666;">在出现莫名滑动时这个值不为0</span>
            if (!focusedChildVisible || isFocusedChildVisibleAfterScrolling(parent, dx, dy)) {
                if (dx != 0 || dy != 0) {
                    if (immediate) {
                        parent.scrollBy(dx, dy);
                    } else {
                        parent.smoothScrollBy(dx, dy);//<span style="background-color:rgb(255,102,102);">通过这个方法调用recyclerView滑动</span>
                    }
                    return true;
                }
            }
            return false;
        }
```
requestChildRectangleOnScreen这个是RecyclerView LayoutManager 中的方法，源码通过方法getChildRectangleOnScreenScrollAmount获得child的一个scroll数组int[] scrollAmount;当出现异常滑动时dy = scrollAmount[1] 的值便不为0，注意这个方法是public的，所以我们通过重写它就可以解决异常滑动了。
然后我们看看这个方法是在哪里调用的
```
 /**
     * Requests that the given child of the RecyclerView be positioned onto the screen. This method
     * can be called for both unfocusable and focusable child views. For unfocusable child views,
     * the {@param focused} parameter passed is null, whereas for a focusable child, this parameter
     * indicates the actual descendant view within this child view that holds the focus.
     * @param child The child view of this RecyclerView that wants to come onto the screen.
     * @param focused The descendant view that actually has the focus if child is focusable, null
     *                otherwise.
     */
    private void requestChildOnScreen(@NonNull View child, @Nullable View focused) {
        View rectView = (focused != null) ? focused : child;
        mTempRect.set(0, 0, rectView.getWidth(), rectView.getHeight());

        // get item decor offsets w/o refreshing. If they are invalid, there will be another
        // layout pass to fix them, then it is LayoutManager's responsibility to keep focused
        // View in viewport.
        final ViewGroup.LayoutParams focusedLayoutParams = rectView.getLayoutParams();
        if (focusedLayoutParams instanceof LayoutParams) {
            // if focused child has item decors, use them. Otherwise, ignore.
            final LayoutParams lp = (LayoutParams) focusedLayoutParams;
            if (!lp.mInsetsDirty) {
                final Rect insets = lp.mDecorInsets;
                mTempRect.left -= insets.left;
                mTempRect.right += insets.right;
                mTempRect.top -= insets.top;
                mTempRect.bottom += insets.bottom;
            }
        }

        if (focused != null) {
            offsetDescendantRectToMyCoords(focused, mTempRect);
            offsetRectIntoDescendantCoords(child, mTempRect);
        }
        mLayout.requestChildRectangleOnScreen(this, child, mTempRect, !mFirstLayoutComplete,
                (focused == null));//<span style="background-color:rgb(255,102,102);">注意这是layoutmanager中的方法</span>
    }
```
接下来看看mLayout在哪里被设置的
```
/**
     * Set the {@link LayoutManager} that this RecyclerView will use.
     *
     * <p>In contrast to other adapter-backed views such as {@link android.widget.ListView}
     * or {@link android.widget.GridView}, RecyclerView allows client code to provide custom
     * layout arrangements for child views. These arrangements are controlled by the
     * {@link LayoutManager}. A LayoutManager must be provided for RecyclerView to function.</p>
     *
     * <p>Several default strategies are provided for common uses such as lists and grids.</p>
     *
     * @param layout LayoutManager to use
     */
    public void setLayoutManager(LayoutManager layout) {
        if (layout == mLayout) {
            return;
        }
        stopScroll();
        // TODO We should do this switch a dispatchLayout pass and animate children. There is a good
        // chance that LayoutManagers will re-use views.
        if (mLayout != null) {
            // end all running animations
            if (mItemAnimator != null) {
                mItemAnimator.endAnimations();
            }
            mLayout.removeAndRecycleAllViews(mRecycler);
            mLayout.removeAndRecycleScrapInt(mRecycler);
            mRecycler.clear();

            if (mIsAttached) {
                mLayout.dispatchDetachedFromWindow(this, mRecycler);
            }
            mLayout.setRecyclerView(null);
            mLayout = null;
        } else {
            mRecycler.clear();
        }
        // this is just a defensive measure for faulty item animators.
        mChildHelper.removeAllViewsUnfiltered();
        mLayout = layout; //<span style="background-color:rgb(255,102,102);">在这里将layout赋给mLayout</span>
        if (layout != null) {
            if (layout.mRecyclerView != null) {
                throw new IllegalArgumentException("LayoutManager " + layout +
                        " is already attached to a RecyclerView: " + layout.mRecyclerView);
            }
            mLayout.setRecyclerView(this);
            if (mIsAttached) {
                mLayout.dispatchAttachedToWindow(this);
            }
        }
        mRecycler.updateViewCacheSize();
        requestLayout();
    }
```
看到这里是不是很熟悉的感觉，setLayoutManager(LayoutManager layout)这个不就是我们初始化RecyclerView时设置的LayoutManager吗?
