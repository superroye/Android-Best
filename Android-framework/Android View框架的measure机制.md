# [Android View框架的measure机制](https://www.cnblogs.com/xyhuangjinfu/p/5435201.html)



# 概述

​        Android中View框架的工作机制中，主要有三个过程：

​                1、View树的测量（measure）[**Android View框架的measure机制**](http://www.cnblogs.com/xyhuangjinfu/p/5435201.html)

​                2、View树的布局（layout） **Android View框架的layout机制**

​                3、View树的绘制（draw）[**Android View框架的draw机制**](http://www.cnblogs.com/xyhuangjinfu/p/5435277.html)

​        View框架的工作流程为：测量每个View大小（measure）-->把每个View放置到相应的位置（layout）-->绘制每个View（draw）。

 

​         **本文主要讲述三大流程中的measure过程。**

 

 

 

# 带着问题来思考整个measure过程。

 

## 1、系统为什么要有measure过程？

​        开发人员在绘制UI的时候，基本都是通过XML布局文件的方式来配置UI，而每个View必须要设置的两个群属性就是layout_width和layout_height，这两个属性代表着当前View的尺寸。

官方文档截图：

![img](https://images2015.cnblogs.com/blog/940692/201604/940692-20160426150932283-2105571282.png)

​        所以这两个属性的值是必须要指定的，这两个属性的取值只能为三种类型：

​                 **1、固定的大小，比如100dp。**

​                 **2、刚好包裹其中的内容，wrap_content。**

​                 **3、想要和父布局一样大，match_parent / fill_parent。**

​        由于Android希望提供一个更优雅的GUI框架，所以提供了自适应的尺寸，也就是 wrap_content 和 match_parent 。

​        试想一下，那如果这些属性只允许设置固定的大小，那么每个View的尺寸在绘制的时候就已经确定了，所以可能都不需要measure过程。但是由于需要满足自适应尺寸的机制，所以需要一个measure过程。

 

 

## 2、measure过程都干了点什么事？

​        由于上面提到的自适应尺寸的机制，所以在用自适应尺寸来定义View大小的时候，View的真实尺寸还不能确定。但是View尺寸最终需要映射到屏幕上的像素大小，所以measure过程就是干这件事，把各种尺寸值，经过计算，得到具体的像素值。measure过程会遍历整棵View树，然后依次测量每个View真实的尺寸。具体是每个ViewGroup会向它内部的每个子View发送measure命令，然后由具体子View的onMeasure()来测量自己的尺寸。最后测量的结果保存在View的mMeasuredWidth和mMeasuredHeight中，保存的数据单位是像素。

 

 

## 3、对于自适应的尺寸机制，如何合理的测量一颗View树？

​        系统在遍历完布局文件后，针对布局文件，在内存中生成对应的View树结构，这个时候，整棵View树种的所有View对象，都还没有具体的尺寸，因为measure过程最终是要确定每个View打的准确尺寸，也就是准确的像素值。但是刚开始的时候，View中layout_width和layout_height两个属性的值，都只是自适应的尺寸，也就是match_parent和wrap_content，这两个值在系统中为负数，所以系统不会把它们当成具体的尺寸值。所以当一个View需要把它内部的match_parent或者wrap_content转换成具体的像素值的时候，他需要知道两个信息。

​        **1、针对于match_parent，父布局当前具体像素值是多少，因为match_parent就是子View想要和父布局一样大。**

​        **2、针对wrap_content，子View需要根据当前自己内部的content，算出一个合理的能包裹所有内容的最小值。但是如果这个最小值比当前父布局还大，那不行，父布局会告诉你，我只有这么大，你也不应该超过这个尺寸。**

​        由于树这种数据结构的特殊性，我们在研究measure的过程时，可以只研究一个ViewGroup和2个View的简单场景。大概示意图如下：

![img](https://images2015.cnblogs.com/blog/940692/201604/940692-20160426150945517-1508475801.png)

​        也就是说，在measure过程中，ViewGroup会根据自己当前的状况，结合子View的尺寸数据，进行一个综合评定，然后把相关信息告诉子View，然后子View在onMeasure自己的时候，一边需要考虑到自己的content大小，一边还要考虑的父布局的限制信息，然后综合评定，测量出一个最优的结果。

 

## 4、那么ViewGroup是如何向子View传递限制信息的？

​        谈到传递限制信息，那就是MeasureSpec类了，该类贯穿于整个measure过程，用来传递父布局对子View尺寸测量的约束信息。简单来说，该类就保存两类数据。

​                **1、子View当前所在父布局的具体尺寸。**

​                **2、父布局对子View的限制类型。**

​        那么限制类型又分为三种类型：

​                **1、UNSPECIFIED，不限定。意思就是，子View想要多大，我就可以给你多大，你放心大胆的measure吧，不用管其他的。也不用管我传递给你的尺寸值。（其实Android高版本中推荐，只要是这个模式，尺寸设置为0）**

​                **2、EXACTLY，精确的。意思就是，根据我当前的状况，结合你指定的尺寸参数来考虑，你就应该是这个尺寸，具体大小在MeasureSpec的尺寸属性中，自己去查看吧，你也不要管你的content有多大了，就用这个尺寸吧。**

​                **3、AT_MOST，最多的。意思就是，根据我当前的情况，结合你指定的尺寸参数来考虑，在不超过我给你限定的尺寸的前提下，你测量一个恰好能包裹你内容的尺寸就可以了。**

 

 

 

# 源代码分析

​        在View的源代码中，提取到了下面一些关于measure过程的信息。

​        我们知道，整棵View树的根节点是DecorView，它是一个FrameLayout，所以它是一个ViewGroup，所以整棵View树的测量是从一个ViewGroup对象的measure方法开始的。

 

## View：

### 1、measure

/** 开始测量一个View有多大，parent会在参数中提供约束信息，实际的测量工作是在onMeasure()中进行的，该方法会调用onMeasure()方法，所以只有onMeasure能被也必须要被override */
public final void measure(int widthMeasureSpec, int heightMeasureSpec);

父布局会在自己的onMeasure方法中，调用child.measure ，这就把measure过程转移到了子View中。

 

### 2、onMeasure


/** 具体测量过程，测量view和它的内容，来决定测量的宽高（mMeasuredWidth  mMeasuredHeight ）。该方法中必须要调用setMeasuredDimension(int, int)来保存该view测量的宽高。 */
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec);

子View会在该方法中，根据父布局给出的限制信息，和自己的content大小，来合理的测量自己的尺寸。

 

### 3、setMeasuredDimension

 
/** 保存测量结果 */
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight);

当View测量结束后，把测量结果保存起来，具体保存在mMeasuredWidth和mMeasuredHeight中。

 

##  ViewGroup：

### 1、measureChildren

/** 让所有子view测量自己的尺寸，需要考虑当前ViewGroup的MeasureSpec和Padding。跳过状态为gone的子view */
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec);-->getChildMeasureSpec()-->child.measure();

测量所有的子View尺寸，把measure过程交到子View内部。

 

### 2、measureChild

/** 测量单个View，需要考虑当前ViewGroup的MeasureSpec和Padding。 */
protected void measureChild(View child, int parentWidthMeasureSpec, int parentHeightMeasureSpec);-->getChildMeasureSpec()-->child.measure();

对每一个具体的子View进行测量。

 

### 3、measureChildWithMargins

 

/** 测量单个View，需要考虑当前ViewGroup的MeasureSpec和Padding、margins。 */
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, int parentHeightMeasureSpec, int heightUsed);-->getChildMeasureSpec()-->child.measure();

对每一个具体的子View进行测量。但是需要考虑到margin等信息。

 

### 4、getChildMeasureSpec


/** measureChildren过程中最困难的一部分，为child计算MeasureSpec。该方法为每个child的每个维度（宽、高）计算正确的MeasureSpec。目标就是把当前viewgroup的MeasureSpec和child的LayoutParams结合起来，生成最合理的结果。
比如，当前ViewGroup知道自己的准确大小，因为MeasureSpec的mode为EXACTLY，而child希望能够match_parent，这时就会为child生成一个mode为EXACTLY，大小为ViewGroup大小的MeasureSpec。
 */
public static int getChildMeasureSpec(int spec, int padding, int childDimension);

根据当前自身的状况，以及特定子View的尺寸参数，为特定子View计算一个合理的限制信息。

**源代码：**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

 

**伪代码：**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        获取限制信息中的尺寸和模式。
        switch (限制信息中的模式) {
            case 当前容器的父容器，给当前容器设置了一个精确的尺寸:
                if (子View申请固定的尺寸) {
                    你就用你自己申请的尺寸值就行了;
                } else if (子View希望和父容器一样大) {
                    你就用父容器的尺寸值就行了;
                } else if (子View希望包裹内容) {
                    你最大尺寸值为父容器的尺寸值，但是你还是要尽可能小的测量自己的尺寸，包裹你的内容就足够了;
                } 
                    break;
            case 当前容器的父容器，给当前容器设置了一个最大尺寸:
                if (子View申请固定的尺寸) {
                    你就用你自己申请的尺寸值就行了;
                } else if (子View希望和父容器一样大) {
                    你最大尺寸值为父容器的尺寸值，但是你还是要尽可能小的测量自己的尺寸，包裹你的内容就足够了;
                } else if (子View希望包裹内容) {
                    你最大尺寸值为父容器的尺寸值，但是你还是要尽可能小的测量自己的尺寸，包裹你的内容就足够了;
                } 
                    break;
            case 当前容器的父容器，对当前容器的尺寸不限制:
                if (子View申请固定的尺寸) {
                    你就用你自己申请的尺寸值就行了;
                } else if (子View希望和父容器一样大) {
                    父容器对子View尺寸不做限制。
                } else if (子View希望包裹内容) {
                    父容器对子View尺寸不做限制。
                }
                    break;
        } return 对子View尺寸的限制信息;
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

 


​        **当自定义View的时候，也需要处理measure过程，主要有两种情况。**

​        **1、继承自View的子类。**

​                需要覆写onMeasure来正确测量自己。最后都需要调用setMeasuredDimension来保存测量结果

​                一般来说，自定义View的measure过程伪代码为：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
int mode = MeasureSpec.getMode(measureSpec);
int size = MeasureSpec.getSize(measureSpec);

int viewSize = 0;

swith (mode) {
    case MeasureSpec.EXACTLY:
        viewSize = size; //当前View尺寸设置为父布局尺寸
        break;
    case MeasureSpec.AT_MOST:
        viewSize = Math.min(size, getContentSize()); //当前View尺寸为内容尺寸和父布局尺寸当中的最小值
        break;
    case MeasureSpec.UNSPECIFIED:
        viewSize = getContentSize(); //内容有多大，就设置尺寸为多大
        break;
    default:
        break;
}

setMeasuredDimension(viewSize);
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

 

​        **2、继承自ViewGroup的子类。**

​                不但需要覆写onMeasure来正确测量自己，可能还要覆写一系列measureChild方法，来正确的测量子view，比如ScrollView。或者干脆放弃父类实现的measureChild规则，自己重新实现一套测量子view的规则，比如RelativeLayout。最后都需要调用setMeasuredDimension来保存测量结果。

​                一般来说，自定义ViewGroup的measure过程的伪代码为：

​                

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
//ViewGroup开始测量自己的尺寸
viewGroup.onMeasure();
//ViewGroup为每个child计算测量限制信息（MeasureSpec）
viewGroup.getChildMeasureSpec();
//把上一步生成的限制信息，传递给每个子View，然后子View开始measure自己的尺寸
child.measure();
//子View测量完成后，ViewGroup就可以获取每个子View测量后的尺寸
child.getChildMeasuredSize();
//ViewGroup根据自己自身状况，比如Padding等，计算自己的尺寸
viewGroup.calculateSelfSize();
//ViewGroup保存自己的尺寸
viewGroupsetMeasuredDimension();
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 


 

 

# 案例分析

​        很多开发人员都遇到过这种需求，就是ScrollView内部嵌套ListView，而该ListView数据条数是不确定的，所以需要设置为包裹内容，然后就会发现ListView就会显示第一行出来。然后就会百度到一条解决方案，继承ListView，覆写onMeasure方法。

```
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int expandSpec = MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2, MeasureSpec.AT_MOST);
        super.onMeasure(widthMeasureSpec, expandSpec);
    }
```

 


 

问题是解决了，但是很多开发人员并不知道为什么。

 

下面会从ScrollView和ListView的measure过程来分析一下。

## 1、为什么会出现上述问题？

**备注：截取部分问题相关代码，并不是完整代码。**

看看ListView的onMeasure：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        final View child = obtainView(0, mIsScrap);
        childHeight = child.getMeasuredHeight();
        if (heightMode == MeasureSpec.UNSPECIFIED) {
            heightSize = mListPadding.top + mListPadding.bottom + childHeight + getVerticalFadingEdgeLength() * 2;
            if (heightMode == MeasureSpec.AT_MOST) {
                // TODO: after first layout we should maybe start at the first visible position, not 0 
                heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
            }
            setMeasuredDimension(widthSize, heightSize);
            mWidthMeasureSpec = widthMeasureSpec;
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

 

当MeasureSpec mode为UNSPECIFIED的时候，只测量第一个item打的高度，跟问题描述相符，所以我们猜测可能是因为ScrollView传递了一个UNSPECIFIED限制给ListView。

 

再来看ScrollView的onMeasure代码：

```
 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }
```

 

 

调用了父类的onMeasure：

 

看看FrameLayout的onMeasure：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            }
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

 

调用了measureChildWithMargins，但是因为ScrollView覆写了该方法，所以看看ScrollView的measureChildWithMargins方法：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
@Override
    protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed, 
                                           int parentHeightMeasureSpec, int heightUsed) {
        final int childHeightMeasureSpec = 
                MeasureSpec.makeSafeMeasureSpec(MeasureSpec.getSize(parentHeightMeasureSpec), 
                        MeasureSpec.UNSPECIFIED);
        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 


 

果然，它向ListView的onMeasure传递了一个UNSPECIFIED的限制。

 

为什么呢，想想，因为ScrollView，本来就是可以在竖直方向滚动的布局，所以，它对它所有的子View的高度就是UNSPECIFIED，意思就是，不限制子View有多高，因为我本来就是需要竖直滑动的，它的本意就是如此，所以它对子View高度不做任何限制。

 

## 2、为什么这种解决方法可以解决这个问题？

看看ListView的onMeasure：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        final View child = obtainView(0, mIsScrap);
        childHeight = child.getMeasuredHeight();
        if (heightMode == MeasureSpec.UNSPECIFIED) {
            heightSize = mListPadding.top + mListPadding.bottom + childHeight + getVerticalFadingEdgeLength() * 2;
            if (heightMode == MeasureSpec.AT_MOST) {
                // TODO: after first layout we should maybe start at the first visible position, not 0 
                heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
            }
            setMeasuredDimension(widthSize, heightSize);
            mWidthMeasureSpec = widthMeasureSpec;
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

 

 

只要让heightMode == MeasureSpec.AT_MOST，它就会测量它的完整高度，所以第一个数据，限制mode的值就确定下来了。第二个数据就是尺寸上限，如果给个200，那么当ListView数据过多的时候，该ListView最大高度就是200了，还是不能完全显示内容，怎么办？那么就给个最大值吧，最大值是多少呢，Integer.MAX_VALUE?

先看一下MeasureSpec的代码说明：

```
        private static final int MODE_SHIFT = 30;
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;
        public static final int EXACTLY = 1 << MODE_SHIFT;
        public static final int AT_MOST = 2 << MODE_SHIFT;
```

 

 

他用最高两位存储mode，用其他剩余未存储size。所以Integer.MAX_VALUE >> 2，就是限制信息所能携带的最大尺寸数据。所以最后就需要用这两个值做成一个限制信息，传递给ListView的height维度。

也就是如下代码：

```
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int expandSpec = MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2, MeasureSpec.AT_MOST);
        super.onMeasure(widthMeasureSpec, expandSpec);
    }
```

 


 

 

# 自己动手

​        下面我们自己写一个自定义的ViewGroup，让它内部的每一个子View都垂直排布，并且让每一个子View的左边界都距离上一个子View的左边界一定的距离。并且支持wrap_content。大概看起来如下图所示：

![img](https://images2015.cnblogs.com/blog/940692/201604/940692-20160426151005939-2115917987.png)


 实际运行效果如下图所示：
![img](https://images2015.cnblogs.com/blog/940692/201604/940692-20160426151015455-884266698.png)

 

 **代码如下：**

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public class VerticalOffsetLayout extends ViewGroup {

    private static final int OFFSET = 100;

    public VerticalOffsetLayout(Context context) {
        super(context);
    }

    public VerticalOffsetLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public VerticalOffsetLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        int width = 0;
        int height = 0;

        int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            ViewGroup.LayoutParams lp = child.getLayoutParams();
            int childWidthSpec = getChildMeasureSpec(widthMeasureSpec, 0, lp.width);
            int childHeightSpec = getChildMeasureSpec(heightMeasureSpec, 0, lp.height);
            child.measure(childWidthSpec, childHeightSpec);
        }

        switch (widthMode) {
            case MeasureSpec.EXACTLY:
                width = widthSize;
                break;
            case MeasureSpec.AT_MOST:
            case MeasureSpec.UNSPECIFIED:
                for (int i = 0; i < childCount; i++) {
                    View child = getChildAt(i);
                    int widthAddOffset = i * OFFSET + child.getMeasuredWidth();
                    width = Math.max(width, widthAddOffset);
                }
                break;
            default:
                break;

        }

        switch (heightMode) {
            case MeasureSpec.EXACTLY:
                height = heightSize;
                break;
            case MeasureSpec.AT_MOST:
            case MeasureSpec.UNSPECIFIED:
                for (int i = 0; i < childCount; i++) {
                    View child = getChildAt(i);
                    height = height + child.getMeasuredHeight();
                }
                break;
            default:
                break;

        }

        setMeasuredDimension(width, height);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int left = 0;
        int right = 0;
        int top = 0;
        int bottom = 0;

        int childCount = getChildCount();

        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            left = i * OFFSET;
            right = left + child.getMeasuredWidth();
            bottom = top + child.getMeasuredHeight();

            child.layout(left, top, right, bottom);

            top += child.getMeasuredHeight();
        }
    }
}
```

（转载：https://www.cnblogs.com/xyhuangjinfu/p/5435201.html）