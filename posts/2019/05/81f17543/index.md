# Flink源码学习-准备工作

本文是学习flink源码前的准备工作。在开始之前，先做好准备工作：

- 安装JDK8+，Flink 依赖 Java 8 或更新的版本来进行构建。
  - 自flink1.15 开始需要Java 11(如无必要不使用更高版本的JDk去build)
- 安装Maven 3 ( Maven 3.3.x 可以构建 Flink，但是不能正确地屏蔽掉指定的依赖。Maven 3.2.5 可以正确地构建库文件)
- 安装Scala 2.11或2.12
  - 自flink1.15 开始需要Scala 2.12

<!--more-->

## 编译flink

下载 Flink 代码到本地。

```sh
git clone https://github.com/apache/flink.git
```

> 国内下载慢的话可以使用gitee的镜像仓库:`https://gitee.com/apache/flink.git`

maven配置(设置镜像):

```xml
<!--.m2/settings.xml-->
<mirrors>
	<mirror>
  		<id>aliyunmaven</id>
  		<mirrorOf>central</mirrorOf>
  		<name>阿里云公共仓库</name>
  		<url>https://maven.aliyun.com/repository/public</url>
	</mirror>
  </mirrors>
```
项目配置（设置远程仓库）:

```xml
<repository>
  <id>confluent</id>
  <url>http://packages.confluent.io/maven/</url>
  <releases>
    <enabled>true</enabled>
  </releases>
  <snapshots>
    <enabled>false</enabled>
  </snapshots>
</repository>
<repository>
  <id>mapr-public</id>
  <url>https://maven.aliyun.com/repository/mapr-public</url>
  <releases>
    <enabled>true</enabled>
  </releases>
  <snapshots>
    <enabled>false</enabled>
  </snapshots>
</repository>
```

编译源码，编译结果在flink/flink-dist/target/下。

```sh
mvn clean install -DskipTests

## 加速
mvn clean install \
	-DskipTests \ # 跳过测试部分
	-Dfast \ # 跳过QA 的插件和 JavaDocs 的生成
	-Pskip-webui-build  \ # 跳过 WebUI 编译, flink 15后支持该参数
	-T 1C \ # 可选，支持多处理器或者处理器核数参数,加快构建速度
```

> maven-shade-plugin 现存的 bug 可能会在并行构建时产生死锁。建议分2步进行构建：首先使用并行方式运行 mvn validate/test-compile/test，然后使用单线程方式运行 mvn package/verify/install。

## 依赖屏蔽

Flink [屏蔽](https://maven.apache.org/plugins/maven-shade-plugin/)了一些它使用的包，这样做是为了避免与程序员自己引入的包的存在的可能的版本冲突。屏蔽掉的包包括 *Google Guava*,*Asm*,*Apache Curator*,*Apache HTTP Components*,*Netty* 等。

这种依赖屏蔽机制最近在 Maven 中有所改变。需要用户根据 Maven 的的不同版本来执行不同的命令。

**对于Maven 3.1.x and 3.2.x**
直接在 Flink 源码根目录执行命令 `mvn clean install -DskipTests` 就足够了。

### v1.12
**Maven 3.3.x**
如下的构建需要两步走：第一步需要在基础目录下执行编译构建；第二步需要在编译后的 flink-dist 目录下执行：

```bash
mvn clean install -DskipTests
cd flink-dist
mvn clean install
```
### 1.13 above

**Maven 3.3.x**
如下的构建需要两步走：第一步需要在基础目录下执行编译构建；第二步需要在shade模块目录下执行：

```bash
## build overall project
mvn clean install -DskipTests

## build shaded modules used in dist again, for example:
cd flink-filesystems/flink-s3-fs-presto/
mvn clean install -DskipTests
## ... and other modules

## build dist again to include shaded modules
cd flink-dist
mvn clean install
```

> 注意:运行 `mvn --version` 以查看Maven的版本。

## scala 版本

flink默认是基于scala 2.11和2.12打包(15后只支持2.12)。但是可以在打包时选择scala的版本:

```sh
mvn clean install -DskipTests -Dscala-2.12 -Dscala.version=<scala version>
```

## 编译问题

### unsafe 类找不到

```
符号: 方法 defineClass(java.lang.String,byte[],int,int,java.lang.ClassLoader,java.security.ProtectionDomain)
位置: 类 sun.misc.Unsafe
```

1. 设置环境变量JAVA_HOME
2. 设置环境变量CLASSPATH,加上unsafe类所在jar包路径。

### 找不到jar包

```sh
[ERROR] Failed to execute goal on project flink-avro-confluent-registry: Could not resolve dependencies for project org.apache.flink:flink-avro-confluent-registry:jar:1.8-SNAPSHOT: Could not find artifact io.confluent:kafka-schema-registry-client:jar:3.3.1 in nexus-aliyun (http://maven.aliyun.com/nexus/content/groups/public) -> [Help 1]
```

1. 可以在pom.xml中加上如下配置:

```xml
<repository>
  <id>confluent</id>
  <url>http://packages.confluent.io/maven/</url>
</repository>
```

2. 也可以手动下载[kafka-schema-registry-client-3.3.1.jar](http://packages.confluent.io/maven/io/confluent/kafka-schema-registry-client/3.3.1/kafka-schema-registry-client-3.3.1.jar).将下载的kafka-schema-registry-client-3.3.1.jar安装到本地仓库:

```sh
mvn install:install-file -DgroupId=io.confluent -DartifactId=kafka-schema-registry-client -Dversion=3.3.1 -Dpackaging=jar  -Dfile=/home/hadoop/downloads/kafka-schema-registry-client-3.3.1.jar -DpomFile=/home/hadoop/downloads/kafka-schema-registry-client-3.3.1.pom
```

### cannot access io.confluent.kafka.serializers.AbstractKafkaSchemaSerDe

maven 中心仓提供的kafka-avro-serializer的pom文件没有依赖信息，手动安装依赖或者修改本地maven库中的pom文件.

```sh
$ mvn install:install-file -DgroupId=io.confluent -DartifactId=kafka-avro-serializer -Dversion=5.5.2 -Dpackaging=jar -Dfile=kafka-avro-serializer-5.5.2.jar -DpomFile=kafka-avro-serializer-5.5.2.pom
$ mvn install:install-file -DgroupId=io.confluent -DartifactId=kafka-schema-serializer -Dversion=5.5.2 -Dpackaging=jar -Dfile=kafka-schema-serializer-5.5.2.jar -DpomFile=kafka-schema-serializer-5.5.2.pom
$ mvn install:install-file -DgroupId=io.confluent -DartifactId=kafka-schema-registry-client -Dversion=5.5.2 -Dpackaging=jar -Dfile=kafka-schema-registry-client-5.5.2.jar -DpomFile=kafka-schema-registry-client-5.5.2.pom
$ mvn install:install-file -DgroupId=io.confluent -DartifactId=kafka-schema-registry-parent -Dversion=5.5.2 -Dpackaging=pom -Dfile=kafka-schema-registry-parent-5.5.2.pom -DpomFile=kafka-schema-registry-parent-5.5.2.pom
$ mvn install:install-file -DgroupId=io.confluent -DartifactId=common -Dversion=5.5.2 -Dpackaging=pom -Dfile=common-5.5.2.pom -DpomFile=common-5.5.2.pom
$ mvn install:install-file -DgroupId=io.confluent -DartifactId=common-parent -Dversion=5.5.2 -Dpackaging=pom -Dfile=common-parent-5.5.2.pom -DpomFile=common-parent-5.5.2.pom
$ mvn install:install-file -DgroupId=io.confluent -DartifactId=common-config -Dversion=5.5.2 -Dpackaging=jar -Dfile=common-config-5.5.2.jar -DpomFile=common-config-5.5.2.pom
$ mvn install:install-file -DgroupId=io.confluent -DartifactId=common-utils -Dversion=5.5.2 -Dpackaging=jar -Dfile=common-utils-5.5.2.jar -DpomFile=common-utils-5.5.2.pom
```

问题具体描述参见[邮件列表](http://apache-flink-user-mailing-list-archive.2336050.n4.nabble.com/Test-failed-in-flink-end-to-end-tests-flink-end-to-end-tests-common-kafka-td40824.html)

### 编译卡在 Running 'npm ci --cache-max=0 --no-save'

flink 编译中需要安装node 包,在flink-runtime-web/pom.xml 中。

将npm install 部分的arguments`ci --cache-max=0 --no-save` 修改为 `install -registry=https://registry.npm.taobao.org --cache-max=0 --no-save`

或者在`<configuration>`下面添加元素 `<npmRegistryURL>https://registry.npm.taobao.org</npmRegistryURL>`

### ng: command not found

环境没有ng命令，使用`npm install @angular/cli -registry=https://registry.npm.taobao.org`。

> 注意在 flink/flink-runtime-web/web-dashboard 目录下执行 npm,目的是在目录下安装 angular/cli

### javac-9+181-r4173-1.jar因为阿里云的maven仓库的403权限问题无法下载

```log
[ERROR] Failed to execute goal com.diffplug.spotless:spotless-maven-plugin:2.4.2:check (spotless-check) on project flink-annotations: Execution spotless-check of goal com.diffplug.spotless:spotless-maven-plugin:2.4.2:check failed: Unable to resolve dependencies: Failed to collect dependencies at com.google.googlejavaformat:google-java-format:jar:1.7 -> com.google.errorprone:javac-shaded:jar:9+181-r4173-1: Failed to read artifact descriptor for com.google.errorprone:javac-shaded:jar:9+181-r4173-1: Could not transfer artifact com.google.errorprone:javac-shaded:pom:9+181-r4173-1 from/to nexus-aliyun (https://maven.aliyun.com/repository/public/): Authorization failed for https://maven.aliyun.com/repository/public/com/google/errorprone/javac-shaded/9+181-r4173-1/javac-shaded-9+181-r4173-1.pom 403 Forbidden -> [Help 1]
```

将flink 根pom中的 plugin 注释掉:

```xml
<!--			<plugin>-->
<!--				<groupId>com.diffplug.spotless</groupId>-->
<!--				<artifactId>spotless-maven-plugin</artifactId>-->
<!--			</plugin>-->
```

由于flink使用 Spotless plugin 和 google-java-format 一起格式化代码。所以需要 `com.google.errorprone:javac-shaded:jar:9+181-r4173-1` ，注释掉打包插件即可。

## 调试

### 本地调试

flink 在编译完成后可以本地调试,例如使用flink-examples/flink-examples-streaming 下的wordcount任务直接可以debug。

> 注意,将 run/debug configuration中的before launch设置为`build,no error check`防止出现build中异常导致无法启动。

### 远程调试( jvm remote port)

在jvm启动参数中添加远程调试参数:

1. 如果是调试Client，可以将上述参数加到bin/flink脚本的最后一行中，形如：
```sh
JVM_REMOTE_DEBUG_ARGS='-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005'
exec $JAVA_RUN $JVM_ARGS $JVM_REMOTE_DEBUG_ARGS "${log_setting[@]}" -classpath "`manglePathList "$CC_CLASSPATH:$INTERNAL_HADOOP_CLASSPATHS"`" org.apache.flink.client.CliFrontend "$@"
```
2. 如果是调试JobManager或TaskManager，可以在conf/flink-conf.yaml中添加：

```yml
env.java.opts: -agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5006
```
3. 启动flink client或jobmanager或taskmanager，此时程序会suspend等待debuger连接（通过suspend=y来配置）。
4. 配置IDEA中的remote：host配置为localhost，配置port（参考1中的配置的address端口）。
5. 在Flink源码中设置断点，连接远程host，然后就可以开始debug跟踪了。

### 单元测试

在IDEA中，项目目录如果有一个类报错，Junit测试用例都不能运行，即使测试用例中并没有应用到这个类，在Eclipse中是可以的成功运行的。这是因为IDEA运行Junit或者运行main方法时候会默认先编译整个项目。

```
run
|-> edit configurations
|-> Before Launch
|-> 取消对build的勾选
```
## 参考

- [1] [idea 导入flink](https://ci.apache.org/projects/flink/flink-docs-master/zh/flinkDev/ide_setup.html)
- [2] [从源码构建 Flink](https://ci.apache.org/projects/flink/flink-docs-master/zh/flinkDev/building.html)

