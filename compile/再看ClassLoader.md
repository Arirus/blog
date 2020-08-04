# 再看 ClassLoader

之前我们在读《深入理解JVM》时，简单的了解过 ClassLoader。而 Android 系统使用的 art 虚拟机不算是标准的 JVM，但也有相应的 ClassLoader 的实现。那么我们本篇就来看看这两种虚拟机的 ClassLoader 的异同点。

## 类与类加载器

大家知道，虚拟机中类的使用，和类似C/C++中，类在编译时就有链接过程不同。虚拟机中，一个个类是在载入之后才链接的，因此在载入时，我们可以认为一个一个的类都是单独存在的。那么其都是由类加载器进行加载载入的。
比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个Java虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。

### 双亲委派模型（Parents Delegation Model）

双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应有自己的父类加载器。不过这里类加载器之间的父子关系一般不是以继承（Inheritance）的关系来实现的，而是通常使用组合（Composition）关系来复用父加载器的代码。

其模型是这样的：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载。

这种模型的优点在于：Java中的类随着它的类加载器一起具备了一种带有优先级的层次关系，无论哪个类加载器加载 Object 类，最终都是由最顶端的类加载器来加载。


### Java ClassLoader

Java一直保持着三层类加载器，这里的三层主要是：
- 启动类加载器（Bootstrap Class Loader），这个类加载器负责加载存放在<JAVA_HOME>\lib目录
- 扩展类加载器（Extension Class Loader），它负责加载<JAVA_HOME>\lib\ext目录中
- 应用程序类加载器（Application Class Loader）：它负责加载用户类路径（ClassPath）上所有的类库，开发者同样可以直接在代码中使用这个类加载器。如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

### Android ClassLoader

类似的，Android 系统也有自己的类加载结构：
- BootClassLoader，系统启动的时候创建的，用于加载一些系统Framework层级需要的类。
- PathClassLoader，应用启动时创建的，用于加载“/data/app/PACKAGE_NAME/base.apk”里面的类。

一个运行的Android应用至少有2个ClassLoader。

但除此之外，还有另一个 ClassLoader：DexClassLoader

## DexClassLoader 和 PathClassLoader

在Android中，ClassLoader是一个抽象类，实际开发过程中，我们一般是使用其具体的子类DexClassLoader、PathClassLoader这些类加载器来加载类的，它们的不同之处是：

- DexClassLoader可以加载jar/apk/dex，可以从SD卡中加载未安装的apk；
- PathClassLoader只能加载系统中已经安装过的apk；

二者其实都是对 BaseDexClassLoader 的一个封装。这里我们以 5.0 的源码为例来进行定义分析：

- DexClassLoader：用来加载包含 classes.dex 实体的 jar/apk 文件，这个可以从你应用外部来加载执行。
- PathClassLoader：一个可以加载文件列表或者目录的，但是不能从网络加载的加载器，也是 App 的类加载器。

二者都是直接调用了 BaseDexClassLoader 的构造函数。唯一的区别在于：**PathClassLoader 的 optimizedDirectory 为空**。

### BaseDexClassLoader
```java
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
        String libraryPath, ClassLoader parent) {
    super(parent);
    this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
}

@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    List<Throwable> suppressedExceptions = new ArrayList<Throwable>();
    Class c = pathList.findClass(name, suppressedExceptions);
    if (c == null) {
        ClassNotFoundException cnfe = new ClassNotFoundException("Didn't find class \"" + name + "\" on path: " + pathList);
        for (Throwable t : suppressedExceptions) {
            cnfe.addSuppressed(t);
        }
        throw cnfe;
    }
    return c;
}
```
BaseDexClassLoader 类中，也没有在 ClassLoader 上面做过多额外操作。一切都是围绕 DexPathList 来做的操作。包括findClass 都是直接在 PathList 中进行查找，那我们看看 DexPathList 。

### DexPathList
```java
private final Element[] dexElements;

public DexPathList(ClassLoader definingContext, String dexPath,
        String libraryPath, File optimizedDirectory) {
    ...

    this.definingContext = definingContext;
    ArrayList<IOException> suppressedExceptions = new ArrayList<IOException>();
    this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                        suppressedExceptions);
    if (suppressedExceptions.size() > 0) {
        this.dexElementsSuppressedExceptions =
            suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
    } else {
        dexElementsSuppressedExceptions = null;
    }
    this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
}

private static Element[] makeDexElements(ArrayList<File> files, File optimizedDirectory, ArrayList<IOException> suppressedExceptions) {
    ...
    for (File file : files) {
        File zip = null;
        DexFile dex = null;
        String name = file.getName();
        ...
        dex = loadDexFile(file, optimizedDirectory);
        ...     
        if ((zip != null) || (dex != null)) {
            elements.add(new Element(file, false, zip, dex));
        }
    }

    return elements.toArray(new Element[elements.size()]);
}


private static DexFile loadDexFile(File file, File optimizedDirectory)
        throws IOException {
    if (optimizedDirectory == null) {
        return new DexFile(file);
    } else {
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);
        return DexFile.loadDex(file.getPath(), optimizedPath, 0);
    }
}
```
DexPathList 的构造函数中，创建了一个 dexElements 的数组。最后是读取每个dex 文件，如果 optimizedDirectory 为空，则直接创建一个 DexFile 否则从给定路径中读取一个。

### DexFile
```java
/**
    * Opens a DEX file from a given filename. This will usually be a ZIP/JAR
    * file with a "classes.dex" inside.
    *
    * The VM will generate the name of the corresponding file in
    * /data/dalvik-cache and open it, possibly creating or updating
    * it first if system permissions allow.  Don't pass in the name of
    * a file in /data/dalvik-cache, as the named file is expected to be
    * in its original (pre-dexopt) state.
    */
public DexFile(String fileName) throws IOException {
    mCookie = openDexFile(fileName, null, 0);
    mFileName = fileName;
    guard.open("close");
    //System.out.println("DEX FILE cookie is " + mCookie + " fileName=" + fileName);
}

/**
    * Opens a DEX file from a given filename, using a specified file
    * to hold the optimized data.
    */
private DexFile(String sourceName, String outputName, int flags) throws IOException {
    ...
    mCookie = openDexFile(sourceName, outputName, flags);
    mFileName = sourceName;
    guard.open("close");
    //System.out.println("DEX FILE cookie is " + mCookie + " sourceName=" + sourceName + " outputName=" + outputName);
}
```
其实到这里 我们根据方法注释应该也能理解清除了，如果没有 optimizedDirectory 那么，就会从`/data/dalvik-cache`读取相应的文件，这里应该应该就是安装时通过 dexopt 转化 odex 格式的文件。
而后者根据放置的位置，就是用户自定义的位置。

到这里我们就可以得出结论了，optmizedDirectory 不为空时，使用用户定义的目录作为 DEX 文件优化后产物 .odex 的存储目录，为空时，会使用默认的 /data/dalvik-cache/ 目录。

### 加载类
```java
//  DexPathList
public Class findClass(String name, List<Throwable> suppressed) {
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;

        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    if (dexElementsSuppressedExceptions != null) {
        suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
    }
    return null;
}
```
BaseDexClassLoader findClass 使用的就是 DexPathList 的 findClass，这里其实就是遍历 Dex 文件进行查找，最后将找到的结果进行返回。

## 8.0 的变化

其实到 8.0 中，optimizedDirectory 被弃用了，传给 DexPathList 的 optimizedDirectory 直接为空，不管外面传进来什么值。 也就是说，在 8.0 上，PathClassLoader 和 DexClassLoader 其实已经没有什么区别了。DexClassLoader 也不能指定 optimizedDirectory 了。