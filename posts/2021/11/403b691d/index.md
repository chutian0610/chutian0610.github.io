# maven default http blocker问题

最近在maven打包的时候遇到一个问题:

```
Could not transfer artifact io.airlift:airbase:pom:128 from/to maven-default-http-blocker 
```

使用的maven版本是 3.8.4. 原因是maven自3.8.1升级后需要https协议的仓库地址，见 [Release Notes – Maven 3.8.1](https://maven.apache.org/docs/3.8.1/release-notes.html)。

有2种解决方案：

1. 修改maven远程仓库地址为支持https的地址。比如：添加ali的maven仓库地址。
2. 替换为3.8.1之前的maven版本，例如 3.6.3。

