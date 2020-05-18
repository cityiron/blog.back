---
title: 多个springboot运行在同一个JVM
date: 2019-09-28 17:00:11
tags: [springboot, classloader, 记录, 原创]
categories: springboot
---

[springboot] springboot瞎折腾之一个JVM多个springboot项目

<!-- more --> 

# 概述

> 当下作为Java开发人员，运行一个服务基本上都会直接基于springboot，因此启动N个服务是需要启动N个springboot程序的。特别在本地环境通过intellij运行多个服务的时候，会需要占用较大的资源，电脑往往会出现卡顿现象。

本文主要介绍如何通过启动一个（入口） ` main ` 方法来运行多个服务，从而提高本地的开发爽度。

# 如何运行

> 本文例子参考于github的地址，文末有提供

## 创建项目

### 基本结构 
> 通过 intellij 创建一个项目，并创建如下module

- launcher
- common-service
- first-service
- second-service

### 导入必要的依赖
```xml
    <dependencyManagement>
        <dependencies>
            <!-- springboot -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter</artifactId>
                <version>${spring.boot.version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
                <version>${spring.boot.version}</version>
            </dependency>
            <!-- 会引起一个JMX的问题，下文会说明如何解决 -->
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-actuator</artifactId>
                <version>${spring.boot.version}</version>
            </dependency>
            <dependency>
                <groupId>junit</groupId>
                <artifactId>junit</artifactId>
                <version>4.12</version>
                <scope>test</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

### 单个服务说明
![服务结构](https://res.cloudinary.com/dogbaobao/image/upload/v1571537636/blog/springbootOneJvm/sbojvm-3-wm-1571537594_sucxbc.jpg)

在 first-service 和 second-service 编写最基本的 web 所需要的类 Application 和 Controller

代码如下:

```java
@SpringBootApplication
public class FirstApplication {

    public static void main(String[] args) {
        SpringApplication.run(FirstApplication.class, args);
    }

}
```

```java
@RestController
public class FirstController {

    @GetMapping(value = "/index")
    public String index() throws Exception {
        return "Hello from first microservice!";
    }

}
```

```java
public class FirstServiceBackendRunner extends BackendRunner {

    public FirstServiceBackendRunner() {
        super("first", FirstApplication.class, CustomizationBean.class);
    }

}
```

```java
// 如果是springboot1.x
public class CustomizationBean implements EmbeddedServletContainerCustomizer {

    @Value( "${backend.apps.first.contextPath}" )
    private String contextPath;

    @Value( "${backend.apps.first.port}" )
    private Integer port;

    @Override
    public void customize(ConfigurableEmbeddedServletContainer container) {
        container.setContextPath(contextPath);
        container.setPort(port);
    }

}

// 如果是springboot2.x
public class CustomizationBean implements WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> {

    @Value( "${backend.apps.first.contextPath}" )
    private String contextPath;

    @Value( "${backend.apps.first.port}" )
    private Integer port;

    @Override
    public void customize(ConfigurableServletWebServerFactory factory) {
        factory.setContextPath(contextPath);
        factory.setPort(port);
    }

}
```

图片中的两个 yml 写下面一个内容即可
```yml
spring:
  application:
    name: FirstService
server:
  port: 8110
  servlet:
    contextPath: /first    
```

### 运行核心类

> 在common模块的核心类

```java
public abstract class BackendRunner {

    // 区分不通服务的配置
    private String profile;

    private ConfigurableApplicationContext appContext;
    private final Class<?>[] backendClasses;

    private Object monitor = new Object();
    private boolean shouldWait;

    protected BackendRunner(String profile, final Class<?>... backendClasses) {
        this.backendClasses = backendClasses;
        this.profile = profile;
    }

    public void run() {
        if (appContext != null) {
            throw new IllegalStateException("AppContext must be null to run this backend");
        }
        runBackendInThread();
        waitUntilBackendIsStarted();
    }

    private void waitUntilBackendIsStarted() {
        try {
            synchronized (monitor) {
                if (shouldWait) {
                    monitor.wait();
                }
            }
        } catch (InterruptedException e) {
            throw new IllegalStateException(e);
        }
    }

    private void runBackendInThread() {
        final Thread runnerThread = new BackendRunnerThread();
        shouldWait = true;
        runnerThread.setContextClassLoader(backendClasses[0].getClassLoader());
        runnerThread.start();
    }

    public void stop() {
        SpringApplication.exit(appContext);
        appContext = null;
    }

    private class BackendRunnerThread extends Thread {
        @Override
        public void run() {
            // 真正启动
            // 设置环境，加载不同配置
            System.setProperty("spring.profiles.active", profile);
            appContext = SpringApplication.run(backendClasses, new String[] {});
            synchronized (monitor) {
                shouldWait = false;
                monitor.notify();
            }
        }
    }

}
```

### 启动核心类

```java
public class MicroservicesStarter {

    private static final List<Backend> activeBackends = new ArrayList<>();

    public MicroservicesStarter() {
    }

    public static void startBackends() throws Exception {
        startBackend("first-software", "com.gongdao.middleware.first.backendRunner.FirstServiceBackendRunner");
        startBackend("second-software", "com.gongdao.middleware.second.backendRunner.SecondServiceBackendRunner");
    }

    /**
     * 启动入口
     *
     * @param backendProjectName 项目名称，对应文件路径
     * @param backendClassName   每个服务的启动类
     * @throws Exception
     */
    private static void startBackend(final String backendProjectName, final String backendClassName) throws Exception {
        URL runnerUrl = new File(
            System.getProperty("user.dir") + "/" + backendProjectName + "/target/classes/").toURI()
            .toURL();

        URL[] urls = new URL[] {runnerUrl};

        URLClassLoader cl = new URLClassLoader(urls, MicroservicesStarter.class.getClassLoader());
        Class<?> runnerClass = cl.loadClass(backendClassName);

        Object runnerInstance = runnerClass.newInstance();

        final Backend backend = new Backend(runnerClass, runnerInstance);
        activeBackends.add(backend);

        runnerClass.getMethod("run").invoke(runnerInstance);
    }

    public static void stopAllBackends()
        throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {
        for (Backend b : activeBackends) {
            b.runnerClass.getMethod("stop").invoke(b.runnerInstance);
        }
    }

    private static class Backend {
        private Class<?> runnerClass;
        private Object runnerInstance;

        public Backend(final Class<?> runnerClass, final Object runnerInstance) {
            this.runnerClass = runnerClass;
            this.runnerInstance = runnerInstance;
        }
    }

}
```

### 启动

> 点击 com.gongdao.middleware.LauncherApplication#main 启动

```java
public class LauncherApplication {

    public static void main(String[] args) {
        try {
            MicroservicesStarter.startBackends();
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }

}
```

![first](https://res.cloudinary.com/dogbaobao/image/upload/v1571537636/blog/springbootOneJvm/sbojvm-1-wm-1571537591_ndonnh.jpg)

![second](https://res.cloudinary.com/dogbaobao/image/upload/v1571537636/blog/springbootOneJvm/sbojvm-2-wm-1571537593_t4avao.jpg)

# 补充说明

## 原理
利用了不同的classloader（默认类+指定的路径的class文件）去加载不同的服务

## actuate包的JMX注册异常
> 在 springboot1.x 的时候出现

```text
2019-09-27 15:13:11.343 ERROR 51322 --- [       Thread-5] o.s.b.a.e.jmx.EndpointMBeanExporter      : Could not register JmxEndpoint [auditEventsEndpoint]

org.springframework.jmx.export.UnableToRegisterMBeanException: Unable to register MBean [org.springframework.boot.actuate.endpoint.jmx.AuditEventsJmxEndpoint@65a39e41] with key 'auditEventsEndpoint'; nested exception is javax.management.InstanceAlreadyExistsException: org.springframework.boot:type=Endpoint,name=auditEventsEndpoint
	at org.springframework.jmx.export.MBeanExporter.registerBeanNameOrInstance(MBeanExporter.java:628) ~[spring-context-4.3.23.RELEASE.jar:4.3.23.RELEASE]
	at org.springframework.boot.actuate.endpoint.jmx.EndpointMBeanExporter.registerJmxEndpoints(EndpointMBeanExporter.java:174) [spring-boot-actuator-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at org.springframework.boot.actuate.endpoint.jmx.EndpointMBeanExporter.locateAndRegisterEndpoints(EndpointMBeanExporter.java:162) [spring-boot-actuator-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at org.springframework.boot.actuate.endpoint.jmx.EndpointMBeanExporter.doStart(EndpointMBeanExporter.java:158) [spring-boot-actuator-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at org.springframework.boot.actuate.endpoint.jmx.EndpointMBeanExporter.start(EndpointMBeanExporter.java:337) [spring-boot-actuator-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at org.springframework.context.support.DefaultLifecycleProcessor.doStart(DefaultLifecycleProcessor.java:173) [spring-context-4.3.23.RELEASE.jar:4.3.23.RELEASE]
	at org.springframework.context.support.DefaultLifecycleProcessor.access$200(DefaultLifecycleProcessor.java:50) [spring-context-4.3.23.RELEASE.jar:4.3.23.RELEASE]
	at org.springframework.context.support.DefaultLifecycleProcessor$LifecycleGroup.start(DefaultLifecycleProcessor.java:350) [spring-context-4.3.23.RELEASE.jar:4.3.23.RELEASE]
	at org.springframework.context.support.DefaultLifecycleProcessor.startBeans(DefaultLifecycleProcessor.java:149) [spring-context-4.3.23.RELEASE.jar:4.3.23.RELEASE]
	at org.springframework.context.support.DefaultLifecycleProcessor.onRefresh(DefaultLifecycleProcessor.java:112) [spring-context-4.3.23.RELEASE.jar:4.3.23.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.finishRefresh(AbstractApplicationContext.java:880) [spring-context-4.3.23.RELEASE.jar:4.3.23.RELEASE]
	at org.springframework.boot.context.embedded.EmbeddedWebApplicationContext.finishRefresh(EmbeddedWebApplicationContext.java:146) [spring-boot-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:545) [spring-context-4.3.23.RELEASE.jar:4.3.23.RELEASE]
	at org.springframework.boot.context.embedded.EmbeddedWebApplicationContext.refresh(EmbeddedWebApplicationContext.java:124) [spring-boot-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:693) [spring-boot-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:360) [spring-boot-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:303) [spring-boot-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1118) [spring-boot-1.5.20.RELEASE.jar:1.5.20.RELEASE]
	at com.gongdao.middleware.common.BackendRunner$BackendRunnerThread.run(BackendRunner.java:58) [classes/:na]
    ......
```

通过引入 profile 让每个服务调用自己的配置文件，通过配置如下内容解决：

需要在 launcher 的resource下添加对应的 yml 文件
```yml
spring:
  jmx:
    default-domain: FirstService

endpoints:
  jmx:
    unique-names: true
    domain: FirstService
```

# 参考资料
https://github.com/rameez4ever/springboot-demo/tree/master/springboot-multi-service-launcher
https://www.davidtanzer.net/david's%20blog/2015/04/01/running-multiple-spring-boot-apps-in-the-same-jvm.html
