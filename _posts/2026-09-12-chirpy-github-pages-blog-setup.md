---
title: "Chirpy + GitHub Pages 搭建博客：从空仓库到第一篇博文"
date: 2026-09-12 15:40:00 +0800
categories: [博客搭建]
tags: [Chirpy, GitHub Pages, Jekyll, 博客]
---

博客搭建的成败，不在主题炫不炫，在第一篇文章的路径够不够短。本站用的是 Chirpy 主题 + GitHub Pages：免费、零服务器、推送即发布，从空仓库到第一篇博文用不了一天。这篇文章把这条路径完整走一遍，所有配置都对照本仓库的真实状态，照着做就能跑起来。

## 第一步：环境就绪，三条命令验完

Chirpy 是 Jekyll 主题，本地只需要 Ruby、Bundler 和 git。三个版本号都能打出来，就算就绪：

```bash
ruby -v
bundler -v
git --version
```

Ruby 版本建议 3.1+；Windows 用户可以看 Chirpy 官方文档的安装指引，macOS/Linux 一般直接可用。

## 第二步：拿 Starter，而不是从头拼主题

不要自己把主题源码拖进仓库慢慢配，直接用官方 Starter 模板。GitHub 网页上点 `Use this template`，或者命令行克隆：

```bash
git clone https://github.com/cotes2020/chirpy-starter.git testerdiary.github.io
cd testerdiary.github.io
```

仓库名必须是 `<username>.github.io`，这个命名决定线上访问地址。本仓库 `testerdiary.github.io` 对应 `https://testerdiary.github.io`。

## 第三步：先本地跑起来，再谈配置

克隆后第一件事不是改配置，是把网站在本地跑起来，确认依赖和环境是通的：

```bash
bundle install
bundle exec jekyll serve
```

浏览器打开 `http://localhost:4000`，看到主题默认首页就是成功。这一步排除掉"环境问题"和"配置问题"，后面改什么心里都有底。

## 第四步：_config.yml 只改这五处

`_config.yml` 是这个站的总开关。对照本仓库现状，最常改的就是五处：

```yaml
title: testerdiary
tagline: A tester's diary on software quality
lang: en
timezone: Asia/Shanghai
url: "https://testerdiary.github.io"
```

`url` 和 `timezone` 是最容易踩坑的两个：`url` 留空，Jekyll 生成的 SEO 标签、站点地图与站点内链接都会缺域名；`timezone` 留空，文章日期按 UTC 显示，比北京时间少 8 小时。github 下的 `username` 填你的 GitHub 用户名，social 部分按需改，不影响发布。

## 第五步：写第一篇文章，front matter 必须有

文章放在 `_posts/` 目录，文件名的日期就是发布时间，格式 `YYYY-MM-DD-短横线标题.md`。front matter 至少要有 `title`、`date`、`categories`、`tags`，否则文章不会出现在列表里：

```yaml
---
title: "我的第一篇文章"
date: 2026-09-12 10:00:00 +0800
categories: [测试相关]
tags: [测试, 博客]
---
```

date 里的 `+0800` 是时区偏移，和第四步设置的 `timezone` 配合，保证显示的是北京时间。categories 和 tags 是主题聚合页的依据，建议第一次就定好，避免后面改名导致分类目录分裂。

## 第六步：推送，剩下交给 GitHub Actions

Chirpy Starter 自带部署工作流 `.github/workflows/pages-deploy.yml`，本仓库也有这份文件。推送即发布，无需手动构建：

```bash
git add .
git commit -m "init: chirpy blog"
git push origin main
```

推完后到 GitHub 仓库的 `Actions` 页看部署任务，变绿后访问 `https://<username>.github.io`。如果 Actions 不可用，备用方案是把构建产物推到 `gh-pages` 分支，在 `Settings → Pages → Build and deployment` 里把 Source 指过去。用 Starter 推荐的方式则保持 `GitHub Actions` 不变。

## 踩过的坑，一张表记完

| 现象 | 原因 | 解法 |
| --- | --- | --- |
| 文章显示的时间比本地少 8 小时 | `timezone` 留空，按 UTC 显示 | `_config.yml` 设 `Asia/Shanghai` |
| 本地正常，线上链接没有域名前缀 | `url` 留空 | 填 `https://<username>.github.io` |
| 文章写了一堆，首页一篇都没有 | 缺 front matter 或文件名日期不是今天 | 补 front matter；文件名日期改为当天 |
| 头像不显示或变形 | 图片路径、比例不符 | 图片放 `assets/img/`，配置 `avatar: /assets/img/avatar.svg`，保持 1:1 |
| 分类/标签聚合页看起来少了一块 | 大小写不一致 | categories/tags 首次写入时定好大小写 |

最后一个建议：本地预览用 `bundle exec jekyll serve --livereload`，改 `_posts` 和 `_config.yml` 都是即时刷新，验证成本低到可以忽略。

## 结论：成功的标准是"十分钟能把文章发出去"

主题、字体、插件的折腾可以无限延长，但博客的及格线是另一件事：今天想写，十分钟内能发出去。本站的几篇文章全部走这条链路——写 `_posts` 里的 Markdown，推送，剩下交给 GitHub Actions。先把路径打通，再谈美化。
