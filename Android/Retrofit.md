### Retrofit

[深入分析Java ClassLoader原理](https://blog.csdn.net/xyang81/article/details/7292380)
而程序在启动的时候，并不会一次性加载程序所要用的所有class文件，而是根据程序的需要，通过Java的类加载机制（ClassLoader）来动态加载某个class文件到内存当中的，从而只有class文件被载入到了内存之后，才能被其它class所引用。所以ClassLoader就是用来动态加载class文件到内存当中用的。



**双亲委派模型**
**双亲委派模型过程**:某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。

使用双亲委派模型的好处在于Java类随着它的类加载器一起具备了一种带有优先级的层次关系。例如类java.lang.Object，它存在在rt.jar中，无论哪一个类加载器要加载这个类，最终都是委派给处于模型最顶端的Bootstrap ClassLoader进行加载，因此Object类在程序的各种类加载器环境中都是同一个类。相反，如果没有双亲委派模型而是由各个类加载器自行加载的话，如果用户编写了一个java.lang.Object的同名类并放在ClassPath中，那系统中将会出现多个不同的Object类，程序将混乱。因此，如果开发者尝试编写一个与rt.jar类库中重名的Java类，可以正常编译，但是永远无法被加载运行。

双亲委派模型的系统实现

在java.lang.ClassLoader的loadClass()方法中，先检查是否已经被加载过，若没有加载则调用父类加载器的loadClass()方法，若父加载器为空则默认使用启动类加载器作为父加载器。如果父加载失败，则抛出ClassNotFoundException异常后，再调用自

---

[JDK中的proxy动态代理原理剖析](https://www.jianshu.com/p/d5d9215bf8ad)

主要几个方法

```java
/*
 * Look up or generate the designated proxy class.
 */
private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces)//Proxy
public Class<?> apply(ClassLoader loader, Class<?>[] interfaces)//ProxyClassFactory
public static byte[] generateProxyClass(final String var0, Class<?>[] var1, int var2)//ProxyGenerator
 private static native Class<?> defineClass0(ClassLoader loader, String name,
                                                byte[] b, int off, int len);//Proxy
 cons.newInstance(new Object[]{h})
```



1. 代理类的class样本

```java
package com.sun.proxy;

import com.czq.proxy.IPackageManager;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy
  implements IPackageManager
{
  private static Method m3; // 生成对应的方法对象
  private static Method m1;
  private static Method m0;
  private static Method m2;
// proxy0 继承Proxy，实现IPackageManager 接口，需要传入 InvocationHandler，初始化对应的h对象。
// 我们的h对象就是PackageManagerWoker，所以我们会调用到PackageManagerWoker的 invoke方法。
// 所以是proxy0，调用InvocationHandler的 invoke 方法，传入对应的方法。InvocationHandler 放射调用对应的tagret中的方法。
  public $Proxy0(InvocationHandler paramInvocationHandler)
    throws
  {
    super(paramInvocationHandler);
  }

  public final String getPackageInfo()
    throws
  {
    try
    {
      return (String)this.h.invoke(this, m3, null);
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  public final boolean equals(Object paramObject)
    throws
  {
    try
    {
      return ((Boolean)this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  public final int hashCode()
    throws
  {
    try
    {
      return ((Integer)this.h.invoke(this, m0, null)).intValue();
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  public final String toString()
    throws
  {
    try
    {
      return (String)this.h.invoke(this, m2, null);
    }
    catch (RuntimeException localRuntimeException)
    {
      throw localRuntimeException;
    }
    catch (Throwable localThrowable)
    {
    }
    throw new UndeclaredThrowableException(localThrowable);
  }

  static
  {
    try
    {
     // 把各个方法，对应到成员变量上
      m3 = Class.forName("com.czq.proxy.IPackageManager").getMethod("getPackageInfo", new Class[0]);
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
    }
    throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
  }
}


```

1. newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)调用后会生成上面一个class文件，可以看到上面的文件实现了interfaces中的接口即IPackageManager接口，除了实现接口中的方法getPackageInfo() 还默认实现了toString,equals,hashcode的方法。还有一个InvocationHandler 成员。

2. loader就是用于从class文件中加载class用的，通过cl.getConstructor(constructorParams)及 cons.newInstance(new Object[]{h})实例化的到Proxy0对象并把h传入。 

3. 最后就是调用Proxy0这个对象的方法，可以看到每次都是去调用h.invoke方法，这就是我们要在

    InvocationHandler的invoke方法下写响应逻辑的原因.

4. 每调用newProxyInstance会以interfaces生成一个key,Proxy0对象做为一个值存入map中

   下一次同样调用直接从map中取出.

---

**retrofit是一个用注解修饰方法来定义如何去做一个http请求。**

![image-20181128113516304](/Users/meiweibuyondeng/Library/Application Support/typora-user-images/image-20181128113516304.png)

1. args到request放在请求这步是由于args是变的 而Method其他注解返回值类型是不变的。其中有一个 Map<Method, ServiceMethod<?>> serviceMethodCache缓存。

2. 不同的 ParameterHandler上有不同的Converter： Converter<?, RequestBody>,Converter<?, String> stringConverter。用于区分转换。ParameterHandler也是为了解决第一个问题吧

3. 需要直接使用ResposeBody的source 需要将resposeType==ResposeBody.这样的话BuiltInConverters  会去拦截converter。

4. Method反射这块没暂时没深入 写了Test

   ```java
   Observable<List<String>> test = ((Inner) Proxy.newProxyInstance(Inner.class.getClassLoader(), new Class<?>[] { Inner.class }, new InvocationHandler() {
       @Override
       public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
           Class<?> declaringClass = method.getDeclaringClass();//interface retrofit2.adapter.rxjava.ZhouTest$Inner
           Type genericReturnType = method.getGenericReturnType();//rx.Observable<java.util.List<java.lang.String>>
           Class<?> returnType = method.getReturnType();//class rx.Observable
           Annotation[] annotations = method.getAnnotations();//@retrofit2.http.HEAD(value=test head),@retrofit2.http.GET(value=test get)
           //{Annotation[1][]} @retrofit2.http.Path(encoded=false, value=test path),@retrofit2.http.Query(encoded=false, value=test qury )
           Annotation[][] parameterAnnotations = method.getParameterAnnotations();
           return Observable.just("test");
       }
   })).justTest("test");
   
   
    interface Inner{
           @HEAD("test head")
           @GET("test get")
           Observable<List<String>> justTest(@Path("test path")@Query("test qury ")String test);
       }
   ```



**最后理一下下面这些参数作用。**

```java
Retrofit retrofit = new Retrofit.Builder()
        .baseUrl(baseUrl)
        .client(client)
        .addConverterFactory(GsonConverterFactory.create())
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
        .build();
```

* client 传入的的是OkhttpClient。和callFactory()功能是一样。用于创建一个正在的调用网络请求的Call

  OkhttpClient中是RealCall。

*  Converter.Factory 用于创建responseBodyConverter，requestBodyConverter，stringConverter

   分别用于返回时ResponseBody的转换，请求时转换成RequestBody和转为String 

*  CallAdapter.Factory用于创建CallAdapter创建时会传入一个returnType用于确定responseType.

   responseBodyConverter需要转换为啥就拿的这个参数。CallAdapter的作用就是设配如何去调用Call。

  比如Rxjava，subscibe就会去调用Call