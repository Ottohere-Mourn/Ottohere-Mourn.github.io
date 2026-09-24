# Mourn's blog 更新与发布说明

这份文档说明如何修改文章、添加新文章、在本地预览，并推送到公网。

博客地址：<https://ottohere-mourn.github.io>

## 1. 重要目录

博客项目目录：

```text
/home/mjh/Projects/blog
```

文章 Markdown 源文件：

```text
src/content/blog/
```

当前文章：

```text
src/content/blog/where-vit-free-vlms-stand.md
src/content/blog/goodbye-three-years.md
```

个人资料和外部链接：

```text
src/config.ts
```

头像和文章正文中使用的图片：

```text
public/avatar.jpg
public/images/
```

不要直接修改以下目录或文件：

```text
dist/
.astro/
node_modules/
```

它们都是构建产物或依赖，下一次构建会自动生成或更新。

## 2. 修改已有文章

例如修改 ViT-free 文章：

```bash
cd /home/mjh/Projects/blog
编辑 src/content/blog/where-vit-free-vlms-stand.md
```

文章由两部分组成：顶部的 frontmatter 和下面的 Markdown 正文。

```md
---
title: "文章标题"
description: "首页显示的文章摘要"
pubDate: 2026-09-24
category: "Efficient Inference"
tags: ["ViT-free", "VLM"]
readingTime: "10 min read"
featured: false
---

这里开始写正文。
```

目前允许的分类必须完全匹配下面四个值：

```text
Long Video Understanding
Efficient Inference
Agent Systems
Experience Sharing
```

文章顺序按 `pubDate` 倒序排列，日期越新越靠上。日期建议使用完整格式：

```yaml
pubDate: 2026-09-24
```

文件名会自动成为文章 URL。例如：

```text
src/content/blog/my-new-note.md
```

对应地址是：

```text
/blog/my-new-note
```

文件名建议只使用小写英文、数字和连字符，不要使用空格或中文。

## 3. 添加新文章

先进入文章目录：

```bash
cd /home/mjh/Projects/blog
```

然后新建一个 Markdown 文件，例如：

```text
src/content/blog/reading-notes-on-agents.md
```

填写以下基本模板：

```md
---
title: "Reading Notes on Agents"
description: "一段用于首页展示的简短摘要。"
pubDate: 2026-10-01
category: "Agent Systems"
tags: ["Agents", "Learning note"]
readingTime: "6 min read"
featured: false
---

## 01 / 问题

正文内容。

## 02 / 观察

正文内容。
```

当前版本没有文章封面字段，文章详情页也不会自动显示 cover 图片。

## 4. 在文章中使用图片

把图片放进：

```text
public/images/
```

比如：

```text
public/images/agent-diagram.png
```

在 Markdown 中这样引用：

```md
![Agent 系统结构图](/images/agent-diagram.png)
```

头像文件是：

```text
public/avatar.jpg
```

如果要替换头像，保持文件名为 `avatar.jpg`，或者同步修改 `src/config.ts` 中的 `avatar` 字段。

## 5. 本地预览

启动开发服务器：

```bash
cd /home/mjh/Projects/blog
npm run dev
```

然后打开：

```text
http://localhost:4321
```

开发服务器会自动热更新。修改 Markdown 后刷新浏览器即可看到结果。

正式推送前，建议运行一次生产构建：

```bash
npm run build
```

如果看到下面的结果，说明构建检查通过：

```text
0 errors
0 warnings
```

## 6. 推送到公网

先查看当前修改：

```bash
git status
```

只修改文章时，可以这样提交：

```bash
git add src/content/blog
git commit -m "Update blog posts"
git push origin master
```

如果同时新增了图片：

```bash
git add src/content/blog public/images
git commit -m "Add a new blog note"
git push origin master
```

如果修改了个人资料、样式或说明文档：

```bash
git add src/config.ts src/styles docs
git commit -m "Update blog profile and design"
git push origin master
```

也可以在确认内容无误时使用：

```bash
git add -A
git commit -m "Update blog"
git push origin master
```

## 7. push 之后发生什么

仓库中的 `.github/workflows/deploy.yml` 会自动触发 GitHub Actions：

```text
push master
  ↓
npm ci
  ↓
npm run build
  ↓
上传 dist/
  ↓
部署 GitHub Pages
```

通常等待几十秒到几分钟，公网地址就会更新：

<https://ottohere-mourn.github.io>

可以在 GitHub 仓库的 Actions 页面查看部署状态：

<https://github.com/Ottohere-Mourn/Ottohere-Mourn.github.io/actions>

如果部署失败，不要继续反复 push。先打开失败的 workflow，查看 `Build Astro site` 步骤中的具体错误。

## 8. 常见问题

### frontmatter 报错

优先检查：

- `pubDate` 是否为 `YYYY-MM-DD`
- `category` 是否完全匹配允许的英文分类
- YAML 冒号、引号和缩进是否正确
- `tags` 是否使用数组格式

### 本地能看，push 后没有更新

依次检查：

1. `git status` 确认修改已经提交；
2. `git log -1` 确认最新 commit 是自己的修改；
3. GitHub Actions 是否成功；
4. 浏览器使用强制刷新，或在网址后加 `?v=日期`；
5. 确认访问的是 <https://ottohere-mourn.github.io>。

### 资料模板和网站文章不是同一个源

`/home/mjh/Projects/ViT-free/blog-profile-and-content-template.md` 是写作和收集资料用的模板，不会被 Astro 自动读取。

填写完模板后，需要把最终内容复制到博客项目的：

```text
/home/mjh/Projects/blog/src/content/blog/
```

网站真正读取的是这个目录里的 Markdown 文件。
