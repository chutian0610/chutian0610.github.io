# My Effective Maven

我的Effective Maven使用笔记

## 常用 mvn 命令

### mvn输出增加线程ID

```sh
MAVEN_OPTS=-Dorg.slf4j.simpleLogger.showThreadName=true mvn <goals>
```

> 见 [https://maven.apache.org/maven-logging.html](https://maven.apache.org/maven-logging.html)

## pom.xml 设置

### 定义源码位置

设置Maven 源代码的位置

```xml
<properties>
    <src.dir>src/main/java</src.dir>
</properties>
```

### jar包名

maven的包名可以通过finalName配置。

```xml
<build>
    <finalName> ${project.artifactId}-${project.version}</finalName>
</build>
```

#### 时间戳

如果想在包名上增加UTC时间戳，可以通过在finalName中添加属性`maven.build.timestamp`.

```xml
<build>
    <finalName> ${project.artifactId}-${maven.build.timestamp}</finalName>
</build>
```

`maven.build.timestamp`的格式是通过`maven.build.timestamp.format`参数设置的。

```
<properties>
    <maven.build.timestamp.format>yyyyMMddHHmmss</maven.build.timestamp.format>
</properties>
```

maven自带时间组件时区是只能使用UTC，要使用正确的时间，需要另一个插件`build-helper-maven-plugin`的帮助。

```xml
<build>
    <!-- 要将maven.build.timestamp替换成build.time -->
    <finalName>${project.artifactId}-${build.time}</finalName>
    <plugins>
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>build-helper-maven-plugin</artifactId>
            <version>3.2.0</version>
            <executions>
                <execution>
                    <id>timestamp-property</id>
                    <goals>
                        <goal>timestamp-property</goal>
                    </goals>
                    <configuration>
                        <name>build.time</name>
                        <pattern>yyyyMMddHHmm</pattern>
                        <locale>zh_CN</locale>
                        <timeZone>GMT+8</timeZone>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

## Build Helper

[`build-helper-maven-plugin`](https://www.mojohaus.org/build-helper-maven-plugin/index.html)插件提供了 Maven 使用中的多个常用小功能。

- 管理源码目录
  - add-source：添加源码目录
  - add-test-source：添加测试源码目录
- 管理资源目录
  - add-resource：添加资源目录
  - add-test-resource：添加测试资源目录
- 版本管理
  - parse-version：解析项目版本为属性
  - released-version：解析项目最新发布版本
- attach-artifact: 附件管理
- timestamp-property：设置时间戳属性
- rootlocation: 多模块构建的根文件夹
- ... ...

## 依赖问题

当项目开发维护了一段时间时，经常会有项目打包速度慢，jar依赖多，依赖关系错综复杂，这种问题是项目维护最常见的问题，由于开发人员在bugfix或者feature开发时往往只是往项目中添加jar依赖，我们需要分析出项目中哪些依赖是用到的，哪些依赖是多余的。

为了解决这一问题，可以通过 `mvn dependency:analyze` 来查看依赖问题(analyze命令用于单独执行，也可以在pom配置中加上`analyze-only`goal和mvn打包一起使用)。

通常会有如下的输出日志:

```log
[INFO] --- maven-dependency-plugin:2.8:analyze (default-cli) @ xxxproject ---
[WARNING] Used undeclared dependencies found:
[WARNING]    org.springframework:spring-beans:jar:4.0.0.RELEASE:compile
[WARNING]    org.springframework:spring-context:jar:4.0.0.RELEASE:compile
[WARNING] Unused declared dependencies found:
[WARNING]    com.alibaba:dubbo:jar:2.5.3:compile
[WARNING]    com.baidu.disconf:disconf-client:jar:2.6.32:compile
```

对于日志中的`Used undeclared dependencies found`和`Unused declared dependencies found`我们分别介绍下处理方案。

### Used undeclared dependencies found

`Used undeclared dependencies found` 是指某些依赖的包在代码中有用到它的代码，但是它并不是直接的依赖（就是说没有在pom中直接声明），是通过引入传递的包。

因此，对于`Used undeclared dependencies found`的包，可以通过直接在pom文件的依赖中显式声明。

### Unused declared dependencies found

`Unused declared dependencies found` 指我们在pom中声明了依赖，但是在实际代码中并没有用到这个包！也就是多余的包。 这个时候我们就可以把这个依赖从pom中剔除。

> 注意： 这里说的代码没有用到，指的是在代码中没有显式引用，并不是意味着真的没有用到这些包(例如spi方式引入的代码).对于这种情况可以在 `<configuration>` 中的`<ignoredUsedUndeclaredDependencies>` 配置中增加这些包。


## 格式化

### maven-surefire-plugin

maven-surefire-plugin的[Junit格式化配置](https://maven.apache.org/surefire/maven-surefire-plugin/examples/junit-platform.html)

- 在TestCase中增加`@DisplayName`注解，用于展示自定义测试名。
- 树状结构打印单元测试结果

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <!-- 注意版本要高于3.0.0-M4 -->
    <version>3.1.2</version>
    <dependencies>
        <dependency>
            <groupId>me.fabriciorby</groupId>
            <artifactId>maven-surefire-junit5-tree-reporter</artifactId>
            <version>0.1.0</version>
        </dependency>
    </dependencies>
    <configuration>
        <reportFormat>plain</reportFormat>
        <consoleOutputReporter>
            <disable>true</disable>
        </consoleOutputReporter>
        <statelessTestsetInfoReporter implementation="org.apache.maven.plugin.surefire.extensions.junit5.JUnit5StatelessTestsetInfoTreeReporter"/>
    </configuration>
</plugin>
```

日志输出样例:

```xml
[INFO] |
[INFO] +-- 语法解析测试
[INFO] | +-- [OK] parse01 - 0.001 ss
[INFO] | +-- [OK] parse02 - 0.001 ss
[INFO] | +-- [OK] parse03 - 0.010 ss
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 70, Failures: 0, Errors: 0, Skipped: 0
```

### Sortpom Maven Plugin

[Sortpom](https://github.com/Ekryd/sortpom)插件帮助排序maven pom 文件。

## 重复资源

### duplicate-finder-maven-plugin

duplicate-finder-maven-plugin 用于查找classpath下重复的class和resource。

```xml
<plugin>
  <groupId>org.basepom.maven</groupId>
  <artifactId>duplicate-finder-maven-plugin</artifactId>
  <version>1.3.0</version>
  <executions>
    <execution>
      <id>default</id>
      <phase>verify</phase>
      <goals>
        <goal>check</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
      <configuration>
        <!-- 用于设置忽略的resource-->
        <ignoredResourcePatterns>
            <ignoredResourcePattern>parquet.thrift</ignoredResourcePattern>
            <ignoredResourcePattern>about.html</ignoredResourcePattern>
        </ignoredResourcePatterns>
        <!-- 用于设置忽略的class-->
        <ignoredClassPatterns>
            <ignoredClassPattern>shaded.parquet.it.unimi.dsi.fastutil.*</ignoredClassPattern>
            <ignoredClassPattern>module-info</ignoredClassPattern>
        </ignoredClassPatterns>
      </configuration>
  </configuration>
</plugin>
```

如果在日志中发现重复的类，可以将通过`mvn dependency:tree`找到对应的依赖将其exclude掉。

## 规则检查

### Maven Enforcer

Maven Enforcer插件是一个用于强制执行特定规则的Maven插件。它可以帮助您确保项目中的依赖项和构建环境符合特定的要求，例如强制使用特定版本的Java、强制执行依赖项收敛规则等。Maven Enforcer插件非常有用，可以确保项目的稳定性和可靠性。

Maven Enforcer插件提供了许多规则，可以根据需要进行配置。例如:

- Require Java Version：强制要求使用特定版本的Java。
- Require Maven Version：强制要求使用特定版本的Maven。
- Require Plugin Versions：强制要求所有插件定义版本。
- Dependency Convergence：强制要求依赖项收敛，即确保项目中只使用一个版本的依赖项。
- Banned Dependencies：禁止使用特定的依赖项。
- Require Property：强制要求使用特定的属性。
- Require Upper Bound Dependencies: 要求所有依赖的符合最低版本。


## 参考

- [1] [Apache Maven Dependency Plugin](https://maven.apache.org/plugins/maven-dependency-plugin/index.html)
- [2] [duplicate-finder-maven-plugin](https://basepom.github.io/duplicate-finder-maven-plugin/development/)

