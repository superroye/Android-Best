# AndroidX概述

AndroidX是Android团队用于在[Jetpack中](https://developer.android.google.cn/jetpack)开发，测试，打包，版本和发布库的开源项目 。

AndroidX是对原始Android [支持库](https://developer.android.google.cn/topic/libraries/support-library/index)的重大改进 。与支持库一样，AndroidX与Android操作系统分开提供，并提供跨Android版本的向后兼容性。AndroidX通过提供功能奇偶校验和新库完全取代了支持库。此外，AndroidX还包括以下功能：

- AndroidX中的所有软件包都以字符串开头，位于一致的命名空间中`androidx`。支持库包已映射到相应的`androidx.*`包中。有关所有旧类和构建工件的完整映射到新构件，请参阅“ [包重构”](https://developer.android.google.cn/jetpack/androidx/refactor)页面。
- 与支持库不同，AndroidX软件包是单独维护和更新的。这些`androidx`包使用 从版本1.0.0开始的严格[语义版本控制](https://semver.org/)。您可以单独更新项目中的AndroidX库。
- 所有新的支持库开发都将在AndroidX库中进行。这包括维护原始支持库工件和引入新的Jetpack组件。

## 使用AndroidX

请参阅[迁移到AndroidX](https://developer.android.google.cn/jetpack/androidx/migrate)以了解如何迁移现有项目。

如果要在新项目中使用AndroidX，则需要将compile SDK设置为Android 9.0（API级别28）或更高版本，并`true`在[`gradle.properties`文件中](https://developer.android.google.cn/studio/build/#properties-files)设置以下两个Android Gradle插件标志。

- `android.useAndroidX`：设置`true`为时，Android插件使用相应的AndroidX库而不是支持库。`false`如果未指定，则默认情况下为该标志 。
- `android.enableJetifier`：设置`true`为时，Android插件会自动迁移现有的第三方库，通过重写其二进制文件来使用AndroidX。`false`如果未指定，则默认情况下为该标志。

## AndroidX参考

AndroidX中的所有包和类都可以在 [AndroidX参考部分找到](https://developer.android.google.cn/reference/androidx/packages)。

## 其他资源

Jetpack组件是AndroidX库的一部分。在Jetpack [主页上](https://developer.android.google.cn/jetpack)了解有关组件的更多信息。

有关从支持库到AndroidX的包重构的更多信息， [请参阅博客文章](https://android-developers.googleblog.com/2018/05/hello-world-androidx.html)。