# 将Blog从Hexo迁移至Hugo

最近完成了博客从 Hexo 到 Hugo 的迁移，这篇文章记录了迁移的完整历程，包括技术选型、具体实施步骤、遇到的问题以及解决方案。

<!--more-->

## Hexo的问题

博客自 2016 年开始使用 Hexo + NexT 主题搭建，在使用过程中，一些问题逐渐显现：

- 升级问题: Node.js 版本升级后，某些 Hexo 插件不再兼容。每次升级 Node.js 都需要花时间寻找替代方案或降级 Node 版本。
- 性能瓶颈: 随着文章数量增加到 200+ 篇，每次构建都需要几分钟。

经过几种静态站点生成器的比较，最终选择迁移到 Hugo 上。

- 性能好: 构建速度快。
- 扩展多: 插件和主题丰富。
- 学习成本低: 活跃的社区论坛。

## 迁移过程

### 项目初始化

```sh
# 安装 hugo(mac)
$ brew install hugo

# 创建 hugo 项目
$ hugo new site blog
$ cd blog

# 安装主题
$ git init
$ git submodule add https://github.com/HEIGE-PCloud/DoIt.git themes/DoIt
```

### 内容迁移

Hexo 中的内容都在`./source`目录下

```sh
.
├── _posts
│   ├── 2016-06-16-Memory-of-C
│   │   ├── memorychart.webp
│   │   ├── stackmemory2.webp
│   │   ├── stackmemory31.webp
│   │   ├── stackmemory4.webp
│   │   └── stackvsheap.webp
│   ├── 2016-06-16-Memory-of-C.md
│   ├── ... ...
│   └── 2026-01-19-JavaCC-Token-Manager
└── images
    ├── logo.png
    └── logo.webp
```

Hugo的内容都在`./content`目录,公共资源可以放在 `./static` 中:

```sh
.
├── content
│   ├── about
│   │   └── index.md
│   └── posts
│       └── 2016-06-16-Memory-of-C
│           ├── memorychart.webp
│           ├── stackmemory2.webp
│           ├── stackmemory31.webp
│           ├── stackmemory4.webp
│           └── stackvsheap.webp
└── static
    └── images
        ├── logo.png
        └── logo.webp
```

### 永久链接

之前使用 Hexo 的时候，用的是 hexo-abbrlink 插件来处理永久链接，而换到了 Hugo 之后,需要一种生成永久链接的方案。Hugo 在永久链接中支持参数：slug。简单来说，我们可以针对每一篇文章指定一个 slug，然后在 config.toml 中配置 permalinks 包含 slug 参数，就可以生成唯一的永久链接。我们的目的就是对每篇文章自动生成一个 slug。

修改 archetypes/default.md 添加如下一行：

```md
---
#...
slug: {{ substr (md5 (printf "%s%s" .Date (replace .TranslationBaseName "-" " " | title))) 4 8 }}
#...
---
```

这样在每次使用 hugo new 的时候就会自动填写一个永久链接了。然后修改 config.toml:

```toml
[permalinks]
  post = "/post/:slug"
```

### mermaid 升级

`themes/DoIt/layouts/_partials/assets.html` 文件中指定了 mermaid 的版本。

```git
-   import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
+   import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
```

### algolia 集成

在我们本地 blog 站点的根目录下执行以下命令：

```sh
npm init   
npm install atomic-algolia --save 
```

执行完后会生成node_modules 文件夹和 package.json文件。，打开该文件，在`scripts:{}`中添加"algolia": "atomic-algolia"后结果如下：

```json
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "algolia": "atomic-algolia"
  },
```

在 blog 项目根目录下建立一个.env文件，文件内容如下:

```env
ALGOLIA_APP_ID=Your Application ID   
ALGOLIA_INDEX_NAME= Your Index Name   
ALGOLIA_INDEX_FILE=public/index.json   
ALGOLIA_ADMIN_KEY= Your Admin API Key
```

在 blog 项目根目录下执行下面的命令便可将索引文件自动上传至更新 algolia。

```sh
hugo
npm run algolia
```

