## 前言

距离上篇分享已经过去半年了，懒癌一直发作，再加之表述能力不行一直拖更。

讲讲为啥又重新营业吧？有阅读源码习惯，这过程会思考很多，但是总是会忘记思考过程的思路。

所谓好记性不如烂笔头，我觉得作为一个搞技术的，要经常记录自己学过的东西，方便以后温习，写文章就相当于做笔记一样。

## Dubbo服务暴露之配置

Dubbo提供三种暴露服务的方式：

1. api类
2. xml配置
3. 注解配置

其中后两种方式是在spring基础上完成的。三种方式最终目的都是生成一个ServiceConfig，服务的暴露逻辑都封装在此类中。后续文章会详解ServiceConfig是如何暴露服务的，敬请期待。

## Api类方式
```java
public class Application {
    public static void main(String[] args) throws Exception {
 		//创建ServiceConfig对象
        ServiceConfig<DemoServiceImpl> service = new ServiceConfig<>();
        //设置应用配置
        service.setApplication(new ApplicationConfig("dubbo-demo-api-provider"));
        //设置注册中心配置
        service.setRegistry(new RegistryConfig("nacos://192.168.15.129:8848"));
        //设置服务接口类型，通过接口方法对外暴露配置
        service.setInterface(DemoService.class);
        //设置服务接口实现类
        service.setRef(new DemoServiceImpl());
        //执行服务暴露逻辑
        service.export();
    }
}
```

服务暴露其实就是向注册中心注册服务的一些元数据信息，所以需要给服务配置设置应用配置和设置注册中心地址等信息。用过dubbo的人应该都会好奇为什么从zookeeper切换到nacos注册中心只需要修改zookeeper://变成nacos://，其实这是dubbo的spi机制完成，至于spi机制是什么，后续文章会详解。服务的一些元数据信息来源是服务的接口与设置的应用配置。

## Xml配置方式

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 			   http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd">

    <dubbo:application name="dubbo-demo-api-provider"/>

    <dubbo:registry address="nacos://192.168.15.129:884" />

    <dubbo:protocol name="dubbo" />

    <bean id="demoService" class="org.apache.dubbo.demo.provider.DemoServiceImpl"/>

    <dubbo:service interface="org.apache.dubbo.demo.DemoService" ref="demoService"/>

</beans>
```

Xml配置其实就是将调用api的行为映射到Xml文件。思考下，为什么配置dubbo:**就能映射这些行为？其底层做了什么操作?

### NamespaceHandler机制

Spring应用启动时，创建AbstractXmlApplicationContext上下文，解析Xml配置文件时，当遇到dubbo:这种开头的配置。其会获取beans节点中配置的dubbo对应的namespace，也就是上面的http://dubbo.apache.org/schema/dubbo。然后他会根据这个namespace获取到类路径下所有的spring.handlers配置文件，获取其namespace对应的NamespaceHandler。在dubbo中，http://dubbo.apache.org/schema/dubbo对应org.apache.dubbo.config.spring.schema.DubboNamespaceHandler处理类。通过在NamespaceHandler注册BeanDefinitionParser解析器，Spring会回调解析器的parse方法返回BeanDefinition，注册到spring容器中。

dubbo-config/dubbo-config-spring/src/java/resources/META-INF/spring.hanlders

```properties
http\://dubbo.apache.org/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
http\://code.alibabatech.com/schema/dubbo=org.apache.dubbo.config.spring.schema.DubboNamespaceHandler
```

### DubboNamespaceHandler

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    @Override
    public void init() {
		//此处只贴部分代码
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
    
}
```

init方法中注册多个DubboBeanDefinitionParser，其中注册的名字如service对应的是解析dubbo:service配置的。而传入DubboBeanDefinitionParser中的Class对象作用是解析配置，将配置映射进Class对象中。如service解析器传入的是ServiceBean，既解析完会在Spring容器中创建一个ServiceBean，而这个ServiceBean是ServiceConfig的子类。

### ServiceBean

```java
public class ServiceBean<T> extends ServiceConfig<T> implements InitializingBean, DisposableBean,
        ApplicationContextAware, ApplicationListener<ContextRefreshedEvent>, BeanNameAware,
        ApplicationEventPublisherAware {
            public void afterPropertiesSet() throws Exception {
               //此处省略一系列配置代码
        	  if (!supportedApplicationListener) {
                  	//如果不支持监听器，则直接在对象初始化的时候暴露服务
           			 export();
   				}
    		}
            
            @Override
   		 public void onApplicationEvent(ContextRefreshedEvent event) {
        		if (!isExported() && !isUnexported()) {
                	if (logger.isInfoEnabled()) {
               		 logger.info("The service ready on spring started. service: " + getInterface());
            		}
              		//监听Context刷新事件，暴露服务
           			export();
       		 }
    	}       
  }
```

Service的afterPropertiesSet大部分逻辑就是判断一下配置为空，从Spring容器中获取对应配置信息，这里就不细说了，想研究的话下载Dubbo源码慢慢看。

## 注解配置方式

### 启动类

```java
public class Application {
    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context = new 			     AnnotationConfigApplicationContext(ProviderConfiguration.class);
        context.start();
        System.in.read();
    }

    @Configuration
    //配置dubbo服务扫描的包
    @EnableDubbo(scanBasePackages = "org.apache.dubbo.demo.provider")
    @PropertySource("classpath:/spring/dubbo-provider.properties")
    static class ProviderConfiguration {
        @Bean
        public RegistryConfig registryConfig() {
            //注册中心配置
            RegistryConfig registryConfig = new RegistryConfig();
            registryConfig.setAddress("nacos://192.168.15.129:8848");
            return registryConfig;
        }
    }
}
```

@EnableDubbo配置扫描的org.apache.dubbo.demo.DemoService注解的类，并注册到Spring容器中

### 服务暴露类

```java
import org.apache.dubbo.config.annotation.Service;
import org.apache.dubbo.demo.DemoService;
import org.apache.dubbo.rpc.RpcContext;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class DemoServiceImpl implements DemoService {
    private static final Logger logger = LoggerFactory.getLogger(DemoServiceImpl.class);

    @Override
    public String sayHello(String name) {
        logger.info("Hello " + name + ", request from consumer: " + RpcContext.getContext().getRemoteAddress());
        return "Hello " + name + ", response from provider: " + RpcContext.getContext().getLocalAddress();
    }

}
```

注意@Service不是Spring框架提供的，是org.apache.dubbo.demo.DemoService。

### @EnableDubbo

```java
@Deprecated
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {

    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};
 
    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};

    @AliasFor(annotation = EnableDubboConfig.class, attribute = "multiple")
    boolean multipleConfig() default false;
}
```

可以看到@EnbaleDubbo注解中有一个@DubboComponentScan

### DubboComponentScan

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {

    String[] value() default {};


    String[] basePackages() default {};

  
    Class<?>[] basePackageClasses() default {};

}
```

可以看到@EnableDubbo注解里面有一个@Import(DubboComponentScanRegistrar.class)（后续解析springboot源码会解析Import作用，不清楚的可以了解下Import机制）

### DubboComponentScanRegistrar

```java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		//获取@EnableDubbo中配置的scanBasePackages
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
		//注册ServiceAnnotationBeanPostProcessor到Spring容器中
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);
		//注册到ReferenceAnnotationBeanPostProcessor到Spring容器中
        registerReferenceAnnotationBeanPostProcessor(registry);

    }
}
```

DubboComponentScanRegistrar是实现ImportBeanDefinitionRegistrar的实现类，当Spring在解析Configuration的时候会回调registerBeanDefinitions方法传入BeanDefinitionRegistry。DubboComponentScanRegistrar在此方法中会注册一个关键的ServiceAnnotationBeanPostProcessor。org.apache.dubbo.demo.DemoService注解的类扫描逻辑都封装在ServiceAnnotationBeanPostProcessor这个类。

### ServiceAnnotationBeanPostProcessor

```java
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {
        	@Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
           	//扫描包路径注册服务
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }
    }
           
        private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {
        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);
        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);
        scanner.setBeanNameGenerator(beanNameGenerator);
        //设置符合规则的过滤其@Service注解
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));
        scanner.addIncludeFilter(new AnnotationTypeFilter(com.alibaba.dubbo.config.annotation.Service.class));

        for (String packageToScan : packagesToScan) {

            // Registers @Service Bean first
            //扫描的时候注册@Service的实现类到Spring容器中
            scanner.scan(packageToScan);

            // Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {

                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    //遍历，注册ServiceBean到spring容器中
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }

                if (logger.isInfoEnabled()) {
                    logger.info(beanDefinitionHolders.size() + " annotated Dubbo's @Service Components { " +
                            beanDefinitionHolders +
                            " } were scanned under package[" + packageToScan + "]");
                }

            } else {

                if (logger.isWarnEnabled()) {
                    logger.warn("No Spring Bean annotating Dubbo's @Service was found under package["
                            + packageToScan + "]");
                }

            }

        }

    }
       
     }
```

ServiceAnnotationBeanPostProcessor继承BeanDefinitionRegistryPostProcessor会在Spring容器创建对象前回调postProcessBeanDefinitionRegistry，传入BeanDefinitionRegistry，通过DubboClassPathBeanDefinitionScanner扫描包注册实现类到Spring容器中，进而通过执行registerServiceBean方法创建实现类对应的ServiceBean注册到Spring容器中，通过@Service注解元数据给ServiceBean设置对应配置。

## 总结

本次源码解析只是说了一些片面的东西，没有做一下深入思考，后面会继续更新详细分析。