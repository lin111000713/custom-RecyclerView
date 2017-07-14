# ViewPager源码分析
## 0.使用demo

链接：[使用实例](http://www.jianshu.com/p/6b1008fcc082)

1.准备布局文件

```xml

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
## 1、核心数据
	```java
			// Core data structure for every page
	67      static class ItemInfo {
				//object为PagerAdapter的instantiateItem函数返回的对象
	68          Object object;
				//position为页面的序号，即第几个页面--对应Object的页面下标
	69          int position;
				//是否正在滚动
	70          boolean scrolling;
	71      }
	
	```
参考博客链接：[ViewPager源码分析：与PagerAdapter 交互](http://www.jianshu.com/p/204efa98a18d)
來源：简书


## 2、入口函数1: setAdapter

```java
266     public void setAdapter(PagerAdapter adapter) {
			// ignore it!
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
				// if we have datas to restore
289             if (mRestoredCurItem >= 0) {
290                 mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
291                 setCurrentItemInternal(mRestoredCurItem, false, true);
292                 mRestoredCurItem = -1;
293                 mRestoredAdapterState = null;
294                 mRestoredClassLoader = null;
295             } else {
					// Tested On API 24 platform, the function was not called for the first time when setAdapter is called.But it is called in this version of sdk
296                 populate();
297             }
298         }
299     }


由于ViewPager并不是将所有页面作为子View，而是最多缓存用户指定缓存个数*2（左右两边，可能左边或右边没有那么多页面）因此需要创建和销毁页面，populate主要工作就是这些		

607     void populate() {
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
629 		// **added by lby:default value is 1-->to cache a page for left and right
630         final int pageLimit = mOffscreenPageLimit;
			// ** added by lby:// mCurItem: Index of currently displayed page.
			// ** added by lby:Index of the first page
631         final int startPos = Math.max(0, mCurItem - pageLimit);
632         final int N = mAdapter.getCount();
			// ** added by lby:Index of the last page
633         final int endPos = Math.min(N-1, mCurItem + pageLimit);
634 
636 
637         // 1.Add and remove pages in the existing list.
638         int lastPos = -1;
639         for (int i=0; i<mItems.size(); i++) {
640             ItemInfo ii = mItems.get(i);
				//** added by lby:remove all items where index is not between startPos and endPos while ViewPager is not scrolling
641             if ((ii.position < startPos || ii.position > endPos) && !ii.scrolling) {
642                 if (DEBUG) Log.i(TAG, "removing: " + ii.position + " @ " + i);
643                 mItems.remove(i);
644                 i--;
645                 mAdapter.destroyItem(this, ii.position, ii.object);
646             } 
				// The following piece of code can't be understood.
				else if (lastPos < endPos && ii.position > startPos) {
647                 // The next item is outside of our range, but we have a gap
648                 // between it and the last item where we want to have a page
649                 // shown.  Fill in the gap.  希望有一个item去填充空白
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
664         // 2.Add any new pages we need at the end.
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
			//** added by lby:get current item
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

		/*
		 	SDK version:4.1.1_r1
		 */
775     void populate(int newCurrentItem) {
776         ItemInfo oldCurInfo = null;
777         if (mCurItem != newCurrentItem) {
778             oldCurInfo = infoForPosition(mCurItem);
779             mCurItem = newCurrentItem;
780         }
781 
			...
801 
802         mAdapter.startUpdate(this);
803 
804         final int pageLimit = mOffscreenPageLimit;
805         final int startPos = Math.max(0, mCurItem - pageLimit);
806         final int N = mAdapter.getCount();
807         final int endPos = Math.min(N-1, mCurItem + pageLimit);
808 
809         // Locate the currently focused item or add it if needed.
810         int curIndex = -1;
811         ItemInfo curItem = null;
812         for (curIndex = 0; curIndex < mItems.size(); curIndex++) {
813             final ItemInfo ii = mItems.get(curIndex);
814             if (ii.position >= mCurItem) {
815                 if (ii.position == mCurItem) curItem = ii;
816                 break;
817             }
818         }
819 
			//** added by lby:add current item to last position of mItems
820         if (curItem == null && N > 0) {
821             curItem = addNewItem(mCurItem, curIndex);
822         }
823 
824         // Fill 3x the available width or up to the number of offscreen
825         // pages requested to either side, whichever is larger.
826         // If we have no current item we have no work to do.
			//** added by lby:Caches the page on the left side of the selected page
827         if (curItem != null) {
828             float extraWidthLeft = 0.f;
829             int itemIndex = curIndex - 1;
830             ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
831             final float leftWidthNeeded = 2.f - curItem.widthFactor;
832             for (int pos = mCurItem - 1; pos >= 0; pos--) {
					//** added by lby: If the width on the left exceeds the required width and the current page position is smaller than the first cached page, this page needs Destroy
833                 if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
834                     if (ii == null) {
835                         break;
836                     }

						//** added by lby: destroy current page
837                     if (pos == ii.position && !ii.scrolling) {
838                         mItems.remove(itemIndex);
839                         mAdapter.destroyItem(this, pos, ii.object);
840                         itemIndex--;
841                         curIndex--;
842                         ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
843                     }
844                 } else if (ii != null && pos == ii.position) {
						//** added by lby:If the current location is in need to cache, and the page on this location already exists, then the left page's extraWidthLeft plus the current widthFactor
845                     extraWidthLeft += ii.widthFactor;
846                     itemIndex--;
847                     ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
848                 } else {
						// If the current location is to be cached, and there is no page in this position, then 
you need to add a ItemInfo to ItemInfo list.
849                     ii = addNewItem(pos, itemIndex + 1);
850                     extraWidthLeft += ii.widthFactor;
851                     curIndex++;
852                     ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
853                 }
854             }
855 
				//** added by lby:Caches the page on the right side of the selected page.The principle is similar to the above
856             float extraWidthRight = curItem.widthFactor;
857             itemIndex = curIndex + 1;
858             if (extraWidthRight < 2.f) {
859                 ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
860                 for (int pos = mCurItem + 1; pos < N; pos++) {
861                     if (extraWidthRight >= 2.f && pos > endPos) {
862                         if (ii == null) {
863                             break;
864                         }
865                         if (pos == ii.position && !ii.scrolling) {
866                             mItems.remove(itemIndex);
867                             mAdapter.destroyItem(this, pos, ii.object);
868                             ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
869                         }
870                     } else if (ii != null && pos == ii.position) {
871                         extraWidthRight += ii.widthFactor;
872                         itemIndex++;
873                         ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
874                     } else {
875                         ii = addNewItem(pos, itemIndex);
876                         itemIndex++;
877                         extraWidthRight += ii.widthFactor;
878                         ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
879                     }
880                 }
881             }
882 
883             calculatePageOffsets(curItem, curIndex, oldCurInfo);
884         }
885 
886         if (DEBUG) {
887             Log.i(TAG, "Current page list:");
888             for (int i=0; i<mItems.size(); i++) {
889                 Log.i(TAG, "#" + i + ": page " + mItems.get(i).position);
890             }
891         }
892 
893         mAdapter.setPrimaryItem(this, mCurItem, curItem != null ? curItem.object : null);
894 
895         mAdapter.finishUpdate(this);
896 
897         // Check width measurement of current pages. Update LayoutParams as needed.
898         final int childCount = getChildCount();
899         for (int i = 0; i < childCount; i++) {
900             final View child = getChildAt(i);
901             final LayoutParams lp = (LayoutParams) child.getLayoutParams();
902             if (!lp.isDecor && lp.widthFactor == 0.f) {
903                 // 0 means requery the adapter for this, it doesn't have a valid width.
904                 final ItemInfo ii = infoForChild(child);
905                 if (ii != null) {
906                     lp.widthFactor = ii.widthFactor;
907                 }
908             }
909         }
910 
911         if (hasFocus()) {
912             View currentFocused = findFocus();
913             ItemInfo ii = currentFocused != null ? infoForAnyChild(currentFocused) : null;
914             if (ii == null || ii.position != mCurItem) {
915                 for (int i=0; i<getChildCount(); i++) {
916                     View child = getChildAt(i);
917                     ii = infoForChild(child);
918                     if (ii != null && ii.position == mCurItem) {
919                         if (child.requestFocus(FOCUS_FORWARD)) {
920                             break;
921                         }
922                     }
923                 }
924             }
925         }
926     }

		/*
		 * 1.add ItemInfo to mItems
		 * 2.add View corresponding it's ItemInfo to ViewPager
		 */
545     void addNewItem(int position, int index) {
546         ItemInfo ii = new ItemInfo();
547         ii.position = position;
			//** added by lby:add child View to it's parent view(ViewPager)
548         ii.object = mAdapter.instantiateItem(this, position);
549         if (index < 0) {
550             mItems.add(ii);
551         } else {
552             mItems.add(index, ii);
553         }
554     }

Summary:when setAdapter is called, populate is called at the first time.
	at this time:mItems.size is equal 2
		--->mItems.get(0):addNewItem(0, -1);   [position=0]
		--->mItems.get(1):addNewItem(1, -1);	[position=1]
		
  
```

## 3.构造函数

```java
227     public ViewPager(Context context) {
228         super(context);
229         initViewPager();
230     }
231 
232     public ViewPager(Context context, AttributeSet attrs) {
233         super(context, attrs);
234         initViewPager();
235     }
236 
237     void initViewPager() {
			// **added by lby: When ViewGroup calls invadilate, onDraw will be called
238         setWillNotDraw(false);
239         setDescendantFocusability(FOCUS_AFTER_DESCENDANTS);
240         setFocusable(true);
241         final Context context = getContext();
			// init Scroller for ViewPager's slidding
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
255     private void setScrollState(int newState) {
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
834     protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
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
900     protected void onLayout(boolean changed, int l, int t, int r, int b) {
901         mInLayout = true;
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
912                 int loff = (width + mPageMargin) * ii.position; // mPageMargin default value is 0
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
806     ItemInfo infoForChild(View child) {
807         for (int i=0; i<mItems.size(); i++) {
808             ItemInfo ii = mItems.get(i);
809             if (mAdapter.isViewFromObject(child, ii.object)) {
810                 return ii;
811             }
812         }
813         return null;
814     }
815 
816     ItemInfo infoForAnyChild(View child) {
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

总结：
初始化的时候调用过程
1.setAdapter-->populate   	结果：产生下标为0和1的两个item
2.onMeasure-->onLayout  	结果：下标为0和1的两个item被测量和布局，测量后的大小为父布局的宽高（除了padding外），布局后为屏幕第一屏和右边第二屏

## 4. 入口2：setCurrentItem

```java
420     public void setCurrentItem(int item) {
421         mPopulatePending = false;
422         setCurrentItemInternal(item, !mFirstLayout, false);
423     }

430 
431     public void More ...setCurrentItem(int item, boolean smoothScroll) {
432         mPopulatePending = false;
433         setCurrentItemInternal(item, smoothScroll, false);
434     }
439 
440     void More ...setCurrentItemInternal(int item, boolean smoothScroll, boolean always) {
441         setCurrentItemInternal(item, smoothScroll, always, 0);
442     }
443 
444     void More ...setCurrentItemInternal(int item, boolean smoothScroll, boolean always, int velocity) {
			...
			
459         final int pageLimit = mOffscreenPageLimit;
460         if (item > (mCurItem + pageLimit) || item < (mCurItem - pageLimit)) {
461             // We are doing a jump by more than one page.  To avoid
462             // glitches, we want to keep all current pages in the view
463             // until the scroll ends.
464             for (int i=0; i<mItems.size(); i++) {
465                 mItems.get(i).scrolling = true;
466             }
467         }
468         final boolean dispatchSelected = mCurItem != item;
			
469         populate(item);
470         final ItemInfo curInfo = infoForPosition(item);
471         int destX = 0;
472         if (curInfo != null) {
473             final int width = getWidth();
474             destX = (int) (width * Math.max(mFirstOffset,
475                     Math.min(curInfo.offset, mLastOffset)));
476         }
477         if (smoothScroll) {
478             smoothScrollTo(destX, 0, velocity);
479             if (dispatchSelected && mOnPageChangeListener != null) {
480                 mOnPageChangeListener.onPageSelected(item);
481             }
482             if (dispatchSelected && mInternalPageChangeListener != null) {
483                 mInternalPageChangeListener.onPageSelected(item);
484             }
485         } else {
486             if (dispatchSelected && mOnPageChangeListener != null) {
487                 mOnPageChangeListener.onPageSelected(item);
488             }
489             if (dispatchSelected && mInternalPageChangeListener != null) {
490                 mInternalPageChangeListener.onPageSelected(item);
491             }
492             completeScroll();
493             scrollTo(destX, 0);
494         }
495     }

```

总结：

![布局变化](https://github.com/lin111000713/custom-RecyclerView/blob/master/%E5%8F%98%E5%8C%96%E8%BF%87%E7%A8%8B.png)

```java
假设当前的ViewPager维护5个Pager，刚开始的时候选择下标为0的pager，然后选中下标为4的pager（setCurrentItem）.Item从0变到4的过程：
1.当下标从0-->4滚动的过程中，维护的数据链表：mItems（维护四个数据项）
		mItems = {ArrayList@4681}  size = 4
			 0 = {ViewPager$ItemInfo@4713} 
			  object = {ImageView@4717} "android.widget.ImageView{ba43fb8 V.ED..... ........ 0,0-1080,600}"
			  offset = 0.0
			  position = 0		// 第一个Pager
			  scrolling = true
			  widthFactor = 1.0
			  shadow$_klass_ = {Class@4585} "class android.support.v4.view.ViewPager$ItemInfo"
			  shadow$_monitor_ = -1923628340
			 1 = {ViewPager$ItemInfo@4714} 
			  object = {ImageView@4720} "android.widget.ImageView{dc854f7 V.ED..... ......ID 1080,0-2160,600}"
			  offset = 1.0
			  position = 1		// 第二个Pager
			  scrolling = true
			  widthFactor = 1.0
			  shadow$_klass_ = {Class@4585} "class android.support.v4.view.ViewPager$ItemInfo"
			  shadow$_monitor_ = -2122299115
			 2 = {ViewPager$ItemInfo@4715} 
			  object = {ImageView@4723} "android.widget.ImageView{a89e82 V.ED..... ......ID 0,0-0,0}"
			  offset = 3.0
			  position = 3		// 第四个Pager
			  scrolling = false
			  widthFactor = 1.0
			  shadow$_klass_ = {Class@4585} "class android.support.v4.view.ViewPager$ItemInfo"
			  shadow$_monitor_ = -2061388758
			 3 = {ViewPager$ItemInfo@4716} 
			  object = {ImageView@4726} "android.widget.ImageView{81db1c9 V.ED..... ......ID 0,0-0,0}"
			  offset = 4.0
			  position = 4		// 第五个Pager
			  scrolling = false
			  widthFactor = 1.0
			  shadow$_klass_ = {Class@4585} "class android.support.v4.view.ViewPager$ItemInfo"
			  shadow$_monitor_ = -2034585061
2.将步骤1的四个数据项，根据Item数据中的position，将每个Pager布局到0，1，3，4位置
3.根据mItems的四个数据项重新布局，然后滚动4个屏幕的距离，当前选中的位置从0变到4
4.清理掉mItems链表中的0和1两个page对应的数据项，此时mItems中的数据项仅剩下两个元素（position=3和position=4两项）
```



