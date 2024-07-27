## 流程
结构：
1.drouter-plugin-proxy
2.drouter-plugin
3.drouter-api

其中 drouter-api 有个task
```java
task generateVersion {
    doLast {
        File file = new File(buildDir, "/generated/drouter/META-INF")
        file.mkdirs()
        file = new File(file, "drouter")
        file.text = "api-version : ${version}" + "\n" + "plugin-min-support : 0.0.5.5"
    }
}
```
会打入 drouter的 META-INF 文件。当前api-version的版本  和 plugin最小支持版本


1/2 可以二选一，如果选 1的话 会在在transform的时候 尝试获取元信息文件中的 plugin-min-support 版本，下载到本地，通过反射获取到 RouterTransform，直接调用其transform方法。如果找不到或者异常的话 直接仅做copy操作。

如果选2的话 则走正常的流程。会忽略元信息文件。

router的transform 会把input的来源来存储到compileClassPath，然后会启动一个任务，遍历查询每个类，看是否有对应的注解。
任务这里使用一个线程池来跑，同时分为多个任务。阻塞等待所有的任务结束。
```java
private void loadFullPaths(final Queue<File> files) throws ExecutionException, InterruptedException {
    final List<Future<Void>> taskList = new ArrayList<>();
    for (int i = 0; i < CPU_COUNT; i++) {
        taskList.add(executor.submit(new Callable<Void>() {
            @Override
            public Void call() throws Exception {
                File file;
                while ((file = files.poll()) != null) {
                    loadFullClass(file);
                }
                return null;
            }
        }));
    }
    for (Future<Void> task : taskList) {
        task.get();
    }
}
```
遍历的时候 判断如果这个类满足 注解的条件，则把这个类统一收集起来，方便之后进行生成文件。
最后使用javaassist来生成对应的文件。

## 缓存
router本身支持缓存配置 routertransform 本身 支持增量编译，同时其也会维护一个 cacheFile 里面有每次生成文件后生成的 所有类路径
如果增量编译的时候，会统一把add或者change的class写入这个文件，然后根据cacheFile来直接判断注解 最后生成文件，如果每次注解校验不过则删除这个class

## 优化点
使用缓存的时候，使用javaassist来生成文件，虽然我们使用了cacheFile来生成最后的注册文件，但是很多情况下我们仅会一次添加几个文件，每次都是全量生成注册文件(约300个注册class)，比较费时费力，考虑进行增量修改，changeFile 内容有变化的时候 存入 cacheFile 增加标志位，进行增量修改。

## Booster支持
Booster 是增量编译的，transform的时候可以每次都会获取当前处理ctclass（InputChanges），然后将其增加到 changeFile 中,最后生成对应的class文件。



