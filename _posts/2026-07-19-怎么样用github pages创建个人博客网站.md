---
layout: post
title: 怎么样用 GitHub Pages 创建个人博客网站
subtitle: 零基础从选模板到上线，半小时搞定
date: 2026-07-19
tags: [tutorial, github-pages, jekyll, beginner]
author: babi
---

一直想搭一个个人博客，但不想买服务器、不想折腾数据库、不想写前后端代码？**GitHub Pages + Jekyll** 就是为你准备的方案——完全免费，纯静态，连构建部署都自动化。

这篇文章带你走完从零到上线的全过程。

---

## 什么是 GitHub Pages？

[GitHub Pages](https://pages.github.com/) 是 GitHub 提供的免费静态网站托管服务。你只需要把网站文件推送到仓库里，GitHub 会自动帮你托管到 `https://你的用户名.github.io` 这个域名上。

优点：

- **免费**——托管、HTTPS 证书、自定义域名全免费
- **无服务器维护**——不用管 Nginx、Apache 或任何后端
- **和 Git 深度集成**——写文章 = 提交代码，天然版本控制
- **支持 Jekyll 原生构建**——不需要在本地生成静态文件再上传

## 什么是 Jekyll？

[Jekyll](https://jekyllrb.com/) 是一个静态站点生成器，用 Ruby 编写。它把 Markdown（写作格式）和 Liquid 模板（页面布局）结合，生成完整的 HTML 网站。

关键概念：

| 概念 | 说明 |
|---|---|
| `_posts/` | 博客文章目录，用 `YYYY-MM-DD-标题.md` 命名 |
| `_layouts/` | 页面模板（首页、文章页、关于页等） |
| `_includes/` | 可复用的组件片段（导航栏、页脚等） |
| 前置元数据 (Front Matter) | 每篇 Markdown 文件开头的 `---` 之间的配置项 |

你只需要用 Markdown 写文章，Jekyll 自动处理剩下的。

---

## 第一步：找一个喜欢的模板

自己从零写一套网站样式费时费力，好在 Jekyll 社区有大量免费/付费模板。

推荐去 **[jekyllthemes.io](https://jekyllthemes.io/)** 挑选，上面有几百个模板，按分类、风格、功能筛选。

### 本次示例：Beautiful Jekyll

我们使用 [Beautiful Jekyll](https://jekyllthemes.io/theme/beautiful-jekyll) 作为示例——它也是这个博客正在使用的主题。

![](https://beautifuljekyll.com/assets/img/beautiful-jekyll-theme.png)

它的特点：

- **颜值高**——开箱即用就很好看，首页大图、响应式设计
- **配置丰富**——导航栏、社交链接、评论系统、搜索、Google Analytics 全在 `_config.yml` 里开关
- **GitHub Pages 一键部署**——不需要在本地装 Ruby 环境，直接在 GitHub 上操作
- **社区活跃**——4000+ Star，持续维护

> 你可以在 **[jekyllthemes.io](https://jekyllthemes.io/)** 上浏览更多模板，选择自己喜欢的风格。大部分模板都提供了类似的 GitHub 一键部署流程。

---

## 第二步：Fork 模板仓库

推荐的做法是直接 Fork（复制）模板仓库到你的 GitHub 账号：

1. 打开 [Beautiful Jekyll 的 GitHub 仓库](https://github.com/daattali/beautiful-jekyll)
2. 点击右上角的 **Fork** 按钮
3. 在 **Repository name** 处填 **`你的用户名.github.io`**（例如我的就是 `babi.github.io`）
4. 点击 **Create fork**

> ⚠️ 仓库名必须是 `你的用户名.github.io`，GitHub Pages 才会把这个仓库当作个人站点。如果想用自定义域名可以取其他名字，参考 [FAQ](https://beautifuljekyll.com/faq/#custom-domain)。

> ⚠️ 仓库名必须是 `你的用户名.github.io`，GitHub Pages 才会把这个仓库当作个人站点。

## 第三步：启用 GitHub Pages

1. 进入你刚创建的仓库 → **Settings** → **Pages**
2. 在 "Branch" 下拉选择 `master`（或 `main`），目录选 `/ (root)`
3. 点击 **Save**
4. 等一两分钟，访问 `https://你的用户名.github.io`，你的网站就上线了

## 第四步：配置网站信息

克隆仓库到本地（或者直接在 GitHub 网页上编辑）：

```bash
git clone https://github.com/你的用户名/你的用户名.github.io.git
cd 你的用户名.github.io
```

修改根目录下的 `_config.yml` 文件，这是网站的核心配置文件：

```yaml
title: 我的个人博客
author: 你的名字
avatar: /assets/img/avatar-icon.png

# 社交链接
social-network-links:
  github: 你的GitHub用户名
  twitter: 你的Twitter用户名

# 页脚显示
rss-description: 我的个人博客
```

每个配置项都有注释说明，按需修改即可。


## 第五步：写你的第一篇文章

在 `_posts/` 目录下新建文件，命名格式必须是 `YYYY-MM-DD-标题.md`：

```markdown
---
layout: post
title: Hello World
subtitle: 我的第一篇博客
date: 2026-07-19
tags: [hello, first]
---

这是我的第一篇博客文章！

用 **Markdown** 写作，支持粗体、*斜体*、`代码`、[链接](https://example.com) 等格式。

### 可以插入图片

![描述文字](/assets/img/某张图片.jpg)

### 可以插入代码

```python
print("Hello World")
```
```

提交推送，GitHub Actions 会自动构建并部署：

```bash
git add .
git commit -m "第一篇文章"
git push
```

---

## 自动化构建：GitHub Actions

当你把代码推送到 GitHub 后，GitHub Actions 会自动运行构建流程。Beautiful Jekyll 项目自带了 CI 配置 [`.github/workflows/ci.yml`](https://github.com/daattali/beautiful-jekyll/blob/master/.github/workflows/ci.yml)：


每次 push，GitHub Actions 都会：
1. 启动一个 Ubuntu 容器
2. 安装 Ruby 和 Jekyll 依赖
3. 执行 Jekyll 构建，生成 `_site/` 静态文件
4. 上传构建产物
5. GitHub Pages 自动把构建产物部署上线

所以你只需要专注于写 Markdown，剩下的全是自动化的。

---

## 绑定自定义域名（可选）

如果你有自己的域名，想用 `blog.com` 而不是 `你的用户名.github.io`：

1. 在仓库 **Settings** → **Pages** → **Custom domain** 里填入你的域名
2. 在你的域名 DNS 管理后台添加一条 **CNAME 记录**，指向 `你的用户名.github.io`
3. 在 `_config.yml` 里设置 `url: https://你的域名`
4. 提交后 GitHub 会自动为你的域名申请 HTTPS 证书

---

## 进阶功能

网站搭好后，还可以按需添加：

| 功能 | 配置方式 |
|---|---|
| **评论系统** | `_config.yml` 中配置 Disqus、Utterances 或 giscus |
| **搜索** | Beautiful Jekyll 自带搜索功能 |
| **Google Analytics** | 在 `_config.yml` 填入 GA 跟踪 ID |
| **数学公式** | 启用 MathJax 支持 LaTeX 公式 |
| **标签分类** | 文章 Front Matter 里加 `tags:`，自动生成标签索引 |
| **多语言** | 用 `_data/` 配合 Liquid 模板实现 |
| **自定义样式** | 修改 `assets/css/beautifuljekyll.css` 中的 CSS 变量 |

---

## 总结

从零到上线只需要：

1. **选模板** — [jekyllthemes.io](https://jekyllthemes.io/) 挑选喜欢的
2. **创建仓库** — Fork 模板仓库到你的账号并重命名为 `你的用户名.github.io`
3. **启用 Pages** — 在仓库 Settings → Pages 选择分支
4. **写文章** — `_posts/` 下加 Markdown 文件，推送即发布

整个流程不需要购买服务器、不需要配置数据库、不需要写一行后端代码。你唯一要做的事就是——**开始写**。

---

*这篇文章本身就是用 Jekyll + GitHub Pages 发布的。如果你能看到它，说明这套流程确实管用 😄*
