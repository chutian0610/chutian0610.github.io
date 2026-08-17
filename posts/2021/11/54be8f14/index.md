# Java项目打包时HeapSpace OOM

## maven 打包时HeapSpace OOM

maven打包时出现HeapSpace OOM问题。由于Maven是Java启动的，显然我们只要修改maven进程的JVM配置就可以了。

在系统的环境变量中，设置MAVEN_OPTS，用以存放JVM的参数: 

```sh
export MAVEN_OPTS=-Xms256m -Xmx768m -XX:PermSize=128m -XX:MaxPermSize=256M
```

除了Maven主进程之外，单测插件surefire起的进程也可能出现OOM异常。surefire插件可以拉出几个JVM进程，以及每个进程的JVM配置是如何的，都是可以配置的(见[官方文档](http://maven.apache.org/surefire/maven-surefire-plugin/examples/fork-options-and-parallel-execution.html))。

```xml
<!--注意argLine配置可以用于传递JVM参数-->
<plugins>
[...]
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.0.0-M5</version>
    <configuration>
        <forkCount>3</forkCount>
        <reuseForks>true</reuseForks>
        <argLine>-Xmx1024m -XX:MaxPermSize=256m</argLine>
        <systemPropertyVariables>
            <databaseSchema>MY_TEST_SCHEMA_${surefire.forkNumber}</databaseSchema>
        </systemPropertyVariables>
        <workingDirectory>FORK_DIRECTORY_${surefire.forkNumber}</workingDirectory>
    </configuration>
  </plugin>
[...]
</plugins>
```

## gradle 打包时HeapSpace OOM

在gradle工程的gradle.properties里配置`org.gradle.jvmargs=-Xmx2000m -Xms500M -XX:+HeapDumpOnOutOfMemoryError`

