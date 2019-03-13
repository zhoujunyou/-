### Annotation Processor

 [annotationProcessor理解](https://blog.csdn.net/xx326664162/article/details/68490059)

自己瞎几把理解后续有时间再深入吧主要为了看glide实现：

[官方文档](https://docs.oracle.com/javase/7/docs/technotes/tools/solaris/javac.html#commandlineargfile)里有这么一段

![image-20181128174139833](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181128174139833.png)

apt的源码里AndroidAptPlugin.configureVariant有这么一段配置

```java
  if (processors) {
            javaCompile.options.compilerArgs += [
                    '-processor', processors
            ]
        }

        if (!(processors && aptExtension.disableDiscovery())) {
            javaCompile.options.compilerArgs += [
                    '-processorpath', processorPath
            ]
        }
```

拿Glide的compiler来说。所以应该就是打包编译时会执行一段处理就是去执行GlideAnnotationProcessor下的代码逻辑。

---

**gradle plugin的调试方法**。 Android Studio中按照如下步骤操作： *Menu → Run → Edit Configurations... → Add New Configuration → Remote → 自定义配置name → host: localhost → port: 5005 → OK*

glidei 项目中这里我在GlideAnnotationProcessor的init方法中打好断点  用的./gradlew :samples:contacturi:build -Dorg.gradle.debug=true --no-daemon命令行执行。这时会卡在Starting Daemon。自己配置的name是remote所以切到remote。然后点击调试按钮等待进入断点.

---

看了一下com.bumptech.glide.annotation.compiler的源码按下面步骤

1. libraryModuleProcessor.processModules(env)如果有带GlideModule注解的 并派生于LibraryGlideModule ,使用IndexerGenerator写入一个如下的类

   ```java
   @com.bumptech.glide.annotation.compiler.Index(}
         modules = "com.bumptech.glide.integration.okhttp3.OkHttpLibraryGlideModule"
     )
     public class Indexer_GlideModule_com_bumptech_glide_integration_okhttp3_OkHttpLibraryGlideModule
    {
    }
   ```

2.  extensionProcessor.processExtensions(env) 如果有带GlideExtension注解的类，使用GlideExtensionValidator验证可用性，也就是注解是否写对。比如构造方法是否用Private修饰，构造方法参数是否为空。如果类中还有带有GlideOption或GlideType修饰的方法。拿GlideOption来说需要验证方法上是否有NotNull注解，

    BaseRequestOptions是否是一个参数，如果是重写BaseRequestOptions方法是否有override值，返回值是否是BaseRequestOptions。验证通过，使用IndexerGenerator写入以下类

   ```java
   @Index(
       extensions = "com.bumptech.glide.samples.flickr.FlickrGlideExtension"
   )
   public class GlideIndexer_GlideExtension_com_bumptech_glide_samples_flickr_FlickrGlideExtension {
   }
   ```

3. 步骤1和2生成的类主要用于在AAR中写入这样在application 模块下处理时找到这些类去处理

4.  appModuleProcessor.processModules(set, env);获取带有GlideModule 的AppGlideModule判断是否有多个

5.  appModuleProcessor.maybeWriteAppModule()去处理，获取Index修饰的类得到参数

    requestOptionsGenerator.generate 生成GlideOptions类

    requestBuilderGenerator.generate生成GlideRequest类

    requestManagerGenerator.generate生成GlideRequests类

    requestManagerFactoryGenerator.generate生成GeneratedRequestManagerFactory

    glideGenerator.generate生成GlideApp类

    appModuleGenerator.generate生成GeneratedAppGlideModule类



   使用GlideApp时，调用GeneratedAppGlideModule中的几个方法里面会去调用我们在GlideModule修饰类的方法做初始配置，GeneratedRequestManagerFactory替换 RequestManagerRetriever.RequestManagerFactory build时得到

    RequestManager的子类GlideRequests。GlideRequests as时得到RequestBuilder的子类GlideRequest。

    GlideRequest可以直接使用我们在GlideExtension修饰的类中定义的静态方法，也可以apply一个GlideOptions，GlideOptions中也有GlideExtension修饰的类中定义的静态方法的相应方法。



    