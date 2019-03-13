### Android 打包构建流程

点击android studio  Run按钮后发生了什么？ [知乎答案](https://www.zhihu.com/question/65289196)

1. **检查项目和读取基本配置(这一步主要在IDE的代码中[JetBrains/android](https://link.zhihu.com/?target=https%3A//github.com/JetBrains/android))**
2. **Gradle Build(主要用的com.android.build.gradle中的逻辑)**
3. **Apk Install & LaunchActivity(主要用的adb)**

---

主要理解一下build tool执行的过程  以下分析基于**com.android.tools.build:gradle:3.3.0**

* 可以这样调式 新建一个目录buildSrc下新建一个build.gradle文件 

  ```groovy
  //用于查看打包脚本
  apply plugin: 'java'
  apply plugin: 'groovy'
  repositories {
      google()
      jcenter()
     mavenLocal()
  
  }
  
  dependencies {
       compile gradleApi()
      compile localGroovy()
      compile 'com.android.tools.build:gradle:3.3.0'
  }
  ```

  执行assemble 找到TaskManager  chooseSource 找到相关sources.jar  

  https://mvnrepository.com/artifact/com.android.tools.build/gradle/3.3.0

  设置remote ，com.android.build.gradle.BasePlugin上的apply方法中打上断点。
  执行task命令后加上  -Dorg.gradle.debug=true --no-daemon 点击调式的按钮.

* Task的依赖关系可以查看ApplicationTaskManager中的createTasksForVariantScope方法

---

从 gradlew文件说起 

