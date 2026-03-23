---
title: "Setting Up Access Control for timshi.net"
date: 2026-03-23
---

今天对 timshi.net 做了一次比较完整的访问控制改造，记录一下过程和思路。

## 需求

希望网站能做到：

- 自己可以看到所有内容
- 分享给别人时，对方只能看到我希望给他们看的主题（佛学 / 射箭 / 文具）
- 个人笔记（notes）只有自己能看
- 直接访问根域名 `timshi.net` 的陌生人看不到任何内容

## 方案

选择了两种机制结合：

**Cloudflare Access（真正的访问拦截）** 用于：
- 根路径 `/` — 只有自己登录后可见
- `/notes/*` — 同上，继承根路径保护

**导航隔离（隐藏入口）** 用于：
- `/dharma/*`、`/archery/*`、`/pens/*` — 公开可访问，但页面内不出现其他主题的链接

## Cloudflare Access 配置

在 Zero Trust → Access → Applications 里建了四个独立应用：

| 应用 | 路径 | 策略 |
|------|------|------|
| timshi.net | `/` | Allow → 自己的 email |
| timshi.net dharma | `/dharma/*` | Bypass → Everyone |
| timshi.net archery | `/archery/*` | Bypass → Everyone |
| timshi.net pens | `/pens/*` | Bypass → Everyone |

关键点：每个路径需要独立建一个应用，不能在同一个应用里混用 Allow 和 Bypass——因为 Cloudflare 会优先执行 Bypass，会把整站变成公开。

## Hugo 模板调整

原来的 `_default/list.html` 里有 `← Home` 链接，公开主题的访客点了会跳到受保护的首页，体验不好。

解决方式：为每个公开主题创建专属模板，去掉 Home 链接：
```
layouts/archery/list.html   ← 无 Home 链接
layouts/pens/list.html      ← 无 Home 链接
layouts/notes/list.html     ← 保留 Home 链接（只有自己看）
layouts/notes/single.html   ← 新增：文章底部显示 GitHub commit 历史
```

## Notes 的版本历史功能

`layouts/notes/single.html` 在文章底部加了一段 JavaScript，通过 GitHub API 动态加载当前文件的 commit 历史：
```javascript
const filePath = 'content/' + parts.join('/') + '/index.md';
const apiUrl = `https://api.github.com/repos/timsotapanna/timshi-net/commits?path=${filePath}`;
```

每次修改文章并 push 到 GitHub，版本历史会自动更新，显示修改日期和 commit message。第一条 commit 会标注 `(initial)`。

## 效果

- 陌生人访问 `timshi.net` → Cloudflare 登录页
- 朋友访问 `timshi.net/dharma` → 直接进入，只看到佛学内容
- 自己登录后访问 `/notes` → 完整笔记列表，每篇文章底部有完整修改历史
