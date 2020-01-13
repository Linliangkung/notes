## 1、前言

在[Dubbo 服务暴露(一)之配置](https://mp.weixin.qq.com/s?__biz=Mzg4OTA2ODE4Nw==&mid=2247483744&idx=1&sn=9350af87ec5c9d53c34484038d28f4f9&chksm=cff0ce94f8874782cd53eaf55ed8e9c4cdb0ade6a507ae9a7eb5990a633efeda2311d521c5fc&token=1015887673&lang=zh_CN#rd)一文中讲到服务暴露的配置，其中提到一个关键的类 ServiceConfig，该类封装了服务提供者的所有配置信息，以及提供了暴露服务的关键方法 export。

Dubbo 与 Spring 集成时，通过 ServiceBean 继承 ServiceConfig 类，封装一些与 Spring 相关的操作，最终都是通过 ServiceConfig 的 export 方法进行服务的暴露。

## 2、ServiceBean

ServiceBean 大致可以分为两大块逻辑，首先封装服务提供者一些配置信息校验与初始化，其次是监听 Spring 容器的 ContextRefreshedEvent 事件执行父类 export 方法进行服务暴露。

### 2.1 afterPropertiesSet

ServiceBean 实现 InitializingBean 接口，Spring 创建对象后回调 afterPropertiesSet 方法。在此方法中执行一些配置信息的非空校验，如果为空会在 Spring 容器中获取相关对象进行默认配置的初始化。

```java
public void afterPropertiesSet() throws Exception {
//此处省略一系列校验代码
if (CollectionUtils.isEmpty(getProtocols())
                && (getProvider() == null || CollectionUtils.isEmpty(getProvider().getProtocols()))) {
            //如果applicationContext不为空，则通过spring容器获取类型为ProtocolConfig的对象，
            // 并封装成一个Map，Map的key为ProtocolConfig的名称，value为对象本身
            Map<String, ProtocolConfig> protocolConfigMap = applicationContext == null ? null : BeanFactoryUtils.beansOfTypeIncludingAncestors(applicationContext, ProtocolConfig.class, false, false);
            if (protocolConfigMap != null && protocolConfigMap.size() > 0) {
                List<ProtocolConfig> protocolConfigs = new ArrayList<ProtocolConfig>();
                if (StringUtils.isNotEmpty(getProtocolIds())) {
                    Arrays.stream(COMMA_SPLIT_PATTERN.split(getProtocolIds()))
                            .forEach(id -> {
                                if (protocolConfigMap.containsKey(id)) {
                                    protocolConfigs.add(protocolConfigMap.get(id));
                                }
                            });
                }

                if (protocolConfigs.isEmpty()) {
                    for (ProtocolConfig config : protocolConfigMap.values()) {
                        if (StringUtils.isEmpty(protocolIds)) {
                            //如果当前ServiceBean的协议配置为空且协议id配置为空，则添加spring容器的所有协议
                            protocolConfigs.add(config);
                        }
                    }
                }

                if (!protocolConfigs.isEmpty()) {
                    super.setProtocols(protocolConfigs);
                }
            }
        }
    if (!supportedApplicationListener) {
            //如果spring版本不支持事件，则在初始化参数方法中直接暴露服务
            export();
    }
}
```

### 2.2 onApplicationEvent

监听 ContextRefreshedEvent 事件，进行服务的暴露

```java
@Override
public void onApplicationEvent(ContextRefreshedEvent event) {
  //监听ContextRefreshedEvent事件
    if (!isExported() && !isUnexported()) {
      	//如果服务需要暴露且尚未暴露
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
        export();
    }
}
```

## 3、ServiceConfig

### 3.1 提供者配置信息

ServiceConfig 中的一些提供者的配置信息是被抽象到不同的父类中。
![ServiceConfig类继承图](https://imgkr.cn-bj.ufileos.com/9e74cb95-a3eb-4450-84e1-83b503f1e098.jpg)

1. ServiceConfig 继承 AbstractServiceConfig，其封装了服务提供者独有的元数据信息，如 version（版本号）、group（分组）、weight（权重）、protocols（协议）等信息。
2. AbstractServiceConfig 继承 AbstractInterfaceConfig，其封装了服务提供者与消费者公共的元数据信息，如 stub（本地存根），filter（过滤器），listener（监听器），connections（连接数），registries(注册中心)等信息。
3. AbstractInterfaceConfig 继承 AbstractMethodConfig，其封装了方法的配置信息。
4. AbstractMethodConfig 继承 AbstractConfig，其封装了一些常量信息。

具体提供者的配置的作用篇幅较长，不一一阐述，可参照[官方文档](http://dubbo.apache.org/zh-cn/docs/user/configuration/xml.html "官方文档")。

### 3.2 提供者服务暴露

#### 3.2.1 export

服务暴露关键入口是 export 方法，此方法中先是服务提供者一些关键信息的校验。校验通过后，判断服务是延迟暴露还是实时暴露，如果延迟暴露则通过线程池进行服务的延迟暴露。

```java
public synchronized void export() {
    //校验ServiceConfig各种配置信息
    checkAndUpdateSubConfigs();

    //是否需要暴露，对应<dubbo:service export="true"/>
    if (!shouldExport()) {
        return;
    }

    if (shouldDelay()) {
        //延迟暴露,对应<dubbo:service delay="5"/>
        DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
    } else {
        //执行暴露服务具体逻辑
        doExport();
    }
}
```

#### 3.2.2 doExport

export 方法中会调用 doExport 方法进行服务暴露，在 doExport 方法中只是做些键盘的变量修改，将 exported 修改成 true，标示当前服务已经暴露。

```java
protected synchronized void doExport() {
    if (unexported) {
        throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
    }
    if (exported) {
        return;
    }
    exported = true;

    if (StringUtils.isEmpty(path)) {
        path = interfaceName;
    }
    //暴露url
    doExportUrls();
}
```

#### 3.2.3 doExportUrls

doExport 方法中又会调用 doExportUrls，加载注册中心配置信息，遍历当前提供者所有协议配置逐一向所有注册中心暴露服务

```java
private void doExportUrls() {
    //加载注册中心URL  registry://
    List<URL> registryURLs = loadRegistries(true);
    for (ProtocolConfig protocolConfig : protocols) {
        //遍历当前服务所有协议，逐一暴露
        //构建pathKey,group/contextPath(优先级协议>ProviderConfig)/path:version
        String pathKey = URL.buildKey(getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), group, version);
        ProviderModel providerModel = new ProviderModel(pathKey, ref, interfaceClass);
        //初始化model到ApplicationModel
        ApplicationModel.initProviderModel(pathKey, providerModel);
        //执行使用配置的协议，真正暴露服务到注册中心
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```

#### 3.2.4 doExportUrlsFor1Protocol

doExportUrls 方法中会遍历服务提供者的协议，执行 doExportUrlsFor1Protocol 方法，将服务提供者注册到所有配置的注册中心中。

> 在 doExportUrlsFor1Protocol 方法中，先是构建服务提供者 URL，其中 URL 包含服务提供者的协议，host(ip)，port(服务监听端口)、接口全限定类名等关键信息，以及一些服务提供者的元数据如 timeout、connection 等信息。
>
> ```java
> if (StringUtils.isEmpty(name)) {
> //如果协议配置name为空，则默认位dubbo
> name = DUBBO;
> }
> 
> //创建map，map用于暴露服务的URL初始化参数
> Map<String, String> map = new HashMap<String, String>();
> map.put(SIDE_KEY, PROVIDER_SIDE);
> 
> //追加运行时参数到map中
> appendRuntimeParameters(map);
> appendParameters(map, metrics);
> appendParameters(map, application);
> appendParameters(map, module);
> // remove 'default.' prefix for configs from ProviderConfig
> // appendParameters(map, provider, Constants.DEFAULT_KEY);
> appendParameters(map, provider);
> appendParameters(map, protocolConfig);
> appendParameters(map, this);
> //如果methods配置不为空，追加methods配置信息到map中，由于method是配置信息代码过多，此处省略
> // export service
> //获取暴露服务使用的host
> String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
> //获取暴露服务使用的port端口
> Integer port = this.findConfigedPorts(protocolConfig, name, map);
> //生成暴露服务协议的URL链接：dubbo://192.168.15.1:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-api-provider&bind.ip=192.168.15.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=59828&release=&side=provider&timestamp=1574667968493
>   URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);
> 
> ```
>
> 其中 findConfigedHosts 获取 host 地址，其大致获取步骤是先从系统配置信息中获取，再从服务提供者协议配置或者提供者全局配置中获取，再从本地系统 InetAddress.getLocalHost().getHostAddress()中获取（linux 中配置在 host 文件的 ip 地址），最后再通过 tcp 连接到注册中心，获取 tcp 连接的 host 地址。
>
> ```java
> private String findConfigedHosts(ProtocolConfig protocolConfig, List<URL> registryURLs, Map<String, String> map) {
> boolean anyhost = false;
> //从系统环境配置中获取===>System.getProperty("DUBBO_IP_TO_BIND")和System.getProperty("protocolConfig.getName()_DUBBO_IP_TO_BIND")
> String hostToBind = getValueFromConfig(protocolConfig, DUBBO_IP_TO_BIND);
> if (hostToBind != null && hostToBind.length() > 0 && isInvalidLocalHost(hostToBind)) {
>   //如果系统配置的绑定ip不为空，且是回环地址，则抛出异常
>   throw new IllegalArgumentException("Specified invalid bind ip from property:" + DUBBO_IP_TO_BIND + ", value:" + hostToBind);
> }
> 
> // if bind ip is not found in environment, keep looking up
> if (StringUtils.isEmpty(hostToBind)) {
>   //如果系统配置的host为空,则获取协议配置的host
>   hostToBind = protocolConfig.getHost();
>   if (provider != null && StringUtils.isEmpty(hostToBind)) {
>       //如果协议没有配置且provider配置不为空，获取provider配置的host
>       hostToBind = provider.getHost();
>   }
>   if (isInvalidLocalHost(hostToBind)) {
>       //如果协议配置和提供则全局配置获取的host都无效
>       anyhost = true;
>       try {
>           logger.info("No valid ip found from environment, try to find valid host from DNS.");
>           //如果系统完全没有配置host属性，则通过InetAddress.getLocalHost().getHostAddress()，也是linux中配置在host文件的ip地址
>           hostToBind = InetAddress.getLocalHost().getHostAddress();
>       } catch (UnknownHostException e) {
>           logger.warn(e.getMessage(), e);
>       }
>       if (isInvalidLocalHost(hostToBind)) {
>           //如果读取InetAddress.getLocalHost()仍不可用
>           if (CollectionUtils.isNotEmpty(registryURLs)) {
>               for (URL registryURL : registryURLs) {
>                   if (MULTICAST.equalsIgnoreCase(registryURL.getParameter("registry"))) {
>                       // skip multicast registry since we cannot connect to it via Socket
>                       continue;
>                   }
>                   try (Socket socket = new Socket()) {
>                       //通过与注册中心进行tcp连接，获取连接socket的本地host地址
>                       SocketAddress addr = new InetSocketAddress(registryURL.getHost(), registryURL.getPort());
>                       socket.connect(addr, 1000);
>                       hostToBind = socket.getLocalAddress().getHostAddress();
>                       break;
>                   } catch (Exception e) {
>                       logger.warn(e.getMessage(), e);
>                   }
>               }
>           }
>           if (isInvalidLocalHost(hostToBind)) {
>               hostToBind = getLocalHost();
>           }
>       }
>   }
> }
> 
> map.put(Constants.BIND_IP_KEY, hostToBind);
> 
> // registry ip is not used for bind ip by default
> //获取注册到注册中心的host地址配置
> String hostToRegistry = getValueFromConfig(protocolConfig, DUBBO_IP_TO_REGISTRY);
> if (hostToRegistry != null && hostToRegistry.length() > 0 && isInvalidLocalHost(hostToRegistry)) {
>   // 如果系统环境配置中获取===>System.getProperty("DUBBO_IP_TO_REGISTRY")和System.getProperty("protocolConfig.getName()_DUBBO_IP_TO_REGISTRY")，配置到注册中心的注册地址不为空且是回环地址，则抛出异常
>   throw new IllegalArgumentException("Specified invalid registry ip from property:" + DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
> } else if (StringUtils.isEmpty(hostToRegistry)) {
>   //如果找不注册到注册中心的host配置，则使用绑定的host配置
>   // bind ip is used as registry ip by default
>   hostToRegistry = hostToBind;
> }
> 
> map.put(ANYHOST_KEY, String.valueOf(anyhost));
> 
> return hostToRegistry;
> }
> ```
>
> findConfigedPorts 获取服务提供者绑定端口配置，与获取 host 类似
>
> ```java
> private Integer findConfigedPorts(ProtocolConfig protocolConfig, String name, Map<String, String> map) {
> Integer portToBind = null;
> 
> // parse bind port from environment
> //获取系统中配置的端口
> String port = getValueFromConfig(protocolConfig, DUBBO_PORT_TO_BIND);
> portToBind = parsePort(port);
> 
> // if there's no bind port found from environment, keep looking up.
> if (portToBind == null) {
>   //获取协议中配置的端口
>   portToBind = protocolConfig.getPort();
>   if (provider != null && (portToBind == null || portToBind == 0)) {
>       //获取提供者全局配置的端口
>       portToBind = provider.getPort();
>   }
>   //通过协议获取默认端口,如果是dubbo协议则是20880端口
>   final int defaultPort = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(name).getDefaultPort();
>   if (portToBind == null || portToBind == 0) {
>       portToBind = defaultPort;
>   }
>   if (portToBind <= 0) {
>       //根据协议名称随机获取端口
>       portToBind = getRandomPort(name);
>       if (portToBind == null || portToBind < 0) {
>           //获取65535以下的有效端口
>           portToBind = getAvailablePort(defaultPort);
>           putRandomPort(name, portToBind);
>       }
>   }
> }
> 
> // save bind port, used as url's key later
> map.put(Constants.BIND_PORT_KEY, String.valueOf(portToBind));
> 
> // registry port, not used as bind port by default
> // 获取注册到注册中心的系统端口配置
> String portToRegistryStr = getValueFromConfig(protocolConfig, DUBBO_PORT_TO_REGISTRY);
> Integer portToRegistry = parsePort(portToRegistryStr);
> if (portToRegistry == null) {
>   //如果该配置为空，使用绑定端口
>   portToRegistry = portToBind;
> }
> 
> return portToRegistry;
> }
> ```

> 构建完 URL 后，执行 exportLocal 进行本地服务暴露，本地服务暴露也就是 jvm 服务内部暴露，这里不细说。
>
> 本地服务暴露后，遍历所有注册中心进行服务的远程暴露。远程暴露服务主要流程是通过 ProxyFactory 创建 Invoker，通过 Protocol.export 进行服务的具体暴露
>
> ```java
> for (URL registryURL : registryURLs) {
> //暴露服务到所有注册中心
> //if protocol is only injvm ,not register
> if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
>   //如果协议是injvm则跳过远程服务暴露
>   continue;
> }
> url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
> URL monitorUrl = loadMonitor(registryURL);
> if (monitorUrl != null) {
>   url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
> }
> if (logger.isInfoEnabled()) {
>   if (url.getParameter(REGISTER_KEY, true)) {
>       logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
>   } else {
>       logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
>   }
> }
> 
> // For providers, this is used to enable custom proxy to generate invoker
> String proxy = url.getParameter(PROXY_KEY);
> if (StringUtils.isNotEmpty(proxy)) {
>   //将服务url的proxy参数加到注册url中
>   registryURL = registryURL.addParameter(PROXY_KEY, proxy);
> }
> //暴露服务的关键
> //PROXY_FACTORY 是  ProxyFactory适配器类
> //如果registryURL中没有proxy参数，默认会使用JavassistProxyFactory
> //通过ProxyFactory生成Invoker对象,生成的invoker是最底层的,依赖接口实现的
> Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
> //封装成依赖ServiceConfig的代理调用者
> DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
> 
> //使用协议暴露服务,此protocol时Protocol适配器类，由于wrapperInvoker中依赖的URL是registryURL,故此处会获取RegistryProtocol执行export
> Exporter<?> exporter = protocol.export(wrapperInvoker);
> exporters.add(exporter);
> }
> ```
>
> PROXY_FACTORY 对象是一个 ProxyFactory 适配器类，会根据 URL 中的 proxy 参数获取具体实现类，如果 proxy 参数为空，则会获取 JavassistProxyFactory 工厂来创建 Invoker，JavassistProxyFactory 会构建一个 Wrapper 来实现服务提供者具体实现类的方法调用。Wrapper 是通过 Javassist 工具动态生成的一个类，从反编译其中代码可见，其最大的作用就是直接调用实现类方法，减去反射的开销。
>
> Invoker 是 Dubbo 中的一个关键组件，在服务者与消费者都充当服务调用的一个职责。

> ```java
> public class JavassistProxyFactory extends AbstractProxyFactory {
> 
> @Override
> @SuppressWarnings("unchecked")
> public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
>   return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
> }
> 
> @Override
> public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
>   // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
>   //使用Wrapper的原因，减少反射带来的开销
>   final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
>   return new AbstractProxyInvoker<T>(proxy, type, url) {
>       @Override
>       protected Object doInvoke(T proxy, String methodName,
>                                 Class<?>[] parameterTypes,
>                                 Object[] arguments) throws Throwable {
>           return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
>       }
>   };
> }
> 
> }
> ```
>
> DelegateProviderMetaDataInvoker 是对 PROXY_FACTORY 生成的 invoker 进一步包装，使其依赖服务提供者的一些元数据信息。
>
> 构建完 Invoker 后，就是通过 Protocol 进行服务暴露，由于 PROXY_FACTORY 中生成的 Invoker 是传入的是 registryURL，这里通过 Protocol 适配器类获取的实现类就是 RegistryProtocol，即通过 RegistryProtocol 进行服务暴露。具体逻辑后续再详解。

## 4、总结

ServiceConfig 中我们能学到的东西很多，其中一个很重要的概念就是分层复用，将一些公有的配置放到父类，通过继承子类达到一个复用配置的效果。