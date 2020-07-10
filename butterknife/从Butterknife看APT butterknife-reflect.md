# 写在最前面

上一篇中，我们简单的分析了下 Butterknife 的项目结构，同时也分析了反射效率较低的原因，本篇就正式看看 Butterknife-reflect 这个库，看看如何用反射实现一个 ButterKnife。

# 项目分析
 
好的，最开始我们还是先来分析下，整个工程的项目结构。

```java
.
├── README.md
├── build.gradle
├── butterknife-reflect.iml
├── gradle.properties
├── proguard-rules.txt
└── src
    └── main
        ├── AndroidManifest.xml
        └── java
            └── butterknife
                ├── ButterKnife.java
                ├── CompositeUnbinder.java
                ├── EmptyTextWatcher.java
                ├── FieldUnbinder.java
                └── ListenerUnbinder.java
```
整个结构简单的出奇，ok，那么直接来看代码。

## 视图绑定

ok 首先我们回忆下，通常我们在绑定一个 View 时，会有怎样的操作

- @BindView(R.id.user) EditText username;     指出要绑定的类。
- ButterKnife.bind(this);                     在onCreate方法中，调用 bind 方法。

那么源码中我们也按照这样分析，看下 Butterknife 中的相关函数：
```java
public static Unbinder bind(@NonNull Activity target) {
    View sourceView = target.getWindow().getDecorView();
    return bind(target, sourceView);
}

public static Unbinder bind(@NonNull Object target, @NonNull View source) {
    List<Unbinder> unbinders = new ArrayList<>();
    Class<?> targetClass = target.getClass();
    if ((targetClass.getModifiers() & PRIVATE) != 0) {
      throw new IllegalArgumentException(targetClass.getName() + " must not be private.");
    }

    while (true) {
      String clsName = targetClass.getName();
      if (clsName.startsWith("android.") || clsName.startsWith("java.")
          || clsName.startsWith("androidx.")) {
        break;
      }

      for (Field field : targetClass.getDeclaredFields()) { ... }

      for (Method method : targetClass.getDeclaredMethods()) { ... }

      targetClass = targetClass.getSuperclass();
    }

    if (unbinders.isEmpty()) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return Unbinder.EMPTY;
    }

    if (debug) Log.d(TAG, "HIT: Reflectively found " + unbinders.size() + " bindings.");
    return new CompositeUnbinder(unbinders);
}
```

结构很清晰：获取当前的 DecorView，和 Activity 这个实例类绑定到一起，当然也有可能是别的实例类，这里我们用 Clazz 来代替。那这个`bind`函数做了这么几件事：

- 获取当前 Clazz 的可见性，private 的不可以。其实如果要用反射来绑定时，private 也是完全可以的，不过为了和编译时注解保持一致，因此不许 private。
- 递归遍历 Field 和 Method，直到达到基类为 官方类 时。对 Field 和 Method 进行而外操作，这里我们先不深究。
- 将所有的 binder 对象放回，方便统一管理

接下来看看他对遍历的 Field 对象都做了什么事情：
```java
for (Field field : targetClass.getDeclaredFields()) {
    int unbinderStartingSize = unbinders.size();
    Unbinder unbinder;

    unbinder = parseBindView(target, field, source);
    if (unbinder != null) unbinders.add(unbinder);

    ...

    if (unbinders.size() - unbinderStartingSize > 1) {
        throw new IllegalStateException(
            "More than one bind annotation on " + targetClass.getName() + "." + field.getName());
    }
}
```
其实就是遍历尝试，看看 这个 Field 是否被相应的注解所修饰，这里我们以 `@BindView` 作为例子来看看其内部是怎么操作的。

### parseBindView
```java
private static @Nullable Unbinder parseBindView(Object target, Field field, View source) {

    //查找 Field 是否被 BindView 所修饰，不是就跳过，进行下一个 注解的判断，例如 BindViews
    BindView bindView = field.getAnnotation(BindView.class);
    if (bindView == null) {
      return null;
    }

    //校验 Field 是否是 private 或者 static 的。其实可以直接修改，但是原因还是 要和 编译时注解保持一致。
    validateMember(field);

    //获取 BindView 的 id：value
    int id = bindView.value();

    //获取 Field 对象的类信息
    Class<?> viewClass = field.getType();

    //如果 Field 不是不是继承于 View 的话，或者一个 接口也行（Checkable），就抛异常
    if (!View.class.isAssignableFrom(viewClass) && !viewClass.isInterface()) {
      throw new IllegalStateException(
          "@BindView fields must extend from View or be an interface. ("
              + field.getDeclaringClass().getName()
              + '.'
              + field.getName()
              + ')');
    }

    String who = "field '" + field.getName() + "'";

    // 在 DecroView 中根据 id 获取相应的 View，并且将这个 View 进行强转成 定义的 Field 类型。
    Object view = Utils.findOptionalViewAsType(source, id, who, viewClass);

    // 最后一个 设置 类似与 TextView tv = findViewById(R.id.tv);
    // 这样便把一个 未初始化的 View 和 相应对象 绑定到了一起。
    trySet(field, target, view);

    return new FieldUnbinder(target, field);
}

public static <T> T findOptionalViewAsType(View source, @IdRes int id, String who,
      Class<T> cls) {
    View view = source.findViewById(id);
    return castView(view, id, who, cls);
}

public static <T> T castView(View view, @IdRes int id, String who, Class<T> cls) {
    try {
      return cls.cast(view);
    } catch (ClassCastException e) {
      String name = getResourceEntryName(view, id);
      throw new IllegalStateException("View '"
          + name
          + "' with ID "
          + id
          + " for "
          + who
          + " was of the wrong type. See cause for more info.", e);
    }
}

static void trySet(Field field, Object target, @Nullable Object value) {
    try {
      field.set(target, value);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to assign " + value + " to " + field + " on " + target, e);
    }
}
```

上面的注解已经把整个流程分析的很透彻了，整体来看主要分成这几步：

- 类可见性校验。
- 迭代查找类，遍历其 Field 和 Method 对象。
- 针对 Field
    - 尝试解析相应的注解，对注解的校验等
    - 校验 Field 对象的可见性
    - 检验 Field 的具体类是否继承与 View
    - 从 DecorView 获取相应的 视图，并强转为 对应的 Field 的类型
    - 对 Field 进行 set value

## 操作绑定

上面我们说了 BindView 相关的内容，那这里我们看看 OnClick 又会有怎样的操作。

```java
for (Method method : targetClass.getDeclaredMethods()) {
    Unbinder unbinder;
    ...
    unbinder = parseOnClick(target, method, source);
    if (unbinder != null) unbinders.add(unbinder);
    ...
}

private static @Nullable Unbinder parseOnClick(final Object target, final Method method,
      View source) {

    // 查找 Method 是否被 OnClick 所修饰，不是就跳过，进行下一个 注解的判断，例如 OnTouch
    OnClick onClick = method.getAnnotation(OnClick.class);
    if (onClick == null) {
      return null;
    }

    // 校验 Method
    validateMember(method);

    // 校验返回类型。对于 OnClick 没啥用，主要是对于有返回值的方法有有用，例如 onTouch 这样的
    validateReturnType(method, void.class);

    // 参数传递，默认情况只有一个 View，用 ON_CLICK_TYPES 来表示 Class<?>[] ON_CLICK_TYPES = { View.class };
    final ArgumentTransformer argumentTransformer =
        createArgumentTransformer(method, ON_CLICK_TYPES);

    //获取视图列表
    List<View> views =
        findViews(source, onClick.value(), isRequired(method), method.getName(), View.class);

    //把 视图 操作 实现 完全解藕
    ViewCollections.set(views, ON_CLICK,
        v -> tryInvoke(method, target, argumentTransformer.transform(v)));

    return new ListenerUnbinder<>(views, ON_CLICK);
}

// 这里其实就是讨论一个问题，用户自己绑定定义的函数签名，应该和目标函数签名怎么匹配的问题。
// 因为我们知道 以 onClick 为例，虽然有一个参数 view ，但如果我不关心的话，我自己定义时是可以不写的。
// 但是如果 我写了但是和提供函数的参数对不上怎么办？
private static ArgumentTransformer createArgumentTransformer(Method method,
      Class<?>[] callbackParameterTypes) {
    Class<?>[] targetParameterTypes = method.getParameterTypes();

    // 用户定义的函数 完全没有参数。好啊，你不用参数，我就不给你传了。
    int targetParameterLength = targetParameterTypes.length;
    if (targetParameterLength == 0) {
      // Special case the common case of no arguments.
      return ArgumentTransformer.EMPTY;
    }

    // 用户定义的函数 要求的参数比能提供的还要多，淦，没了。抛异常。
    int callbackParameterLength = callbackParameterTypes.length;
    if (targetParameterLength > callbackParameterLength) {
      throw new IllegalStateException(method.getDeclaringClass().getName()
          + "."
          + method.getName()
          + " must have at most "
          + callbackParameterLength
          + " parameter(s).");
    }

    // 用户定义的函数 要求的参数和能提供的参数 完 全 一 致。ok 那就有多少返回多少
    if (Arrays.equals(targetParameterTypes, callbackParameterTypes)) {
      // Special case the common case of exact argument match.
      return ArgumentTransformer.IDENTITY;
    }

    // 如果个数不一致，或者顺序不一致，那就只能遍历来挑选了。
    boolean[] callbackIndexUsed = new boolean[callbackParameterLength];
    final int[] indexMap = new int[targetParameterLength];
    nextTarget: for (int targetIndex = 0; targetIndex < targetParameterLength; targetIndex++) {
      Class<?> targetParameterType = targetParameterTypes[targetIndex];
      for (int callbackIndex = 0; callbackIndex < callbackParameterLength; callbackIndex++) {
        if (callbackIndexUsed[callbackIndex]) {
          continue; // 我们已经用过这个返回参数了.
        }
        Class<?> callbackParameterType = callbackParameterTypes[callbackIndex];

        if (/* 精确匹配 */
            callbackParameterType.equals(targetParameterType)
            /* 或者是个View子类 */
            || (View.class.isAssignableFrom(callbackParameterType)
                && callbackParameterType.isAssignableFrom(targetParameterType))
            /* 或者是个接口 (like Checkable) */
            || targetParameterType.isInterface()) {
          
          //那么就设置上匹配关系，修改标志位
          indexMap[targetIndex] = callbackIndex;
          callbackIndexUsed[callbackIndex] = true;
          continue nextTarget; // This avoids the error handling code if loop exits normally.
        }
      }
      //如果无法精确匹配的话，那就说明 自定义函数的某个参数无法通过 提供的函数来提供只能抛出异常。
      throw new IllegalStateException("参数不匹配");
    }

    // 返回一个参数列表 根据 indexMap 来返回吧。
    return new ArgumentTransformer() {
      @Override public Object[] transform(Object... arguments) {
        Object[] newArguments = new Object[indexMap.length];
        for (int i = 0; i < indexMap.length; i++) {
          newArguments[i] = arguments[indexMap[i]];
        }
        return newArguments;
      }

      @Override public String toString() {
        StringBuilder builder = new StringBuilder("ArgumentTransformer[");
        for (int i = 0; i < indexMap.length; i++) {
          if (i > 0) {
            builder.append(", ");
          }
          builder.append(i).append(" => ").append(indexMap[i]);
        }
        return builder.append(']').toString();
      }
    };
}
```
整体来说绑定操作是这么一个步骤：

- 类可见性校验。
- 迭代查找类，遍历其 Field 和 Method 对象。
- 针对 Method
    - 尝试解析相应的注解，对注解的校验等
    - 校验 Method 对象的可见性
    - 校验 Method 放回类型是否为 void 主要是给 onTouch 这类的有返回值的用的
    - 检验 Method 的参数和能提供的方法参数是否匹配（可以少 可以没 可以乱序，不可多，不可不匹配）
    - 获取视图列表
    - 使用 ViewCollections 来进行绑定操作等。

整体来说，思路比较简单，但是思路和抽象能力很棒。