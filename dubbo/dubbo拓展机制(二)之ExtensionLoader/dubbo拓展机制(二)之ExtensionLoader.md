## 1、前言

上期[Dubbo 拓展机制(一)之 SPI](https://mp.weixin.qq.com/s?__biz=Mzg4OTA2ODE4Nw==&mid=2247483751&idx=1&sn=edd18e03c1f8793cb245faaebcbedff2&chksm=cff0ce93f8874785dab92733717358e3c048431ed076a4b980b4895f57d47a360ad8375db705&token=1151108396&lang=zh_CN#rd)讲到 dubbo SPI 的一些基础知识，本期结合 dubbo SPI 最核心的类 ExtensionLoader 源码分析其内部实现。

## 2、ExtensionLoader 变量

```java
public class ExtensionLoader<T> {
    /**
     * 静态变量缓存，拓展接口对应的ExtensionLoader缓存，key：拓展接口Class对象  value：对应的ExtensionLoader对象
     */
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>();
    /**
     * 静态变量缓存，拓展接口实现类对象缓存，key：拓展接口实现类Class对象  value：实现类对象
     */
    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<>();
    /**
     * 拓展接口Class对象
     */
    private final Class<?> type;
    /**
     * 对象工厂，创建拓展接口实现类对象时ioc注入时获取对象的地方
     */
    private final ExtensionFactory objectFactory;
    /**
     * 实现类与标记名缓存，key:实现类Class对象  value：标记名
     */
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<>();
    /**
     * 实现类缓存，key：标记名  value：实现类Class对象
     */
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();
    /**
     * Activate注解信息缓存，key：标记名  value：Activate注解对象
     */
    private final Map<String, Object> cachedActivates = new ConcurrentHashMap<>();
    /**
     *拓展接口实现类对象缓存，key：标记名   value：实现类对象Holder
     */
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();
    /**
     * 拓展接口适配类对象缓存
     */
    private final Holder<Object> cachedAdaptiveInstance = new Holder<>();
    /**
     * 保存拓展接口的适配器类
     */
    private volatile Class<?> cachedAdaptiveClass = null;
    /**
     * 保存拓展接口默认标记名，也就是拓展接口上@SPI的value属性
     */
    private String cachedDefaultName;
}
```

cachedAdaptiveClass 变量上有 volatile 是为了保证线程操作修改的可见性。

## 3、ExtensionLoader 源码

### 3.1 getExtensionLoader

```java
 public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
 	//校验拓展接口是否接口类型，是否声明SPI接口）（代码略）

        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
        		//判断是否在缓存中存在，不存在创建
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

EXTENSION_LOADERS 是 ConcurrentMap，能保证线程安全即使多个线程同时执行 key 相同的 putIfAbsent 操作，也只有一个线程能真正 put 成功，所以下面的 get 方法返回的是执行 put 成功线程放入的 ExtensionLoader 对象，所以某个类型的拓展接口 type 的 ExtensionLoader 全局唯一

---

### 3.2 拓展接口实现类加载

拓展接口实现类加载过程由多个方法组成，getExtensionClasses 方法主要是判断实现类缓存是否存在，不存在执行 loadExtensionClasses 方法加载实现类。loadExtensionClasses 方法又会调用 loadDirectory 加载类路径的配置文件，最终通过 loadClass 方法加载实现类。关键逻辑都在 loadClass 方法中。

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        //校验实现类是否拓展接口实现类,省略
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            //如果实现类有Adaptive注解
            cacheAdaptiveClass(clazz);
        } else if (isWrapperClass(clazz)) {
            //如果实现类是包装类
            cacheWrapperClass(clazz);
        } else {
            //实现类必须有无参构造函数
            clazz.getConstructor();
            if (StringUtils.isEmpty(name)) {
                //如果标记名为空，获取实现类上Extension注解值作为标记名或实现类SimpleName
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                }
            }

            //逗号分割标记名
            String[] names = NAME_SEPARATOR.split(name);
            if (ArrayUtils.isNotEmpty(names)) {
                //缓存ActivateClass
                cacheActivateClass(clazz, names[0]);
                for (String n : names) {
                    //缓存标记名
                    cacheName(clazz, n);
                    saveInExtensionClass(extensionClasses, clazz, n);
                }
            }
        }
    }
```

如果实现类声明了@Adaptive 注解，则执行 cacheAdaptiveClass 方法，缓存拓展接口的适配器类信息

```java
private void cacheAdaptiveClass(Class<?> clazz) {
        if (cachedAdaptiveClass == null) {
            //缓存拓展接口适配器类到cachedAdaptiveClass对象
            cachedAdaptiveClass = clazz;
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            //如果适配器类缓存对象不为空且不等于当前实现类，说明拓展接口存在两个拥有@Adaptive实现类
            throw new IllegalStateException("More than 1 adaptive class found: "
                    + cachedAdaptiveClass.getName()
                    + ", " + clazz.getName());
        }
}
```

如果实现类存在形参为拓展接口 type 的构造方法，将实现类缓存到包装类缓存集合中

```java
private void cacheWrapperClass(Class<?> clazz) {
        if (cachedWrapperClasses == null) {
            cachedWrapperClasses = new ConcurrentHashSet<>();
        }
        //缓存包装类cachedWrapperClasses
        cachedWrapperClasses.add(clazz);
}
```

### 3.3 getExtension

```java
	public T getExtension(String name) {
  	//校验name是否为空（代码略）
        //如果name=ture的话，获取默认Default Extension
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        final Holder<Object> holder = getOrCreateHolder(name);
        Object instance = holder.get();
        //double check判断对象是否创建
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    //创建标记名对应实现类的对象
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

getOrCreateHolder 方法根据标记名创建唯一 holder 对象，通过 double check 检查 holder 里的实现类对象是否为空，如果为空执行 createExtension 创建实现类对象（dcl 在很多源码中都非常常见，建议深入理解）

```java
	 private T createExtension(String name) {
        //获取标识名对应的实现类
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            //ioc注入
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    //aop,返回包装对象
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```

createExtension 方法会去 EXTENSION_INSTANCES 缓存中获取对象，如果不存在，创建实现类对象。然后执行 injectExtension 对实现类对象进行元素注入和使用 cachedWrapperClasses 缓存的包装类来包装实现类对象（ioc 和 aop）。

```java
private T injectExtension(T instance) {
        if (objectFactory == null) {
            //如果对象工厂不存在，不执行注入逻辑
            return instance;
        }
        try {
            for (Method method : instance.getClass().getMethods()) {
                if (!isSetter(method)) {
                    //判断是否set方法
                    //非set方法跳过
                    continue;
                }
                if (method.getAnnotation(DisableInject.class) != null) {
                    //方法声明了DisableInject注解跳过
                    continue;
                }
                Class<?> pt = method.getParameterTypes()[0];
                if (ReflectUtils.isPrimitives(pt)) {
                    //如果方法参数是原始类型跳过（原始类型：原始类型、包装类、String、Date）
                    continue;
                }

                try {
                    String property = getSetterProperty(method);
                    //通过对象工厂获取需要注入对象 //ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension(),objectFactory初始化代码，其应该是一个适配器类
                    //ExtensionFactory的适配器实现类为AdaptiveExtensionFactory，其逻辑是遍历所有的ExtensionFactory实现类，执行getExtension方法，直到返回的object不为空
                    Object object = objectFactory.getExtension(pt, property);
                    if (object != null) {
                        method.invoke(instance, object);
                    }
                } catch (Exception e) {
                    logger.error("Failed to inject via method " + method.getName()
                            + " of interface " + type.getName() + ": " + e.getMessage(), e);
                }

            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

objectFactory 对象在 ExtensionLoader 构造方法中通过 getAdaptiveExtension 创建，即 AdaptiveExtensionFactory 实现类创建的对象，该类具体逻辑是遍历所有的 ExtensionFactory 实现类，执行 getExtension 方法，直到返回的 object 不为空。ExtensionFactory 有两个实现类，分别是 SpiExtensionFactory 和 SpringExtensionFactory。SpiExtensionFactory 是通过 ExtensionLoader 来获取实现类对象，而 SpringExtensionFactory 是基于 Spring 容器来获取实现类对象。

### 3.4 getAdaptiveExtension

```java
	public T getAdaptiveExtension() {
        Object instance = cachedAdaptiveInstance.get();
        //double check
        if (instance == null) {
            if (createAdaptiveInstanceError != null) {
                throw new IllegalStateException("Failed to create adaptive instance: " +
                        createAdaptiveInstanceError.toString(),
                        createAdaptiveInstanceError);
            }

            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        //创建适配器类对象
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        }

        return (T) instance;
    }
```

先判断拓展接口配器类对象缓存是否存在，如果不存在调用 createAdaptiveExtension()方法创建

```java
private T createAdaptiveExtension() {
        try {
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }

private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            //如果拓展接口适配器类不为空直接返回
            return cachedAdaptiveClass;
        }
        //动态编译生成拓展接口适配器类
        return cachedAdaptiveClass = createAdaptiveExten获取拓展接口适配器类，如果缓存sionClass();
    }
```

createAdaptiveExtension 方法会调用 getAdaptiveExtensionClass 获取拓展接口适配器类，如果缓存 cachedAdaptiveClass 为空，会调用 getAdaptiveExtensionClass 方法为拓展接口动态生成字节码创建一个适配器类，类名为 XXX\$Adaptive。

```java
  private Class<?> createAdaptiveExtensionClass() {
        //生成拓展接口适配类源码
        String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
        ClassLoader classLoader = findClassLoader();
        //编译代码
        org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
    }
```

可以看到 code 就是拓展接口适配类的源码。

### 3.5 getActivateExtension

```java
 public List<T> getActivateExtension(URL url, String[] values, String group) {
        List<T> exts = new ArrayList<>();
        List<String> names = values == null ? new ArrayList<>(0) : Arrays.asList(values);
        if (!names.contains(REMOVE_VALUE_PREFIX + DEFAULT_KEY)) {
            //如果传入得values不包含"-default"
            getExtensionClasses();
            for (Map.Entry<String, Object> entry : cachedActivates.entrySet()) {
                String name = entry.getKey();
                Object activate = entry.getValue();

                String[] activateGroup, activateValue;

                if (activate instanceof Activate) {
                    activateGroup = ((Activate) activate).group();
                    activateValue = ((Activate) activate).value();
                } else if (activate instanceof com.alibaba.dubbo.common.extension.Activate) {
                    activateGroup = ((com.alibaba.dubbo.common.extension.Activate) activate).group();
                    activateValue = ((com.alibaba.dubbo.common.extension.Activate) activate).value();
                } else {
                    continue;
                }
                if (isMatchGroup(group, activateGroup)
                        && !names.contains(name)
                        && !names.contains(REMOVE_VALUE_PREFIX + name)
                        //当前activateValue为空返回true，或者不为空时需要url上有对应parameter且值不为空返回true
                        && isActive(activateValue, url)) {
                    exts.add(getExtension(name));
                }
            }
            exts.sort(ActivateComparator.COMPARATOR);
        }
        List<T> usrs = new ArrayList<>();
        for (int i = 0; i < names.size(); i++) {
            String name = names.get(i);
            if (!name.startsWith(REMOVE_VALUE_PREFIX)
                    && !names.contains(REMOVE_VALUE_PREFIX + name)) {
                if (DEFAULT_KEY.equals(name)) {
                    if (!usrs.isEmpty()) {
                        //讲自定义的实现类放到默认实现类前面
                        exts.addAll(0, usrs);
                        usrs.clear();
                    }
                } else {
                    usrs.add(getExtension(name));
                }
            }
        }
        if (!usrs.isEmpty()) {
            exts.addAll(usrs);
        }
        return exts;
    }
```

getActivateExtension 方法大致逻辑是先从拓展接口的所有默认实现类中，根据实现类的@Activate 注解信息判断默认实现类是否符合传入 URL 规则。然后再获取 values 数组的标记名的实现类。**-** 字符是关键字，用来排除某个标记名的实现类。-default 是不获取默认实现类，只加载 values 数组的标记名的实现类。

## 4、最后

源码解析已上传到 [github](https://github.com/Linliangkung/DubboCodeAnalysis "github")，结合源码看看吧。