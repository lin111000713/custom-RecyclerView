# ViewPager源码分析
## 0.使用demo

链接：[使用实例](http://www.jianshu.com/p/6b1008fcc082)

1.准备布局文件

```java

<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="cn.ucai.viewpager.MainActivity">
    <android.support.v4.view.ViewPager
        android:id="@+id/vpGoods"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
</RelativeLayout>

```



2.准备数据源

3.填充适配器


```java
public class MainActivity extends AppCompatActivity {
    ViewPager mvpGoods;
    ArrayList<Integer>mGoodsList;
    ArrayList<ImageView> mivGoodsList;
    GoodsAdapter mAdapter;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        initView();
    }
    private void initView(){
         // prepare data source
        mGoodsList=new ArrayList<>();
        mGoodsList.add(R.drawable.goods01);
        mGoodsList.add(R.drawable.goods02);
        mGoodsList.add(R.drawable.goods03);
        mGoodsList.add(R.drawable.goods04);
        mGoodsList.add(R.drawable.goods05);

        mivGoodsList=new ArrayList<>();
        for(int i=0;i<mGoodsList.size();i++){
            ImageView iv=new ImageView(this);
           iv.setImageResource(mGoodsList.get(i));
            mivGoodsList.add(iv);
        }
        mAdapter=new GoodsAdapter(this,mivGoodsList);
        mvpGoods= (ViewPager) findViewById(R.id.vpGoods);
        // ViewPager's onMeasure and onLayout method is called after setAdapter
        mvpGoods.setAdapter(mAdapter);
    }
    
  	// prepare adapter class
    class GoodsAdapter extends PagerAdapter {
        Context context;
        ArrayList<ImageView> ivGoodsList;
        public GoodsAdapter(Context context,ArrayList<ImageView>ivGoodsList){
            this.context=context;
            this.ivGoodsList=ivGoodsList;
        }

        @Override
        public int getCount() {
            return ivGoodsList.size();
        }

        @Override
        public boolean isViewFromObject(View view, Object object) {
            return view==object;
        }

        @Override
        public ImageView instantiateItem(ViewGroup container, int position) {
            ImageView imageView=ivGoodsList.get(position);
            container.addView(imageView);
            return imageView;
        }

        @Override
        public void destroyItem(ViewGroup container, int position, Object object) {
            ImageView imageView=(ImageView)object;
            container.removeView(imageView);
        }
    }
}

```

# 二、源码分析：查看早期的ViewPager源码,代码量较少，能较快熟悉实现机制[4.0.1_r1](http://grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/4.0.1_r1/android/support/v4/view/ViewPager.java#ViewPager).
## 0、入口函数

```java
266     public void More ...setAdapter(PagerAdapter adapter) {
			// 第一次调用setAdapter，mAdapter为空
267         if (mAdapter != null) {
				// added by lby:clear Observer
268             mAdapter.setDataSetObserver(null);
				// **added by lby:Call-back the startUpdate function and tell PagerAdapter to start updating the page that you want to display; Default implementation of startUpdate is null implementation(in future version of PagerAdapter)
269             mAdapter.startUpdate(this);
270             for (int i = 0; i < mItems.size(); i++) {
271                 final ItemInfo ii = mItems.get(i);
					//**added by lby:  destroy all history pages
272                 mAdapter.destroyItem(this, ii.position, ii.object);
273             }
				// **added by lby:default implementation of finishUpdate is null implementation(in future version of PagerAdapter)
274             mAdapter.finishUpdate(this);
				//**added by lby:  clear all history ItemInfos
275             mItems.clear();
276             removeAllViews();
277             mCurItem = 0;
				//**added by lby:  scroll to first coordinate of (0,0)
278             scrollTo(0, 0);
279         }
280 
281         mAdapter = adapter;
282 
283         if (mAdapter != null) {
				//**added by lby:The observer is used to monitor changes in the contents of the data source
284             if (mObserver == null) {
285                 mObserver = new DataSetObserver();
286             }
				//**added by lby:add observer
287             mAdapter.setDataSetObserver(mObserver);
288             mPopulatePending = false;
				
289             if (mRestoredCurItem >= 0) {
290                 mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
291                 setCurrentItemInternal(mRestoredCurItem, false, true);
292                 mRestoredCurItem = -1;
293                 mRestoredAdapterState = null;
294                 mRestoredClassLoader = null;
295             } else {
					// tested On API 24 platform, the function was not called for the first time when setAdapter is called.
296                 populate();
297             }
298         }
299     }


由于ViewPager并不是将所有页面作为子View，而是最多缓存用户指定缓存个数*2（左右两边，可能左边或右边没有那么多页面）因此需要创建和销毁页面，populate主要工作就是这些		

607     void More ...populate() {

611 
612         // Bail now if we are waiting to populate.  This is to hold off
613         // on creating views from the time the user releases their finger to
614         // fling to a new position until we have finished the scroll to
615         // that position, avoiding glitches from happening at that point.
616         if (mPopulatePending) {
617             if (DEBUG) Log.i(TAG, "populate is pending, skipping for now...");
618             return;
619         }
620 
621         // Also, don't populate until we are attached to a window.  This is to
622         // avoid trying to populate before we have restored our view hierarchy
623         // state and conflicting with what is restored.
624         if (getWindowToken() == null) {
625             return;
626         }
627 
628         mAdapter.startUpdate(this);
629 
630         final int pageLimit = mOffscreenPageLimit;
631         final int startPos = Math.max(0, mCurItem - pageLimit);
632         final int N = mAdapter.getCount();
633         final int endPos = Math.min(N-1, mCurItem + pageLimit);
634 
635         if (DEBUG) Log.v(TAG, "populating: startPos=" + startPos + " endPos=" + endPos);
636 
637         // Add and remove pages in the existing list.
638         int lastPos = -1;
639         for (int i=0; i<mItems.size(); i++) {
640             ItemInfo ii = mItems.get(i);
641             if ((ii.position < startPos || ii.position > endPos) && !ii.scrolling) {
642                 if (DEBUG) Log.i(TAG, "removing: " + ii.position + " @ " + i);
643                 mItems.remove(i);
644                 i--;
645                 mAdapter.destroyItem(this, ii.position, ii.object);
646             } else if (lastPos < endPos && ii.position > startPos) {
647                 // The next item is outside of our range, but we have a gap
648                 // between it and the last item where we want to have a page
649                 // shown.  Fill in the gap.
650                 lastPos++;
651                 if (lastPos < startPos) {
652                     lastPos = startPos;
653                 }
654                 while (lastPos <= endPos && lastPos < ii.position) {
655                     if (DEBUG) Log.i(TAG, "inserting: " + lastPos + " @ " + i);
656                     addNewItem(lastPos, i);
657                     lastPos++;
658                     i++;
659                 }
660             }
661             lastPos = ii.position;
662         }
663 
664         // Add any new pages we need at the end.
665         lastPos = mItems.size() > 0 ? mItems.get(mItems.size()-1).position : -1;
666         if (lastPos < endPos) {
667             lastPos++;
668             lastPos = lastPos > startPos ? lastPos : startPos;
669             while (lastPos <= endPos) {
670                 if (DEBUG) Log.i(TAG, "appending: " + lastPos);
671                 addNewItem(lastPos, -1);
672                 lastPos++;
673             }
674         }
675 
676         if (DEBUG) {
677             Log.i(TAG, "Current page list:");
678             for (int i=0; i<mItems.size(); i++) {
679                 Log.i(TAG, "#" + i + ": page " + mItems.get(i).position);
680             }
681         }
682 
683         ItemInfo curItem = null;
684         for (int i=0; i<mItems.size(); i++) {
685             if (mItems.get(i).position == mCurItem) {
686                 curItem = mItems.get(i);
687                 break;
688             }
689         }
690         mAdapter.setPrimaryItem(this, mCurItem, curItem != null ? curItem.object : null);
691 
692         mAdapter.finishUpdate(this);
693 
694         if (hasFocus()) {
695             View currentFocused = findFocus();
696             ItemInfo ii = currentFocused != null ? infoForAnyChild(currentFocused) : null;
697             if (ii == null || ii.position != mCurItem) {
698                 for (int i=0; i<getChildCount(); i++) {
699                     View child = getChildAt(i);
700                     ii = infoForChild(child);
701                     if (ii != null && ii.position == mCurItem) {
702                         if (child.requestFocus(FOCUS_FORWARD)) {
703                             break;
704                         }
705                     }
706                 }
707             }
708         }
709     }
  
```



## 1.构造函数

```java

227     public More ...ViewPager(Context context) {
228         super(context);
229         initViewPager();
230     }
231 
232     public More ...ViewPager(Context context, AttributeSet attrs) {
233         super(context, attrs);
234         initViewPager();
235     }
236 
237     void More ...initViewPager() {
			// **added by lby: When ViewGroup calls invadilate, onDraw will be called
238         setWillNotDraw(false);
239         setDescendantFocusability(FOCUS_AFTER_DESCENDANTS);
240         setFocusable(true);
241         final Context context = getContext();
242         mScroller = new Scroller(context, sInterpolator);
243         final ViewConfiguration configuration = ViewConfiguration.get(context);
244         mTouchSlop = ViewConfigurationCompat.getScaledPagingTouchSlop(configuration);
245         mMinimumVelocity = configuration.getScaledMinimumFlingVelocity();
246         mMaximumVelocity = configuration.getScaledMaximumFlingVelocity();
247         mLeftEdge = new EdgeEffectCompat(context);
248         mRightEdge = new EdgeEffectCompat(context);
249 
250         float density = context.getResources().getDisplayMetrics().density;
251         mBaseLineFlingVelocity = 2500.0f * density;
252         mFlingVelocityInfluence = 0.4f;
253     }
254 
255     private void More ...setScrollState(int newState) {
256         if (mScrollState == newState) {
257             return;
258         }
259 
260         mScrollState = newState;
261         if (mOnPageChangeListener != null) {
262             mOnPageChangeListener.onPageScrollStateChanged(newState);
263         }
264     }

```


```java
 
833     @Override
834     protected void More ...onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
835         // For simple implementation, or internal size is always 0.
836         // We depend on the container to specify the layout size of
837         // our view.  We can't really know what it is since we will be
838         // adding and removing different arbitrary views and do not
839         // want the layout to change as this happens.
840         setMeasuredDimension(getDefaultSize(0, widthMeasureSpec),
841                 getDefaultSize(0, heightMeasureSpec));
842 
843         // Children are just made to fill our space.
844         mChildWidthMeasureSpec = MeasureSpec.makeMeasureSpec(getMeasuredWidth() -
845                 getPaddingLeft() - getPaddingRight(), MeasureSpec.EXACTLY);
846         mChildHeightMeasureSpec = MeasureSpec.makeMeasureSpec(getMeasuredHeight() -
847                 getPaddingTop() - getPaddingBottom(), MeasureSpec.EXACTLY);
848 
849         // Make sure we have created all fragments that we need to have shown.
850         mInLayout = true;
851         populate();
852         mInLayout = false;
853 
854         // Make sure all children have been properly measured.
855         final int size = getChildCount();
856         for (int i = 0; i < size; ++i) {
857             final View child = getChildAt(i);
858             if (child.getVisibility() != GONE) {
859                 if (DEBUG) Log.v(TAG, "Measuring #" + i + " " + child
860 		        + ": " + mChildWidthMeasureSpec);
					// **added by lby: because child's SpecMode is set as MeasureSpec.EXACTLY, default onMeasure implementation set 
member variables, mMeasuredWidth and mMeasuredHeight, as the SpecSize value.(MeasureSpec=SpecMode+SpecSize)  
861                 child.measure(mChildWidthMeasureSpec, mChildHeightMeasureSpec);
862             }
863         }
864     }
 
 Summary:
 1、use default measure function to measure itself.
 2.measure all child elements:the main purpose of the onMeasure function is to enable all child to fill the parent (minus padding).
 
 
  ```
  
  
  ```java
899     @Override
900     protected void More ...onLayout(boolean changed, int l, int t, int r, int b) {
901         mInLayout = true;
			// This function package all child view to mItems
902         populate();
903         mInLayout = false;
904 
905         final int count = getChildCount();
906         final int width = r-l;
907 
908         for (int i = 0; i < count; i++) {
909             View child = getChildAt(i);
910             ItemInfo ii;
911             if (child.getVisibility() != GONE && (ii=infoForChild(child)) != null) {
912                 int loff = (width + mPageMargin) * ii.position;
913                 int childLeft = getPaddingLeft() + loff;
914                 int childTop = getPaddingTop();
915                 if (DEBUG) Log.v(TAG, "Positioning #" + i + " " + child + " f=" + ii.object
916 		        + ":" + childLeft + "," + childTop + " " + child.getMeasuredWidth()
917 		        + "x" + child.getMeasuredHeight());
					// layout all child from left to right
918                 child.layout(childLeft, childTop,
919                         childLeft + child.getMeasuredWidth(),
920                         childTop + child.getMeasuredHeight());
921             }
922         }
923         mFirstLayout = false;
924     }


		//**added by lby:get ItemInfo from View
806     ItemInfo More ...infoForChild(View child) {
807         for (int i=0; i<mItems.size(); i++) {
808             ItemInfo ii = mItems.get(i);
809             if (mAdapter.isViewFromObject(child, ii.object)) {
810                 return ii;
811             }
812         }
813         return null;
814     }
815 
816     ItemInfo More ...infoForAnyChild(View child) {
817         ViewParent parent;
818         while ((parent=child.getParent()) != this) {
819             if (parent == null || !(parent instanceof View)) {
820                 return null;
821             }
822             child = (View)parent;
823         }
824         return infoForChild(child);
825     }

  
```

## 






