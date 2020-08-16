
# MultiDex 相关总结

## 原因

Android 的 classLoader 在加载 APK 的时候限制了class.dex 包含的 Java 方法数，其总数不能超过65535。Google 官方给出的解决方案是使用 Multidex。

## 使用 Multidex
```java
//配置文件中，开启
multiDexEnabled true

//依赖中使用相应的库
implementation 'com.android.support:multidex:1.0.3'

//默认情况下，Dalvik限制每个APK只能使用一个Classes.dex
//运行时多Dex加载， 继承MultiDexApplication最终也是调用这个方法
MultiDex.install(this);
```
在 5.0 一下需要手动调用 multidex。

在 5.0 之后的版本，使用ART运行时，它本身支持从APK文件中加载多个Dex文件。并且ART在应用安装时执行预编译，会扫描所有的ClassesN.dex，统一优化为.oat文件。因此其实是在安装的时候，统一生产一个新的 oat 文件，包含所有需要的dex文件内容。所以对于 5.0 以上的机子无需使用 multidex。

## MainDex 的类

MainDex中的类主要是由mainDexListFile指定的，而mainDexListFile的生成是通过SDK中的mainDexClasses、mainDexClasses.rules、mainDexClassesNoAapt.rules等相关脚本生成。里面包含了
Application 继承类，Instrumentation 继承类，Annotation 继承类。以及四大组件继承类相关。以及所有的直接引用类。

当然也支持人工额外配置，既processDebugResources方法之后生成 manifest_keep.txt ，在这里添加额外需要保留的类。

最终根据这个文件，确定主 Dex 中需要保留的文件。这些内容都写在 ManifestDexListFile 文件中。


## MultiDex 加载过程
- ART

    - 安装时会把各个dex合成一个可执行oat文件，==不存在运行期加载==

- Dalvik(对于5.0以下的系统，我们需要在启动时手动加载其他的dex)

    - 判断本地是否已经有可运行的mainodex，并使用(安装阶段，会dexopt对MainClass.dex转化为odex)
    - 触发Applaction生命周期
    - Applaction生命周期中，研发人员主动调用 MultiDex.install(this);进行合包
        -解压APK，通过DexFile来加载Secondary等附属DEX，如果有对应odex则直接使用，没有的话，再次进行dexopt

直接看下 Dalvik 加载的过程，一个伪代码来表示
```java
fun install(){
    ClassLoader loader = context.getClassLoader();
    File dexDir = new File(applicationInfo.dataDir, SECONDARY_FOLDER_NAME);
    List<File> files = MultiDexExtractor.load(context, applicationInfo, dexDir, false);
    installSecondaryDexes(loader, dexDir, files);
}

private static void installSecondaryDexes(ClassLoader loader, File dexDir, List<File> files) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException, IOException {
    if (!files.isEmpty()) {
        if (VERSION.SDK_INT >= 19) {
            MultiDex.V19.install(loader, files, dexDir);
        } else if (VERSION.SDK_INT >= 14) {
            MultiDex.V14.install(loader, files, dexDir);
        } else {
            MultiDex.V4.install(loader, files);
        }
    }

}
```
对于不同 Api 级别，使用不同的载入，这里我们只是看 19 以上的情况：
```java
// 类加载器，额外要加载的内容，odex路径
private static void install(ClassLoader loader, List<File> additionalClassPathEntries, File optimizedDirectory) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException {
    //获取 classLoader 的 pathList
    Field pathListField = MultiDex.findField(loader, "pathList");
    Object dexPathList = pathListField.get(loader);
    ArrayList<IOException> suppressedExceptions = new ArrayList();

    //添加额外的dex文件，到 dexElements 中
    MultiDex.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries), optimizedDirectory, suppressedExceptions));
    ...

}

private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files, File optimizedDirectory, ArrayList<IOException> suppressedExceptions) throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {
    Method makeDexElements = MultiDex.findMethod(dexPathList, "makeDexElements", ArrayList.class, File.class, ArrayList.class);
    return (Object[])((Object[])makeDexElements.invoke(dexPathList, files, optimizedDirectory, suppressedExceptions));
}
```
整个过程就是 hook classLoader 的加载过程。


## 可能遇到的问题

### 背景知识

#### Art和Dalvik的区别
仅从编译的角度来看，在Dalvik中(实际为android2.2以上引入的技术),如同其他大多数jvm一样,都采用的是jit来做及时翻译(动态翻译),将dex或odex中并排的dalvik code运行态翻译成native code去执行。

Art中,完全抛弃了dalvik的jit,使用了aot直接在安装时用dex2oat将其完全翻译成native code。

7.0中引入了混合编译，就是既有 jit 又有 oat。在第一次安装的时候，使用 jit，保证速度。之后在手机空闲的时间会在后台进行 oat。

#### Dexopt

Dalvik 虚拟机把 dex 文件优化成 odex 文件这个过程就是 dexopt，此时不同版本允许分配的最大内存区域不同（linearAlloc）。因此可能出现opt失败。


### NoClassDefFoundError

Multidex默认的分dex实现保证了应用内四大组件的class都在主dex中，但仍然会有NoClassXXX类型的crash出现。因为Android 加载Dex files采用的是Lazy Load，这会导致虚拟机中即使已经加载了某个class，但如果这个class不在主dex的class列表中，则主dex有可能引用不到这个class，从而导致NoClassDefFoundError

解决方法 将这个类配置到mainDex中。

