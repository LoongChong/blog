---
title: 从零搭建 Hexo + Butterfly 博客：我踩过的 5 个坑
date: 2026-09-04 15:30:00
updated:
categories:
  - 建站日志
tags:
  - Hexo
  - Butterfly
  - 踩坑
cover: /img/default-cover.jpg
description: 用 Hexo 8 + Butterfly 5.7 搭技术博客的完整记录，包含 5 个真实卡住我的问题：runtime_date 类型错误、root 子路径部署、4.x 遗留配置失效、图片 404、GitHub Actions 权限。
keywords: Hexo, Butterfly, 博客搭建, GitHub Pages, GitHub Actions
top_img:
comments: true
toc: true
mathjax: false
copyright: true
---

<!-- more -->

## 背景

想搭一个自己的技术博客，参考了 [Guo12181 的博客](https://guo12181.github.io/)，确认是 Hexo + Butterfly 之后就照着搭了。过程比想象中曲折，大部分时间花在排查配置错误上。

这篇文章记录真实踩到的 5 个坑，希望能帮你少花两小时。

当前版本组合：**Hexo 8.1.2 + Butterfly 5.7.0 + Node 22**。

## 坑 1：runtime_date 写成对象，构建直接报错

这是最要命的一个，构建直接崩：

```
ERROR RangeError: ...card_webinfo.pug:14
  .item-count#runtimeshow(data-publishDate=date_xml(theme.aside.card_webinfo.runtime_date))
Invalid time value
```

我最初按老教程写成这样：

```yaml
aside:
  card_webinfo:
    runtime_date:
      enable: true
      date: 2026/09/04 15:00:00
```

但 Butterfly 5.7 的模板是这样判断的：

```pug
if theme.aside.card_webinfo.runtime_date
  .item-count#runtimeshow(data-publishDate=date_xml(theme.aside.card_webinfo.runtime_date))
```

`runtime_date` 是个对象 → 布尔判断为真 → 传给 `date_xml()` 一个非日期值 → 抛异常。**整个侧栏渲染失败**。

正确写法是**字符串**：

```yaml
aside:
  card_webinfo:
    enable: true
    post_count: true
    last_push_date: true
    runtime_date: '2026/09/04 00:00:00'
```

留空就是关闭这个功能。

## 坑 2：Butterfly 4.x 的配置键在 5.x 已经失效

网上大量教程是针对 Butterfly 4.x 的，直接抄会出现一堆**不报错但完全不起作用**的配置。我中招的几处：

| 我写的（4.x 风格） | 5.7 的正确写法 |
| --- | --- |
| `aside.card_webinfo.wordcount: true` | 顶层 `wordcount.enable: true` |
| `aside.card_webinfo.website_running_time: true` | `aside.card_webinfo.runtime_date: '日期'` |
| `highlight.theme: mac-dark` | `code_blocks.theme: darker` |
| `runtime.date: 2026/09/01` | `aside.card_webinfo.runtime_date` |

排查方法很直接 —— 拿你的配置和主题默认配置对比：

```bash
# 主题的完整默认配置
cat node_modules/hexo-theme-butterfly/_config.yml
```

**凡是默认配置里没有的键，基本都是无效的。** 我的习惯是改动前先在这个文件里搜一遍键名。

## 坑 3：GitHub Pages 子路径部署，root 必须写对

我的 `LoongChong.github.io` 已经被作品集网站占用了，而**每个 GitHub 账号只能有一个 `<用户名>.github.io` 仓库**。所以博客只能走子路径：

```yaml
# _config.yml
url: https://LoongChong.github.io/blog
root: /blog/
```

`root` 写错的表现很有迷惑性：页面能打开，但 CSS 和 JS 全部 404，看起来像一堆没排版的纯文本。

对应的 GitHub Actions 里也要指定子目录：

```yaml
- uses: peaceiris/actions-gh-pages@v4
  with:
    external_repository: LoongChong/LoongChong.github.io
    destination_dir: blog
    publish_dir: ./public
```

这样博客写在 `/blog/`，和作品集共存，互不覆盖。

## 坑 4：默认深色模式，start 不能填 0

Butterfly 的深色模式逻辑在 `scripts/helpers/inject_head_js.js` 里：

```js
const start = darkmode.start || 6
const end = darkmode.end || 18
const isNight = start < end
  ? hour < start || hour >= end
  : hour >= start || hour < end
```

我想让首次访问默认深色。直觉是 `start: 0, end: 0`，但 **`0 || 6` 的结果是 6**，0 会被当成假值吃掉。

可行解是 `start: 1, end: 1`：

- `start < end` 为假 → 走 `hour >= start || hour < end`
- `hour >= 1 || hour < 1` 对 0~23 每一个小时都成立
- 结果：恒为夜间，即默认深色

```yaml
darkmode:
  enable: true
  button: true
  autoChangeMode: 2
  start: 1
  end: 1
```

访客手动切换后会写入 localStorage，之后以访客的选择为准，切换按钮照常可用。

## 坑 5：图片统一放 source/img/

一开始想用 `post_asset_folder: true`（每篇文章一个同名文件夹放图），但 `hexo-asset-image` 插件在 Hexo 5+ 上有兼容问题，图片经常 404。

最省事的方案：所有图片丢进 `source/img/`，文章里用绝对路径引用：

```markdown
![架构图](/img/rag-architecture.png)
```

封面图同理：

```yaml
cover: /img/rag-architecture.png
```

好处是路径简单、文章改名不会失效、CI 构建也不会丢文件。

另外注意：`favicon` 默认值是 `/img/favicon.png`，但主题自带的只有 `favicon.ico`。**不自己放一个 favicon.png 的话，图标就是 404。**

## 最终的目录结构

```
blog/
├── _config.yml              # 站点配置
├── _config.butterfly.yml    # 主题配置（覆盖式，升级主题不丢）
├── scaffolds/post.md        # 文章模板
├── source/
│   ├── _posts/              # 文章
│   ├── img/                 # 全部图片
│   ├── about/
│   ├── categories/
│   ├── tags/
│   └── link/
└── .github/workflows/
    └── deploy.yml
```

## 小结

搭静态博客本身不难，真正的成本在**配置细节**。三个能省时间的习惯：

1. **改配置前先搜主题默认配置文件**，确认键名和值类型
2. **构建报错看第一行 `ERROR` 后面的模板路径**，直接去 `node_modules/hexo-theme-butterfly/layout/` 里翻对应的 pug 文件，比搜教程快得多
3. **改完配置先 `hexo clean` 再 `hexo g`**，缓存会掩盖很多问题

## 参考

- [Butterfly 官方文档](https://butterfly.js.org/)
- [Hexo 官方文档](https://hexo.io/zh-cn/docs/)
- [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages)
