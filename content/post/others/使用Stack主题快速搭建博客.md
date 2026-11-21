---
title: "使用Stack主题快速搭建博客"
description: "记录从 CaiJimmy/hugo-theme-stack-starter 模板初始化仓库到修改配置的完整过程，并解释我实际改动过的每一项配置。"
date: 2026-11-21T14:49:48+08:00
lastmod: 2026-11-21T15:10:00+08:00
image: 
math: false
license: 
comments: false
draft: false
build:
    list: always    # 改为 "never" 可以把文章从列表中隐藏
tags: ["Hugo", "Stack", "博客搭建"]
categories: ["其他"]
---

# 前言

之前用 PaperMod 主题搭过一次博客，后来想换一个更精致的主题，于是选择了 [Hugo Theme Stack](https://github.com/CaiJimmy/hugo-theme-stack)。为了避免从零开始折腾，直接使用了作者提供的模板仓库：

- 模板：<https://github.com/CaiJimmy/hugo-theme-stack-starter>
- 主题：<https://github.com/CaiJimmy/hugo-theme-stack>
- 主题文档：<https://stack.cai.im/zh/>

这篇文章记录我的实际操作过程：先从模板初始化仓库，再修改配置，最后把这次改过的每一项配置的作用整理出来。

# 1. 初始化仓库

## 1.1 用模板创建远程仓库

打开模板仓库 <https://github.com/CaiJimmy/hugo-theme-stack-starter>，点击右上角的 **Use this template → Create a new repository**。

仓库名填写 `ostwind1.github.io`，可见性选择 **Public**。这样最终网站地址就是 `https://ostwind1.github.io`，不需要额外的子路径。

> 如果仓库名不是 `用户名.github.io`，网站会变成 `https://用户名.github.io/仓库名/`，此时 `baseurl` 也要跟着加上仓库名。

## 1.2 设置 GitHub Pages

进入仓库的 **Settings → Pages**，把 **Build and deployment → Source** 设置为 **GitHub Actions**。模板已经准备好了工作流，之后每次推送代码都会自动构建并发布。

## 1.3 克隆到本地

```bash
git clone https://github.com/ostwind1/ostwind1.github.io.git
cd ostwind1.github.io
```

这一步对应仓库里的第一次提交 `6ba0fde Initial commit`，内容就是模板的原始文件。

## 1.4 本地预览

```bash
hugo server -D
```

打开 <http://localhost:1313> 就能看到站点。`-D` 表示把草稿（`draft: true`）也渲染出来。

> 这个模板使用 **Hugo Modules** 管理主题，所以本机除了 Hugo extended 之外，还需要安装 **Go**。如果只装了 Hugo，运行时会报 `binary with name "go" not found in PATH`。不想折腾环境的话，可以直接用仓库自带的 devcontainer 或 GitHub Codespaces。

# 2. 项目结构

模板的目录不多，和后面修改配置相关的主要是这几个：

```text
.
├── .github/workflows/      # GitHub Actions 工作流
│   ├── deploy.yml          # 推送后自动构建并部署到 GitHub Pages
│   └── update-theme.yml    # 每天自动更新主题
├── assets/
│   ├── img/                # 头像、favicon 等图片资源
│   └── scss/custom.scss    # 自定义样式
├── config/_default/        # 站点配置，按功能拆成多个 TOML 文件
├── content/                # 文章和页面
├── go.mod / go.sum         # 主题依赖及版本锁定
├── public/                 # 构建产物（已被 .gitignore 忽略）
└── resources/              # Hugo 资源缓存（已被 .gitignore 忽略）
```

`config/_default/` 下的配置会由 Hugo 自动合并，拆成多个文件只是为了好维护。下面只记录我实际改过的内容，模板自带的其它配置保持原样。

# 3. 我的配置修改记录

这部分对应提交 `f29e04d 修改配置和自定义展示项`。

## 3.1 config.toml：站点基础信息

修改内容：

```diff
-baseurl = "https://starter.stack.cai.im/"
-locale  = "en-us"
-title   = "Hugo Theme Stack Starter"
+baseurl = "https://ostwind1.github.io"
+locale  = "zh-cn"
+title   = "朱林的博客"

-defaultContentLanguage = "en"
+defaultContentLanguage = "zh"

-hasCJKLanguage = false
+hasCJKLanguage = true
```

每一项的作用：

| 配置 | 修改为 | 作用 |
| --- | --- | --- |
| `baseurl` | `https://ostwind1.github.io` | 站点根地址。生成绝对链接、RSS、sitemap 时都会用到。模板默认是主题作者的演示地址，不改的话样式和图片可能加载失败。 |
| `locale` | `zh-cn` | 站点区域设置，影响日期等本地化格式，也会影响页面的语言标记。 |
| `title` | `朱林的博客` | 站点标题，显示在浏览器标签页和侧边栏。 |
| `defaultContentLanguage` | `zh` | 默认内容语言。主题会根据它去 `i18n/` 目录找对应的翻译文件（这里是 `zh.toml`），决定“阅读更多”“目录”等界面文字。 |
| `hasCJKLanguage` | `true` | 开启中日韩文字处理。开启后 `.Summary` 和 `.WordCount` 才能正确按字统计中文，否则中文摘要和字数会算错。 |

## 3.2 markup.toml：让目录包含一级标题

修改内容：

```diff
 [tableOfContents]
     endLevel   = 4
     ordered    = true
-    startLevel = 2
+    startLevel = 1
```

`tableOfContents` 控制文章目录抓取哪些标题：

- `startLevel`：从几级标题开始收集。模板是 `2`，只收集 `##` 及以下的标题；我改成 `1`，这样文章开头的 `#` 一级标题也会出现在目录里。
- `endLevel`：收集到几级标题，这里保持 `4`，即最多到 `####`。
- `ordered`：目录使用有序列表，保持 `true` 不变。

## 3.3 menu.toml：去掉模板自带的社交链接

模板原本在侧边栏底部放了主题作者的 GitHub 和 Twitter 链接：

```toml
[[social]]
    identifier = "github"
    name       = "GitHub"
    url        = "https://github.com/CaiJimmy/hugo-theme-stack"

    [social.params]
        icon = "brand-github"

[[social]]
    identifier = "twitter"
    name       = "Twitter"
    url        = "https://twitter.com"

    [social.params]
        icon = "brand-twitter"
```

我把它们注释掉了：

```toml
#[[social]]
#    identifier = "github"
#    name       = "GitHub"
#    url        = "https://github.com/"
#
#    [social.params]
#        icon = "brand-github"
```

这样侧边栏就不会出现指向主题作者和 Twitter 首页的链接。以后想加自己的 GitHub、邮箱、RSS 等链接，把注释取消并改成自己的地址即可。

> 主菜单不在这里配置，而是写在各页面的 front matter 里，见 3.6 节。

## 3.4 params.toml：修改侧边栏简介

修改内容：

```diff
 [sidebar]
     emoji    = "🍥"
-    subtitle = "Lorem ipsum dolor sit amet, consectetur adipiscing elit."
+    subtitle = "世界上只有一种真正的英雄主义，就是在认清生活的真相之后依然热爱生活"
     avatar   = "img/avatar.png"
```

`sidebar.subtitle` 是侧边栏头像下方的一句话简介。模板里是一段拉丁文占位文本，我换成了罗曼·罗兰的这句话。`emoji` 和 `avatar` 的配置没有改，只是替换了对应的图片文件。

## 3.5 替换头像和站点图标

这次提交还替换了 `assets/img/` 下的两张图片：

| 文件 | 作用 |
| --- | --- |
| `assets/img/avatar.png` | 侧边栏头像，路径由 `params.toml` 的 `sidebar.avatar` 指定。 |
| `assets/img/favicon.png` | 浏览器标签页图标，路径由 `params.toml` 的 `favicon` 指定。 |

两个路径都是相对于 `assets/` 目录的，所以只替换文件即可，不需要改配置。

## 3.6 页面标题与导航中文化

Stack 的主菜单可以直接写在页面 front matter 的 `menu.main` 里。`weight` 控制顺序，数字越小越靠前；`params.icon` 是图标名，使用 Tabler Icons。

我修改了四个页面：

| 文件 | 修改 | 效果 |
| --- | --- | --- |
| `content/_index.md` | `name: Home` → `name: 首页` | 导航第一项显示“首页”。 |
| `content/page/archives/index.md` | `title: "Archives"` → `title: "文章"` | 归档页标题和导航文字变成“文章”。 |
| `content/page/search/index.md` | `title: "Search"` → `title: "搜索"` | 搜索页标题和导航文字变成“搜索”。 |
| `content/page/links/index.md` | 注释掉 `menu` 部分 | 友链页仍然存在，但暂时不显示在导航里。 |

以首页为例：

```yaml
---
menu:
    main:
        name: 首页
        weight: 1
        params:
            icon: home
---
```

归档页和搜索页的写法类似，分别使用 `archives` 和 `search` 图标，`weight` 分别是 2 和 3。最终导航顺序就是：首页 → 文章 → 搜索。

友链页原本也注册在导航第四位，我把 `menu` 部分注释掉了：

```yaml
# menu:
#     main: 
#         weight: 4
#         params:
#             icon: link
```

页面本身还在，直接访问 `/links/` 仍然可以打开，只是顶部导航不再显示。

# 4. 修改后的检查

改完配置后，重新启动本地预览：

```bash
hugo server -D
```

主要检查了这几项：

1. 浏览器标签页标题是否变成“朱林的博客”；
2. 导航是否按“首页 → 文章 → 搜索”的顺序显示；
3. 侧边栏头像、简介和站点图标是否生效；
4. 文章目录是否包含一级标题；
5. 中文摘要和字数是否正常。

确认无误后，再生成一次静态文件：

```bash
hugo
```

构建产物会输出到 `public/`，这个目录不会提交到仓库。

# 5. 提交与自动部署

修改完成后提交并推送：

```bash
git add .
git commit -m "修改配置和自定义展示项"
git push
```

这次提交就是 `f29e04d`。推送后，`.github/workflows/deploy.yml` 会自动触发，完成安装依赖、构建和发布，几分钟后就能在 `https://ostwind1.github.io` 看到修改后的站点。

# 6. 新建文章

环境准备好之后，新建文章只需要一条命令：

```bash
hugo new content post/others/使用Stack主题快速搭建博客.md
```

Hugo 会使用主题自带的文章模板，在 `content/post/others/` 下生成 Markdown 文件。写完内容后把 front matter 里的 `draft` 改成 `false`，再提交推送即可。

# 7. 参考链接

- Stack 主题仓库：<https://github.com/CaiJimmy/hugo-theme-stack>
- Stack 模板仓库：<https://github.com/CaiJimmy/hugo-theme-stack-starter>
- Stack 中文文档：<https://stack.cai.im/zh/>
- Hugo 官方文档：<https://gohugo.io/documentation/>
- Hugo Modules：<https://gohugo.io/hugo-modules/>
- GitHub Pages 文档：<https://docs.github.com/pages>

以上就是我从模板初始化仓库到完成配置修改的完整记录。整体感受是：Stack 模板已经把 GitHub Actions、主题依赖、示例页面都准备好了，真正需要改的只有 `config/_default/` 下的几个配置项和页面 front matter，剩下的就是专心写文章。
