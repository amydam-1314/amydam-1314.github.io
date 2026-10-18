# 小舟的笔记

基于 **Hexo 8.1.2** 的静态博客，部署在 GitHub Pages。

**线上地址：https://amydam-1314.github.io/**

之前那套 Astro 版本已完整备份在 `.workbuddy/backup/`（`blog-v1-20261018.zip` + `git-history/`）。

---

## 一、记命令

安装依赖之后的日常操作，记住这三条就够：

```bash
node node_modules/hexo-cli/bin/hexo new "文章标题"     # 新建
node node_modules/hexo-cli/bin/hexo server             # 本地预览 localhost:4000
node node_modules/hexo-cli/bin/hexo generate           # 生成静态文件
```

加个别名会舒服很多（Git Bash 里执行一次即可）：

```bash
alias hx='node node_modules/hexo-cli/bin/hexo'
```

之后就是 `hx new "标题"`、`hx server`。

> 为什么不直接 `hexo`？因为 `hexo` 这个命令行工具需要全局安装，而 `npm install -g` 在这台机器上不一定顺。直接调 `node_modules` 里的入口最稳。

---

## 二、写一篇文章

```bash
node node_modules/hexo-cli/bin/hexo new "CTR 模式里计数器错一位就全废"
```

会生成 `source/_posts/CTR-模式里计数器错一位就全废.md`。

文件开头是这样：

```yaml
---
title: CTR 模式里计数器错一位就全废
date: 2026-10-18 17:00:00
tags:
---
```

几个要点：

- **URL 由文件名决定**。默认 `permalink: :year/:month/:day/:title/`，所以上面的文章会落在 `/2026/10/18/CTR-模式里计数器错一位就全废/`。想要干净的英文链接，把文件名改成英文即可。
- **摘要分隔符**是 `<!-- more -->`，写在它上面的内容会显示在首页列表里。
- `tags` 是数组，写成：

```yaml
tags:
  - 密码学
  - CTF
```

草稿：`hexo new draft "标题"` 会生成到 `source/_drafts/`，默认不发布。

---

## 三、预览与发布

本地预览（改文件自动刷新）：

```bash
node node_modules/hexo-cli/bin/hexo server
```

浏览器打开 http://localhost:4000 。`Ctrl + C` 停止。

发布：

```bash
git add -A
git commit -m "post: 文章标题"
git push
```

Actions 会自动跑 `hexo generate` 并部署，约一分钟生效。**不需要本地 `hexo generate`**，那是给本地预览和排查用的。

---

## 四、配置在哪

| 改什么 | 文件 |
|---|---|
| 站名、副标题、描述、作者 | `_config.yml` 顶部 |
| 站点地址、永久链接格式 | `_config.yml` 的 `url` / `permalink` |
| 每页文章数 | `_config.yml` 的 `per_page` |
| 主题 | `_config.yml` 的 `theme` |
| 主题自己的设置 | `_config/themes/<主题名>.yml`（换主题后出现） |

⚠️ **`url` 必须是 `https://amydam-1314.github.io`，`root` 保持 `/`**。改错了全站链接和 CSS 都会 404。

---

## 五、目录结构

```
├── _config.yml              # 站点配置
├── _config.butterfly.yml    # 主题配置（1132 行，改样式改这里）
├── package.json             # 依赖与 hexo 版本
├── source/
│   ├── _posts/              # ← 文章都放这里
│   ├── _drafts/             # 草稿
│   ├── about/index.md       # 关于页（type: about）
│   └── tags/index.md        # 标签总览页（type: tags）
├── scaffolds/               # hexo new 的模板
├── themes/                  # 空的，主题走 npm 包
├── public/                  # 生成产物，已 gitignore
└── .github/workflows/deploy.yml
```

当前主题是 npm 包 `hexo-theme-landscape`，装在 `node_modules/` 里，所以 `themes/` 目录是空的。

---

## 六、主题：Butterfly

当前是 [Butterfly](https://butterfly.js.org/) **5.7.0**，以 npm 包形式装在 `node_modules/hexo-theme-butterfly`，所以 `themes/` 目录是空的。

**改样式不要碰 `node_modules`**，配置全在根目录的 `_config.butterfly.yml`。那份文件 1132 行，别被吓到——99% 是带注释的默认值，你只用改想看的那几段。

已经配好的：

| 配置项 | 现在的值 |
|---|---|
| `menu` | 首页 / 归档 / 标签 / 关于 |
| `social` | GitHub 链接 |
| `subtitle` | 打字机效果，两行文案（`sub` 数组里改） |
| `search.use` | `local_search`，本地搜索，靠 `hexo-generator-searchdb` |
| `darkmode` | 已开，右上角可手动切换 |
| `index_layout` | `3`，封面左右交替 |
| `avatar` | 暂时是主题自带的蝴蝶图标 |
| `index_img` | 空，所以首页 banner 是默认渐变 |

想给首页加一张头图：`_config.butterfly.yml` 的 `index_img: /img/xxx.jpg`，图片放 `source/img/`。

### 换别的主题

```bash
npm install hexo-theme-xxx
```

`_config.yml` 改 `theme: xxx`，主题配置写到根目录 `_config.xxx.yml`。

> 老教程里写的 `_config/themes/主题名.yml` 是 Hexo 5 之前的约定，现在不生效了。

---

## 七、常见问题

**首页空白 / 没有 index.html** → `source/_posts/` 里没有文章。Hexo 不会为 0 篇文章生成首页。

**改了文章但线上没变** → 看 Actions 有没有跑完；Pages 有十几秒 CDN 缓存，硬刷一次。

**CSS 丢失、链接全 404** → `_config.yml` 的 `url` 或 `root` 写错了。

**`hexo` 命令报 "not a hexo folder"** → `package.json` 里的 `hexo.version` 字段丢了，补上 `"hexo": { "version": "8.1.2" }`。

**`/tags/` 或 `/about/` 404** → 这两个页面不会自动生成，需要在 `source/` 下自己建。`hexo new page tags` 后把 frontmatter 改成 `type: tags`；关于页是 `type: about`。归档 `/archives/` 才是自动生成的。

**Butterfly 构建报模板找不到** → 缺 `hexo-renderer-pug`，`npm install hexo-renderer-pug`。

**Actions 报 `npm ci` 失败** → `package-lock.json` 没提交，或者与 `package.json` 不一致。本地跑 `npm install` 后把 lock 一起提交。
