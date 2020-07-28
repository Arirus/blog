# include 标签
如果给include标签 和 include所加载的布局 都添加id的话，那么id要保持一致，如例子中都是container，否则是在代码中获取不到RelativeLayout容器的。 

include布局里元素的id 要和 include所在页面布局里的其他元素id 不同，如例子中的两个textview，如果把id设置相同了，程序运行起来并不会报错，但是textview的赋值只会赋值给其中的一个。

如果需要给include标签设置位置属性的话，如例子中的layout_below、layout_marginTop，这时候 必须 同时设置include标签的宽高属性layout_width、layout_height，否则编译器是会报错的。同时include中定义的 layout_* 相关熟悉会失效。

# merge 标签

merge标签可用于减少视图层级来优化布局，可以配合include使用，如果include标签的父布局 和 include布局的根容器是相同类型的，那么根容器的可以使用merge代替。

