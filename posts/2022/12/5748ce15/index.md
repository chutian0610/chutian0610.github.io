# Trino源码学习-基础框架Airlift

[Airlift](https://github.com/airlift/airlift)是Trino的系统基础，是一个用Java构建REST服务的轻量级框架。Airlift并不是像Spring一样的重量级框架，相反，它像是由一组开源工具组成的工具包，将来自 Java 生态系统的稳定、成熟的库汇集到一个简单、轻量级的包中，让您专注于完成工作，并包括对配置、指标、日志记录、依赖注入等的内置支持，使开发者能够在最短的时间内交付生产质量的 Web 服务。

Airlift包含以下标准开源库:

|Library|Domain|
|:---|:---|
|[Jetty](https://www.eclipse.org/jetty/)|Industry standard HTTP server and client|
|[JAX-RS/Jersey](https://eclipse-ee4j.github.io/jersey/)|The Java standard for REST servers|
|[Jackson](https://github.com/FasterXML/jackson)|Industry standard JSON serialization|
|[Guava](https://github.com/google/guava)|Swiss army knife for Java|
|[Guice](https://github.com/google/guice)|The best dependency injection framework for Java|
|[jmxutils](https://github.com/martint/jmxutils)|Simple library for exposing JMX endpoints|

<!--more-->

## Quick Start

下面是一个Airlift的Quick Start例子(依赖 JDK17,Maven和Git)。项目结构如下:

```
├── pom.xml
└── src
    └── main
        └── java
            └── org
                └── example
                    ├── Service.java
                    ├── ServiceModule.java
                    └── ServiceResource.java
```

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>sample-server</groupId>
  <artifactId>sample-server</artifactId>
  <name>sample-server</name>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>

  <parent>
    <groupId>io.airlift</groupId>
    <artifactId>airbase</artifactId>
    <version>132</version>
  </parent>

  <properties>
    <!--声明了airlift的版本-->
    <dep.airlift.version>223</dep.airlift.version>
    <!--license header检查开关-->
    <air.check.skip-license>true</air.check.skip-license>
    <!--确保项目构建和打包时使用相同版本的airbase-->
    <dep.packaging.version>${dep.airlift.version}</dep.packaging.version>
    <!--项目使用的JDK版本-->
    <project.build.targetJdk>17</project.build.targetJdk>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>io.airlift</groupId>
        <artifactId>bom</artifactId>
        <version>${dep.airlift.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>com.google.inject</groupId>
      <artifactId>guice</artifactId>
    </dependency>

    <dependency>
      <groupId>javax.ws.rs</groupId>
      <artifactId>javax.ws.rs-api</artifactId>
    </dependency>

    <dependency>
      <groupId>io.airlift</groupId>
      <artifactId>bootstrap</artifactId>
    </dependency>

    <dependency>
      <groupId>io.airlift</groupId>
      <artifactId>http-server</artifactId>
    </dependency>

    <dependency>
      <groupId>io.airlift</groupId>
      <artifactId>json</artifactId>
    </dependency>

    <dependency>
      <groupId>io.airlift</groupId>
      <artifactId>node</artifactId>
    </dependency>

    <dependency>
      <groupId>io.airlift</groupId>
      <artifactId>event</artifactId>
    </dependency>

    <dependency>
      <groupId>io.airlift</groupId>
      <artifactId>jaxrs</artifactId>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-surefire-plugin</artifactId>
          <configuration>
              <failIfNoTests>false</failIfNoTests>
          </configuration>
      </plugin>
  </plugins>
  </build>
</project>
```
### Rest service

1. 创建Rest Endpoint

```java
package org.example;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/v1/service")
public class ServiceResource
{
    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public String hello()
    {
        return "Hello Airlift!";
    }
}

```

2. 设置Guice Bindings

```java
package org.example;

import com.google.inject.Binder;
import com.google.inject.Module;

import static io.airlift.jaxrs.JaxrsBinder.jaxrsBinder;

public class ServiceModule
        implements Module
{
    @Override
    public void configure(Binder binder)
    {
        jaxrsBinder(binder).bind(ServiceResource.class);
    }
}

```

3. 创建启动类

```java
package org.example;

import io.airlift.bootstrap.Bootstrap;
import io.airlift.event.client.EventModule;
import io.airlift.http.server.HttpServerModule;
import io.airlift.jaxrs.JaxrsModule;
import io.airlift.json.JsonModule;
import io.airlift.node.NodeModule;

public class Service
{
    public static void main(String[] args)
    {
        Bootstrap app = new Bootstrap(new ServiceModule(),
                new NodeModule(),
                new HttpServerModule(),
                new EventModule(),
                new JsonModule(),
                new JaxrsModule());
        // this method start app
        app.initialize();
    }

    private Service()
    {
    }
}

```

### 构建 & 运行

1. 将修改提交到git

```sh
git init
git add .
git commit -a -m "initial commit"
```
2. 构建项目

```sh
mvn clean verify
```

3. 运行项目

```sh
mvn exec:java -Dexec.mainClass=example.Service -Dnode.environment=test
```

4. 测试项目运行

```sh
curl http://localhost:8080/v1/service
```

### 配置

Airlift的配置支持是非常简单和直接的。

首先，我们需要一些额外的依赖用于配置功能。在pom中添加

```xml
<dependency>
    <groupId>javax.validation</groupId>
    <artifactId>validation-api</artifactId>
</dependency>

<dependency>
    <groupId>io.airlift</groupId>
    <artifactId>configuration</artifactId>
</dependency>
<dependency>
    <groupId>javax.inject</groupId>
    <artifactId>javax.inject</artifactId>
</dependency>        
```

接下来创建一个配置类，创建如下文件`src/main/java/org/example/ServiceConfig.java`。

```java
package org.example;

import io.airlift.configuration.Config;
import javax.validation.constraints.NotBlank;

public class ServiceConfig
{
    private String helloMessage = "Hello Airlift!";

    @NotBlank
    public String getHelloMessage()
    {
        return helloMessage;
    }

    @Config("hello.message")
    public ServiceConfig setHelloMessage(String helloMessage)
    {
        this.helloMessage = helloMessage;
        return this;
    }
}

```

接下来我们需要将配置类绑定到Guice模块上。Airlift使用`io.airlift.configuration.ConfigBinder.configBinder`来绑定配置。修改`ServiceModule.java`:

```java
package org.example;

import com.google.inject.Binder;
import com.google.inject.Module;

import static io.airlift.configuration.ConfigBinder.configBinder; // Config
import static io.airlift.jaxrs.JaxrsBinder.jaxrsBinder;

public class ServiceModule
        implements Module
{
    @Override
    public void configure(Binder binder)
    {
        jaxrsBinder(binder).bind(ServiceResource.class);
        configBinder(binder).bindConfig(ServiceConfig.class);   // Config
    }
}

```


修改ServiceResource来使用配置对象。



```java
package org.example;

import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/v1/service")
public class ServiceResource
{
    private final ServiceConfig config;

    @Inject
    public ServiceResource(ServiceConfig config)
    {
        this.config = config;
    }

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public String hello()
    {
        return config.getHelloMessage();
    }
}

```


编译运行，可以看到接口响应变成了配置类中的配置。

```sh
mvn clean verify
mvn exec:java -Dexec.mainClass=org.example.Service -Dnode.environment=test
curl http://localhost:8080/v1/service # 查看接口响应
```

接下来，就是将配置绑定到配置文件上。

首先在项目根节点添加配置文件`etc/config.properties`

```properties
node.environment=test
hello.message=Hello from a config file!
```

然后将启动命令改为:

```sh
mvn exec:java -Dexec.mainClass=org.example.Service -Dconfig=etc/config.properties
```

### 日志

Airlift包含了一个简单的日志API基于JDK的logging包。

首先，我们添加日志的maven依赖。

```xml
<dependency>
  <groupId>io.airlift</groupId>
  <artifactId>log</artifactId>
</dependency>

<dependency>
  <groupId>io.airlift</groupId>
  <artifactId>log-manager</artifactId>
</dependency>
```

然后，我们在ServiceResource中打印日志。


```java
package org.example;

import io.airlift.log.Logger;

import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/v1/service")
public class ServiceResource
{
    // 获取logger
    private static final Logger LOG = Logger.get(ServiceResource.class);
    private final ServiceConfig config;

    @Inject
    public ServiceResource(ServiceConfig config)
    {
        this.config = config;
    }

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public String hello()
    {
        String message = config.getHelloMessage();
        // 打印日志
        LOG.info("call Hello Function and return %s", message);
        return message;
    }
}

```

最后，在配置文件config.properties中添加日志打印的相关配置。

```properties
## etc/config.properties

## log
## 格式可以是text或json，json格式时，会被打印成一行
log.format=json 
## 可以是本地文件，也可以通过tcp传输，tcp时配置格式为tcp://<host>:<port>
log.output-file=var/log/server.log 
## 可以向日志中添加环境参数
log.annotation-file=etc/annotations.properties

## etc/annotations.properties
hostIp=${ENV:HOST_IP}
podName=${ENV:POD_NAME}
```

日志样例如下:

```json
{
  "timestamp": "2021-12-06T16:23:41.352519093Z",
  "level": "DEBUG",
  "thread": "main",
  "logger": "TestLogger",
  "message": "Test Log Message",
  "annotations": {
    "hostIp": "127.0.0.1",
    "podName": "mypod"
  }
}
```

### 监控

Airlift 包含 [jmxutils](https://github.com/martint/jmxutils)，可以非常方便地暴露JMX指标。

首先我们在pom.xml中添加必备的依赖项:

```xml
<dependency>
  <groupId>org.weakref</groupId>
  <artifactId>jmxutils</artifactId>
</dependency>

<dependency>
  <groupId>io.airlift</groupId>
  <artifactId>jmx</artifactId>
</dependency>

<dependency>
  <groupId>io.airlift</groupId>
  <artifactId>jmx-http</artifactId>
</dependency>

<dependency>
  <groupId>io.airlift</groupId>
  <artifactId>jmx-http-rpc</artifactId>
</dependency>
```

然后我们Service中新增JMX模块:


```java
package org.example;

import io.airlift.bootstrap.Bootstrap;
import io.airlift.event.client.EventModule;
import io.airlift.http.server.HttpServerModule;
import io.airlift.jaxrs.JaxrsModule;
import io.airlift.jmx.JmxHttpModule;
import io.airlift.jmx.JmxModule;
import io.airlift.jmx.http.rpc.JmxHttpRpcModule;
import io.airlift.json.JsonModule;
import io.airlift.node.NodeModule;
import org.weakref.jmx.guice.MBeanModule;

public class Service
{
    public static void main(String[] args)
    {
        Bootstrap app = new Bootstrap(new ServiceModule(),
                new JmxModule(),
                new JmxHttpModule(),
                new JmxHttpRpcModule(),
                new MBeanModule(),
                new NodeModule(),
                new HttpServerModule(),
                new EventModule(),
                new JsonModule(),
                new JaxrsModule());
        app.initialize();
    }

    private Service()
    {
    }
}

```

添加完模块化，我们可以对外暴露出一些指标，例如访问Rest接口的计数指标。



```java
package org.example;

import io.airlift.log.Logger;
import org.weakref.jmx.Managed;

import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

import java.util.concurrent.atomic.AtomicLong;

@Path("/v1/service")
public class ServiceResource
{
    // 获取logger
    private static final Logger LOG = Logger.get(ServiceResource.class);

    private final ServiceConfig config;
    private final AtomicLong helloCount = new AtomicLong();

    @Inject
    public ServiceResource(ServiceConfig config)
    {
        this.config = config;
    }

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public String hello()
    {
        String message = config.getHelloMessage();
        Long count = helloCount.incrementAndGet();
        // 打印日志
        LOG.info("call Hello Function %d and return %s", count, message);
        return message;
    }

    @Managed
    public long getHelloCount()
    {
        return helloCount.get();
    }
}

```

最后，在module上注册绑定。


```java
package org.example;

import com.google.inject.Binder;
import com.google.inject.Module;

import static io.airlift.configuration.ConfigBinder.configBinder;
import static io.airlift.jaxrs.JaxrsBinder.jaxrsBinder;
import static org.weakref.jmx.guice.ExportBinder.newExporter;

public class ServiceModule
        implements Module
{
    @Override
    public void configure(Binder binder)
    {
        jaxrsBinder(binder).bind(ServiceResource.class);
        configBinder(binder).bindConfig(ServiceConfig.class);
        newExporter(binder).export(ServiceResource.class).withGeneratedName();
    }
}

```

启动服务后，通过jconsole连接jmx，定位名为`org.example:name=ServiceResource`的mbean。或者通过访问`http://localhost:8080/v1/jmx/mbean/org.example:name=ServiceResource`获取jmx信息。

