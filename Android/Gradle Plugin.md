### Gradle Plugin 

虽然不常用 起码要看懂

- resources/META-INF/gradle-plugins 这个文件夹结构是强制要求的，否则不能识别成插件。

   implementation-class=com.tencent.matrix.plugin.MatrixPlugin

* **在Gradle插件开发中，所有的插件都要继承`org.gradle.api.Plugin`接口，并且需要重写void apply(Project project) 方法,这个方法将会传入使用这个插件的 project 的实例，这是一个重要的 context**。

* ![image-20181226112314229](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226112314229.png)

  create的第一个参数`matrix`是我们自定义配置的DSL名字，第二个参数是参数类的名字

  通过`matrix`这个DSL这个名字，我们可以任意的改变参数类中相应字段的值。这样就带来了很大的便利。

* Gradle脚本的执行分为三个过程：

  **初始化** :分析有哪些module将要被构建，为每个module创建对应的 project实例。这个时候settings.gradle文件会被解析。

  **配置**：处理所有的模块的 build 脚本，处理依赖，属性等。这个时候每个模块的build.gradle文件会被解析并配置，这个时候会构建整个task的链表（这里的链表仅仅指存在依赖关系的task的集合，不是数据结构的链表）。

  **执行：**根据task链表来执行某一个特定的task，这个task所依赖的其他task都将会被提前执行.

* ![image-20181226112735974](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181226112735974.png)

  配置完了以后，有一个重要的回调`project.afterEvaluate`，它表示所有的模块都已经配置完了，可以准备执行task了；

* [Gradle Transform](http://tools.android.com/tech-docs/new-build-system/transform-api)是Android官方提供给开发者在项目构建阶段即由class到dex转换期间修改class文件的一套api。目前比较经典的应用是字节码插桩、代码注入技术