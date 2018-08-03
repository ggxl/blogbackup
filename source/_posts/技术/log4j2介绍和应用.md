---
date: 2018-8-1 15:45:28
title: log4j2介绍和应用
no_word_count: false
tags: [技术,后端,log4j2]
---

## 介绍

[官网](https://logging.apache.org/log4j/2.x/)

### 概述

> Apache Log4j 2是对Log4j的升级，它比其前身Log4j 1.x提供了重大改进，并提供了Logback中可用的许多改进，同时修复了Logback架构中的一些固有问题。 
>

<!--more-->
 
### 特性

> #### api分离
> Log4j的API与实现分开，使应用程序开发人员可以清楚地了解他们可以使用哪些类和方法，同时确保向前兼容性。这使Log4j团队能够以兼容的方式安全地改进实施。

> #### 性能提升
> Log4j 2包含基于LMAX Disruptor库的下一代异步记录器。在多线程场景中，异步记录器的吞吐量比Log4j 1.x和Logback高18倍，延迟低。Log4j 2明显优于Log4j 1.x，Logback和java.util.logging，尤其是在多线程应用程序中。

> #### 支持多个API
> Log4j 2支持Log4j 1.2，SLF4J，Commons Logging和java.util.logging（JUL）API。

> #### 避免锁定
> 编码到Log4j 2 API的应用程序始终可以选择使用任何符合SLF4J的库作为log4j-to-slf4j适配器的记录器实现

> #### 自动重新加载配置
> 与Logback一样，Log4j 2可以在修改时自动重新加载其配置。与Logback不同，它在重新配置发生时不会丢失日志事件。

> #### 高级过滤
> 与Logback一样，Log4j 2支持基于Log事件中的上下文数据，标记，正则表达式和其他组件进行过滤。可以指定过滤以在传递给Loggers之前或通过Appender时应用于所有事件。此外，过滤器还可以与记录器关联。与Logback不同，您可以在任何这些情况下使用通用的Filter类。

> #### 插件架构
> Log4j使用插件模式配置组件。因此，您无需编写代码来创建和配置Appender，Layout，Pattern Converter等。Log4j自动识别插件并在配置引用它们时使用它们。

> #### 属性支持
> 您可以在配置中引用属性，Log4j将直接替换它们，或者Log4j将它们传递给将动态解析它们的底层组件。属性来自配置文件，系统属性，环境变量，ThreadContext Map以及事件中存在的数据中定义的值。用户可以通过添加自己的Lookup插件来进一步自定义属性提供程序。

> ##### Java 8 Lambda支持
> 以前，如果构造日志消息的代价很高，您通常会在构造消息之前显式检查是否启用了请求的日志级别。在Java 8上运行的客户端代码可以从Log4j的lambda支持中受益。如果未启用请求的日志级别，Log4j将不会评估lambda表达式，因此可以使用更少的代码实现相同的效果

> #### 自定义日志级别
> 在Log4j 2中，可以在代码或配置中轻松定义自定义日志级别。不需要子类化

> #### 垃圾分类
> 在稳态日志记录期间，Log4j 2 在独立应用程序中是无垃圾的，在Web应用程序中是低垃圾。这减少了垃圾收集器的压力，并且可以提供更好的响应时间性能。

##### Log4j 1.x已被广泛采用并用于许多应用中。然而，经过多年的发展，它已经放缓。由于需要符合Java的旧版本并且在2015年8月成为End of Life，因此维护变得更加困难 。它的替代方案SLF4J / Logback对框架进行了许多必要的改进。
>### 那么为什么要使用Log4j 2呢？以下是一些原因。
>1. Log4j 2旨在用作审计日志记录框架。Log4j 1.x和Logback都会在重新配置时丢失事件。Log4j 2不会。在Logback中，Appender中的异常永远不会对应用程序可见。在Log4j中，可以将Appender配置为允许异常渗透到应用程序。
>2. Log4j 2包含基于LMAX Disruptor库的下一代异步记录器。在多线程场景中，异步记录器的吞吐量比Log4j 1.x和Logback高10倍，延迟低几个数量级。
>3. Log4j 2 对于独立应用程序是无垃圾的，对于稳定状态日志记录期间的Web应用程序来说是低垃圾。这减少了垃圾收集器的压力，并且可以提供更好的响应时间性能
>4. Log4j 2使用插件系统，通过添加新的Appender， 过滤器，布局，查找和模式转换器，可以非常轻松地 扩展框架，而无需对Log4j进行任何更改。
>5. 由于插件系统配置更简单。配置中的条目不需要指定类名。
>6. 支持自定义日志级别。可以在代码或配置中定义自定义日志级别。
>7. 支持lambda表达式。仅当启用了请求的日志级别时，在Java 8上运行的客户端代码才能使用lambda表达式延迟构造日志消息。不需要显式级别检查，从而使代码更清晰。
>8. 支持Message对象。消息允许支持有趣和复杂的构造通过日志系统传递并被有效地操作。用户可以自由创建自己的 消息 类型，并编写自定义布局，过滤器和 查找来操纵它们
>9. Log4j 1.x支持Appender上的过滤器。Logback添加了TurboFilters，允许在事件由Logger处理之前过滤事件。Log4j 2支持可以配置为在Logger处理事件之前处理事件的过滤器，因为它们由Logger或Appender处理。
>10. 许多Logback Appender不接受布局，只会以固定格式发送数据。大多数Log4j 2 Appender接受布局，允许以任何所需格式传输数据。
>11. Log4j 1.x和Logback中的布局返回一个String。这导致了Logback Encoders中讨论的问题。Log4j 2采用更简单的方法，Layouts总是返回一个字节数组。这样做的好处是，它意味着它们几乎可以在任何Appender中使用，而不仅仅是写入OutputStream的Appender。
>12. 该系统日志追加程序支持TCP和UDP以及支持用于BSD系统日志和RFC 5424种格式。
>13. Log4j 2利用Java 5并发支持并在尽可能低的级别执行锁定。Log4j 1.x已知死锁问题。其中许多都是在Logback中修复的，但许多Logback类仍然需要在相当高的级别进行同步。


## <font color="red">从1.x升到2.x</font>
> ### 官方提醒
1. 版本1中的主程序包是org.apache.log4j，在版本2中它是 org.apache.logging.log4j
2. 必须将对`org.apache.log4j.Logger.getLogger（）`的 调用修改为 `org.apache.logging.log4j.LogManager.getLogger（）`。
3. 必须使用`org.apache.logging.log4j.LogManager.getRootLogger（）`替换 对`org.apache.log4j.Logger.getRootLogger（）`或 `org.apache.log4j.LogManager.getRootLogger（）`的调用。
4. 调用`org.apache.log4j.Logger.getLogger`接受一个的LoggerFactory必须删除`org.apache.log4j.spi.LoggerFactory`和使用的Log4j 2的其他扩展机制之一。
5. 替换为调用`org.apache.log4j.Logger.getEffectiveLevel（）`与 `org.apache.logging.log4j.Logger.getLevel（）` 。
6. 删除对`org.apache.log4j.LogManager.shutdown（）`的调用，版本2中不需要它们，因为Log4j Core现在会在启动时自动添加JVM关闭挂钩以执行任何Core清理。
	1. 从Log4j 2.1开始，您可以指定自定义 ShutdownCallbackRegistry 来覆盖默认的JVM关闭挂钩策略。
	2. 从Log4j 2.6开始，您现在可以使用`org.apache.logging.log4j.LogManager.shutdown（）` 手动启动关闭。
7. API不支持 调用`org.apache.log4j.Logger.setLevel（）`或类似方法。应用程序应删除这些。Log4j 2实现类中提供了等效功能，请参阅`org.apache.logging.log4j.core.config.Configurator.setLevel（）`，但可能会使应用程序容易受到Log4j 2内部更改的影响
8. 在适当的情况下，应用程序应转换为使用参数化消息而不是字符串连接。
9. `org.apache.log4j.MDC`和 `org.apache.log4j.NDC` 已被线程上下文取代。



>### 需要的jar包
- log4j-core
- lo4j-api
```XML
<dependency>
	<groupId>org.apache.logging.log4j</groupId>
	<artifactId>log4j-core</artifactId>
	<version>2.8.2</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-api -->
<dependency>
	<groupId>org.apache.logging.log4j</groupId>
	<artifactId>log4j-api</artifactId>
	<version>2.8.2</version>
</dependency>

<!-- https://mvnrepository.com/artifact/org.slf4j/slf4j-api -->
<dependency>
	<groupId>org.slf4j</groupId>
	<artifactId>slf4j-api</artifactId>
	<version>1.7.25</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-slf4j-impl -->
<dependency>
	<groupId>org.apache.logging.log4j</groupId>
	<artifactId>log4j-slf4j-impl</artifactId>
	<version>2.8.2</version>
</dependency>
<!-- https://mvnrepository.com/artifact/org.apache.logging.log4j/log4j-jcl -->
<dependency>
	<groupId>org.apache.logging.log4j</groupId>
	<artifactId>log4j-jcl</artifactId>
	<version>2.8.2</version>
</dependency>

```


>### 配置文件
>##### log4j2.xml ，名字要注意不要起错了这个加了2

```XML
<?xml version="1.0" encoding="UTF-8"?>
<Configuration>
    <!--定义属性-->
    <Properties>
        <!--./logs-->
        <Property name="LOG_HOME">./logs</Property>
        <Property name="INFO_FILE_NAME">info-sf-customer</Property>
        <Property name="WARN_FILE_NAME">warn-sf-customer</Property>
        <Property name="ERROR_FILE_NAME">error-sf-customer</Property>
    </Properties>

    <Appenders>
    <!-- 常用的三个子节点 ：Console、RollingFile、File-->
        <!--定义超过指定大小自动删除旧的创建新的的Appender-->
        <RollingFile name="info-sf-customer" fileName="${LOG_HOME}/${INFO_FILE_NAME}.log"
                     filePattern="${LOG_HOME}/%d{yyyy-MM}/${INFO_FILE_NAME}-%d{MM-dd-yy}-%i.log"
                     append="true">
            <!--输出格式-->
            <PatternLayout pattern="[%-5level] %d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %c{1} - %msg%n"/>
            <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <Filters>
                <!--Accept（接受）、Deny（拒绝）或Neutral（中立）
                onMatch:默认NEUTRAL onMismatch:默认DENY
                只接受INFO级别的日志，其余的全部拒绝处理-->
                <ThresholdFilter level="info" onMatch="NEUTRAL" onMismatch="DENY"/>
                <ThresholdFilter level="warn" onMatch="DENY" onMismatch="NEUTRAL"/>
            </Filters>
            <!--<ThresholdFilter level="info" onMatch="ACCEPT" onMismatch="DENY"/>-->
            <!--日志滚动策略-->
            <Policies>
                <!--时间滚动策略-->
                <TimeBasedTriggeringPolicy/>
                <!--大小滚动策略-->
                <SizeBasedTriggeringPolicy size="1 KB"/>
            </Policies>
            <!--默认最多切分7个 超过7个删除之前的-->
            <DefaultRolloverStrategy max="20"/>
        </RollingFile>

        <RollingFile name="warn-sf-customer" fileName="${LOG_HOME}/${WARN_FILE_NAME}.log"
                     filePattern="${LOG_HOME}/%d{yyyy-MM}/${WARN_FILE_NAME}-%d{MM-dd-yy}-%i.log"
                     append="true">

            <PatternLayout pattern="[%-5level] %d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %c{1} - %msg%n"/>
            <Filters>
                <ThresholdFilter level="warn" onMatch="NEUTRAL" onMismatch="DENY"/>
                <ThresholdFilter level="error" onMatch="DENY" onMismatch="NEUTRAL"/>
            </Filters>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="1 KB"/>
            </Policies>
            <DefaultRolloverStrategy max="20"/>
        </RollingFile>
        <RollingFile name="error-sf-customer" fileName="${LOG_HOME}/${ERROR_FILE_NAME}.log"
                     filePattern="${LOG_HOME}/%d{yyyy-MM}/${ERROR_FILE_NAME}-%d{MM-dd-yy}-%i.log"
                     append="true">
            <PatternLayout pattern="[%-5level] %d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %c{1} - %msg%n"/>
            <ThresholdFilter level="error"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="1 KB"/>
            </Policies>
            <DefaultRolloverStrategy max="20"/>
        </RollingFile>

        <!-- 节点用来定义输出到控制台的Appender-->
        <Console name="STDOUT" target="SYSTEM_OUT">
            <PatternLayout pattern="%-d{yyyy-MM-dd HH:mm:ss} [%p] [%c{1}] [%t] %30.30C.%M():%L - %m%n"/>
        </Console>
    </Appenders>

    <Loggers>


        <!--<logger name="org.spring" level="warn" additivity="false">
            <AppenderRef ref="STDOUT" />
            <AppenderRef ref="sf-tempalte-service" />
        </logger>
        <logger name="org.springframework" level="warn" additivity="false">
            <AppenderRef ref="STDOUT" />
            <AppenderRef ref="sf-tempalte-service" />
        </logger>-->
        <!--dubbo-->
        <Logger name="com.alibaba.dubbo" level="info" additivity="false">
            <AppenderRef ref="STDOUT" />
            <AppenderRef ref="info-sf-customer" />
        </Logger>
        <!-- sql -->
        <Logger name="com.sf.customer.service.core.dao" level="debug" additivity="false">
            <AppenderRef ref="STDOUT"/>
            <AppenderRef ref="info-sf-customer"/>
        </Logger>

        <Root level="info">
            <AppenderRef ref="STDOUT"/>
            <AppenderRef ref="info-sf-customer"/>
            <AppenderRef ref="warn-sf-customer"/>
            <AppenderRef ref="error-sf-customer"/>
        </Root>

    </Loggers>
</Configuration>

```

>### 问题
包冲突
```
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:.m2/repository/org/slf4j/slf4j-log4j12/1.7.11/slf4j-log4j12-1.7.11.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:.m2/repository/org/apache/logging/log4j/log4j-slf4j-impl/2.8.2/log4j-slf4j-impl-2.8.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]

```

##### 在相关依赖下排除log4j的依赖包

```XML

<exclusions>
	<!--  排除log4j1的方式  -->
	<exclusion>
		<groupId>log4j</groupId>
		<artifactId>log4j</artifactId>
	</exclusion>
	<!--  排除logback的方式  -->
	<exclusion>
		<groupId>org.slf4j</groupId>
		<artifactId>slf4j-log4j12</artifactId>
	</exclusion>
</exclusions>
````

##### 巴特，有太多包依赖了log4j，一个一个排除实在不麻烦也不好找。在网上查了下全局排除的方法。
>maven在解析依赖的时候，有两个原则
>
>- 第一原则是路径最短有限原则，例如A-->B-->C-1.0(A依赖B，B依赖C的1.0版本)，同时A-->D-->E-->C-2.0，那么从A来看，最终会依赖C的1.0版本进来，因为路径最短，最可信，这个例子也推翻了“高版本覆盖低版本”的错误言论。
>- 第二原则是优先声明原则(pom中的声明顺序)，这是对第一原则的补充，就是路径长度相同(第一原则好无力)的情况下，第二原则开始决策微调，不过这个原则是在maven2.1.0才加入的，之前的版本如果第一原则无效的情况下，就是不可调控的

```XML
<!--
试了插件无效，也不能改仓库，最后没办法了，随便给个不存在的包 让那些自动依赖的第三方包找不到log4j12
同归于尽，先跑起来试用下再说。
-->
<dependency>
	<groupId>org.slf4j</groupId>
	<artifactId>slf4j-log4j12</artifactId>
	<version>1.7.25</version>
	<scope>system</scope>
	<systemPath>${project.basedir}/lib/not-exist-empty.jar</systemPath>
</dependency>
```
##### 虽然idea maven报红，试了下冲突没了，但是很多日志也没了。。。

### 首先dubbo的日志

>dubbo日志不见了，但是，却给出了一个关键的警告日志。
>log4j:WARN No appenders could be found for logger (com.alibaba.dubbo.common.logger.LoggerFactory).
>log4j:WARN Please initialize the log4j system properly.

##### 看下LoggerFactory的源码
```java
// 查找常用的日志框架
	static {
	    String logger = System.getProperty("dubbo.application.logger");
	    if ("slf4j".equals(logger)) {
    		setLoggerAdapter(new Slf4jLoggerAdapter());
    	} else if ("jcl".equals(logger)) {
    		setLoggerAdapter(new JclLoggerAdapter());
    	} else if ("log4j".equals(logger)) {
    		setLoggerAdapter(new Log4jLoggerAdapter());
    	} else if ("jdk".equals(logger)) {
    		setLoggerAdapter(new JdkLoggerAdapter());
    	} else {
    		try {
    			setLoggerAdapter(new Log4jLoggerAdapter());
            } catch (Throwable e1) {
                try {
                	setLoggerAdapter(new Slf4jLoggerAdapter());
                } catch (Throwable e2) {
                    try {
                    	setLoggerAdapter(new JclLoggerAdapter());
                    } catch (Throwable e3) {
                        setLoggerAdapter(new JdkLoggerAdapter());
                    }
                }
            }
    	}
	}

```
> 看到这我们就明白了，原来dubbo的日志的LoggerAdapter是可以通过dubbo.application.logger变量指定的。
> 那在启动dubbo的时候给他加上。

```java
System.setProperty("dubbo.application.logger","slf4j");
Main.main(args);
```

### 然后sql的日志 配置指定mapper的包路径即可
```XML
<Logger name="com.sf.customer.service.core.dao" level="debug" additivity="false">
	<AppenderRef ref="STDOUT"/>
	<AppenderRef ref="info-sf-customer"/>
</Logger>
```

### 总结

>1. log4j2 性能提升
>2. 解决了Log4j 1.x已知死锁问题
>3. API分离 api包和core包
>4. 支持多个API，slf4j，也正因为如此，此次从1.x升级到2.x我们项目代码才不用改动。
>5. zkclient包中直接使用的log4j 无法获取logger,如：ZkEventThread。不知道咋整，好在这个日志对我们来说没什么用处。
>6. 如果使用log4j2获取logger则使用`LogManager.getLogger()`方式
>7. log4j2的包名`org.apache.logging.log4j`，log4j1.x的报名不带logging
>8. 大部分内容都是复制了官网的介绍，没太注意排版，看起来有点乱，全当个手册吧。
>9. <font color="red">遗留问题:如何全局去除log4j的依赖</font>





