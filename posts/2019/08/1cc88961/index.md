# My Effective Maven

我的Effective Maven使用笔记

## 内置变量

- `${project.basedir}`: 代表项目根目录（pom.xml 所在目录）
- `${project.parent.basedir}/`: 代表Maven 多模块项目，在父工程根目录

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
- ...

## 发布

### flatten-maven-plugin

flatten-maven-plugin 的核心作用是解决 Maven 多模块项目在使用动态版本号（如 `${revision}`）时，install/deploy 到仓库的 POM 文件无法被其他项目解析的问题。它通过在构建时生成一个“扁平化”的 POM（默认名为 .flattened-pom.xml），将占位符替换为实际值，并用它替换原始 POM 进行发布。

```xml
<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>flatten-maven-plugin</artifactId>
  <version>1.2.7</version>
  <configuration>
    <updatePomFile>true</updatePomFile>
    <flattenMode>resolveCiFriendliesOnly</flattenMode>
  </configuration>
  <executions>
    <execution>
      <id>flatten</id>
      <phase>process-resources</phase>
      <goals>
        <goal>flatten</goal>
      </goals>
    </execution>
    <execution>
      <id>flatten.clean</id>
      <phase>clean</phase>
      <goals>
        <goal>clean</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

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

> 注意： 这里说的代码没有用到，指的是在代码中没有显式引用，并不是意味着真的没有用到这些包(例如spi方式引入的代码).对于这种情况可以在 `<configuration>` 中的`<ignoredUnusedDeclaredDependencies>` 配置中增加这些包。


## 格式化

### formatter-maven-plugin

支持 Eclipse/Google/ 自定义 代码风格，可灵活配置缩进、换行、导入排序等规则。

```xml
<build>
  <plugins>
    <!-- Java 代码格式化插件 -->
    <plugin>
      <groupId>net.revelc.code.formatter</groupId>
      <artifactId>formatter-maven-plugin</artifactId>
      <version>2.23.0</version> <!-- 使用最新版本 -->
      <configuration>
        <!-- 基础配置 -->
        <encoding>UTF-8</encoding>
        <lineEnding>LF</lineEnding> <!-- 换行符（Linux/Mac用LF，Windows用CRLF） -->
        <configFile>${project.basedir}/eclipse-formatter.xml</configFile> <!-- 自定义规则文件（可选） -->
        
        <!-- 格式化范围：包含/排除文件 -->
        <includes>
          <include>**/*.java</include> <!-- 格式化所有Java文件 -->
        </includes>
        <excludes>
          <exclude>**/target/**</exclude> <!-- 排除编译产物 -->
          <exclude>**/generated/**</exclude> <!-- 排除自动生成的代码 -->
        </excludes>
        
        <!-- 代码风格：默认Eclipse，也可指定Google -->
        <style>Eclipse</style>
        <!-- <style>Google</style> -->
        
        <!-- 可选：自动排序导入 -->
        <sortImports>true</sortImports>
      </configuration>
      
      <!-- 绑定到 compile 阶段（可选，编译前自动格式化） -->
      <executions>
        <execution>
          <phase>compile</phase>
          <goals>
            <goal>format</goal>
          </goals>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

[配置文件](https://gist.github.com/chutian0610/23adb8a902bb1a4d7631df4c8d36cc4b)参考.

### impsort-maven-plugin

[impsort-maven-plugin](https://github.com/revelc/impsort-maven-plugin) 是一个用于在 Maven 构建过程中自动排序 Java 源文件 import 语句的插件。让项目团队无论使用何种 IDE，都能通过构建流程强制统一 import 的顺序，从而避免因 IDE 设置不同而产生的无意义代码差异。

```xml
<plugin>
  <groupId>net.revelc.code</groupId>
  <artifactId>impsort-maven-plugin</artifactId>
  <version>1.13.0</version>
  <configuration>
    <compliance>17</compliance>
    <!--设置 import 组的顺序-->
    <groups>java.,javax.,*</groups>
    <!--设置 static import 组的顺序-->
    <staticGroups>*</staticGroups>
    <removeUnused>true</removeUnused>
    <excludes>
      <!--动态生成的代码不参与 import sort-->
      <exclude>**/thrift/*.java</exclude>
    </excludes>
  </configuration>
  <executions>
    <execution>
      <id>sort-imports</id>
      <goals>
        <!-- runs at process-sources phase by default -->
        <goal>sort</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

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

```xml
<plugin>
  <groupId>com.github.ekryd.sortpom</groupId>
  <artifactId>sortpom-maven-plugin</artifactId>
  <version>4.0.0</version>
  <executions>
    <execution>
      <goals>
        <goal>sort</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

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

## 健壮性

### spotbugs-maven-plugin 

spotbugs-maven-plugin 将 SpotBugs 静态分析工具集成到 Maven 构建流程中。它通过分析编译后的字节码（而非源代码）来检测 Java 代码中的潜在缺陷、错误模式和安全隐患，并可以配置为在发现特定级别的问题时直接使构建失败，从而充当代码质量门禁。

```xml
<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.8.6.0</version>
    <dependencies>
        <dependency>
            <groupId>com.github.spotbugs</groupId>
            <artifactId>spotbugs</artifactId>
            <version>4.8.6</version>
        </dependency>
    </dependencies>
    <configuration>
        <!-- 分析的努力程度-->
        <effort>Max</effort>
        <!--报告/失败的最低严重级别。低于此级别的缺陷将被忽略。-->
        <threshold>Low</threshold>
        <!--当分析过程中发生错误（如解析失败）时，是否使构建失败。-->
        <failOnError>false</failOnError>
        <plugins>
            <!--集成 Find Security Bugs 进行安全审计-->
            <plugin>
                <groupId>com.h3xstream.findsecbugs</groupId>
                <artifactId>findsecbugs-plugin</artifactId>
                <version>1.12.0</version>
            </plugin>
        </plugins>
        <!-- 可选：通过过滤文件仅关注安全类别的缺陷 -->
        <includeFilterFile>spotbugs-security-include.xml</includeFilterFile>
    </configuration>
    <executions>
        <execution>
            <id>spotbugs-check</id>
            <phase>verify</phase>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### maven-checkstyle-plugin

maven-checkstyle-plugin 将 Checkstyle 静态分析工具集成到 Maven 生命周期中，通过预定义的规则集检查 Java 源代码是否符合编码规范（如缩进、命名、Javadoc、导入顺序等）。与 SpotBugs 分析字节码不同，Checkstyle 直接分析源代码，因此能在编码阶段尽早发现风格问题，并可作为 CI 流水线中的强制性质量门禁。

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-checkstyle-plugin</artifactId>
    <version>3.6.0</version>
    <dependencies>
        <dependency>
            <groupId>com.puppycrawl.tools</groupId>
            <artifactId>checkstyle</artifactId>
            <version>12.3.1</version>
        </dependency>
    </dependencies>
    <configuration>
        <configLocation>src/config/CheckStyle.xml</configLocation>
        <consoleOutput>true</consoleOutput>
        <failsOnError>true</failsOnError>
        <failOnViolation>true</failOnViolation>
    </configuration>
    <executions>
        <execution>
            <id>validate</id>
            <phase>validate</phase>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

[配置文件](https://gist.github.com/chutian0610/9bca2b89fb78138cadb0eda499f9ff46)参考:

### jacoco-maven-plugin

JaCoCo（Java Code Coverage）是 Java 生态中主流的代码覆盖率工具。jacoco-maven-plugin 将其集成到 Maven 生命周期中，负责在测试执行时动态注入探针以收集覆盖率数据，并基于这些数据生成可视化报告或执行覆盖率门禁检查。与 SpotBugs（字节码缺陷）和 Checkstyle（源码风格）不同，JaCoCo 关注的是“测试到底覆盖了多少代码”。

```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.14</version>
  <executions>
    <execution>
      <goals>
          <goal>prepare-agent</goal>
      </goals>
    </execution>
    <!-- CI 门禁 -->
    <execution>
      <id>check-coverage</id>
      <goals>
          <goal>check</goal>
      </goals>
      <configuration>
          <!-- 仅警告不阻断-->
          <haltOnFailure>false</haltOnFailure>
          <rules>
              <rule>
                <!--规则列表。每条规则可以指定 element：BUNDLE（整个项目，默认）、
                PACKAGE、CLASS、SOURCEFILE 或 METHOD。-->
                  <element>BUNDLE</element>
                  <limits>
                      <limit>
                          <counter>LINE</counter>
                          <value>COVEREDRATIO</value>
                          <minimum>0.80</minimum>
                      </limit>
                      <limit>
                          <counter>CLASS</counter>
                          <value>MISSEDCOUNT</value>
                          <maximum>0</maximum>
                      </limit>
                  </limits>
              </rule>
          </rules>
      </configuration>
    </execution>
    <execution>
      <id>generate-report</id>
      <phase>test</phase>
      <goals>
          <goal>report</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

- prepare-agent 不可省略：如果只配置了 report 或 check 而没有 prepare-agent，JaCoCo 将找不到 jacoco.exec 数据文件，报告为空或 check 直接报错。prepare-agent 必须绑定在测试执行之前（默认 initialize 阶段即可）

> jacoco csv生成结果[解析脚本](https://gist.github.com/chutian0610/c591ca4515cfaa4c8a461f2ea1649b84)

#### 多模块项目的聚合

默认情况下，每个模块的 check 只检查自身代码。如果父 POM 需要基于所有子模块的汇总数据执行门禁，需要使用 report-aggregate goal 或确保 jacoco.exec 被正确合并。否则父 POM 上的 check 可能因缺少数据而误判。

```xml
<project>
    <artifactId>my-project-coverage</artifactId>
    <packaging>pom</packaging>

    <dependencies>
        <!-- 声明需要聚合的模块，report-aggregate 只会看到这里的直接依赖 -->
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>module-a</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>module-b</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.14</version>
                <executions>
                    <execution>
                        <id>report-aggregate</id>
                        <phase>verify</phase>
                        <goals>
                            <goal>report-aggregate</goal>
                        </goals>
                        <configuration>
                            <!-- 显式指定只生成 CSV（按需调整） -->
                            <formats>
                                <format>CSV</format>
                                <format>XML</format>
                            </formats>
                            <!-- 可选：将本聚合模块自身也纳入报告（默认 false） -->
                            <includeCurrentProject>false</includeCurrentProject>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

## 参考

- [1] [Apache Maven Dependency Plugin](https://maven.apache.org/plugins/maven-dependency-plugin/index.html)
- [2] [duplicate-finder-maven-plugin](https://basepom.github.io/duplicate-finder-maven-plugin/development/)

