---
title: springboot
description: springboot原理，启动流程
categories:
 - springboot
 - interview
tags:
---
<!-- more -->

[官方手册](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#getting-started-first-application)
> springboot



## springboot概述

* description
Spring Boot makes it easy to create stand-alone, production-grade Spring based Applications that you can "just run". We take an opinionated view of the Spring platform and third-party libraries so you can get started with minimum fuss. Most Spring Boot applications need very little Spring configuration.

  总结：简化spring应用搭建而生，最小代价启动应用，减少开发配置量。
* Features
-- Create stand-alone Spring applications
-- Embed Tomcat, Jetty or Undertow directly (no need to deploy WAR files)
-- Provide opinionated 'starter' POMs to simplify your Maven configuration
-- Automatically configure Spring whenever possible
-- Provide production-ready features such as metrics, health checks and externalized configuration
-- Absolutely no code generation and no requirement for XML configuration

总结：创建独立spring应用；集成Tomcat，无需war文件；maven配置starter；spring自动配置；提供扩展配置；无需代码以及xml配置。
     把应用打包成为一个jar/war，这个jar/war可以直接启动的，不需要另外配置一个Web Server。
## springboot启动
* fatjar
jar结构：
```sh
├── META-INF
│   ├── MANIFEST.MF
├── application.properties
├── com
│   └── example
│       └── SpringBootDemoApplication.class
├── lib
│   ├── aopalliance-1.0.jar
│   ├── spring-beans-4.2.3.RELEASE.jar
│   ├── ...
└── org
    └── springframework
        └── boot
            └── loader
                ├── ExecutableArchiveLauncher.class
                ├── JarLauncher.class
                ├── JavaAgentDetector.class
                ├── LaunchedURLClassLoader.class
                ├── Launcher.class
                ├── MainMethodRunner.class
                ├── ...
```

com/example：应用class

lib：依赖的jar文件

org/springframework/boot/loader：springboot loader class

描述文件MANIFEST.MF：

```sh
Manifest-Version: 1.0
Start-Class: com.example.SpringBootDemoApplication
Implementation-Vendor-Id: com.example
Spring-Boot-Version: 1.3.0.RELEASE
Created-By: Apache Maven 3.3.3
Build-Jdk: 1.8.0_60
Implementation-Vendor: Pivotal Software, Inc.
Main-Class: org.springframework.boot.loader.JarLauncher
```

main-Class：jar启动main函数
start-Class：应用main函数
## Archive
归档。
在spring boot里，抽象出了Archive的概念。一个archive可以是一个jar（JarFileArchive），也可以是一个文件目录（ExplodedArchive）。可以理解为Spring boot抽象出来的统一访问资源的层。

```sh
public abstract class Archive {
    public abstract URL getUrl();
    public String getMainClass();
    public abstract Collection<Entry> getEntries();
    public abstract List<Archive> getNestedArchives(EntryFilter filter);
```
getNestedArchives函数，这个实际返回的是demo-0.0.1-SNAPSHOT.jar/lib下面的jar的Archive列表。它们的URL是：

```sh 
jar:file:/tmp/target/demo-0.0.1-SNAPSHOT.jar!/lib/aopalliance-1.0.jar
jar:file:/tmp/target/demo-0.0.1-SNAPSHOT.jar!/lib/spring-beans-4.2.3.RELEASE.jar
```

## JarLauncher
创建一个Archive：

```sh 
    protected final Archive createArchive() throws Exception {
        ProtectionDomain protectionDomain = getClass().getProtectionDomain();
        CodeSource codeSource = protectionDomain.getCodeSource();
        URI location = (codeSource == null ? null : codeSource.getLocation().toURI());
        String path = (location == null ? null : location.getSchemeSpecificPart());
        if (path == null) {
            throw new IllegalStateException("Unable to determine code source archive");
        }
        File root = new File(path);
        if (!root.exists()) {
            throw new IllegalStateException(
                    "Unable to determine code source archive from " + root);
        }
        return (root.isDirectory() ? new ExplodedArchive(root)
                : new JarFileArchive(root));
    }
```

通过getNestedArchives加载lib下包，并新建线程启动main：

```sh 
    /**
     * Launch the application given the archive file and a fully configured classloader.
     */
    protected void launch(String[] args, String mainClass, ClassLoader classLoader)
            throws Exception {
        Runnable runner = createMainMethodRunner(mainClass, args, classLoader);
        Thread runnerThread = new Thread(runner);
        runnerThread.setContextClassLoader(classLoader);
        runnerThread.setName(Thread.currentThread().getName());
        runnerThread.start();
    }

    /**
     * Create the {@code MainMethodRunner} used to launch the application.
     */
    protected Runnable createMainMethodRunner(String mainClass, String[] args,
            ClassLoader classLoader) throws Exception {
        Class<?> runnerClass = classLoader.loadClass(RUNNER_CLASS);
        Constructor<?> constructor = runnerClass.getConstructor(String.class,
                String[].class);
        return (Runnable) constructor.newInstance(mainClass, args);
    }
```

启动流程总结：
spring boot应用打包之后，生成一个fat jar，里面包含了应用依赖的jar包，还有Spring boot loader相关的类
Fat jar的启动Main函数是JarLauncher，它负责创建一个LaunchedURLClassLoader来加载/lib下面的jar，并以一个新线程启动应用的Main函数。
## Embed Tomcat
判断是否在web环境：

```sh 
private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
    "org.springframework.web.context.ConfigurableWebApplicationContext" };

private boolean deduceWebEnvironment() {
    for (String className : WEB_ENVIRONMENT_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return false;
        }
    }
    return true;
}

```

创建AnnotationConfigEmbeddedWebApplicationContext:

```sh 
//org.springframework.boot.SpringApplication
    protected ConfigurableApplicationContext createApplicationContext() {
        Class<?> contextClass = this.applicationContextClass;
        if (contextClass == null) {
            try {
                contextClass = Class.forName(this.webEnvironment
                        ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
            }
            catch (ClassNotFoundException ex) {
                throw new IllegalStateException(
                        "Unable create a default ApplicationContext, "
                                + "please specify an ApplicationContextClass",
                        ex);
            }
        }
        return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
    }
```

获取EmbeddedServletContainerFactory的实现类:
spring boot通过获取EmbeddedServletContainerFactory来启动对应的web服务器。常用的两个实现类是TomcatEmbeddedServletContainerFactory和JettyEmbeddedServletContainerFactory。
启动Tomcat：

```sh 
//TomcatEmbeddedServletContainerFactory
@Override
public EmbeddedServletContainer getEmbeddedServletContainer(
        ServletContextInitializer... initializers) {
    Tomcat tomcat = new Tomcat();
    File baseDir = (this.baseDirectory != null ? this.baseDirectory
            : createTempDir("tomcat"));
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    Connector connector = new Connector(this.protocol);
    tomcat.getService().addConnector(connector);
    customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    tomcat.getEngine().setBackgroundProcessorDelay(-1);
    for (Connector additionalConnector : this.additionalTomcatConnectors) {
        tomcat.getService().addConnector(additionalConnector);
    }
    prepareContext(tomcat.getHost(), initializers);
    return getTomcatEmbeddedServletContainer(tomcat);
}
```

## Actuator
actuator是spring boot的一个附加功能,可帮助你在应用程序生产环境时监视和管理应用程序。可以使用HTTP的各种请求来监管,审计,收集应用的运行情况.0特别对于微服务管理十分有意义

略