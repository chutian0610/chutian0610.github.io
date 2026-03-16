# Trino源码学习-准备工作

本文是学习Trino源码前的准备工作。在开始之前，先做好准备工作：

1. 编译环境 Mac OS X or Linux 
2. 安装Java 17.0.4+, Trino 依赖 Java 17.0.4 或更新的版本来进行构建(本文基于 https://gitee.com/chutian0610/trino/tree/code-study/)。
3. 安装 Maven, Maven的版本不重要，Trino编译时会使用mvnw指定mvn版本
4. 安装 Docker

<!--more-->

## 编译Trino

下载 Trino 代码到本地。

```sh
git clone https://github.com/trinodb/trino.git
```

然后进到到项目根目录，使用maven编译。

```sh
./mvnw clean install -DskipTests
```

这样构建会比较慢。同时还会使用docker(前面有要求环境中安装docker)。可以使用下面的快速构建命令[<sup>1</sup>](#refer-anchor-1):

```sh
./mvnw -pl '!:trino-server-rpm,!:trino-docs,!:trino-proxy,!:trino-verifier,!:trino-benchto-benchmarks' clean install -T1C -nsu -DskipTests -Dmaven.javadoc.skip=true -Dmaven.source.skip=true -Dair.check.skip-all=true -Dskip.npm -Dskip.yarn
```

maven 配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<settings
    xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd"
    xmlns="http://maven.apache.org/SETTINGS/1.1.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <localRepository>/Users/victorchu/.m2/repository</localRepository>
    <mirrors>
        <mirror>
            <id>aliyunmaven</id>
            <mirrorOf>*,!confluent,!jcenter</mirrorOf>
            <name>阿里云公共仓库</name>
            <url>https://maven.aliyun.com/repository/public</url>
          </mirror>
    </mirrors>
</settings>
```

在根目录pom.xml 上添加 repo配置，(解决阿里云中找不到包的问题，要在mirrorOf中忽略该仓库)

```xml
<repositories>
  <repository>
    <releases>
      <enabled>true</enabled>
    </releases>
    <snapshots>
      <enabled>false</enabled>
    </snapshots>
    <id>confluent</id>
    <url>http://packages.confluent.io/maven/</url>
  </repository>
  <repository>
    <releases>
      <enabled>true</enabled>
    </releases>
    <snapshots>
      <enabled>false</enabled>
    </snapshots>
    <id>jcenter</id>
    <url>https://maven.aliyun.com/repository/jcenter</url>
  </repository>
</repositories>
```


## 调试

Trino附带了开箱即用的示例配置，可以用于开发。

- Main Class: `io.trino.server.DevelopmentServer`
- VM Options: `-ea -Dconfig=etc/config.properties -Dlog.levels-file=etc/log.properties -Djdk.attach.allowAttachSelf=true`
- Working directory: `$MODULE_DIR$`
- Use classpath of module: trino-server-dev

工作目录应该是子目录trino-server-dev,在idea中可以使用`$MODULE_DIR$`自动定位。

### CLI

启动 CLI 以连接到服务器并运行 SQL 查询：

```
client/trino-cli/target/trino-cli-*-executable.jar
```

运行查询以查看群集中的节点：

```
SELECT * FROM system.runtime.nodes;
```

针对 TPCH 连接器运行查询：

```
SELECT * FROM tpch.tiny.region;
```

### WebUI

访问 localhost:8080端口可以看到webUI,默认是无用户(即随意输入username)都可以登录。

## Trino 源码结构说明

```sh
$ tree -d -L 2
.
├── client # 客户端相关模块
│   ├── trino-cli 
│   ├── trino-client
│   └── trino-jdbc # Jdbc方式客户端
├── core  # 服务端核心模块 
│   ├── docker
│   ├── trino-main # 最重要的核心模块
│   ├── trino-parser
│   ├── trino-server # trino.xml中定义打包的所有子模块
│   ├── trino-server-main
│   ├── trino-server-rpm
│   └── trino-spi
├── docs # 文档
├── lib # lib库
│   ├── trino-array
│   ├── trino-collect
│   ├── trino-filesystem
│   ├── trino-geospatial-toolkit
│   ├── trino-hadoop-toolkit
│   ├── trino-hdfs
│   ├── trino-matching
│   ├── trino-memory-context
│   ├── trino-orc
│   ├── trino-parquet
│   ├── trino-phoenix5-patched
│   ├── trino-plugin-toolkit
│   ├── trino-rcfile
│   └── trino-record-decoder
├── plugin # 插件体系
│   ├── trino-accumulo
│   ├── trino-accumulo-iterators
│   ├── trino-atop
│   ├── trino-base-jdbc # Jdbc方式Connector的通用父模块
│   ├── trino-bigquery
│   ├── trino-blackhole
│   ├── trino-cassandra
│   ├── trino-clickhouse
│   ├── trino-delta-lake
│   ├── trino-druid
│   ├── trino-elasticsearch
│   ├── trino-example-http # connector开发的样例模块
│   ├── trino-exchange-filesystem
│   ├── trino-geospatial
│   ├── trino-google-sheets
│   ├── trino-hive
│   ├── trino-hive-hadoop2
│   ├── trino-http-event-listener
│   ├── trino-hudi
│   ├── trino-iceberg
│   ├── trino-jmx
│   ├── trino-kafka
│   ├── trino-kinesis
│   ├── trino-kudu
│   ├── trino-local-file
│   ├── trino-mariadb
│   ├── trino-memory
│   ├── trino-ml
│   ├── trino-mongodb
│   ├── trino-mysql
│   ├── trino-oracle
│   ├── trino-password-authenticators
│   ├── trino-phoenix5
│   ├── trino-pinot
│   ├── trino-postgresql
│   ├── trino-prometheus
│   ├── trino-raptor-legacy
│   ├── trino-redis
│   ├── trino-redshift
│   ├── trino-resource-group-managers
│   ├── trino-session-property-managers
│   ├── trino-singlestore
│   ├── trino-sqlserver
│   ├── trino-teradata-functions
│   ├── trino-thrift
│   ├── trino-thrift-api
│   ├── trino-thrift-testing-server
│   ├── trino-tpcds
│   └── trino-tpch
├── service # 其他服务
│   ├── trino-proxy # 代理服务
│   └── trino-verifier # 验证服务
└── testing # 测试相关模块
    ├── bin
    ├── trino-benchmark
    ├── trino-benchmark-queries
    ├── trino-benchto-benchmarks
    ├── trino-faulttolerant-tests
    ├── trino-plugin-reader
    ├── trino-product-tests
    ├── trino-product-tests-launcher
    ├── trino-server-dev
    ├── trino-testing
    ├── trino-testing-containers
    ├── trino-testing-kafka
    ├── trino-testing-resources
    ├── trino-testing-services
    ├── trino-test-jdbc-compatibility-old-driver
    ├── trino-test-jdbc-compatibility-old-server
    └── trino-tests
```

## Trino 安装包结构

```sh
.
├── bin
│   ├── launcher
│   ├── launcher.properties
│   ├── launcher.py
│   └── procname
├── lib # trino jar lib 库
│   ├── accessors-smart-2.4.7.jar
│   ├── aircompressor-0.21.jar
│   ├── antlr4-runtime-4.11.1.jar
│   ├── aopalliance-1.0.jar
│   ├── asm-9.3.jar
│   ├── asm-analysis-9.2.jar
│   ├── ...
│   └── validation-api-2.0.1.Final.jar
├── NOTICE
├── plugin # 插件目录
│   ├── accumulo # 插件 jar lib
│   ├── atop
│   ├── bigquery
│   ├── ...
│   └── tpch
└── README.txt
```

## 多节点配置

```conf
## coordinator
node.id=ffffffff-ffff-ffff-ffff-ffffffffffff
coordinator=true
node-scheduler.include-coordinator=false
http-server.http.port=8080
discovery.uri=http:/localhost:8080

## worker
node.id=ffffffff-ffff-ffff-ffff-fffffffffff1
coordinator=false
http-server.http.port=8081
discovery.uri=http:/localhost:8080
```

## ide 调试

### idea

**trino 推荐使用idea作为开发IDE**,通过如下配置可以调试代码.

- Main Class: io.trino.server.DevelopmentServer
- VM Options: -ea -Dconfig=etc/config.properties -Dlog.levels-file=etc/log.properties -Djdk.attach.allowAttachSelf=true
- Working directory: `$MODULE_DIR$`
- Use classpath of module: trino-server-dev

#### idea 远程debug

除了在本地调试代码，还可以远程debug Trino. 

```sh
cd ~/projects/trino/testing/trino-server-dev;
cpath=`echo ~/projects/trino/testing/trino-server-dev/target/dependency/*.jar| tr ' ' ':'`
/home/victorchu/.sdkman/candidates/java/17.0.5-oracle/bin/java -ea -Dconfig=etc/config.properties -Dlog.levels-file=etc/log.properties -Djdk.attach.allowAttachSelf=true -agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005 -classpath ~/projects/trino/testing/trino-server-dev/target/trino-server-dev-424.jar:${cpath} io.trino.server.DevelopmentServer;
```

注意要在trino-server-dev的pom中增加如下内容

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-dependency-plugin</artifactId>
  <version>3.6.0</version>
  <executions>
    <execution>
      <id>copy-dependencies</id>
      <goals>
        <goal>copy-dependencies</goal>
      </goals>
      <phase>package</phase>
      <configuration>
        <!-- configure the plugin here -->
      </configuration>
    </execution>
  </executions>
</plugin>
```

### ~~vscode~~

~~在.vscode中添加如下文件~~

1. launch.json

```json
{
    "configurations": [
    {
        "type": "java",
        "name": "DevelopmentServer",
        "request": "launch",
        "mainClass": "io.trino.server.DevelopmentServer",
        "vmArgs": "-ea -Dconfig=etc/config.properties -Dlog.levels-file=etc/log.properties -Djdk.attach.allowAttachSelf=true",
        "cwd": "${workspaceFolder}/testing/trino-server-dev",
    }
    ]
}
```

2. ~~tasks.json~~

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "Build Trino",
            "type": "shell",
            "group": "build",
            "command": "bash ./.vscode/build.sh"
        }
    ]
  }
```

3. ~~build.sh~~

```sh
## 使用sdkman设置java 版本
## source /home/victorchu/.sdkman/bin/sdkman-init.sh;
## sdk use java 17.0.5-oracle;
mvn clean;
mvn -pl '!:trino-server-rpm,!:trino-docs,!:trino-proxy,!:trino-verifier,!:trino-benchto-benchmarks' install -T1C -nsu -DskipTests -Dmaven.javadoc.skip=true -Dmaven.source.skip=true -Dair.check.skip-all=true -Dskip.npm -Dskip.yarn;
```

> ~~Vscode 自身的build 常和maven的build冲突，导致项目启动异常，所以不建议使用vscode开发~~

## 参考

<div id="refer-anchor-1"></div>

- [1] [trino-dev-cheat-sheet](https://gist.github.com/findepi/04c96f0f60dcc95329f569bb0c44a0cd)

