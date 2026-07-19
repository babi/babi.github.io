# 项目文件说明

本项目是一个基于 **Beautiful Jekyll** 主题的个人网站/博客，使用 [Jekyll](https://jekyllrb.com/) 静态站点生成器构建，可部署在 GitHub Pages 上。

---

## 根目录文件

### `index.html`
网站首页。使用 `layout: home` 布局，包含网站标题和副标题配置。Jekyll 会将其渲染为站点的主页（`/`）。

### `404.html`
自定义 404 错误页面。当访问不存在的页面时显示，包含友好的错误提示信息和一张趣味图片（`assets/img/404-southpark.jpg`）。使用 `layout: default`。

### `aboutme.md`
"关于我"页面内容（Markdown 格式）。使用 `layout: page`，展示网站所有者的个人信息描述。这是用户可以自定义填写的内容页面。

### `feed.xml`
RSS/Atom 订阅源模板。使用 Liquid 模板语法生成 XML 格式的订阅源，列出最新 20 篇文章，包含标题、作者、摘要和发布日期等信息。访问 `/feed.xml` 可获取。

### `tags.html`
标签索引页面。自动收集所有博客文章的标签，生成带标签计数和对应文章列表的索引页面。每个标签可点击跳转到对应位置，文章按标签分组展示。

### `beautiful-jekyll-theme.gemspec`
Ruby Gem 规范文件。定义 `beautiful-jekyll-theme` 这个 gem 的元数据（名称、版本号 `6.0.1`、作者、依赖项等）。依赖包括 `jekyll`、`jekyll-paginate`、`jekyll-sitemap`、`kramdown-parser-gfm`、`webrick` 等。

### `Gemfile`
Ruby 依赖管理文件。声明项目的 gem 依赖，通过 `gemspec` 引入主 gem 的依赖，并包含 Windows 平台特定的时区和文件监控优化（`tzinfo`、`wdm`）。

### `Appraisals`
多版本依赖测试配置。使用 [Appraisal](https://github.com/thoughtbot/appraisal) 工具，定义了两组测试环境：`jekyll-3`（Jekyll 3.9.4）和 `jekyll-4`（Jekyll 4.3.3），确保主题在多个 Jekyll 主版本下兼容。

### `CHANGELOG.md`
版本更新日志。记录每个版本的新功能、Breaking Changes、Bug 修复等信息。当前对应未发布的最新版本（v6.0.1 之后）。

### `README.md`
项目说明文档。详细介绍了 Beautiful Jekyll 主题的功能特性、安装步骤、配置方法和使用指南。是项目的主要文档入口。

### `LICENSE`
MIT 开源许可证。版权归 Dean Attali (2023) 所有，允许自由使用、修改和分发。

### `staticman.yml`
[Staticman](https://staticman.net/) 评论系统配置文件。定义评论表单的字段（name, email, url, message）、数据存储路径、验证规则、reCAPTCHA 和 Akismet 反垃圾邮件配置等。评论数据以 YAML 格式存储在 `_data/comments/` 目录下。

### `screenshot.png`
主题的预览截图图片，在 README 等地方使用。

---

## `assets/` 目录

### `assets/css/beautifuljekyll.css`
**主样式表**。项目的核心 CSS 文件，包含：
- 通过 Liquid 变量暴露可配置的 CSS 自定义属性（页面颜色、文字颜色、导航栏颜色、页脚颜色等）
- 基础样式（字体、布局、排版）
- 导航栏样式
- 页面头部大图样式
- 文章列表样式
- 页脚样式
- 社交图标样式
- 响应式设计
- 暗色模式支持
- 打印样式

### `assets/css/beautifuljekyll-minimal.css`
极简样式——用于简化布局的页脚。实现了一个固定在底部的页脚栏，内容少时也保持在视图底部。

### `assets/css/bootstrap-social.css`
社交网络按钮样式（第三方库）。为 Bootstrap 框架提供带图标的社交按钮样式，支持不同尺寸（lg, sm, xs），用于在网站上显示社交媒体链接按钮。

### `assets/css/pygment_highlights.css`
代码语法高亮样式（Pygments）。定义代码块中各种语法元素（关键字、注释、字符串、函数等）的颜色和样式。

### `assets/css/staticman.css`
Staticman 评论系统样式。定义评论表单、评论列表、头像、时间戳等元素的样式。

### `assets/js/beautifuljekyll.js`
**主 JavaScript 文件**。功能包括：
- 导航栏滚动效果（下滑后缩短导航栏）
- 移动端菜单交互（展开时隐藏头像，点击菜单项后自动折叠）
- 大图加载效果
- 站点搜索功能
- 响应式行为控制

### `assets/js/staticman.js`
Staticman 评论系统 JavaScript。处理评论表单的异步提交，通过 AJAX 将评论数据发送到 Staticman API，并在提交成功后更新 UI。

### `assets/data/searchcorpus.json`
搜索索引数据（Jekyll 生成）。包含所有文章的标题、描述、分类和 URL，用于站内搜索功能。这是一个 Liquid 模板文件，构建时由 Jekyll 动态生成 JSON 数据。

### `assets/img/`
图片资源目录，包含：
- `404-southpark.jpg` — 404 页面趣味图片
- `avatar-icon.png` — 头像图标
- `bgimage.png` — 页面背景图
- `crepe.jpg` — 示例/内容图片
- `path.jpg` — 示例/内容图片
- `thumb.png` — 缩略图

---

## `.github/` 目录（GitHub 相关配置）

### `.github/FUNDING.yml`
GitHub 赞助配置。链接到项目作者 Dean Attali 的 GitHub Sponsors 和 Patreon 页面。

### `.github/issue_template.md`
Issue 模板。引导用户提交 Issue 时遵循规范，建议赞助用户使用 Office Hours 获取帮助。

### `.github/pull_request_template.md`
Pull Request 模板。提醒贡献者区分自己的网站修改和主题修改，鼓励描述 PR 的内容。

### `.github/workflows/ci.yml`
CI 工作流配置（GitHub Actions）。在每次 push 和 pull request 时自动运行，流程包括：
1. 检出代码
2. 设置 Ruby 3.3 环境
3. 安装依赖（`bundle install` 和 `bundle exec appraisal install`）
4. 配置 GitHub Pages 基础路径
5. 在多版本 Jekyll（通过 Appraisal）下构建站点
6. 上传构建产物

---

## 配置文件（根目录）

### `.gitattributes`
Git 属性配置。定义项目中各类文件的换行符策略（文本文件使用 LF，多媒体文件标记为 binary）和 EOL 标准化规则。

### `.gitignore`
Git 忽略规则。排除 Jekyll 构建输出（`_site`）、Sass 缓存（`.sass-cache`）、系统文件（`.DS_Store`、`Thumbs.db`）、Gemfile.lock 和构建的 gem 包等。
