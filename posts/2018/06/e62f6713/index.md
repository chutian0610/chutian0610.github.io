# Server端图片导出

最近的工作需要在服务端生成报表图片，Java库生成的图片实在是惨不忍睹，酷炫的还是要看JS😂。解决方案是: 服务数据 + html模板 + headless浏览器。

<!--more-->

## highchart

如果你计划使用的前端报表是HighChart，比较好的消息是highchart提供了一个基于nodejs的图片导出工具[highcharts-export-server](https://github.com/highcharts/node-export-server)。

> HighChart 曾今提供过Java版本，但是已经不维护了

```sh
## 确保你已经安装了node.js
## npm 安装
$ npm install highcharts-export-server -g

## 源码安装
$ git clone https://github.com/highcharts/node-export-server
$ npm install
$ npm link
```

### 命令

```sh
## arguments : 参数
$ highcharts-export-server <arguments>
```

|option|说明|
|:---|:---|
|--infile|指定输入文件|
|--instr|指定输入为字符串|
|--options|别名 --instr|
|--outfile|指定输出文件名|
|--allowFileResources|允许从文件系统注入资源。作为服务器运行时无效。默认为true。|
|--type|导出文件的类型。有效的选项是jpg png pdf svg。|
|--scale|图表的比例。使用它可以提高PNG和JPG的分辨率，例如，在600像素的图表上将比例设置为2将导致1200像素的输出。|
|--width|缩放图表以适应提供的宽度，覆盖--scale。|
|--constr|要使用的构造函数。Chart，Map（要求服务器与地图支持安装），或StockChart。|
|--callback|在Highcharts的构造函数中调用的文件，包含JavaScript。|
|--resources|字符串化的JSON。|
|--batch "input.json=output.png;input2.json=output2.png;.."|批量转换|
|--logDest `<path>`|为日志文件设置路径，并启用文件日志记录|
|--logFile `<filename>`|设置日志文件的名称（不含路径）。默认为highcharts-export-server.log。请注意，--logDest还需要设置为启用文件记录。|
|--logLevel `<0..4>`|设置日志级别。0 =关，1 =错误，2 =警告，3 =通知，4 =详细|
|--fromFile "options.json"|从JSON文件读取CLI选项|
|--tmpdir|临时输出文件的路径。|
|--workers|工人数量|
|--workLimit|重新开始后台进程之前可以执行的工作|
|--listenToProcessExits|设置为0以跳过附加process.exit处理程序。请注意，禁用这可能会导致僵尸进程！|
|--globalOptions|带有选项的JSON字符串传递给Highcharts.setOptions|
|--enableServer `<1|0>`|启用服务器（在提供--host时也完成）|
|--host|运行服务器的主机名。|
|--port|侦听传入请求的端口。|
|--sslPath|SSL密钥/证书的路径。间接启用SSL支持。|
|--sslPort|运行HTTPS服务器的端口|
|--sslOnly|设置为true仅通过HTTPS提供服务|
|--rateLimit|参数是一分钟内允许的最大请求数。默认情况下禁用。|

* `-`和`--`在使用CLI时可以互换使用。
* 导出服务器将事件侦听器附加到process.exit。这是为了确保在应用程序终止时所有的后台进程都被正确关闭。侦听器还附加了未捕获的异常 - 如果出现，则整个池将被终止，并且应用程序终止。
* 如果--resources未设置，并且运行cli工具的文件夹中存在resources.json文件，它将使用该resources.json文件。

### 启动服务

```sh
## 服务器 模式
$ highcharts-export-server --enableServer 1
## 永久运行的最简单方法是git clone 导出服务器repo，然后在项目文件夹中运行。
$ forever start ./bin/cli.js --enableServer 1 --killSignal SIGINT
```

服务器接受以下参数：

|option|说明|
|:---|:---|
|infile|包含图表的JSON或SVG的字符串|
|options|别名 infile|
|svg|包含要呈现的SVG的字符串|
|type|格式：png，jpeg，pdf，svg。Mimetypes也可以使用。|
|scale|比例因子。使用它可以提高PNG和JPG的分辨率，例如，在600像素的图表上将比例设置为2将导致1200像素的输出。|
|width|图表宽度（覆盖比例）|
|callback|在highcharts构造函数中执行Javascript。|
|resources|其他资源。|
|constr|要使用的构造函数。无论是Chart或Stock。|
|b64|布尔，设置为true以获取base64而不是二进制。|
|async|获取下载链接而不是文件数据。|
|noDownload|Bool，设置为true，不在响应上发送附件头。|
|asyncRendering|highexp.done()在渲染图表前等待包含的脚本调用。|
|globalOptions|带有要传递给选项的JSON对象Highcharts.setOptions。|
|dataOptions|通过 Highcharts.data(..)|
|customCode|dataOptions提供时，这是一个函数，可以在应用数据选项后调用。它唯一的参数是在返回时将被传递给Highcharts构造函数的完整选项对象。|

> b64选项将覆盖该async选项

highchart 还提供了开放的http接口, 详见 [https://export.highcharts.com.cn/](https://export.highcharts.com.cn/)

## Echart

基于python的导出方案:

* [https://github.com/pyecharts/pyecharts](https://github.com/pyecharts/pyecharts)
* [https://github.com/pyecharts/pyecharts-snapshot](https://github.com/pyecharts/pyecharts-snapshot)

基于Java的导出方案:

* Echart结构类库[https://github.com/abel533/ECharts](https://github.com/abel533/ECharts)
* phantomjs-java-wrapper[https://github.com/moodysalem/java-phantomjs-wrapper](https://github.com/moodysalem/java-phantomjs-wrapper)

## Antv \(蚂蚁金服\)

Antv大致也是相似的思路: freemarker + data + phantomjs

## 总结

翻了翻项目代码，发现大体上还是我的思路使用无头浏览器绘制html页面。需要注意的是phantomjs 不再维护，chrome 目前已经支持[headless特性](https://chromium.googlesource.com/chromium/src/+/lkgr/headless/README.md)。 webdriver + chrome = java wrapper + phantomjs。


## 参考

- [1] [highcharts,export-module-overview](https://www.highcharts.com/docs/export-module/export-module-overview)
- [2] [highcharts,node-export-server](https://github.com/highcharts/node-export-server)
- [3] [chromedriver.chromium](http://chromedriver.chromium.org/)

