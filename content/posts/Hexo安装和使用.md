---
title: Hexo安装和使用
date: 2026-10-12 10:00:00
tags:
  - Hexo
description: 从零开始安装 Hexo,配置 NexT 主题,并整理常用命令与常见问题。
---

Hexo 是一个基于 Node.js 的快速、简洁的静态博客框架,支持 Markdown 写作,可以一键部署到 GitHub Pages、Vercel 等平台。本文介绍 Hexo 的安装流程、NexT 主题的安装与配置,以及日常使用的常用命令。

<!-- more -->

## 一、环境准备

Hexo 依赖 **Node.js** 和 **Git**,请先安装。

### 1. 安装 Node.js

前往 [Node.js 官网](https://nodejs.org/) 下载 **LTS(长期支持)版本**并安装。

### 2. 安装 Git

前往 [Git 官网](https://git-scm.com/) 下载安装,macOS 与 Linux 通常已自带。

### 3. 验证安装

```bash
node -v
npm -v
git --version
```

三条命令都能输出版本号,说明环境就绪。

> 国内网络下 npm 下载较慢,可以切换镜像:
>
> ```bash
> npm config set registry https://registry.npmmirror.com
> ```

## 二、安装 Hexo

### 1. 全局安装 hexo-cli

```bash
npm install -g hexo-cli
```

验证:

```bash
hexo -v
```

### 2. 初始化博客

```bash
hexo init myblog
cd myblog
npm install
```

- `myblog` 是博客目录名,可以自定义。
- 初始化完成后,目录结构如下:

```text
myblog/
├── _config.yml      # 站点配置文件
├── package.json     # 依赖信息
├── scaffolds/       # 文章模板
├── source/          # 资源文件夹
│   ├── _drafts/     # 草稿
│   └── _posts/      # 文章
└── themes/          # 主题文件夹
```

### 3. 本地预览

```bash
hexo server
```

浏览器访问 `http://localhost:4000`,看到默认页面即安装成功。按 `Ctrl + C` 停止服务。

## 三、安装 NexT 主题

NexT 是 Hexo 最流行的主题之一,简洁、文档完善。以下介绍两种安装方式,**推荐方式一**。

### 方式一:npm 安装(推荐)

在博客根目录执行:

```bash
npm install hexo-theme-next
```

**优点**:升级方便,配置与主题源码分离,不会因升级覆盖自己的设置。

**1)将主题配置文件复制到根目录**

```bash
# macOS / Linux
cp node_modules/hexo-theme-next/_config.yml _config.next.yml

# Windows (PowerShell)
copy node_modules\hexo-theme-next\_config.yml _config.next.yml
```

以后所有 NexT 的设置都写在根目录的 `_config.next.yml` 中,它的优先级高于主题自带的配置。

**2)在站点配置中启用主题**

编辑根目录 `_config.yml`:

```yaml
theme: next
```

### 方式二:Git 克隆安装

```bash
git clone https://github.com/next-theme/hexo-theme-next themes/next
```

然后同样在 `_config.yml` 中设置 `theme: next`。主题配置则直接修改 `themes/next/_config.yml`。

> 这种方式升级需要手动 `git pull`,且修改主题配置容易产生冲突,不如 npm 方式省心。

### 验证主题

```bash
hexo clean
hexo server
```

刷新 `http://localhost:4000`,页面变为 NexT 风格即安装成功。

## 四、NexT 基础配置

### 1. 站点基本信息(`_config.yml`)

```yaml
title: 我的博客
subtitle: '副标题'
description: '站点描述'
author: 你的名字
language: zh-CN
timezone: Asia/Shanghai
```

### 2. 选择主题风格(`_config.next.yml`)

NexT 提供四种风格,取消想要的那一行注释即可:

```yaml
scheme: Muse
# scheme: Mist
# scheme: Pisces
# scheme: Gemini
```

### 3. 开启深色模式

```yaml
darkmode: true
```

### 4. 配置菜单

```yaml
menu:
  home: / || fa fa-home
  about: /about/ || fa fa-user
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  archives: /archives/ || fa fa-archive
```

菜单对应的页面需要手动创建(archives 自动生成):

```bash
hexo new page about
hexo new page tags
hexo new page categories
```

分别编辑 `source/tags/index.md` 和 `source/categories/index.md`,加上 `type`:

```yaml
---
title: 标签
type: tags
---
```

```yaml
---
title: 分类
type: categories
---
```

### 5. 开启本地搜索

```bash
npm install hexo-generator-searchdb --save
```

根目录 `_config.yml` 末尾添加:

```yaml
search:
  path: search.xml
  field: post
  content: true
  format: html
```

`_config.next.yml` 中开启:

```yaml
local_search:
  enable: true
```

### 6. 显示阅读量(不蒜子)

```yaml
busuanzi_count:
  enable: true
  total_visitors: true
  total_views: true
  post_views: true
```

> 本地预览时数据可能不准,部署到线上后才会正常显示。

## 五、Hexo 常用命令

### 基础命令

| 命令 | 简写 | 说明 |
| --- | --- | --- |
| `hexo init [folder]` | | 初始化博客 |
| `hexo new "标题"` | `hexo n` | 新建文章 |
| `hexo new page 名称` | | 新建页面 |
| `hexo new draft "标题"` | | 新建草稿 |
| `hexo publish "标题"` | `hexo p` | 将草稿发布为正式文章 |
| `hexo generate` | `hexo g` | 生成静态文件 |
| `hexo server` | `hexo s` | 启动本地服务器 |
| `hexo deploy` | `hexo d` | 部署网站 |
| `hexo clean` | | 清除缓存和已生成的文件 |
| `hexo list <type>` | | 列出站点信息(如 post、page、tag) |
| `hexo version` | `hexo -v` | 查看版本 |

### 常用组合

```bash
# 清理后重新生成并本地预览
hexo clean && hexo g && hexo s

# 生成并部署
hexo clean && hexo g && hexo d

# 生成后立即部署
hexo g -d

# 指定端口启动(默认 4000)
hexo s -p 5000

# 预览草稿
hexo s --draft
```

### 新建文章示例

```bash
hexo new "我的第一篇文章"
```

生成的 `source/_posts/我的第一篇文章.md` 中,在 front-matter 里设置标签和分类:

```yaml
---
title: 我的第一篇文章
date: 2026-10-12 10:00:00
tags:
  - Hexo
  - 教程
categories:
  - 技术
---
```

在正文中插入 `<!-- more -->`,可以让首页只显示摘要。

## 六、部署到 GitHub Pages(简要)

**1)安装部署插件**

```bash
npm install hexo-deployer-git --save
```

**2)配置 `_config.yml`**

```yaml
url: https://你的用户名.github.io

deploy:
  type: git
  repo: git@github.com:你的用户名/你的用户名.github.io.git
  branch: main
```

**3)部署**

```bash
hexo clean && hexo g && hexo d
```

## 七、常见问题

**1. 修改配置后不生效**

执行 `hexo clean`,再重新 `hexo g` 或 `hexo s`。

**2. 启动时报 YAML 解析错误**

检查配置文件:冒号后必须有空格,缩进使用空格而不是 Tab。

**3. 主题设置没有效果**

确认修改的是正确的文件:站点设置改 `_config.yml`,NexT 设置改 `_config.next.yml`。

**4. 端口被占用**

```bash
hexo s -p 5000
```

**5. 部署后样式丢失**

检查 `_config.yml` 中的 `url` 和 `root` 是否与实际访问地址一致。部署在子目录时,`root` 需要写成 `/子目录名/`。

**6. 升级 NexT 主题(npm 方式)**

```bash
npm update hexo-theme-next
```

升级后对比新版 `node_modules/hexo-theme-next/_config.yml`,查看是否有新增配置项需要同步到 `_config.next.yml`。

## 总结

1. 安装 **Node.js** 和 **Git**。
2. `npm install -g hexo-cli` 安装 Hexo,`hexo init` 初始化博客。
3. `npm install hexo-theme-next` 安装主题,并在 `_config.yml` 中设置 `theme: next`。
4. 日常写作只需记住:`hexo new` 写文章,`hexo clean && hexo g && hexo s` 预览,`hexo d` 部署。

按以上步骤操作,一个带有 NexT 主题的 Hexo 博客就搭建完成了。
