## 1、前言

> 你是否有遇到过 ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension()返回的是一个 Protocol\$Adaptive 类型的对象，没办法继续跟踪源码而感到困惑？

> 你是否曾经很好奇从 zookeeper 注册中心切换到 nacos 注册中心，只需要将注册中心配置 zookeeper 修改成 nacos 即可？

> 你是否曾经好奇更换远程调用协议，只需要更改 dubbo:protocol 中的 name 属性即可？

其实这些都是 dubbo SPI 背后所做的，要更加深入了解 dubbo 源码，掌握 dubbo SPI 机制必不可少。

## 2、What is SPI？

[Dubbo SPI 官方文档](http://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html "Dubbo SPI 官方文档")

SPI 全称为 Service Provider Interface，是一种服务发现机制。SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。

---

### 2.1 JDK 中的 SPI

在 JDK 中使用 SPI 其实很简单，通过 ServiceLoader 类从 META-INF/services 对应的接口类全限定名文件中加载实现类配置中的实现类。如大家比较熟悉的 Jdbc，其 DriverManager 就是通过 ServiceLoader 去发现 Driver 接口的所有实现类。

```java
//定义动物接口
public interface Animal {
    void say();
}
//定义狗实现类
public class Dog implements Animal {
    @Override
    public void say() {
        System.out.println("汪汪汪");
    }
}
//定义猫实现类
public class Cat implements Animal {
    @Override
    public void say() {
        System.out.println("喵喵");
    }
}
public class TestJdkSpi {
    public static void main(String[] args){
    	  //加载动物接口实现类对象
        ServiceLoader<Animal> load = ServiceLoader.load(Animal.class);
        Iterator<Animal> iterator = load.iterator();
        while(iterator.hasNext()){
            //执行实现类方法
            iterator.next().say();
        }
    }
}
```

定义类路径下 META-INF/services/org.apache.dubbo.demo.consumer.Animal 配置文件

```properties
org.apache.dubbo.demo.consumer.Dog
org.apache.dubbo.demo.consumer.Cat
```

执行结果

![](http://qiniu.mdnice.com/4c30c62560599ce0c4c9b512c3277d03.png)

---

### 2.2 Dubbo 中的 SPI

dubbo 中的 SPI 也类似，其入口类为 ExtensionLoader，通过 ExtensionLoader.getExtensionLoader(Class<T> type)获取某个拓展接口的加载器。

ExtensionLoader 提供了三个成员方法加载实现类。

1. T getExtension(String name) 获取某个类型 key=name 的拓展接口实现类对象。

2. List\<T\> getActivateExtension(URL url, String[] values, String group) 根据 URL、values 和分组动态筛选符合条件的拓展接口实现类对象，返回集合。

3. T getAdaptiveExtension() 获取拓展接口适配对象，配合@Adaptive 注解可以根据 URL 的 parameter 获取具体的拓展接口实现类。此处需要注意@Adaptive 声明在具体实现类上，则该类就是拓展接口的适配类。如果仅仅声明在拓展接口的方法上，则会动态生成一个 XXX\$Adaptive 的拓展接口适配类。

三个成员方法都会触发 loadExtensionClasses()方法，此方法会寻找类路径下 META-INF/dubbo/internal/、 META-INF/dubbo/、META-INF/services/目录下，拓展接口类全限定名为文件名的配置文件，加载配置文件所有拓展接口的实现类。

```java
private Map<String, Class<?>> loadExtensionClasses() {
        cacheDefaultExtensionName();

        Map<String, Class<?>> extensionClasses = new HashMap<>();
	//加载类路径下 META-INF/dubbo/internal/拓展接口类全限定名 配置
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
  	//兼容旧的dubbo版本
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
 	 //加载类路径下 META-INF/dubbo/拓展接口类全限定名 配置
        loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
 	 //加载类路径下 META-INF/services/拓展接口类全限定名 配置
        loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        return extensionClasses;
  }
```

## 3. Dubbo 中的 SPI 注解

### 3.1 @SPI

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface SPI {
    //default extension name 默认拓展实现类key
    String value() default "";
}
```

@SPI 是接口是否拓展接口的一个标志，如果拓展接口没使用@SPI 注解声明，使用 ExtensionLoader 加载拓展接口会抛出 IllegalArgumentException 异常，而@SPI 上的 value 属性值是声明拓展接口默认实现类对应的 key 的值，value 属性值会存储在 ExtensionLoader 中的 cachedDefaultName,这个值会运用在 getDefaultExtension()方法中返回默认的拓展接口实现类对象和拓展接口适配对象中（拓展接口适配对象从 URL 中获取不到具体实现类的 Key 值时，会返回 cachedDefaultName 对应的拓展接口实现类对象）

---

### 3.2 @Adaptive

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {
    /**
     * @return parameter names in URL
     */
    String[] value() default {};

}
```

1. @Adaptive 可以声明在拓展接口实现类上，该实现类就会成为拓展接口的适配对象，当调用 getAdaptiveExtension 会返回该实现类对象。

```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

}
```

AdaptiveExtensionFactory 是 ExtensionFactory 拓展接口的一个实现类，其声明了@Adaptive。

2. @Adaptive 可以声明在拓展接口方法上，这样拓展接口的适配类就会由 dubbo 动态生成一个拓展接口适配类，
   以 XXX\$Adaptive 命名。

```java
@SPI("dubbo")
  public interface Protocol {
    int getDefaultPort();

  //未声明value值，默认会以URL.getProtocol()返回的值作为获取拓展接口实现类的key;若声明value值，会以URL.getParameter()返回的值作为获取拓展接口实现类的key
    @Adaptive
  	<T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    void destroy();
}
```

其生成的拓展接口适配类源码如下:

```java
package org.apache.dubbo.rpc;
import org.apache.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy()  {
        throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }
    public int getDefaultPort()  {
        throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }
    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
  	//"dubbo" 就是Protocol 的@SPI声明的value值dubbo
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
}
```

如何获取这份适配器类源码，会在下期 SPI 源码分析中细致说明。

---

### 3.3 @Activate

@Activate 是可以根据 URL、values 和分组动态筛选符合条件的拓展接口实现类对象，返回集合。使用场景有 dubbo 根据 CONSUMER, PROVIDER 两大分组，动态获取通过 Filter 拓展接口的实现类对象。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Activate {
  //分组定义，匹配分组
    String[] group() default {};
//匹配URL上的Parameter
    String[] value() default {};

    @Deprecated
    String[] before() default {};

    @Deprecated
    String[] after() default {};
//返回集合实现类集合的排序
    int order() default 0;
}
```

## 总结

SPI 是一个面向接口编程的思想，调用方只需要依赖接口，无需关心接口具体实现，通过某些规则或者条件动态创建实现类，使得系统只需更改某个配置项无需修改源代码即可完成不同实现的动态切换。

下一篇基于 getExtension()、getActivateExtension()、getAdaptiveExtension()三大方法源码分析。