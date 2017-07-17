# RecyclerView源码分析
## 一.使用demo
0.第一次使用RecyclerView，对该控件不是很熟悉。

	初步印象1：查看该继承树，该类继承自ViewGroup，因此应该容器类控件
	
	初步印象2：Developer的描述：A flexible view for providing a limited window into a large data set并且参考网上的Blog，该控件的推出很大程度是为了取代ListView，因此使用上应该和ListView有很大类似：设置Adapter，构造ViewHolder等。对比ListView的使用，在github和blog上找demo，完成对REcyclerView的初步学习

链接：[使用实例](http://blog.csdn.net/lmj623565791/article/details/45059587)

1.准备布局文件

```xml

Activity的布局文件：
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent" >

    <android.support.v7.widget.RecyclerView
        android:id="@+id/id_recyclerview"
         android:divider="#ffff0000"
           android:dividerHeight="10dp"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</RelativeLayout

Item的布局：
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:background="#44ff0000"
    android:layout_height="wrap_content" >

    <TextView
        android:id="@+id/id_num"
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:gravity="center"
        android:text="1" />
</FrameLayout>

```

2.准备数据源
3.填充适配器

```java

package com.zhy.sample.demo_recyclerview;

import java.util.ArrayList;
import java.util.List;

import android.os.Bundle;
import android.support.v7.app.ActionBarActivity;
import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;
import android.support.v7.widget.RecyclerView.ViewHolder;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;

public class HomeActivity extends ActionBarActivity
{

    private RecyclerView mRecyclerView;
    private List<String> mDatas;
    private HomeAdapter mAdapter;

    @Override
    protected void onCreate(Bundle savedInstanceState)
    {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_single_recyclerview);

        initData();
        mRecyclerView = (RecyclerView) findViewById(R.id.id_recyclerview);
        mRecyclerView.setLayoutManager(new LinearLayoutManager(this));
        mRecyclerView.setAdapter(mAdapter = new HomeAdapter());

    }

    protected void initData()
    {
        mDatas = new ArrayList<String>();
        for (int i = 'A'; i < 'z'; i++)
        {
            mDatas.add("" + (char) i);
        }
    }

    class HomeAdapter extends RecyclerView.Adapter<HomeAdapter.MyViewHolder>
    {

        @Override
        public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType)
        {
            MyViewHolder holder = new MyViewHolder(LayoutInflater.from(
                    HomeActivity.this).inflate(R.layout.item_home, parent,
                    false));
            return holder;
        }

        @Override
        public void onBindViewHolder(MyViewHolder holder, int position)
        {
            holder.tv.setText(mDatas.get(position));
        }

        @Override
        public int getItemCount()
        {
            return mDatas.size();
        }

        class MyViewHolder extends ViewHolder
        {

            TextView tv;

            public MyViewHolder(View view)
            {
                super(view);
                tv = (TextView) view.findViewById(R.id.id_num);
            }
        }
    }
}

```

# 二、源码分析：源码分析(com.android.support:recyclerview-v7:24.1.1)

## 0.目标：使用自定义的LayoutManager来管理布局，以实现ViewPager--->几点思考
 思考1：RecyclerView最早出现的时候是在5.0.0版本，里面代码有8297行代码，像ViewPager那样去从入口到View的onMeasure，onLayout，onDraw去分析显然不现实，里面涉及到很多类，很复杂。只有进行局部分析。
 
 思考2：局部分析分析什么呢？需求关注的RecyclerView实现ViewPager功能，因此应该关注RecyclerView是如何调用LayoutMananger以管理布局，进而进一步自定义LayoutManager以实现类似ViewPager的布局

## 1.onMeasure

```java

    @Override
    protected void onMeasure(int widthSpec, int heightSpec) {
        if (mLayout == null) {
            defaultOnMeasure(widthSpec, heightSpec);
            return;
        }
        if (mLayout.mAutoMeasure) {		// if LayoutManager.setAutoMeasureEnabled(true) is called
            final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);
            final boolean skipMeasure = widthMode == MeasureSpec.EXACTLY
                    && heightMode == MeasureSpec.EXACTLY;
                    
            // LinearLayoutManager use defaultOnMeasure to measure RecyclerView.The specMode is MeasureSpec.EXACTLY
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
            if (skipMeasure || mAdapter == null) {
                return;
            }
                   
     	} 
     	...
    }
    
    
    /**
     * Used when onMeasure is called before layout manager is set
     */
    void defaultOnMeasure(int widthSpec, int heightSpec) {
        // Chooses a size from the given specs and parameters that is closest to the desired size and also complies with the spec.
        final int width = LayoutManager.chooseSize(widthSpec,
                getPaddingLeft() + getPaddingRight(),
                ViewCompat.getMinimumWidth(this));
        final int height = LayoutManager.chooseSize(heightSpec,
                getPaddingTop() + getPaddingBottom(),
                ViewCompat.getMinimumHeight(this));

		// This method must be called by {@link #onMeasure(int, int)} to store the measured width and measured height
        setMeasuredDimension(width, height);
    }
    
```
    
    
 ## 2.onLayout  
  
  
  
```java
    /*
    	Wrapper around layoutChildren() that handles animating changes caused by layout.
    */
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        ...
        dispatchLayout();
        ...
    }
    
    
	void dispatchLayout() {
		...
		// relative to animation.Ignore first
		dispatchLayoutStep1();
		...
		// measure and fill subview
		dispatchLayoutStep2();
		...
		// relative to animation.Ignore first
		dispatchLayoutStep3();
    }
    
    private void dispatchLayoutStep2() 
    	--->mLayout.onLayoutChildren(mRecycler, mState); // call LayoutManager to layout child view.Take LinearLayoutManager as example
    		--> LinearLayoutManager.fill(recycler, mLayoutState, state, false); // layout algorithm
    			-->layoutChunk
    				-->measureChildWithMargins(view, 0, 0); // measure child with parentMeasureSpec and childLayoutParams
    				-->layoutDecorated(view, left + params.leftMargin, top + params.topMargin, // layout child
            					right - params.rightMargin, bottom - params.bottomMargin);

```

总结：	

1.测量过程：调用了LayoutManager的onMeasure(测量过程交有LayoutManager管理),内部只是调用了RecyclerView的defaultOnMeasure，忽略对子视图大小的测量。

2.布局过程：调用dispatchLayoutStep1，dispatchLayoutStep2，dispatchLayoutStep3完成布局， dispatchLayoutStep2为真正的测量和填充子视图。布局的过程调用mLayout.onLayoutChildren，将布局转给LayoutManager去处理。

3.测量和布局的核心算法由fill实现（未分析）---该函数为LayoutManager的核心函数




## 参考文献
[RecyclerView](https://developer.android.google.cn/reference/android/support/v7/widget/RecyclerView.html)

[RecyclerView源码分析(一)--整体设计](http://www.jianshu.com/p/9ddfdffee5d3)

[RecyclerView源码分析(二)--测量流程](http://www.jianshu.com/p/4b8d6e5004d5)

[RecyclerView源码分析(三)--布局流程](http://www.jianshu.com/p/898479f103b6)

[Android RecyclerView 使用完全解析 体验艺术般的控件](http://blog.csdn.net/lmj623565791/article/details/45059587)

[RecyclerView 全面的源码分析](http://www.jianshu.com/p/5e20d5d7af6c)









