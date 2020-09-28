# include 标签
如果给include标签 和 include所加载的布局 都添加id的话，那么id要保持一致，如例子中都是container，否则是在代码中获取不到RelativeLayout容器的。 

include布局里元素的id 要和 include所在页面布局里的其他元素id 不同，如例子中的两个textview，如果把id设置相同了，程序运行起来并不会报错，但是textview的赋值只会赋值给其中的一个。

如果需要给include标签设置位置属性的话，如例子中的layout_below、layout_marginTop，这时候 必须 同时设置include标签的宽高属性layout_width、layout_height，否则编译器是会报错的。同时include中定义的 layout_* 相关熟悉会失效。

# merge 标签

merge标签可用于减少视图层级来优化布局，可以配合include使用，如果include标签的父布局 和 include布局的根容器是相同类型的，那么根容器的可以使用merge代替。


# ViewStub按需加载

按需加载 顾名思义需要的时候再去加载，不需要的时候可以不用加载，节约内存使用。比如app中页面里某个布局只需要在特定的情况下才显示，其余情况下可以不用加载显示，这时候可以使用ViewStub。

使用 ViewStub.layout 属性设置需要加载的布局，当需要加载视图的时候，调用 inflate 方法，一旦调用后，ViewStub将从视图中移除，被对应的layout布局取代，同时会保留ViewStub上设置的属性效果。


# 布局加载原理

```java
//AppCompatDelegateImpl
@Override
public void setContentView(int resId) {
    ensureSubDecor();
    ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content); //找到根视图
    contentParent.removeAllViews();
    LayoutInflater.from(mContext).inflate(resId, contentParent);//调用 LayoutInflater inflate 直接获取，并attach到根视图
    mOriginalWindowCallback.onContentChanged();
}

//LayoutInflater 
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    if (DEBUG) {
        Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                + Integer.toHexString(resource) + ")");
    }

    //这里 view 总是空
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }

    //读取xml文件
    XmlResourceParser parser = res.getLayout(resource);
    try {
        //解析xml文件 生成相应视图
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}
```

主要流程就是上面：获取根视图，读取xml文件，生产相应的视图并贴到根视图上面。
好的 分开看，先看xml读取

## 读取xml文件
```java
// ResourceImpl
XmlResourceParser loadXmlResourceParser(@NonNull String file, @AnyRes int id, int assetCookie,
    @NonNull String type)
    throws NotFoundException {

    ...

    final XmlBlock block = mAssets.openXmlBlockAsset(assetCookie, file);

    ...

    return block.newParser();

    ...
}


// AssetsManager
@NonNull XmlBlock openXmlBlockAsset(int cookie, @NonNull String fileName) throws IOException {
    Preconditions.checkNotNull(fileName, "fileName");
    synchronized (this) {
        ensureOpenLocked();
        final long xmlBlock = nativeOpenXmlAsset(mObject, cookie, fileName);
        if (xmlBlock == 0) {
            throw new FileNotFoundException("Asset XML file: " + fileName);
        }
        final XmlBlock block = new XmlBlock(this, xmlBlock);
        incRefsLocked(block.hashCode());
        return block;
    }
}
```
因此归根到底是使用了 AssetsManager 来读取了本地 xml 文件生产了 XmlBlock，最后生成 xmlParser。

## 生成视图
```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {

        ...
        View result = root;

        try {
            advanceToRootNode(parser);
            final String name = parser.getName();

            ...

            // 对于 merge 标签做额外处理
            if (TAG_MERGE.equals(name)) {
                if (root == null || !attachToRoot) {
                    throw new InflateException("<merge /> can be used only with a valid "
                            + "ViewGroup root and attachToRoot=true");
                }

                rInflate(parser, root, inflaterContext, attrs, false);
            } else {
                // Temp is the root view that was found in the xml
                // 真正创建相应的view createViewFromTag 是关键。
                final View temp = createViewFromTag(root, name, inflaterContext, attrs);

                ...
                if (root != null && attachToRoot) {
                    root.addView(temp, params);
                }

                // Decide whether to return the root that was passed in or the
                // top view found in xml.
                if (root == null || !attachToRoot) {
                    result = temp;
                }
            }
            ...
        }
        ...
        return result
    }
    ...
}

View createViewFromTag(View parent, String name, Context context, AttributeSet attrs, boolean ignoreThemeAttr) {
    ...

    try {
        View view = tryCreateView(parent, name, context, attrs);

        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(context, parent, name, attrs);
                } else {
                    view = createView(context, name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }

        return view;
    } 
    ...
}

public final View tryCreateView(@Nullable View parent, @NonNull String name,
    @NonNull Context context,
    @NonNull AttributeSet attrs) {
    if (name.equals(TAG_1995)) {
        // Let's party like it's 1995!
        return new BlinkLayout(context, attrs);
    }

    View view;
    if (mFactory2 != null) {
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {
        view = mFactory.onCreateView(name, context, attrs);
    } else {
        view = null;
    }

    if (view == null && mPrivateFactory != null) {
        view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }

    return view;
}
```

在createViewFromTag方法中，首先会判断mFactory2是否存在，存在就会使用mFactory2的onCreateView方法区创建视图，否则就会调用mFactory的onCreateView方法，接下来，如果此时的tag是一个Fragment，则会调用mPrivateFactory的onCreateView方法，否则的话，最终都会调用LayoutInflater实例的createView方法.

```java
public final View createView(String name, String prefix, AttributeSet attrs)
        throws ClassNotFoundException, InflateException {

   ...

    try {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, name);

        if (constructor == null) {
            // Class not found in the cache, see if it's real, and try to add it
            // 加载相应的类
            clazz = mContext.getClassLoader().loadClass(
                    prefix != null ? (prefix + name) : name).asSubclass(View.class);

            if (mFilter != null && clazz != null) {
                boolean allowed = mFilter.onLoadClass(clazz);
                if (!allowed) {
                    failNotAllowed(name, prefix, attrs);
                }
            }
            // 反射找到构造方法
            constructor = clazz.getConstructor(mConstructorSignature);
            constructor.setAccessible(true);
            sConstructorMap.put(name, constructor);
        } else {
            ...
        }

        ...

        // 构造相应的对象 并返回
        final View view = constructor.newInstance(args);
        if (view instanceof ViewStub) {
            // Use the same context when inflating ViewStub later.
            final ViewStub viewStub = (ViewStub) view;
            viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
        }
        mConstructorArgs[0] = lastContext;
        return view;

    } 
   ...
}
```
整个生成过程，可以看代码中的注释。

最后，我们来总结下Android中的布局加载流程：

- 在setContentView方法中，会通过LayoutInflater的inflate方法去加载对应的布局。
- inflate方法中首先会调用Resources的getLayout方法去通过IO的方式去加载对应的Xml布局解析器到内存中。
- 接着，会通过createViewFromTag根据每一个tag创建具体的View对象。
- 它内部主要是按优先顺序为Factory2和Factory的onCreatView、createView方法进行View的创建，如果都没有的话则使用 LayoutInflater 实例的onCreatView、createView进行创建，而createView方法内部采用了构造器反射的方式实现。







图片大小差别

在子线程 进行view 三步骤

设计一个网络请求框架

线程池核心数确定

app启动 页面拦截

app安装过程 证书

三种版本证书的区别

ssl 交互内容

