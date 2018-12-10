# Gradle依赖项学习总结，dependencies、transitive、force、exclude的使用与依赖冲突解决

（转载）：http://www.paincker.com/gradle-dependencies

Gradle是一个非常好用的编译工具，特别是继承了maven的依赖项管理功能，需要的Library不需要像传统IDE一样手动下载复制到项目中，只需要简单的写一行gradle脚本，就能自动下载下来并编译。

但是有时候会出现各种不明情况的报错，最常见的一种原因就是依赖项版本冲突。

每个模块都可能依赖其他模块，这些模块又会依赖别的模块。而一个项目中的多个模块，对同一个模块的不同版本有依赖，就可能产生冲突。

通过gradle命令查看依赖树，可以比较直观的看到冲突。具体方法是在模块所在的目录，也即build.gradle所在目录下执行`gradle dependencies`（需要将gradle加入PATH环境变量），执行结果如图。

![img](http://www.paincker.com/wp-content/uploads/2016/02/578dd292af0d099ed1a73f427f6b5b4b_ef3932fd-7f78-4bf7-a2c3-c1011520095c.png)

## Transitive

Transitive用于自动处理子依赖项。默认为true，gradle自动添加子依赖项，形成一个多层树形结构；设置为false，则需要手动添加每个依赖项。

### 案例

以安卓单元测试espresso的配置为例，gradle依赖如下：

```undefined
dependencies {   androidTestCompile('com.android.support.test:runner:0.2')   androidTestCompile('com.android.support.test:rules:0.2')   androidTestCompile('com.android.support.test.espresso:espresso-core:2.1')}
```

运行gradle dependencies的结果如下。可以看到每个包的依赖项都被递归分析并添加进来。

```undefined
+--- com.android.support.test:runner:0.2|    +--- junit:junit-dep:4.10|    |    \--- org.hamcrest:hamcrest-core:1.1|    +--- com.android.support.test:exposed-instrumentation-api-publish:0.2|    \--- com.android.support:support-annotations:22.0.0+--- com.android.support.test:rules:0.2|    \--- com.android.support.test:runner:0.2 (*)\--- com.android.support.test.espresso:espresso-core:2.1     +--- com.android.support.test:rules:0.2 (*)     +--- com.squareup:javawriter:2.1.1     +--- org.hamcrest:hamcrest-integration:1.1     |    \--- org.hamcrest:hamcrest-core:1.1     +--- com.android.support.test.espresso:espresso-idling-resource:2.1     +--- org.hamcrest:hamcrest-library:1.1     |    \--- org.hamcrest:hamcrest-core:1.1     +--- javax.inject:javax.inject:1     +--- com.google.code.findbugs:jsr305:2.0.1     +--- com.android.support.test:runner:0.2 (*)     +--- javax.annotation:javax.annotation-api:1.2     \--- org.hamcrest:hamcrest-core:1.1
```

### 统一指定transitive

可以给dependencies统一指定transitive为false，再次执行dependencies可以看到如下结果。

```undefined
configurations.all {   transitive = false}dependencies {   androidTestCompile('com.android.support.test:runner:0.2')   androidTestCompile('com.android.support.test:rules:0.2')   androidTestCompile('com.android.support.test.espresso:espresso-core:2.1')}
+--- com.android.support.test:runner:0.2+--- com.android.support.test:rules:0.2\--- com.android.support.test.espresso:espresso-core:2.1
```

### 单独指定依赖项的transitive

```undefined
dependencies {   androidTestCompile('com.android.support.test:runner:0.2')   androidTestCompile('com.android.support.test:rules:0.2')   androidTestCompile('com.android.support.test.espresso:espresso-core:2.1') {       transitive = false   }}
```

## 版本冲突

在同一个配置下（例如androidTestCompile），某个模块的不同版本同时被依赖时，默认使用最新版，gradle同步时不会报错。例如下面的hamcrest-core和runner。

```undefined
dependencies {   androidTestCompile('com.android.support.test:runner:0.4')   androidTestCompile('com.android.support.test:rules:0.2')   androidTestCompile('com.android.support.test.espresso:espresso-core:2.1')}
+--- com.android.support.test:runner:0.4|    +--- com.android.support:support-annotations:23.0.1|    +--- junit:junit:4.12|    |    \--- org.hamcrest:hamcrest-core:1.3|    \--- com.android.support.test:exposed-instrumentation-api-publish:0.4+--- com.android.support.test:rules:0.2|    \--- com.android.support.test:runner:0.2 -> 0.4 (*)\--- com.android.support.test.espresso:espresso-core:2.1     +--- com.android.support.test:rules:0.2 (*)     +--- com.squareup:javawriter:2.1.1     +--- org.hamcrest:hamcrest-integration:1.1     |    \--- org.hamcrest:hamcrest-core:1.1 -> 1.3     +--- com.android.support.test.espresso:espresso-idling-resource:2.1     +--- org.hamcrest:hamcrest-library:1.1     |    \--- org.hamcrest:hamcrest-core:1.1 -> 1.3     +--- javax.inject:javax.inject:1     +--- com.google.code.findbugs:jsr305:2.0.1     +--- com.android.support.test:runner:0.2 -> 0.4 (*)     +--- javax.annotation:javax.annotation-api:1.2     \--- org.hamcrest:hamcrest-core:1.1 -> 1.3
```

## Force

force强制设置某个模块的版本。

```undefined
configurations.all {   resolutionStrategy {       force 'org.hamcrest:hamcrest-core:1.3'   }}dependencies {   androidTestCompile('com.android.support.test:runner:0.2')   androidTestCompile('com.android.support.test:rules:0.2')   androidTestCompile('com.android.support.test.espresso:espresso-core:2.1')}
```

可以看到，原本对hamcrest-core 1.1的依赖，全部变成了1.3。

```undefined
+--- com.android.support.test:runner:0.2|    +--- junit:junit-dep:4.10|    |    \--- org.hamcrest:hamcrest-core:1.1 -> 1.3|    +--- com.android.support.test:exposed-instrumentation-api-publish:0.2|    \--- com.android.support:support-annotations:22.0.0+--- com.android.support.test:rules:0.2|    \--- com.android.support.test:runner:0.2 (*)\--- com.android.support.test.espresso:espresso-core:2.1     +--- com.android.support.test:rules:0.2 (*)     +--- com.squareup:javawriter:2.1.1     +--- org.hamcrest:hamcrest-integration:1.1     |    \--- org.hamcrest:hamcrest-core:1.1 -> 1.3     +--- com.android.support.test.espresso:espresso-idling-resource:2.1     +--- org.hamcrest:hamcrest-library:1.1     |    \--- org.hamcrest:hamcrest-core:1.1 -> 1.3     +--- javax.inject:javax.inject:1     +--- com.google.code.findbugs:jsr305:2.0.1     +--- com.android.support.test:runner:0.2 (*)     +--- javax.annotation:javax.annotation-api:1.2     \--- org.hamcrest:hamcrest-core:1.1 -> 1.3
```

## Exclude

Exclude可以设置不编译指定的模块

```undefined
configurations {   all*.exclude group: 'org.hamcrest', module: 'hamcrest-core'}dependencies {   androidTestCompile('com.android.support.test:runner:0.2')   androidTestCompile('com.android.support.test:rules:0.2')   androidTestCompile('com.android.support.test.espresso:espresso-core:2.1')}
+--- com.android.support.test:runner:0.2|    +--- junit:junit-dep:4.10|    +--- com.android.support.test:exposed-instrumentation-api-publish:0.2|    \--- com.android.support:support-annotations:22.0.0+--- com.android.support.test:rules:0.2|    \--- com.android.support.test:runner:0.2 (*)\--- com.android.support.test.espresso:espresso-core:2.1     +--- com.android.support.test:rules:0.2 (*)     +--- com.squareup:javawriter:2.1.1     +--- org.hamcrest:hamcrest-integration:1.1     +--- com.android.support.test.espresso:espresso-idling-resource:2.1     +--- org.hamcrest:hamcrest-library:1.1     +--- javax.inject:javax.inject:1     +--- com.google.code.findbugs:jsr305:2.0.1     +--- com.android.support.test:runner:0.2 (*)     \--- javax.annotation:javax.annotation-api:1.2
```

### 单独使用group或module参数

exclude后的参数有group和module，可以分别单独使用，会排除所有匹配项。例如下面的脚本匹配了所有的group为’com.android.support.test’的模块。

```undefined
configurations {   all*.exclude group: 'com.android.support.test'}dependencies {   androidTestCompile('com.android.support.test:runner:0.2')   androidTestCompile('com.android.support.test:rules:0.2')   androidTestCompile('com.android.support.test.espresso:espresso-core:2.1')}
\--- com.android.support.test.espresso:espresso-core:2.1     +--- com.squareup:javawriter:2.1.1     +--- org.hamcrest:hamcrest-integration:1.1     |    \--- org.hamcrest:hamcrest-core:1.1     +--- com.android.support.test.espresso:espresso-idling-resource:2.1     +--- org.hamcrest:hamcrest-library:1.1     |    \--- org.hamcrest:hamcrest-core:1.1     +--- javax.inject:javax.inject:1     +--- com.google.code.findbugs:jsr305:2.0.1     +--- javax.annotation:javax.annotation-api:1.2     \--- org.hamcrest:hamcrest-core:1.1
```

### 单独给某个模块指定exclude

```undefined
dependencies {   androidTestCompile('com.android.support.test:runner:0.2')   androidTestCompile('com.android.support.test:rules:0.2')   androidTestCompile('com.android.support.test.espresso:espresso-core:2.1') {       exclude group: 'org.hamcrest'   }}
+--- com.android.support.test:runner:0.2|    +--- junit:junit-dep:4.10|    |    \--- org.hamcrest:hamcrest-core:1.1|    +--- com.android.support.test:exposed-instrumentation-api-publish:0.2|    \--- com.android.support:support-annotations:22.0.0+--- com.android.support.test:rules:0.2|    \--- com.android.support.test:runner:0.2 (*)\--- com.android.support.test.espresso:espresso-core:2.1     +--- com.android.support.test:rules:0.2 (*)     +--- com.squareup:javawriter:2.1.1     +--- com.android.support.test.espresso:espresso-idling-resource:2.1     +--- javax.inject:javax.inject:1     +--- com.google.code.findbugs:jsr305:2.0.1     +--- com.android.support.test:runner:0.2 (*)     \--- javax.annotation:javax.annotation-api:1.2
```

## 不同配置下的版本冲突

同样的配置下的版本冲突，会自动使用最新版；而不同配置下的版本冲突，gradle同步时会直接报错。可使用exclude、force解决冲突。

例如`compile 'com.android.support:appcompat-v7:23.1.1'`，和`androidTestCompile 'com.android.support.test.espresso:espresso-core:2.1'`，所依赖的`com.android.support:support-annotations`版本不同，就会导致冲突。

```undefined
dependencies {   compile 'com.android.support:appcompat-v7:23.1.1'   androidTestCompile('com.android.support.test:runner:0.2')   androidTestCompile('com.android.support.test:rules:0.2')   androidTestCompile('com.android.support.test.espresso:espresso-core:2.1')}
```

gradle同步时会提示

```undefined
Warning:Conflict with dependency 'com.android.support:support-annotations'. Resolved versions for app and test app differ.
```

执行dependencies会提示

```undefined
FAILURE: Build failed with an exception.* What went wrong:A problem occurred configuring project ':app'.> Conflict with dependency 'com.android.support:support-annotations'. Resolved versions for app and test app differ.* Try:Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.BUILD FAILED
```

## 不兼容

虽然可以通过force、exclude等方式避免依赖项版本冲突，使得grade同步成功，但是并不能代表编译时没有问题。由于不同版本可能不完全兼容，于是会出现各种奇怪的报错。已知的解决思路是更改包的版本、尝试强制使用不同版本的依赖项，找到可兼容的依赖组合。

报错例如：

```undefined
com.android.dex.DexException: Multiple dex files define Lorg/hamcrest/MatcherAssert;at com.android.dx.merge.DexMerger.readSortableTypes(DexMerger.java:596)at com.android.dx.merge.DexMerger.getSortedTypes(DexMerger.java:554)at com.android.dx.merge.DexMerger.mergeClassDefs(DexMerger.java:535)at com.android.dx.merge.DexMerger.mergeDexes(DexMerger.java:171)at com.android.dx.merge.DexMerger.merge(DexMerger.java:189)at com.android.dx.command.dexer.Main.mergeLibraryDexBuffers(Main.java:454)at com.android.dx.command.dexer.Main.runMonoDex(Main.java:303)at com.android.dx.command.dexer.Main.run(Main.java:246)at com.android.dx.command.dexer.Main.main(Main.java:215)at com.android.dx.command.Main.main(Main.java:106)Error:Execution failed for task ':app:dexDebugAndroidTest'.> com.android.ide.common.process.ProcessException: org.gradle.process.internal.ExecException: Process 'command '/Library/Java/JavaVirtualMachines/jdk1.8.0_40.jdk/Contents/Home/bin/java'' finished with non-zero exit value 2BUILD FAILED
```

又例如Android执行Espresso单元测试时出现：

```undefined
Running testsTest running startedjava.lang.NoSuchMethodError: org.hamcrest.core.AnyOf.anyOfat org.hamcrest.Matchers.anyOf(Matchers.java:87)at android.support.test.espresso.Espresso.<clinit>(Espresso.java:158)at com.jzj1993.unittest.test.MainActivityEspressoTest.sayHello(MainActivityEspressoTest.java:28)at java.lang.reflect.Method.invokeNative(Native Method)at java.lang.reflect.Method.invoke(Method.java:525)at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:45)at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:15)at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:42)at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:20)at android.support.test.internal.statement.UiThreadStatement.evaluate(UiThreadStatement.java:55)at android.support.test.rule.ActivityTestRule$ActivityStatement.evaluate(ActivityTestRule.java:257)at org.junit.rules.RunRules.evaluate(RunRules.java:18)at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:263)at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:68)at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:47)at org.junit.runners.ParentRunner$3.run(ParentRunner.java:231)at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:60)at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:229)at org.junit.runners.ParentRunner.access$000(ParentRunner.java:50)at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:222)at org.junit.runners.ParentRunner.run(ParentRunner.java:300)at org.junit.runners.Suite.runChild(Suite.java:128)at org.junit.runners.Suite.runChild(Suite.java:24)at org.junit.runners.ParentRunner$3.run(ParentRunner.java:231)at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:60)at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:229)at org.junit.runners.ParentRunner.access$000(ParentRunner.java:50)at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:222)at org.junit.runners.ParentRunner.run(ParentRunner.java:300)at org.junit.runner.JUnitCore.run(JUnitCore.java:157)at org.junit.runner.JUnitCore.run(JUnitCore.java:136)at android.support.test.internal.runner.TestExecutor.execute(TestExecutor.java:54)at android.support.test.runner.AndroidJUnitRunner.onStart(AndroidJUnitRunner.java:228)at android.app.Instrumentation$InstrumentationThread.run(Instrumentation.java:1701)Finish
```