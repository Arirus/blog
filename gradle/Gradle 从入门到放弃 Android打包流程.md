# Android打包流程

``` java
> Task :app:preBuild UP-TO-DATE
--- task name preBuild null
---------------------------------------------------

> Task :app:preDebugBuild UP-TO-DATE
--- task name preDebugBuild null
---------------------------------------------------

// 编译 Aidl
> Task :app:compileDebugAidl NO-SOURCE
--- task name compileDebugAidl null
input file:/Users/arirus/Library/Android/sdk/platforms/android-33/framework.aidl
input file:/Users/arirus/.gradle/caches/transforms-3/4f83aa4847b63a897093e7bf61199839/transformed/core-1.7.0/aidl
input file:/Users/arirus/.gradle/caches/transforms-3/4f80d4483e49a803b2299fe497a7bb56/transformed/versionedparcelable-1.1.1/aidl
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/generated/aidl_source_output_dir/debug/out

// 编译 Renderscript
> Task :app:compileDebugRenderscript NO-SOURCE
--- task name compileDebugRenderscript null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/renderscript_lib/debug/lib
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/rs/debug/obj
output file:/Users/arirus/Desktop/Plugin/app/build/generated/res/rs/debug
output file:/Users/arirus/Desktop/Plugin/app/build/generated/renderscript_source_output_dir/debug/out

// 生成 BuildConfig
> Task :app:generateDebugBuildConfig UP-TO-DATE
--- task name generateDebugBuildConfig null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/generated/source/buildConfig/debug

// check AAR meta
> Task :app:checkDebugAarMetadata UP-TO-DATE
--- task name checkDebugAarMetadata null
input file:/Users/arirus/.gradle/caches/transforms-3/39dfdd2a629358a9953e4308f28b229b/transformed/appcompat-1.4.1/META-INF/com/android/build/gradle/aar-metadata.properties
...
input file:/Users/arirus/.gradle/caches/transforms-3/01bf24068b4652298f53a40e5ad8b190/transformed/annotation-experimental-1.1.0/META-INF/com/android/build/gradle/aar-metadata.properties
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/aar_metadata_check/debug

// 生成 ResValues
> Task :app:generateDebugResValues UP-TO-DATE
--- task name generateDebugResValues null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/generated/res/resValues/debug

> Task :app:generateDebugResources UP-TO-DATE
--- task name generateDebugResources null
---------------------------------------------------

// 合并总 Resources 
> Task :app:mergeDebugResources UP-TO-DATE
--- task name mergeDebugResources null
input file:/Users/arirus/.gradle/caches/transforms-3/ae2c05efad78dce426803156ac577da6/transformed/material-1.2.0/res
...
input file:/Users/arirus/Desktop/Plugin/app/src/main/res
input file:/Users/arirus/Desktop/Plugin/app/src/debug/res
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_res_blame_folder/debug/out
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/data_binding_layout_info_type_merge/debug/out
output file:/Users/arirus/Desktop/Plugin/app/build/generated/res/pngs/debug
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/incremental/mergeDebugResources
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_res/debug

// 打包 Resources 
> Task :app:packageDebugResources UP-TO-DATE
--- task name packageDebugResources null
input file:/Users/arirus/Desktop/Plugin/app/src/main/res
input file:/Users/arirus/Desktop/Plugin/app/src/debug/res
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/data_binding_layout_info_type_package/debug/out
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/incremental/packageDebugResources
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/packaged_res/debug

> Task :app:parseDebugLocalResources UP-TO-DATE
--- task name parseDebugLocalResources null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/packaged_res/debug
input file:/Users/arirus/.gradle/caches/transforms-3/23c4123027e0e3ee5063a064181cd43d/transformed/R.txt
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/local_only_symbol_list/debug/R-def.txt

// 开始manifest相关处理
> Task :app:createDebugCompatibleScreenManifests UP-TO-DATE
--- task name createDebugCompatibleScreenManifests null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/compatible_screen_manifest/debug

> Task :app:extractDeepLinksDebug UP-TO-DATE
--- task name extractDeepLinksDebug null
input file:/Users/arirus/Desktop/Plugin/app/src/debug/res/navigation
input file:/Users/arirus/Desktop/Plugin/app/src/main/res/navigation
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/navigation_json/debug/navigation.json

> Task :app:processDebugMainManifest UP-TO-DATE
--- task name processDebugMainManifest null
input file:/Users/arirus/Desktop/Plugin/app/src/main/AndroidManifest.xml
...
input file:/Users/arirus/.gradle/caches/transforms-3/01bf24068b4652298f53a40e5ad8b190/transformed/annotation-experimental-1.1.0/AndroidManifest.xml
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/navigation_json/debug/navigation.json
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/manifest_merge_blame_file/debug/manifest-merger-blame-debug-report.txt
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_manifest/debug/AndroidManifest.xml
output file:/Users/arirus/Desktop/Plugin/app/build/outputs/logs/manifest-merger-debug-report.txt

> Task :app:processDebugManifest UP-TO-DATE
--- task name processDebugManifest null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/compatible_screen_manifest/debug
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_manifest/debug/AndroidManifest.xml
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_manifests/debug

// 结束manifest相关处理
> Task :app:processDebugManifestForPackage UP-TO-DATE
--- task name processDebugManifestForPackage null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_manifests/debug
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/packaged_manifests/debug

> Task :app:processDebugResources UP-TO-DATE
--- task name processDebugResources null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/aar_metadata_check/debug

input file:/Users/arirus/.gradle/caches/transforms-3/d7c18818464ae001e62e013a23a8cc82/transformed/androidx.annotation.experimental-r.txt
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_res/debug
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/local_only_symbol_list/debug/R-def.txt
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/packaged_manifests/debug
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_manifests/debug
input file:/Users/arirus/Library/Android/sdk/platforms/android-33/android.jar
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/compile_and_runtime_not_namespaced_r_class_jar/debug/R.jar
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/incremental/processDebugResources
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/processed_res/debug/out
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/symbol_list_with_package_name/debug/package-aware-r.txt
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/runtime_symbol_list/debug/R.txt

// 编译开始 先是kotlin
> Task :app:compileDebugKotlin UP-TO-DATE
--- task name compileDebugKotlin Compiles the debug kotlin.
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/compile_and_runtime_not_namespaced_r_class_jar/debug/R.jar
...
input file:/Users/arirus/Desktop/Plugin/app/src/main/java/com/example/plugin/MainActivity.kt
input file:/Users/arirus/Desktop/Plugin/app/build/generated/source/buildConfig/debug/xx/xxx/xxx/BuildConfig.java
input file:/Users/arirus/Desktop/Plugin/app/src/main/res/layout
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/tmp/kotlin-classes/debug

> Task :app:javaPreCompileDebug UP-TO-DATE
--- task name javaPreCompileDebug null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/annotation_processor_list/debug/annotationProcessors.json
// 编译class文件
> Task :app:compileDebugJavaWithJavac UP-TO-DATE
--- task name compileDebugJavaWithJavac null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/compile_and_runtime_not_namespaced_r_class_jar/debug/R.jar
...
input file:/Users/arirus/Desktop/Plugin/app/build/tmp/kotlin-classes/debug
input file:/Users/arirus/Desktop/Plugin/app/build/generated/source/buildConfig/debug/xx/xxx/xxx/BuildConfig.java
input file:/Users/arirus/Library/Android/sdk/platforms/android-33/android.jar
input file:/Users/arirus/Library/Android/sdk/build-tools/30.0.2/core-lambda-stubs.jar
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/annotation_processor_list/debug/annotationProcessors.json
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/javac/debug/classes
output file:/Users/arirus/Desktop/Plugin/app/build/generated/ap_generated_sources/debug/out
output file:/Users/arirus/Desktop/Plugin/app/build/tmp/compileDebugJavaWithJavac/previous-compilation-data.bin
// 编译class文件完成

> Task :app:compileDebugSources UP-TO-DATE
--- task name compileDebugSources null
---------------------------------------------------

> Task :app:mergeDebugNativeDebugMetadata NO-SOURCE
--- task name mergeDebugNativeDebugMetadata null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/outputs/native-debug-symbols/debug/native-debug-symbols.zip

> Task :app:mergeDebugShaders UP-TO-DATE
--- task name mergeDebugShaders null
input file:/Users/arirus/Desktop/Plugin/app/src/main/shaders
input file:/Users/arirus/Desktop/Plugin/app/src/debug/shaders
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/incremental/mergeDebugShaders
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_shaders/debug/out

// 编译 shaders
> Task :app:compileDebugShaders NO-SOURCE
--- task name compileDebugShaders null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_shaders/debug/out
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/shader_assets/debug/out

// 生成assets
> Task :app:generateDebugAssets UP-TO-DATE
--- task name generateDebugAssets null
---------------------------------------------------

> Task :app:mergeDebugAssets UP-TO-DATE
--- task name mergeDebugAssets null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/shader_assets/debug/out
input file:/Users/arirus/Desktop/Plugin/app/src/main/assets
input file:/Users/arirus/Desktop/Plugin/app/src/debug/assets
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/incremental/mergeDebugAssets
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_assets/debug/out

> Task :app:compressDebugAssets UP-TO-DATE
--- task name compressDebugAssets null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_assets/debug/out
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/compressed_assets/debug/out

> Task :app:processDebugJavaRes NO-SOURCE
--- task name processDebugJavaRes null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/java_res/debug/out

> Task :app:mergeDebugJavaResource UP-TO-DATE
--- task name mergeDebugJavaResource null
input file:/Users/arirus/.gradle/caches/modules-2/files-2.1/com.squareup.okhttp3/okhttp/4.11.0/436932d695b2c43f2c86b8111c596179cd133d56/okhttp-4.11.0.jar
...
input file:/Users/arirus/Desktop/Plugin/app/build/tmp/kotlin-classes/debug/META-INF/app_debug.kotlin_module
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/incremental/debug-mergeJavaRes/zip-cache
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_java_res/debug/base.jar

// 检查重复类
> Task :app:checkDebugDuplicateClasses UP-TO-DATE
--- task name checkDebugDuplicateClasses null
input file:/Users/arirus/.gradle/caches/transforms-3/001e8a22fbca2ca11485ba33e2ff8b2b/transformed/okhttp-4.11.0
...
input file:/Users/arirus/.gradle/caches/transforms-3/692a3c2a629e62d5ccb84bb3ee36f3be/transformed/listenablefuture-1.0
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/duplicate_classes_check/debug

> Task :app:desugarDebugFileDependencies UP-TO-DATE
--- task name desugarDebugFileDependencies null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/external_file_lib_dex_archives/debug

// 合并dex文件
> Task :app:mergeExtDexDebug UP-TO-DATE
--- task name mergeExtDexDebug null
...
input file:/Users/arirus/.gradle/caches/transforms-3/7362858bac2ae608fdb358a5ee00d138/transformed/listenablefuture-1.0
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/duplicate_classes_check/debug
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/dex/debug/mergeExtDexDebug

> Task :app:mergeLibDexDebug UP-TO-DATE
--- task name mergeLibDexDebug null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/duplicate_classes_check/debug
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/dex/debug/mergeLibDexDebug

> Task :app:dexBuilderDebug UP-TO-DATE
--- task name dexBuilderDebug null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/compile_and_runtime_not_namespaced_r_class_jar/debug/R.jar
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/javac/debug/classes
input file:/Users/arirus/Desktop/Plugin/app/build/tmp/kotlin-classes/debug
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/external_libs_dex_archive_with_artifact_transforms/debug/out
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/external_libs_dex_archive/debug/out
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/mixed_scope_dex_archive/debug/out
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/project_dex_archive/debug/out
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/sub_project_dex_archive/debug/out

> Task :app:mergeProjectDexDebug UP-TO-DATE
--- task name mergeProjectDexDebug null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/project_dex_archive/debug/out
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/mixed_scope_dex_archive/debug/out
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/duplicate_classes_check/debug
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/dex/debug/mergeProjectDexDebug

// 合并so文件
> Task :app:mergeDebugJniLibFolders UP-TO-DATE
--- task name mergeDebugJniLibFolders null
input file:/Users/arirus/Desktop/Plugin/app/src/main/jniLibs
input file:/Users/arirus/Desktop/Plugin/app/src/debug/jniLibs
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/incremental/mergeDebugJniLibFolders
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_jni_libs/debug/out

> Task :app:mergeDebugNativeLibs NO-SOURCE
--- task name mergeDebugNativeLibs null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_native_libs/debug/out

> Task :app:stripDebugDebugSymbols NO-SOURCE
--- task name stripDebugDebugSymbols null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_native_libs/debug/out
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/stripped_native_libs/debug/out

//签名检查
> Task :app:validateSigningDebug UP-TO-DATE
--- task name validateSigningDebug null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/validate_signing_config/debug

// app 元信息写入
> Task :app:writeDebugAppMetadata UP-TO-DATE
--- task name writeDebugAppMetadata null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/app_metadata/debug/app-metadata.properties

> Task :app:writeDebugSigningConfigVersions UP-TO-DATE
--- task name writeDebugSigningConfigVersions null
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/signing_config_versions/debug/signing-config-versions.json

// 最后打包
> Task :app:packageDebug UP-TO-DATE
--- task name packageDebug null
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/app_metadata/debug/app-metadata.properties
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/compressed_assets/debug/out
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/dex/debug/mergeExtDexDebug
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/dex/debug/mergeProjectDexDebug
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/dex/debug/mergeLibDexDebug
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/merged_java_res/debug/base.jar
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/stripped_native_libs/debug/out
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/packaged_manifests/debug
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/processed_res/debug/out
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/signing_config_versions/debug/signing-config-versions.json
input file:/Users/arirus/Desktop/Plugin/app/build/intermediates/validate_signing_config/debug
input file:/Users/arirus/.android/debug.keystore
---------------------------------------------------
output file:/Users/arirus/Desktop/Plugin/app/build/outputs/apk/debug/output-metadata.json
output file:/Users/arirus/Desktop/Plugin/app/build/intermediates/incremental/packageDebug/tmp
output file:/Users/arirus/Desktop/Plugin/app/build/outputs/apk/debug

> Task :app:assembleDebug UP-TO-DATE
--- task name assembleDebug Assembles main output for variant debug
---------------------------------------------------
```

