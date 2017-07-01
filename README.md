# WindowSourceCode
[TOC]
## 从setContentView开始，了解View的加载过程

setContentView主要看两个方向，一个是继承Activity，另一个是集成AppCompatActivity(谷歌工程师提供的兼容包)，带着下面的4个问题看源码。

问题：
1. setContentView到底做了些什么，为什么调用后就可以显示出我们想要的布局界面？
2. phoneWindow到底是什么东西，window和他是什么关系？
3. DecorView是干什么用的？和我们的布局有什么关系？
4. getWindow().requestFeature 去除标题栏，为什么要放到setContentView之前才有效？

### 继承自Activity 

点进setContentView的源码，可以看到里面调用了window的setContentView方法，然后初始化了windowDecorView的ActionBar和DecorView
```
 public void setContentView(@LayoutRes int layoutResID) {
        getWindow().setContentView(layoutResID);
        initWindowDecorActionBar();
}
```
-  getWindow().setContentView(layoutResID);
   getWindow()返回值是一个Window对象,而Window是一个抽象类，仅有的实现类是PhoneWindow,这个从Window的注释上就可以看的出来。

```

/**
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.
 *
 * <p>The only existing implementation of this abstract class is
 * android.view.PhoneWindow, which you should instantiate when needing a
 * Window.
 */
 public abstract class Window {
    ......
}
```
所以setContentView真正的调用其实是PhoneWindow的setContentView()方法。

```
@Override
    public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```
首先，先判断mContentParent是否为null，mContentParent是window内容的区域，不是decorView本身，也不是DecorView的子类，如果是，installDecor().从注释区域也写的很明白，window.requestFeature()方法要写到此方法之前。看一下啊installDecor中的方法，因为太长了，只看关键的部分。

```
 if (mDecor == null) {
            mDecor = generateDecor(-1);
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
```
先判断mDecorView是否为null，如果是，就初始化一个DecorView，如果不是，把当前的window塞给这个DecorView。generateDecor这个方法很简单，看下代码，很简单，仅仅初始化了一个decorView。

```
  protected DecorView generateDecor(int featureId) {
        // System process doesn't have application context and in that case we need to directly use
        // the context we have. Otherwise we want the application context, so we don't cling to the
        // activity.
        Context context;
        if (mUseDecorContext) {
            Context applicationContext = getContext().getApplicationContext();
            if (applicationContext == null) {
                context = getContext();
            } else {
                context = new DecorContext(applicationContext, getContext().getResources());
                if (mTheme != -1) {
                    context.setTheme(mTheme);
                }
            }
        } else {
            context = getContext();
        }
        return new DecorView(context, featureId, this, getAttributes());
    }
```
继续看isntallDecor方法

```
 if (mContentParent == null) {
            mContentParent = generateLayout(mDecor);

            // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
            mDecor.makeOptionalFitsSystemWindows();

            final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                    R.id.decor_content_parent);
    .........
}
```
 先判断mContentParent是否为null，如果是，就生成一个mContentParent。看下generateLayout()这个方法。这个方法太长了。看几个关键点。主要思想分为两点：
1. 先判断这个布局的样式是什么样式。
2. 加载不同的布局文件
这也能回到上面的问题4。

```
  if (mIsFloating) {  //判断样式
            setLayout(WRAP_CONTENT, WRAP_CONTENT);
            setFlags(0, flagsToUpdate);
        } else {
            setFlags(FLAG_LAYOUT_IN_SCREEN|FLAG_LAYOUT_INSET_DECOR, flagsToUpdate);
        }

        if (a.getBoolean(R.styleable.Window_windowNoTitle, false)) {
            requestFeature(FEATURE_NO_TITLE);
        } else if (a.getBoolean(R.styleable.Window_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestFeature(FEATURE_ACTION_BAR);
        }

        if (a.getBoolean(R.styleable.Window_windowActionBarOverlay, false)) {
            requestFeature(FEATURE_ACTION_BAR_OVERLAY);
        }

        if (a.getBoolean(R.styleable.Window_windowActionModeOverlay, false)) {
            requestFeature(FEATURE_ACTION_MODE_OVERLAY);
        }

        if (a.getBoolean(R.styleable.Window_windowSwipeToDismiss, false)) {
            requestFeature(FEATURE_SWIPE_TO_DISMISS);
        }

        if (a.getBoolean(R.styleable.Window_windowFullscreen, false)) {
            setFlags(FLAG_FULLSCREEN, FLAG_FULLSCREEN & (~getForcedWindowFlags()));
        }

        if (a.getBoolean(R.styleable.Window_windowTranslucentStatus,
                false)) {
            setFlags(FLAG_TRANSLUCENT_STATUS, FLAG_TRANSLUCENT_STATUS
                    & (~getForcedWindowFlags()));
        }
```

```
//加载不同的布局文件
 if ((features & (1 << FEATURE_SWIPE_TO_DISMISS)) != 0) {
            layoutResource = R.layout.screen_swipe_dismiss;
        } else if ((features & ((1 << FEATURE_LEFT_ICON) | (1 << FEATURE_RIGHT_ICON))) != 0) {
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleIconsDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_title_icons;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
            // System.out.println("Title Icons!");
        } else if ((features & ((1 << FEATURE_PROGRESS) | (1 << FEATURE_INDETERMINATE_PROGRESS))) != 0
                && (features & (1 << FEATURE_ACTION_BAR)) == 0) {
            // Special case for a window with only a progress bar (and title).
            // XXX Need to have a no-title version of embedded windows.
            layoutResource = R.layout.screen_progress;
            // System.out.println("Progress!");
        } else if ((features & (1 << FEATURE_CUSTOM_TITLE)) != 0) {
            // Special case for a window with a custom title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogCustomTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else {
                layoutResource = R.layout.screen_custom_title;
            }
            // XXX Remove this once action bar supports these features.
            removeFeature(FEATURE_ACTION_BAR);
        } else if ((features & (1 << FEATURE_NO_TITLE)) == 0) {
            // If no other features and not embedded, only need a title.
            // If the window is floating, we need a dialog layout
            if (mIsFloating) {
                TypedValue res = new TypedValue();
                getContext().getTheme().resolveAttribute(
                        R.attr.dialogTitleDecorLayout, res, true);
                layoutResource = res.resourceId;
            } else if ((features & (1 << FEATURE_ACTION_BAR)) != 0) {
                layoutResource = a.getResourceId(
                        R.styleable.Window_windowActionBarFullscreenDecorLayout,
                        R.layout.screen_action_bar);
            } else {
                layoutResource = R.layout.screen_title;
            }
            // System.out.println("Title!");
        } else if ((features & (1 << FEATURE_ACTION_MODE_OVERLAY)) != 0) {
            layoutResource = R.layout.screen_simple_overlay_action_mode;
        } else {
            // Embedded, so no decoration is needed.
            layoutResource = R.layout.screen_simple; //这个布局看一下
            // System.out.println("Simple!");
        }
```
随便看一个布局，screen_simple，布局文件很简单：看代码

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true"
    android:orientation="vertical">
    <ViewStub android:id="@+id/action_mode_bar_stub"
              android:inflatedId="@+id/action_mode_bar"
              android:layout="@layout/action_mode_bar"
              android:layout_width="match_parent"
              android:layout_height="wrap_content"
              android:theme="?attr/actionBarTheme" />
    <FrameLayout
         android:id="@android:id/content"
         android:layout_width="match_parent"
         android:layout_height="match_parent"
         android:foregroundInsidePadding="false"
         android:foregroundGravity="fill_horizontal|top"
         android:foreground="?android:attr/windowContentOverlay" />
</LinearLayout>
```
整个布局如下图所示：
![image](http://opy4iwqsf.bkt.clouddn.com/1_Activity%E5%8A%A0%E8%BD%BDUI-%E7%B1%BB%E5%9B%BE%E5%85%B3%E7%B3%BB%E5%92%8C%E8%A7%86%E5%9B%BE%E7%BB%93%E6%9E%84.png)

继续看isntallDecor方法，后面的方法就是把主题样式设置给布局。installDecor就看完了，返回去看PhoneWindow的setContentView方法。

```
 public void setContentView(int layoutResID) {
        // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
        // decor, when theme attributes and the like are crystalized. Do not check the feature
        // before this happens.
        if (mContentParent == null) {
            installDecor();
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();
        }
        mContentParentExplicitlySet = true;
    }
```
用加载器加载一个布局出来，mContentParent从新绘制。这样window就把整个布局加好了,然后在Activity的attach方法中实例化出Window，在ActivityThread中启动Activity的时候添加出来。

### 继承自AppCompatActivity

继承自AppCompatActivity做了兼容的操作，但是原理还是一样的。看代码，首先先调用了AppCompatDelegate的setContentView();


```
  @Override
    public void setContentView(@LayoutRes int layoutResID) {
        getDelegate().setContentView(layoutResID);
    }
```
AppCompatDelegate的setContentView()是个抽象方法，真正的实现类为：AppCompatDelegateImplV9的setContentView();

```
 @Override
    public void setContentView(int resId) {
        ensureSubDecor();
        ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
        contentParent.removeAllViews();
        LayoutInflater.from(mContext).inflate(resId, contentParent);
        mOriginalWindowCallback.onContentChanged();
    }
```
从方法体的第一行可以看出这个方法就是确定一个subDecorView的。看代码：

```
private void ensureSubDecor() {
        if (!mSubDecorInstalled) {
            mSubDecor = createSubDecor();

            // If a title was set before we installed the decor, propagate it now
            CharSequence title = getTitle();
            if (!TextUtils.isEmpty(title)) {
                onTitleChanged(title);
            }

            applyFixedSizeWindow();

            onSubDecorInstalled(mSubDecor);

            mSubDecorInstalled = true;

            // Invalidate if the panel menu hasn't been created before this.
            // Panel menu invalidation is deferred avoiding application onCreateOptionsMenu
            // being called in the middle of onCreate or similar.
            // A pending invalidation will typically be resolved before the posted message
            // would run normally in order to satisfy instance state restoration.
            PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
            if (!isDestroyed() && (st == null || st.menu == null)) {
                invalidatePanelMenu(FEATURE_SUPPORT_ACTION_BAR);
            }
        }
    }
```
大体意思就是这么几个方向：
- 创建以个subDecorView
- 获取title
- 将View和Window的大小相匹配
- 安装subDecor
接下来看下createSubDecor()这个方法。代码太多了，看其中的重要的部分，其思想也是和上面Activity一样：
- 先判断feature
- 再加载相应的布局。

```
    /**
        先判断feature
    */
   if (a.getBoolean(R.styleable.AppCompatTheme_windowNoTitle, false)) {
            requestWindowFeature(Window.FEATURE_NO_TITLE);
        } else if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBar, false)) {
            // Don't allow an action bar if there is no title.
            requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR);
        }
        if (a.getBoolean(R.styleable.AppCompatTheme_windowActionBarOverlay, false)) {
            requestWindowFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
        }
        if (a.getBoolean(R.styleable.AppCompatTheme_windowActionModeOverlay, false)) {
            requestWindowFeature(FEATURE_ACTION_MODE_OVERLAY);
        }
        mIsFloating = a.getBoolean(R.styleable.AppCompatTheme_android_windowIsFloating, false);
        a.recycle();
```

```
/**
加载其中的一个布局的一部分代码
*/
  if (mIsFloating) {
                // If we're floating, inflate the dialog title decor
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_dialog_title_material, null);

                // Floating windows can never have an action bar, reset the flags
                mHasActionBar = mOverlayActionBar = false;
            } else if (mHasActionBar) {
                /**
                 * This needs some explanation. As we can not use the android:theme attribute
                 * pre-L, we emulate it by manually creating a LayoutInflater using a
                 * ContextThemeWrapper pointing to actionBarTheme.
                 */
                TypedValue outValue = new TypedValue();
                mContext.getTheme().resolveAttribute(R.attr.actionBarTheme, outValue, true);

                Context themedContext;
                if (outValue.resourceId != 0) {
                    themedContext = new ContextThemeWrapper(mContext, outValue.resourceId);
                } else {
                    themedContext = mContext;
                }

                // Now inflate the view using the themed context and set it as the content view
                subDecor = (ViewGroup) LayoutInflater.from(themedContext)
                        .inflate(R.layout.abc_screen_toolbar, null);

                mDecorContentParent = (DecorContentParent) subDecor
                        .findViewById(R.id.decor_content_parent);
                mDecorContentParent.setWindowCallback(getWindowCallback());

                /**
                 * Propagate features to DecorContentParent
                 */
                if (mOverlayActionBar) {
                    mDecorContentParent.initFeature(FEATURE_SUPPORT_ACTION_BAR_OVERLAY);
                }
                if (mFeatureProgress) {
                    mDecorContentParent.initFeature(Window.FEATURE_PROGRESS);
                }
                if (mFeatureIndeterminateProgress) {
                    mDecorContentParent.initFeature(Window.FEATURE_INDETERMINATE_PROGRESS);
                }
            }
        } else {
            if (mOverlayActionMode) {
                subDecor = (ViewGroup) inflater.inflate(
                        R.layout.abc_screen_simple_overlay_action_mode, null);
            } else {
                subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
            }

```
如果是浮动的就加载浮动的布局，如果不是就加载正常布局，从代码中可以看出这个布局是abc_screen_simple，看下这个布局中写了些什么。

```
<android.support.v7.widget.FitWindowsLinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/action_bar_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:fitsSystemWindows="true">

    <android.support.v7.widget.ViewStubCompat
        android:id="@+id/action_mode_bar_stub"
        android:inflatedId="@+id/action_mode_bar"
        android:layout="@layout/abc_action_mode_bar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content" />

    <include layout="@layout/abc_screen_content_include" />

</android.support.v7.widget.FitWindowsLinearLayout>
```
abc_screen_content_include的布局如下
```
<merge xmlns:android="http://schemas.android.com/apk/res/android">

    <android.support.v7.widget.ContentFrameLayout
            android:id="@id/action_bar_activity_content"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:foregroundGravity="fill_horizontal|top"
            android:foreground="?android:attr/windowContentOverlay" />

</merge>
```
可以看出，和Activity中的代码很相似，上面一个StubView，下面一个FrameLayout，最外面一个LinearLayout。这不过这个做了兼容而已。大体是一致的。返回去继续看setContentView()方法：

```
 public void setContentView(int resId) {
        ensureSubDecor();
        ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);//拿到布局
        contentParent.removeAllViews();//删除顶层ViewGroup的所有的View
        LayoutInflater.from(mContext).inflate(resId, contentParent);//用给定id的布局和contentParent加载一个布局出来
        mOriginalWindowCallback.onContentChanged();//刷新界面。
    }
```
从一张图来看下AppCompatActivity的布局设置

![image](http://opy4iwqsf.bkt.clouddn.com/AppCompatActivity%E7%9A%84View%E5%B8%83%E5%B1%80%E7%BB%93%E6%9E%84.png)

## LayoutInflater
上面代码中有这么一行代码：

```
    LayoutInflater.from(mContext).inflate(resId, contentParent);
```
那他是怎么加载出来的呢？带着3个问题来看下LayoutInflater的加载布局的源码

1. LayoutInflater到底怎么把xml添加到decorView?
2. include 为什么不能xml资源布局的根节点？
3. merge为什么作为xml资源的根节点？

跟着调用链，最终会调用到这里：从代码中可以看到先定义了一个xml解析器，然后开始解析xml布局文件。

```
 public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```
看下inflate(parser, root, attachToRoot)这个方法里面具体做了些什么

```
  public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();
                
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }

                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                final InflateException ie = new InflateException(e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } catch (Exception e) {
                final InflateException ie = new InflateException(parser.getPositionDescription()
                        + ": " + e.getMessage(), e);
                ie.setStackTrace(EMPTY_STACK_TRACE);
                throw ie;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;

                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }

            return result;
        }
    }
```
先不看代码的意思，很明显能看到一个字符串throw new InflateException("<merge /> can be used only with a valid "+ "ViewGroup root and attachToRoot=true");能够看出merge标签只能作为根标签。上面的代码就是一层一层的解析xml中的node标签，很简单,然后调用了如下的方法进行解析：

```
 rInflate(parser, root, inflaterContext, attrs, false);
```
其实这个才是我们真正解析xml的地方。

```
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;

        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            final String name = parser.getName();
            
            if (TAG_REQUEST_FOCUS.equals(name)) {
                parseRequestFocus(parser, parent);
            } else if (TAG_TAG.equals(name)) {
                parseViewTag(parser, parent, attrs);
            } else if (TAG_INCLUDE.equals(name)) {
                if (parser.getDepth() == 0) {
                    throw new InflateException("<include /> cannot be the root element");
                }
                parseInclude(parser, context, parent, attrs);
            } else if (TAG_MERGE.equals(name)) {
                throw new InflateException("<merge /> must be the root element");
            } else {
                final View view = createViewFromTag(parent, name, context, attrs);
                final ViewGroup viewGroup = (ViewGroup) parent;
                final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
                rInflateChildren(parser, view, attrs, true);
                viewGroup.addView(view, params);
            }
        }

        if (finishInflate) {
            parent.onFinishInflate();
        }
    }
```
依然不先看代码，很明显能看到两个字符串，<include /> cannot be the root element和<merge /> must be the root element 这不就是我们想要的答案吗？然后先解析父节点，再解析子节点。一层一层的解析下来，所以，在布局设计的时候，层级不能太多，这儿用了一个while循环来循环节点，层级太多效率会降低。看下图的inflator的加载过程。

![image](http://opy4iwqsf.bkt.clouddn.com/LayoutInflater.png)


小结：每一个Activity都有一个关联的window对象，用来描述应用程序的窗口。每一个窗口内部又包含了一个DecorView对象，DecorView对象用来描述窗口的视图——xml布局。


## DecorView的添加到Window的过程
![image](http://opy4iwqsf.bkt.clouddn.com/DecorView%E6%B7%BB%E5%8A%A0%E8%87%B3%E7%AA%97%E5%8F%A3%E7%9A%84%E8%BF%87%E7%A8%8B.png)

源码比较烧脑，多看几遍就好了。。。加油！[TOC]
